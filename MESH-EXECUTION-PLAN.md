# Mesh execution plan — M1→M3 sprints, concrete deliverables

> **Companion to [`MESH.md`](./MESH.md).** That doc is the
> destination + form. This doc is the *order of operations* — what to
> build in what week, what each step proves, and what the merge gates
> are. The unit is a 1-2 week sprint. Phases beyond M3 are sketched
> further out but not sprint-decomposed yet.
>
> **Demo Aplicacao**: openclaw (cartorio + lacre + publisher-pki).
> Already running on pleme-dev with public hostnames; gives a tight
> feedback loop (~minutes, not days) for every meshing change.

---

## Milestone gates summary

| Gate | What proves it | State | Dependencies |
|---|---|---|---|
| **M1.1** ✅ | Every cartorio/lacre/publisher-pki pod has `/run/spiffe.io/socket` mounted (SPIFFE Workload API socket reachable) | **shipped** — SPIRE on pleme-dev; cartorio pod registered with `spiffe://pleme.io/ns/openclaw/sa/openclaw-stack-cartorio`; SVID delivered via probe pod | spire-server + spire-agent deployed; openclaw helmrelease patched |
| **M1.2** 🟡 | Rust crate `kakuin-workload-api` reads an SVID from a process running inside cartorio's pod | **library shipped** at `pleme-io/kakuin-workload-api`; live-pod integration test deferred to in-cluster CI | M1.1 |
| **M1.3** 🟡 | Two Rust demo Servicos handshake mTLS via SVIDs (no proxy yet, just direct rustls + workload API SVID source) | **library shipped** at `pleme-io/kakuin-rustls`; live two-Servico handshake demo is the remaining gate | M1.2 |
| **M2.1** 🟡 | A new `pleme-io/aresta` Rust crate builds inbound mTLS path on top of kakuin-workload-api + kakuin-rustls; vendoring linkerd2-proxy is deferred (kept simpler scope first) | **library shipped** at `pleme-io/aresta`; outbound + retries + CB queued for M2.3 | independent of M1 |
| **M2.2** 🟡 | Sidecar injector caixa Servico — MutatingAdmissionWebhook adds proxy + iptables init-container to pods labeled `mesh.pleme.io/inject=true` | **library + binary shipped** at `pleme-io/enxerto`; Helm chart + k8s deploy on pleme-dev queued | M2.1 |
| **M2.3** 🟡 | Outbound mTLS path + connect-retry + circuit breaker landed in `aresta`; full openclaw-meshed-on-pleme-dev demo is the remaining live gate | **transport-layer code shipped**; live mesh deploy queued | M1.3 + M2.2 |
| **M3.1** 🟡 | Hanabi gains `:role :sidecar` slot (config schema) + builds in that mode | **typed config surface landed** in `pleme-io/hanabi` main; runtime path that honors `upstream_loopback` + `policy_source` is the remaining slice | independent (small hanabi PR) |
| **M3.2** | Cartorio's pod has 3 sidecars: proxy + hanabi + cartorio. GraphQL request → proxy (mTLS) → hanabi (rate-limit + CSP) → cartorio | M2.3 + M3.1 |
| **M3.3** | One observability story (vector co-sidecar emits W3C trace context propagated through proxy + hanabi + workload) | M3.2 |
| **M4.1** 🟡 | `(defmesh openclaw-mesh …)` form parses, validates type-level invariants | **typed core shipped** at `pleme-io/tatara-mesh` (MeshSpec + validation + YAML round-trip); `#[derive(TataraDomain)]` add-on lands once the proc-macro stabilizes | M3.3 |
| **M4.2** 🟡 | Renders the form to: spire CRDs + ServiceAccount→SPIFFE-ID mapping + sidecar-injector config + hanabi config + observability sinks | **standalone renderer shipped** at `pleme-io/tatara-mesh-render` (deterministic, 13 tests pass); arch-synthesizer integration is a one-liner when M5 starts | M4.1 |
| **M4.3** 🟡 | Same MeshSpec re-deploys openclaw mesh — full reproducibility | **rendered + applied on pleme-dev**: 3 ClusterSPIFFEIDs + 2 ConfigMaps live, 3 SPIRE entries auto-provisioned. Fresh-kind reproducibility (CI gate) queued | M4.2 |

Each sub-gate is the merge bar for the sprint that ends with it.

---

## M1 — Identity baseline (3 sprints, ~6 weeks)

### Sprint M1.1 (week 1) — SPIRE on pleme-dev

**Goal**: SPIFFE Workload API socket reachable from inside every
openclaw pod.

**Decisions made up-front**:
- **Vendor upstream SPIRE** (spiffe/spire) for M1. Reach for a
  pleme-io shikiriya-from-scratch in M5+ once the value is proven.
  Per [`MESH.md`](./MESH.md#open-questions) Q4.
- **Identity shape**: `spiffe://pleme.io/ns/<ns>/sa/<sa>` —
  one SPIFFE-ID per (namespace, ServiceAccount).
- **Trust domain**: `pleme.io` — single trust domain spans the
  fleet (multi-cluster trust bundle propagation is M5+ scope).
- **Agent install style**: SPIRE Agent as a DaemonSet with hostPath
  mount of `/run/spire/sockets/agent.sock`; CSI driver `spiffe-csi`
  exposes the socket inside each pod at `/run/spiffe.io/socket`.

**Concrete tasks**:
1. New repo `pleme-io/shikiriya-helm` (working name; could fold into
   helmworks). Helm chart wrapping spire-server + spire-agent +
   spiffe-csi-driver. One values surface for trust-domain + per-cluster
   SPIRE-server endpoint.
2. Add `k8s/clusters/pleme-dev/apps/spire/` — namespace + helmrelease
   pinning the chart.
3. Patch `lareira-openclaw-pki` chart values to:
   - Mount the spiffe-csi `csi.spiffe.io` volume at `/run/spiffe.io`.
   - Annotate ServiceAccount with `spiffe.io/spiffe-id` registration.
4. Add a `RegistrationEntry` CRD per openclaw ServiceAccount
   (cartorio, lacre, publisher-pki) — initially emitted by the helm
   chart as additional `extraResources`; later moved into the
   `(defmesh …)` renderer.

**Gate**:
```bash
kubectl --context pleme-dev -n openclaw exec deploy/openclaw-stack-cartorio \
  -- ls -l /run/spiffe.io/socket
# Expected: srw-rw-rw- ... /run/spiffe.io/socket
```

---

### Sprint M1.2 (weeks 2-3) — Rust workload API client crate

**Goal**: Rust crate that connects to the workload API socket and
fetches the pod's SVID.

**Concrete tasks**:
1. New crate `pleme-io/kakuin-workload-api` (working name; "kakuin"
   = 確認 = verification). Built via substrate
   `rust-library.nix`. Implements:
   - `WorkloadApiClient::new(socket_path)`
   - `client.fetch_x509_svid().await -> X509Svid` (cert + chain + key)
   - `client.subscribe_svid_updates() -> Stream<X509Svid>` (rotation)
   - `WorkloadApiClient::default()` reads `SPIFFE_ENDPOINT_SOCKET`
     env var; sane fallback to `/run/spiffe.io/socket`.
2. gRPC client wraps SPIFFE Workload API protobuf
   ([spec](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE_Workload_API.md)).
   Use `tonic` + `prost`.
3. Crate depends on substrate's blessed Rust toolchain; published to
   crates.io via the standard release flow.
4. Integration test runs against a transient SPIRE in a kind cluster
   (CI matrix via `nix run .#test-integration`).

**Gate**:
```rust
#[tokio::test]
async fn fetches_svid_from_pleme_dev_cartorio_pod() {
    let client = WorkloadApiClient::default();
    let svid = client.fetch_x509_svid().await.unwrap();
    assert_eq!(svid.spiffe_id().trust_domain(), "pleme.io");
    assert!(svid.spiffe_id().path().starts_with("/ns/openclaw/sa/"));
    assert!(svid.cert_chain().len() >= 1);
}
```

---

### Sprint M1.3 (weeks 4-6) — mTLS via SVIDs (no proxy)

**Goal**: Two Rust Servicos handshake mTLS using SVIDs from the
workload API. NO proxy yet; this is the bare-metal proof of the
identity story.

**Concrete tasks**:
1. New crate `pleme-io/kakuin-rustls` — adapter from
   `kakuin-workload-api` to a `rustls::ServerConfig` /
   `rustls::ClientConfig` that:
   - Uses the SVID's leaf cert as the rustls server/client cert.
   - Uses the trust bundle as the rustls root store.
   - Verifies peer SPIFFE-ID against an allow-list (passed in
     constructor) — *this is the identity-aware policy enforcement
     hook* the higher mesh layers will plug into.
   - Rotates configs on SVID rotation (subscribe path from M1.2).
2. Demo binary `pleme-io/openclaw/cartorio` (already exists) gains
   an experimental `--mtls-only` mode using `kakuin-rustls`. The
   `lacre` binary gains a matching client.
3. Patch lacre's cartorio HTTP client to use
   `reqwest`/`hyper`-with-rustls + the kakuin-rustls config.
4. Manual demo: cartorio rejects connections from any
   non-`spiffe://pleme.io/ns/openclaw/sa/lacre` peer.

**Gate**:
```bash
# Lacre admit succeeds because lacre's SPIFFE-ID is in cartorio's allow-list:
kubectl --context pleme-dev -n openclaw exec deploy/openclaw-stack-lacre \
  -- /bin/lacre admit-test http://openclaw-stack-cartorio:8082/api/v1/admin/health
# OK

# A naked HTTP probe pod gets rejected:
kubectl --context pleme-dev -n openclaw run probe -it --rm --image=curlimages/curl \
  -- curl http://openclaw-stack-cartorio:8082/api/v1/admin/health
# Expected: TLS handshake error (no SVID)
```

---

## M2 — Sidecar injector + proxy MVP (3 sprints, ~7 weeks)

### Sprint M2.1 (weeks 7-9) — proxy crate

**Goal**: a Rust proxy binary that does mTLS termination + retry +
circuit-breaker + timeout, configured statically.

**Decisions**:
- **Vendor `linkerd2-proxy`** as a starting point. It's already
  Rust, idiomatic tokio, MIT-licensed. We wrap it in a pleme-io
  configuration surface and replace its identity layer with
  `kakuin-workload-api`/`kakuin-rustls`.
- The wrapper is a new crate `pleme-io/proxy` (working name;
  alternative: `kabe` 壁 = wall).
- Configuration: typed Rust struct → YAML at startup. Dynamic
  config (xDS) is M5.

**Concrete tasks**:
1. Scaffold `pleme-io/proxy` via substrate `rust-tool-release-flake.nix`.
2. Vendor `linkerd2-proxy` dependencies; replace identity layer.
3. Static config: inbound port + outbound port + per-edge policy
   (timeout, retry, CB) + identity allow-list.
4. iptables-redirect-friendly: bind on `127.0.0.1:15001` (inbound)
   + `127.0.0.1:15006` (outbound).
5. Smoke test: `proxy` runs locally, intercepts a pair of demo
   Rust services, mTLS handshake works.

**Gate**: `nix run .#proxy -- --config demo.yaml` mediates a demo
edge with mTLS, retries on `--inject-failures`, opens circuit
breaker after N failures.

---

### Sprint M2.2 (weeks 10-12) — sidecar injector

**Goal**: a new caixa Servico that intercepts pod creation and
injects the proxy + iptables init-container.

**Concrete tasks**:
1. New caixa `pleme-io/sidecar-injector` (working name).
   MutatingAdmissionWebhook in Rust (via `axum` + `kube-rs`).
2. Selector: pods labeled `mesh.pleme.io/inject=true` OR pods in
   namespaces labeled `mesh.pleme.io/inject=true`.
3. Mutation: add init-container (iptables redirect to 15001/15006)
   + sidecar container (the M2.1 proxy) + volume mounts for
   `/run/spiffe.io/socket`.
4. TLS for the webhook itself: spire-issued (eat the dogfood —
   the injector authenticates via SPIFFE).
5. Helm chart (helmworks) for deployment.

**Gate**: a fresh openclaw helmrelease deploy auto-meshes
cartorio + lacre + publisher-pki — every pod gets the proxy
sidecar without explicit pod-spec changes.

---

### Sprint M2.3 (weeks 13) — openclaw fully meshed

**Goal**: Apply M2.1 + M2.2 to openclaw on pleme-dev. cartorio
receives mTLS-only requests; lacre's cartorio client uses mTLS;
publisher-pki's enrollment endpoint uses mTLS; failure injection
via `kubectl delete pod` shows retries kick in.

**Gate**: black-box test against `https://cartorio-dev.quero.cloud`
still works (browser path goes through cloudflared which
terminates outside the mesh; cluster-internal mTLS doesn't break
public reachability).

---

## M3 — Hanabi sidecar mode (2 sprints, ~5 weeks)

### Sprint M3.1 (weeks 14-15) — `:role :sidecar` slot

**Goal**: hanabi's config gains the four new slots from
[`MESH.md` §V.1](./MESH.md#v1-new-role-sidecar-mode). Standalone
mode unchanged.

**Concrete tasks**:
1. PR to `pleme-io/hanabi`: add `ServerConfig.role` enum
   (`Standalone | Sidecar`), `ServerConfig.upstream_loopback` Option,
   `ServerConfig.policy_source` Option, `ServerConfig.no_static`
   bool. Defaults preserve standalone behavior.
2. Sidecar mode short-circuits static-file serving;
   binds `127.0.0.1` only; expects upstream URL.
3. Unit tests: standalone vs sidecar path.
4. Release: hanabi v1.1.0 (minor — wire-additive).

**Gate**: `cargo test -p hanabi --features sidecar-mode` passes.

---

### Sprint M3.2 (weeks 16-18) — three-sidecar pod

**Goal**: cartorio's pod has proxy + hanabi + cartorio. Request
flow:
```
client → tunnel → cloudflared → kube-svc → proxy (mTLS in)
                                         → hanabi (loopback :15010, applies CSP/rate-limit)
                                         → cartorio (loopback :8082)
```
Same on the way out.

**Concrete tasks**:
1. Sidecar injector (M2.2) gains a per-namespace `inject_l7_hanabi`
   flag.
2. Hanabi sidecar policy is composed from existing chart values
   (CSP / rate-limit / CORS) — no new authoring surface for M3.
3. Demo: rate-limit set to 5 req/s — `hey -n 100 -c 10` shows ~95
   429s and ~5 200s.

**Gate**: demo above passes; openclaw-web at
`openclaw-dev.quero.cloud` continues to render (CSP doesn't break
loading).

---

### Sprint M3.3 (week 19) — observability uniformity

**Goal**: every meshed pod emits the same shape of trace/metric/log.

**Concrete tasks**:
1. Add `vector` co-sidecar to the inject template.
2. Proxy emits W3C `traceparent` headers; hanabi propagates;
   workload propagates.
3. vmagent + VictoriaLogs (or DD on SaaS) receive the unified shape.

**Gate**: a single trace ID follows a request from cloudflared all
the way to cartorio's audit-consistency loop.

---

## M4 — TataraDomain + renderer (2 sprints, ~6 weeks)

### Sprint M4.1 (weeks 20-22) — `(defmesh …)` form

**Goal**: parse + validate the form per [`MESH.md` §VI](./MESH.md#vi-the-defmesh-form).

**Concrete tasks**:
1. New crate `pleme-io/tatara-mesh` — `MeshSpec` Rust struct
   with `#[derive(TataraDomain, …)]`.
2. Validation layer: type-level invariants
   (retry-budget ∈ [0,1], strict mTLS requires identity ≠ off, etc).
3. Lisp parser test: round-trip the example from
   [`MESH.md` §VI](./MESH.md#vi-the-defmesh-form).

**Gate**: `cargo test -p tatara-mesh` passes; `tatara-lisp parse
defmesh examples/openclaw-mesh.lisp` succeeds.

---

### Sprint M4.2 (weeks 23-25) — renderer

**Goal**: `arch-synthesizer/src/mesh.rs` renders `MeshSpec` to k8s
manifests for the "k8s + sidecar + hanabi" backend (the only
backend in M4; others land in M6).

**Concrete tasks**:
1. Add `MeshSpec → Vec<KubeManifest>` morphism in arch-synthesizer.
2. Cover: spire CRDs, RegistrationEntries, sidecar-injector
   ConfigMap, hanabi sidecar policy, vector sidecar, ServiceMonitor.
3. Same form deployed twice produces byte-identical output.

**Gate**: `arch-synthesizer mesh openclaw-mesh.lisp -o ./out` produces
exactly the manifests deployed in M3 — *the form replaces the
hand-authored helmrelease patches*.

---

### Sprint M4.3 (week 26) — fresh-cluster reproducibility

**Goal**: same Lisp form re-deploys mesh from scratch on a kind cluster.

**Concrete tasks**:
1. CI workflow: `kind create` → `(defmesh …)` render → `kubectl
   apply -k out/` → wait for SPIRE ready → run M2.3's gate test.
2. Catch any drift between hand-authored M3 manifests and renderer
   output.

**Gate**: CI green on fresh-cluster reproduction.

---

## Beyond M4 (sketches, not yet sprint-decomposed)

- **M5** xDS dynamic config — Rust xDS server caixa, hot reload,
  canary 5%→50%→100% on a demo edge with no Pod restart.
- **M6** Multi-backend: render the same `(defmesh …)` to Linkerd
  TrafficPolicy / Istio AuthorizationPolicy / Cilium ServiceMesh.
  ~6 weeks per backend.
- **M7+** Production hardening, fleet adoption beyond openclaw.

---

## Open process questions

1. **Sprint review cadence?** Weekly is too noisy; biweekly is
   probably right given agent-augmented throughput.
2. **Where do new caixas live?** Each gets its own repo
   (`pleme-io/<name>`) per the existing pattern, or do we cluster
   them in a `pleme-io/mesh-platform/` monorepo? Mild preference
   for separate repos (matches existing fleet shape; CI is
   already templated via `pleme-actions`).
3. **Versioning during M1-M3?** Pin to git revs across the four
   work-stream crates (`kakuin-*`, `proxy`, `sidecar-injector`,
   `hanabi`) until M4 stabilizes the form, then cut 0.x semver
   releases together.
4. **What kicks off the M4 lift?** When all M3 sprints are merged
   AND openclaw mesh has been stable on pleme-dev for 1 week.
   Don't start M4 against a moving target.

---

## Next concrete action (today)

Begin **Sprint M1.1**: scaffold `shikiriya-helm` chart (or fold
into helmworks) with spire-server + spire-agent + spiffe-csi
declarations. Drop a pleme-dev kustomization layer pointing at it.
Verify `/run/spiffe.io/socket` lands in cartorio's pod. Iterate.
