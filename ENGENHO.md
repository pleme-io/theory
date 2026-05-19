# engenho — the typed, attested, Rust-native Kubernetes runtime

> **Frame.** [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> says every recurring shape becomes a typed substrate primitive, proven by
> construction. [`THEORY.md`](./THEORY.md) §I names the twelve pillars; this
> doc owns the runtime half of **Pillar 7 (Kubernetes control)** — the
> substrate that consumes typed manifests and *runs* them on real hardware.
> [`CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md`](./CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md)
> already governs the *provider-emission* half of Pillar 7 (typed Crossplane
> providers regenerated from OpenAPI). [`MAGMA.md`](./MAGMA.md) governs the
> *execution* half of Pillar 5 (Rust-native Terraform-protocol executor).
> [`NIX-AST.md`](./NIX-AST.md) governs typed Nix emission. [`SHIGOTO.md`](./SHIGOTO.md)
> governs typed work graphs. [`RATE-LIMITED-CONSUMERS.md`](./RATE-LIMITED-CONSUMERS.md)
> governs typed external-API consumption. [`CAIXA-SDLC.md`](./CAIXA-SDLC.md)
> governs the SDLC primitive. **This doc owns the missing primitive: the
> single Rust binary that natively reconciles Kubernetes resources without
> embedding any of upstream `kubernetes/kubernetes`.** Engenho is to Pillar 7
> what magma is to Pillar 5: the molten executive layer beneath the
> declaration crust, recast as a typed pleme-io primitive owned in one place
> and consumed everywhere.
>
> The doc is normative. Every pleme-io workload cluster that today runs k3s
> (rio, home-edge, nexus-staging, lilitu-edge) is an engenho migration
> target — `skip-engenho:` as the first line of a cluster's `engenho.lisp`
> defers migration. The wire compatibility contract is byte-exact at the
> surfaces listed in §III: kubectl, controller-runtime clients, FluxCD,
> Crossplane, Helm, every operator built against k8s 1.32 — all run
> unchanged. No upstream Go is vendored. No `kine` is shipped. No `etcd` is
> shipped. The entire control plane is one Rust binary built by
> `substrate/lib/rust-tool-release-flake.nix`.
>
> **Status.** Draft v1. Pre-implementation. Destination locked first per
> pleme-io/CLAUDE.md Operating Principle #0 ("write the destination before
> the plan"); the M0–M5 phases in §X are the path-down, not the goal —
> time-pressure deviation never collapses §II–IX. Bootstrap consumer is the
> `rio` homelab cluster (single-node k3s today; full conformance target by
> M4). Promotion to fleet default is gated on bit-exact CNCF Certified
> Kubernetes Software Conformance pass on engenho's M0 surface — the first
> non-Go distribution to do so.
>
> **Name.** Brazilian-Portuguese per the [`THEORY.md` §II naming
> convention](./THEORY.md). An **engenho** in colonial Brazil was a
> self-contained sugarmill — a single productive unit that combined the
> field (`terreno`), the mill (the milling apparatus), the boiler, the
> distillery, and the warehouse on one plot of land, run by one capataz.
> The metaphor maps 1:1 onto a Kubernetes single-binary distribution —
> apiserver + datastore + controller-manager + scheduler + kubelet +
> kube-proxy + CNI + DNS + LocalPath + CA, all in one process, on one node,
> run by one supervisor loop. Pairs naturally with `terreno` (the
> declaration-time land where workloads grow — engenho is what runs the
> land), `magma` (engenho shapes containers; magma shapes cloud), `forja`
> (forja shapes CI artifacts; engenho shapes runtime), `feira` (feira
> orchestrates caixa; engenho hosts caixa). Word secondarily means
> *ingenuity* in Portuguese: the clever engine. See §XI for the full
> naming rationale.

---

## I. The repeating pattern

Today every pleme-io workload cluster looks like this:

```
caixa.lisp ─► feira ─► caixa-helm ─► HelmRelease ─► FluxCD ─► k3s ─► containerd
                                                                │
                                                                ├─ vendored Go kube-apiserver  (350k LoC)
                                                                ├─ vendored Go kube-controller-manager (200k LoC)
                                                                ├─ vendored Go kube-scheduler  (80k LoC)
                                                                ├─ vendored Go kubelet         (120k LoC)
                                                                ├─ vendored Go kube-proxy      (25k LoC)
                                                                ├─ vendored kine               (10k Go)
                                                                ├─ vendored flannel            (~30k Go)
                                                                ├─ vendored CoreDNS            (~50k Go)
                                                                └─ vendored Traefik+klipper    (~150k Go)
```

We render typed plans and then hand them to a **1.6 MLOC vendored Go binary
we do not control**. k3s' contribution to its own codebase is ~30k LoC of
glue; the remaining ~99% is upstream Kubernetes recompiled. Nine failure
modes leak from that trust:

| # | Failure mode | Today | What pleme-io loses |
|---|---|---|---|
| 1 | API server bugs | inherited from upstream | proof discipline (Pillar 10) |
| 2 | Watch-stream divergence | tested only by upstream e2e | typed event semantics |
| 3 | Strategic-merge-patch quirks | upstream Go reflection + struct tags | typed AST guarantees |
| 4 | RBAC evaluator subtleties | upstream Go bool soup | typed authorization proofs |
| 5 | Container runtime opacity | containerd dropping events silently | typed CRI-event stream (shigoto §IV) |
| 6 | Secret leakage | k8s Secret = base64 plaintext on disk | typed cofre materialization at pod start |
| 7 | Compliance gaps | k3s ships zero attestation | tameshi BLAKE3 chain (Pillar 10) |
| 8 | Caixa translation overhead | 5–6 step pipeline through Helm + Flux | direct M2/M3 → Pod reconciliation |
| 9 | Resource catalog drift | hand-authored arch-synthesizer types | generated from upstream OpenAPI (Pillar 12) |

The k3s binary handles all nine ad-hoc, in Go, behind a vendored boundary
we cannot extend. Naming the runtime as a typed pleme-io primitive lets
the substrate carry these concerns *once*, in Rust, with macros doing the
boilerplate. Same shape as every other constructive substrate primitive:
declare the type, prove by construction, render mechanically, attest with
BLAKE3, consume everywhere.

The pattern is universal — every "lightweight Kubernetes" project (k3s,
k0s, microk8s, MicroShift, kine, Talos) is the same Go binary repacked.
None expose the runtime as a typed primitive an operator can extend. None
ship attestation. None natively understand the substrate's own SDLC
primitives. Engenho does.

---

## II. The destination

### II.1. One typed primitive — the engenho crate workspace

```
pleme-io/engenho                            (Cargo workspace, substrate's rust-workspace-release)
├── engenho                                 — the shipping binary (~30-50 MB)
├── crates/
│   ├── engenho-types         — generated from upstream OpenAPI v3 via forge-gen.
│   │                           ONE #[derive(KubeResource, TataraDomain)] per kind.
│   │                           Re-exported from arch-synthesizer::k8s for typescape
│   │                           participation. Pillar 12 unlock — never hand-authored.
│   │
│   ├── engenho-datastore     — fjall-backed local KV + tonic-served etcd-v3 gRPC
│   │                           shim (Range / Txn / Watch / Lease / Auth) speaking
│   │                           revision-MVCC. openraft + fjall in HA tier (M1).
│   │                           Wire-compat: byte-exact etcd v3.5 protobuf.
│   │
│   ├── engenho-apiserver     — Axum + tonic + tokio + rustls. Authn (TLS / JWT-SA /
│   │                           bootstrap-token / OIDC), authz (RBAC evaluator on
│   │                           typed rules), admission chain, OpenAPIv3 emitter
│   │                           (generated from engenho-types), watch broadcast,
│   │                           SSA + JSONPatch + StrategicMergePatch (generated
│   │                           from #[strategic_merge_key = …] struct tags),
│   │                           CRDs, conversion webhooks, aggregation API.
│   │
│   ├── engenho-controllers   — 18 controllers, ONE #[derive(TataraController)] per
│   │                           kind. Shared reconcile loop. Workqueue + retry +
│   │                           backoff handled by shigoto::Dag. Leader election
│   │                           via Lease + kube-leader-election. Built-in set:
│   │                           namespace, serviceaccount, sa-token, endpoint,
│   │                           endpointslice, root-ca-cert-publisher, gc, podgc,
│   │                           ttl, csr-approve+sign, deployment, replicaset,
│   │                           daemonset, statefulset, job, cronjob, node, pv.
│   │
│   ├── engenho-scheduler     — 11-plugin default profile. ONE #[derive(SchedPlugin)]
│   │                           per plugin. Filter+Score+Reserve+Permit+Bind hooks.
│   │                           No preemption M0. Pure logic, single watch loop.
│   │
│   ├── engenho-kubelet       — CRI gRPC client (tonic against containerd today;
│   │                           youki+oci-distribution direct M3+). Evented PLEG.
│   │                           Pod-worker state machine. Volume drivers: hostPath,
│   │                           Secret (cofre-projected), ConfigMap, projected SA
│   │                           token. Probes (liveness / readiness / startup ×
│   │                           http / tcp / exec). /exec SPDY shim. cgroup-v2
│   │                           via rustix. Eviction loop.
│   │
│   ├── engenho-cni           — In-tree CNI: bridge + host-local IPAM + portmap +
│   │                           loopback (M0). VXLAN overlay (M0.5). aya-rs eBPF
│   │                           backend (M2). Invoked as $name binary via the
│   │                           existing CNI contract; runs in-process via
│   │                           internal trait + external binary thin shim.
│   │
│   ├── engenho-kubeproxy     — Watch Service+EndpointSlice → rustables/nftnl-rs
│   │                           netlink. Pure-Rust nftables only (no iptables /
│   │                           IPVS support — KEP-3866 nftables GA in 1.33).
│   │                           ClusterIP + NodePort M0; LoadBalancer via the
│   │                           internal ServiceLB in M0.5.
│   │
│   ├── engenho-dns           — hickory-dns server + KubernetesAuthority impl.
│   │                           Watch Services → zone update. UDP+TCP :53 in
│   │                           every pod's DNS config. Drop-in CoreDNS replacement.
│   │
│   ├── engenho-localpath     — local-path-provisioner-equivalent. In-tree
│   │                           controller (lives inside engenho-controllers).
│   │                           Watches PVC with engenho.io/local-path StorageClass
│   │                           + WaitForFirstConsumer → emits hostPath PV.
│   │
│   ├── engenho-ca            — rcgen-based built-in CA. CSR API + signer + approver.
│   │                           kubelet bootstrap flow wire-compatible with k3s.
│   │
│   ├── engenho-caixa         — NATIVE Caixa M2/M3 reconcilers (M1+). One typed
│   │                           controller per Caixa kind (Biblioteca / Binario /
│   │                           Servico / Supervisor / Aplicacao). Emits typed
│   │                           Deployment / Service / NetworkPolicy / Gateway
│   │                           DIRECTLY — no Helm intermediation. Replaces the
│   │                           caixa-helm + HelmRelease + Flux pipeline for
│   │                           in-cluster caixa workloads.
│   │
│   ├── engenho-mesh          — NATIVE Aplicacao mesh (M3+). Enforces M3 typed
│   │                           slots (:contratos / :politicas / :placement /
│   │                           :entrada) at admission time. WIT-typed
│   │                           inter-Servico contracts validated by kensa.
│   │
│   ├── engenho-gateway       — pingora-based Gateway API impl (M3+). Replaces
│   │                           Traefik in-cluster ingress. HTTPRoute / GRPCRoute /
│   │                           TLSRoute typed via engenho-types.
│   │
│   ├── engenho-attest        — tameshi integration. Every Pod create attaches a
│   │                           BLAKE3 receipt; every image pull validates an
│   │                           upstream tameshi seal; sekiban admission webhook
│   │                           runs in-process (not a separate Deployment).
│   │
│   ├── engenho-cli           — kubectl-shaped operator CLI: engenho bootstrap /
│   │                           join / status / drain / attest / replay. Authored
│   │                           via tatara-lisp where the operation graph >3 steps.
│   │
│   └── engenho-test          — Test harness: spawns an in-process engenho cluster
│                               for integration tests. Replaces kind / k3d for
│                               pleme-io's own test suites.
└── flake.nix                              — rust-tool-release-flake.nix builder
```

Every crate is buildable in isolation (`cargo check -p engenho-X`). Every
crate has a `tests/` directory with the contract tests for that surface
(§V). The shipping binary links all of them statically, ~30-50 MB,
fully-fat. M0.0 ships only `engenho-types` and proves the OpenAPI →
generated Rust pipeline is bit-reproducible.

### II.2. The Rust + Lisp authoring surface

Every engenho-aware declaration is a typed Lisp form expanding to a typed
Rust struct via `#[derive(TataraDomain)]`. The operator never writes raw
YAML for an engenho cluster; YAML is the legacy ingress for compatibility
with foreign tools.

```lisp
(defengenho-cluster
  :name          rio
  :node          rio-host
  :datastore     (:embedded-fjall :path "/var/lib/engenho/db")
  :networking    (:cni :bridge      :cidr "10.42.0.0/16")
  :networking    (:proxy :nftables)
  :networking    (:dns   :hickory   :zone "cluster.local")
  :storage       (:local-path       :path "/var/lib/engenho/pv")
  :attestation   (:tameshi          :ca "/etc/engenho/tameshi.pub")
  :compliance    (:overlay :fedramp-moderate)
  :caixa-native  t
  :runtime-classes
    ((:name oci      :handler containerd)
     (:name tlisp    :handler runwasi-tatara)
     (:name wasi-p2  :handler runwasi-wasmtime)))
```

This expands to a `EngenhoClusterSpec` Rust struct. `engenho bootstrap`
reads the form, validates classification + compliance gating via
`tatara-lattice`, materializes the embedded datastore, signs the bootstrap
CA via `engenho-ca`, materializes secrets via cofre, and starts the
supervisor loop. Same pattern for `defengenho-node`, `defengenho-policy`,
`defengenho-runtimeclass`.

### II.3. Testable layers — the three-axis test cube

Every engenho subsystem is testable on **three independent axes** before
any cross-component test runs:

1. **Type-axis** — the Rust compiler refuses invalid composition. Adding a
   StatefulSet field that violates upstream OpenAPI is a build error in
   `engenho-types`. Wiring a Service to a non-existent EndpointSlice kind
   is a build error.
2. **Behavior-axis** — `cargo test -p engenho-X` exercises the subsystem
   in isolation against a Rust trait-based mock of every dependency
   (CRI, datastore, network).
3. **Wire-axis** — `cargo test --features wire-contract` exercises the
   subsystem against the real upstream wire protocol (etcd v3 protobuf
   conformance suite for datastore; CRI v1 conformance suite for kubelet;
   kubectl smoke tests for apiserver). No real cluster required.

A change to a subsystem that breaks any axis fails CI. A change that
passes all three is then exercised in §V's higher-level test pyramid.

### II.4. Determinism contract (binding)

The following are bit-reproducible by construction:

| Artifact | What's reproducible | Test |
|---|---|---|
| engenho binary | identical SHA256 across rebuilds | nix flake's `checks.deterministic` |
| engenho-types generated source | identical SHA256 from same OpenAPI input | `cargo test -p engenho-types --features bit-repro` |
| etcd-v3 Watch responses | identical byte stream for identical (state, since-rev) inputs | `engenho-datastore::tests::wire_replay` |
| Plan output (admission decisions, scheduling decisions) | identical for identical (cluster-state, candidate) inputs | proptest in `engenho-apiserver` + `engenho-scheduler` |
| SSA merge outputs | identical for identical (current, desired, fieldOwners) inputs | proptest in `engenho-apiserver::ssa` |
| Caixa M2/M3 → Deployment lowering | identical for identical Caixa input | `engenho-caixa::tests::lowering_repro` |
| tameshi receipts | identical BLAKE3 chain for identical inputs | tameshi's own conformance suite |

The shigoto scheduler's "concurrent wave execution" is intentionally
non-deterministic in scheduling order; result equivalence is verified via
proptest (order doesn't matter for the typed result).

### II.5. Verifiability surface (binding)

Every engenho release ships:

1. **Type proofs** — the Rust compiler is the verifier. Pillar 10.
2. **Property tests** — proptest cases against the upstream conformance
   shape of every subsystem with a wire spec.
3. **Contract tests** — replay of upstream protocol conformance suites
   (etcd v3, CRI v1, kubectl smoke, CNI v1, CSI v1).
4. **Integration tests** — `engenho-test` spawns in-process clusters for
   pleme-io's own M0-M3 e2e tests.
5. **CNCF e2e** — Sonobuoy `--mode=certified-conformance` on a real
   single-node, dual-node, and three-node engenho cluster. Required to
   pass with zero skips for promotion to fleet default.
6. **tameshi attestation chain** — every release artifact carries a
   typed `ConvergenceAttestation` over (source-tree BLAKE3, build-input
   BLAKE3, output BLAKE3). See [`SAGUAO.md`](./SAGUAO.md) for how the
   chain travels through deploy.
7. **kensa compliance proofs** — FedRAMP-Moderate / SOC-2 / CIS-K8s
   profiles applied as admission rules; admission denials carry typed
   evidence-pointers per [`compliant-systems`](../../.claude/skills/compliant-systems/SKILL.md).

A release that fails any of (1)–(6) does not ship. (7) is gated by
operator selection at cluster create time.

---

## III. Wire-compatibility surfaces

### III.1. kubectl

Engenho speaks the upstream Kubernetes REST API on `:6443`. The reference
target is **v1.32** (current N-2 of upstream v1.34 as of 2026-05-18).
Every endpoint kubectl needs is implemented:

| Path | Verbs | Source |
|---|---|---|
| `/api`, `/apis` | GET (discovery) | `engenho-apiserver::discovery` |
| `/api/v1`, `/apis/$g/$v` | GET (APIResourceList) | `engenho-apiserver::discovery` |
| `/openapi/v3/...` | GET | `engenho-apiserver::openapi` — generated from `engenho-types` |
| `/api/v1/namespaces/{ns}/{r}` | GET/POST/PUT/PATCH/DELETE | `engenho-apiserver::registry` |
| `?watch=true&resourceVersion=N` | GET (chunked / protobuf) | `engenho-apiserver::watch` |
| `?fieldSelector` / `?labelSelector` | GET | `engenho-apiserver::filter` |
| `PATCH application/apply-patch+yaml` | PATCH (SSA) | `engenho-apiserver::ssa` |
| `/api/v1/.../pods/{n}/exec` | GET upgrade (SPDY/v5 WS) | `engenho-kubelet::exec` |
| `/api/v1/.../pods/{n}/log` | GET (chunked) | `engenho-kubelet::log` |
| `/api/v1/.../pods/{n}/portforward` | GET upgrade | `engenho-kubelet::portforward` |
| `/apis/authorization.k8s.io/v1/selfsubjectaccessreviews` | POST | `engenho-apiserver::auth::can_i` |

Compatibility contract: `kubectl version --client=v1.32 --server` succeeds
against engenho. `kubectl apply -f any-valid-1.32-manifest.yaml` succeeds.
`kubectl get crds` returns the engenho-installed CRD set. Operators built
against k8s 1.32 (Crossplane providers, FluxCD, cert-manager,
external-secrets, KEDA, Karpenter, Cilium when used as CNI) run unchanged.

### III.2. etcd v3 gRPC

Engenho-datastore implements the etcd v3 protocol surface that the
apiserver consumes:

| RPC | Source |
|---|---|
| `KV.Range` | `engenho-datastore::kv::range` |
| `KV.Put` | `engenho-datastore::kv::put` |
| `KV.DeleteRange` | `engenho-datastore::kv::delete` |
| `KV.Txn` | `engenho-datastore::kv::txn` (compare-and-swap; the load-bearing primitive for SSA) |
| `KV.Compact` | `engenho-datastore::kv::compact` |
| `Watch.Watch` (bidi stream) | `engenho-datastore::watch` |
| `Lease.LeaseGrant` / `LeaseRevoke` / `LeaseKeepAlive` | `engenho-datastore::lease` |
| `Maintenance.Status` / `Snapshot` / `Defragment` | `engenho-datastore::maint` |
| `Auth.*` | stubs returning `Permission denied`; engenho's authn is at apiserver |

No other RPCs. The contract is "what the upstream kube-apiserver actually
calls" — verified by replaying the apiserver's etcd client trace against
engenho-datastore in `engenho-datastore::tests::apiserver_replay`. The
upstream etcd test suite's KV / Watch / Lease subset is run as a wire-axis
contract test (§II.3).

### III.3. CRI v1

Engenho-kubelet speaks CRI v1 over UDS as a *client* to the runtime
daemon. The reference target is `runtime.v1.RuntimeService` and
`runtime.v1.ImageService` per `kubernetes/cri-api`. Tested against
containerd in M0 + youki/oci-distribution direct in M3+.

### III.4. CNI v1

Engenho-cni emits CNI binaries (or invokes its internal trait
implementations) that obey the CNI v1 contract on stdin JSON + env vars.
Conformance verified against `containernetworking/cnitool`.

### III.5. Gateway API + Ingress

Engenho-gateway (M3+) implements Gateway API v1.1 GA: GatewayClass /
Gateway / HTTPRoute / GRPCRoute / TLSRoute. Legacy Ingress v1 is
forwarded to the Gateway API path with a conversion controller. Sources:
[gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/).

### III.6. CSI v1

Out of scope for M0. local-path is in-tree. CSI plugins (M3+) consume
gRPC `csi.v1` UDS sockets per the CSI spec.

---

## IV. Compounding hierarchy

Per pleme-io/CLAUDE.md ★★★ Operating Principle #6: "Single goals are
anti-goals." Engenho is structured so that every M-milestone *unlocks* a
class of subsequent moves. The dependency graph:

```
M0.0  engenho-types generation pipeline (forge-gen → upstream OpenAPI → typed Rust)
        │
        ├─► M0.1  apiserver registry (every kind compiles for free)
        │         │
        │         ├─► M0.2  controller registry (one #[derive(TataraController)] per kind)
        │         ├─► M0.3  scheduler plugin registry (one #[derive(SchedPlugin)] per plugin)
        │         └─► M0.4  CRD machinery (CRD spec is itself a typed kind)
        │
        └─► M2.1  caixa CRD generation (CaixaSpec is a kind, same pipeline)
                  │
                  └─► M2.2  caixa-native controller (lowering = typed transformation)
```

Engenho-types is the **single load-bearing primitive**. The OpenAPI ingest
proves itself once on a single kind (Pod), then mechanically scales to
~150 kinds with zero per-kind hand authoring. This is the Pillar 12 unlock
that gives engenho its ~15× LoC advantage over k3s — k3s vendors the same
Go types written by hand upstream; engenho generates them mechanically and
the same generator pipeline regenerates them on every minor bump.

**The non-negotiable rule.** Hand-authoring a K8s resource type in engenho
is a CI-rejected anti-pattern. Even if the generator is incomplete for one
kind, the fix is to extend the generator — never to hand-roll the type.
Same rule as Crossplane (`format!()` of Go is forbidden) and NixAST
(string-concat of Nix is forbidden).

The second compounding primitive is **shigoto**. Every reconciliation
loop is a `shigoto::Dag`; every controller is a `shigoto::Job`; watch
dispatch is a `shigoto::Dag` with fan-out wave execution. Once shigoto
hosts the first controller, the other 17 fall in with `#[derive(TataraController)]`
boilerplate.

The third compounding primitive is **tatara's `defguest` daemon mode**
(§IX). Once tatara can supervise a long-running WASM/native daemon under a
typed health predicate, every engenho subsystem (apiserver, kubelet,
controller-manager, scheduler, kube-proxy, DNS) becomes a tatara-supervised
daemon — and every *other* pleme-io daemon (caixa Supervisor, magma
apply-engine, the rate-limited-consumer brokers in `pleme-nats`) benefits
from the same primitive. One small piece of tatara surgery unlocks the
fleet's daemon supervision surface.

The fourth compounding primitive is **engenho-test**. Once the in-process
test-cluster spawner works, every pleme-io repo that today depends on
kind/k3d migrates to `engenho-test` — and engenho's own conformance
testing collapses from "spin up a VM, install k3s, run tests" to "spawn
an in-process cluster, run tests." Test iteration time goes from minutes
to seconds; CI cost drops accordingly.

---

## V. Testing pyramid

The pyramid is bottom-up; each layer is a hard gate before the next runs.

### V.1. L0 — type proofs (the compiler)

The Rust compiler is the first verifier. Pillar 10. Invariants enforced
at the type level:

- Every K8s resource is a typed struct with `#[derive(KubeResource, TataraDomain, Serialize, Deserialize, JsonSchema)]`. Adding a field that violates upstream OpenAPI's required-field semantics is a build error.
- Every controller's `Reconcile` body returns `Result<ReconcileResult, EngenhoError>`. Forgetting to handle a typed error variant is a build error.
- Every storage operation against `engenho-datastore` is typed in terms of `KeyRange<R>` where `R: KubeResource`. Cross-typing a Pod read into a Service registry is a build error.
- Every RBAC rule is typed in terms of `(Verb, Group, Resource, ResourceName)` tuples — there are no string-typed RBAC rules.
- Every watch subscription returns `WatchStream<R>` not `WatchStream<Bytes>` — deserialization is mechanical and validated at the type level.

CI gate: `cargo check --workspace --all-features`. ~5 seconds incremental.

### V.2. L1 — unit tests (cargo test, per crate)

Every crate ships a `tests/` directory. Conventions:

- Mocks for external dependencies via the `trait` pattern (e.g., `trait CriClient`, `trait Datastore`, `trait Network`). Test runners construct an in-memory mock.
- No external network. No real containerd. No real etcd.
- Every public function tested for happy path + at least one error case.
- Snapshot tests for any rendered output (uses `insta` per substrate convention).

CI gate: `cargo test --workspace`. ~30-90 seconds incremental.

### V.3. L2 — property tests (proptest)

The load-bearing subsystems carry property tests:

| Subsystem | Property | Source |
|---|---|---|
| SSA merge | `merge(a, merge(b, c)) ≡ merge(merge(a, b), c)` (associativity) under non-overlapping field owners | `engenho-apiserver::ssa::tests` |
| Watch revision | `for any (state, since-rev): watch_replay(state, since-rev) yields events in revision order` | `engenho-datastore::watch::tests` |
| RBAC eval | `for any (subject, verb, resource, rules): authorize is monotonic in rules` (adding allow never denies) | `engenho-apiserver::authz::tests` |
| Scheduler | `for any (pods, nodes, profile): schedule produces a binding iff at least one node passes Filter` | `engenho-scheduler::tests` |
| Pod lifecycle | `for any (podspec, runtime-events): pod-worker reaches a terminal state` | `engenho-kubelet::worker::tests` |
| Caixa lowering | `for any caixa: lower(caixa) is the same Deployment given the same input` | `engenho-caixa::tests` |

CI gate: `cargo test --workspace --features proptest` on the proptest job
(may take several minutes; runs in parallel with L1).

### V.4. L3 — contract tests (wire conformance)

Each wire-compat surface (§III) carries a contract suite that replays the
upstream conformance shape against engenho. The conformance fixtures are
vendored (and BLAKE3-attested) from upstream:

| Contract | Source fixtures | Test runner |
|---|---|---|
| etcd v3 KV/Watch/Lease | `etcd-io/etcd/tests/integration` subset | `engenho-datastore --features wire-contract` |
| CRI v1 RuntimeService/ImageService | `kubernetes/cri-api/pkg/apis/runtime/v1/conformance` | `engenho-kubelet --features cri-contract` |
| kubectl smoke (~80 commands) | `kubernetes/kubernetes/test/cmd` subset | `engenho-apiserver --features kubectl-contract` |
| CNI v1 ADD/DEL/CHECK/VERSION | `containernetworking/cnitool` | `engenho-cni --features cni-contract` |
| OpenAPI v3 emission | `kubernetes/kubernetes/api/openapi-spec/v3/*.json` round-trip | `engenho-types --features openapi-roundtrip` |

CI gate: `cargo test --workspace --features wire-contract`. ~3-5 minutes.

### V.5. L4 — integration tests (in-process clusters)

`engenho-test` spawns a complete in-process engenho cluster — one tokio
runtime, every subsystem in the same process, real Unix sockets for CRI
(against a `containerd-in-process` test double), in-memory datastore. The
test fixtures cover:

- **Apply / Get / Watch round-trip** — `kubectl apply` a manifest, watch it appear, get it back, verify field ownership.
- **Reconcile cycle** — apply a Deployment, verify ReplicaSet + Pods materialize.
- **Schedule + bind** — apply a Pod, verify it gets a `nodeName`.
- **Pod lifecycle** — pod hits Running, then Ready, then Terminated.
- **Exec + log** — `kubectl exec` and `kubectl logs` work against in-process pods.
- **CRD round-trip** — install a CRD, apply a CR, watch it.
- **SSA conflict** — two field owners SSA the same field → conflict.
- **Caixa lowering** — apply a CaixaSpec → engenho-caixa emits Deployment.

CI gate: `cargo test --workspace --features integration`. ~5-10 minutes.

### V.6. L5 — full e2e (real cluster)

`engenho-test` also supports **out-of-process** mode: spawn a real engenho
binary, attach via real network. The same integration suite runs against
the real binary. Additionally:

- Multi-node bootstrap (M0.5+): three engenho nodes, three control planes, join via CSR.
- HA failover (M1+): kill leader, watch a follower take over within N seconds.
- Caixa-mesh contracts (M3+): two Aplicacaos with conflicting :contratos → admission denial.

CI gate: `cargo test --workspace --features e2e --release`. ~15-30 minutes.

### V.7. L6 — CNCF Certified Kubernetes Software Conformance

The ultimate gate. Sonobuoy `--mode=certified-conformance` against a real
3-node engenho cluster. The full `[Conformance]` tag suite (~300 tests as
of v1.32) must pass with **zero skips**. Test artifacts (`e2e.log` +
`junit_01.xml`) are tameshi-attested and submitted to
[cncf/k8s-conformance](https://github.com/cncf/k8s-conformance).

CI gate: gates **M4 release**. Until M4, partial Sonobuoy passes are
acceptable as long as the running per-SIG pass-rate climbs each PR.

### V.8. L7 — attested release

Every engenho release artifact:

- BLAKE3-attested by tameshi (Pillar 10).
- Signed via lacre.
- Published to cartorio (the compliance-receipt registry).
- Receipted by tabeliao (the notary).

A release missing any of those four does not promote past staging. See
[`compliant-artifact-provability`](../../.claude/skills/compliant-artifact-provability/SKILL.md).

---

## VI. Determinism contract (in detail)

### VI.1. Generated source

`engenho-types` is regenerated by `forge-gen` from a pinned upstream
OpenAPI v3 input. The input is BLAKE3-attested; the output is
BLAKE3-attested; the generator itself is BLAKE3-attested. Regenerating
from the same input yields byte-identical output. Verified in CI by:

```bash
cargo run -p forge-gen -- engenho-types --check
# fails if generated source != tree source
```

### VI.2. Plan / scheduling output

The scheduler is a pure function of `(snapshot, candidate-pod)` →
`Option<NodeName>`. For any fixed `(snapshot, candidate-pod)`, the same
binding is produced — verified by proptest in
`engenho-scheduler::tests::deterministic`. Wave-concurrent scheduling
across multiple pods does NOT preserve order-of-binding, but the *set* of
bindings is deterministic given the same input set.

### VI.3. Watch replay

Given any `(state, since-rev)`, the watch stream replays the same event
sequence. The MVCC keyspace is append-only modulo compaction; the
revision counter is monotonically increasing. Verified by
`engenho-datastore::watch::tests::replay_invariant`.

### VI.4. SSA merge

Strategic-merge-patch is the gnarliest spot for non-determinism in
upstream Go (Go map iteration order leaks into merge output). Engenho's
SSA implementation uses `BTreeMap<FieldOwner, FieldOwnership>` everywhere;
merge output is byte-deterministic. Verified by
`engenho-apiserver::ssa::tests::byte_repro`.

### VI.5. Build determinism

Engenho builds via `substrate/lib/rust-tool-release-flake.nix` under Nix
with `--check-meta`. Two builds from the same git revision produce
byte-identical binaries. Verified by `nix build .#engenho --rebuild` in
CI.

### VI.6. Where non-determinism is intentional

The shigoto scheduler chooses wave execution order based on tokio's
scheduling. This is intentional — order doesn't matter for the typed
result. The verification harness checks **result equivalence under
permutation**, not order equivalence.

---

## VII. Verifiability surface (in detail)

### VII.1. Type proofs

Pillar 10. Rust compiler. The thing.

### VII.2. Property tests

Every load-bearing pure function (SSA merge, scheduling, RBAC eval, watch
replay, caixa lowering) carries proptest cases at L2.

### VII.3. Contract tests

L3 wire-conformance fixtures vendored from upstream. The list:

- `etcd-io/etcd@v3.5.21`'s KV/Watch/Lease integration subset (~120 tests
  reusable; balance is Raft + RAFT-LOG which engenho doesn't need until M1).
- `kubernetes/cri-api@v1.32.0` conformance subset (~80 tests).
- `kubernetes/kubernetes@v1.32.0` test/cmd subset (~120 commands).
- `containernetworking/cnitool@latest`.
- `kubernetes/kubernetes/api/openapi-spec/v3/*.json` round-trip schemas.

### VII.4. Integration tests

L4. In-process. ~30-50 tests covering the kubectl-facing happy paths.

### VII.5. End-to-end tests

L5. Real binary, real network. Multi-node M0.5+; HA M1+.

### VII.6. Conformance tests

L6. Sonobuoy. The ~300 `[Conformance]` tests must pass for CNCF
certification.

### VII.7. Attestation

L7. tameshi BLAKE3 chain, signed via lacre, receipted via cartorio +
tabeliao. Every artifact transferable and verifiable offline.

### VII.8. Compliance proofs

Kensa compliance profiles loaded at cluster bootstrap. Admission rejects
non-compliant resources with typed evidence. FedRAMP-Moderate / SOC-2 /
CIS-K8s profiles ship in-tree. Custom profiles via
[`helm-compliance-overlays`](../../.claude/skills/helm-compliance-overlays/SKILL.md).

### VII.9. Live verification

`engenho status --verify` runs the L0+L4 verification surface against a
live cluster: type-check the cluster state, run the integration suite
against a clone, report drift. Operational verification, not just
build-time.

---

## VIII. Caixa / Aplicacao native integration

### VIII.1. The collapsed path

Today's caixa → pod path through k3s:

```
caixa.lisp ─► feira build ─► caixa-helm renders Chart ─► HelmRelease CR
                                                            │
                                                            └─► FluxCD helm-controller
                                                                  │
                                                                  └─► Helm renders Deployment
                                                                        │
                                                                        └─► kube-apiserver
                                                                              │
                                                                              └─► controller-manager
                                                                                    │
                                                                                    └─► kubelet
                                                                                          │
                                                                                          └─► containerd
                                                                                                │
                                                                                                └─► Pod (5-6 lossy translation steps)
```

Engenho's native path (M2+):

```
caixa.lisp ─► feira build ─► CaixaSpec CR
                                  │
                                  └─► engenho-caixa controller (typed Rust)
                                        │
                                        └─► engenho-controllers emits Deployment + Service
                                              │
                                              └─► engenho-scheduler binds Pod
                                                    │
                                                    └─► engenho-kubelet runs Pod (1 lossy step)
```

The five intermediate translations are eliminated. The typed AST stays
typed end to end. There's no Helm template language between the operator's
intent and the running pod. M2/M3 slot information that gets flattened by
Helm today (limits, behavior, upgrade strategies, mesh contracts) survives
to admission and scheduling.

### VIII.2. CaixaSpec as a first-class CRD

Engenho ships a built-in `caixa.engenho.io/v1alpha1` CRD set:

- `Biblioteca` — typed library (rendered as zero K8s resources; pure consumption)
- `Binario` — typed binary (rendered as Job or one-shot Pod)
- `Servico` — typed service (rendered as Deployment + Service + optionally HPA + PDB)
- `Supervisor` — typed supervisor tree (rendered as parent Deployment + child Pods + restart policy)
- `Aplicacao` — typed mesh of Servicos (rendered as the union of all Servicos + NetworkPolicy + Gateway routes + admission rules for :contratos)

Each CRD is mechanically derived from the corresponding caixa type via
`#[derive(TataraDomain, KubeCustomResource)]`. There is no
"caixa-to-yaml" intermediate format — the caixa Lisp form expands to the
CRD's typed struct directly.

### VIII.3. Aplicacao mesh enforcement

The :contratos slot of an Aplicacao declares WIT-typed inter-Servico
contracts. Engenho-mesh validates at admission time:

- A Servico cannot expose an endpoint not in its WIT export.
- A Servico cannot consume an endpoint not in its WIT import.
- Cross-Servico calls are rejected by NetworkPolicy unless declared.

This is enforcement that today's k3s + Cilium + Gateway API stack cannot
do — there is no typed channel between the WIT declaration and the
runtime network policy. Engenho closes the loop.

### VIII.4. Programs as RuntimeClass

Tatara's tlisp WASI runtime is registered as a Pod RuntimeClass:

```yaml
apiVersion: v1
kind: Pod
metadata: { name: my-program }
spec:
  runtimeClassName: tlisp
  containers:
  - name: app
    image: registry.pleme.io/programs/my-program:v1
```

The kubelet routes runtimeClassName=tlisp to `runwasi-tatara`, which
hosts the tlisp interpreter inside a wasmtime sandbox. A Pod can be an
OCI container OR a tlisp program. This is the capability k3s + krustlet
never achieved — running typed pleme-io programs as first-class k8s
workloads.

---

## IX. Tatara integration

### IX.1. The minimal tatara surgery

For engenho to host its own subsystems as tatara-supervised daemons, three
small extensions to tatara are required:

1. **`WasmSpec::restart_policy: Option<RestartPolicy>`** in `tatara-vm/src/guest.rs`.
   Enum: `Always | OnFailure | Never`. Default `OnFailure`.

2. **`WasmSpec::health_check: Option<HealthPredicate>`** in `tatara-vm/src/guest.rs`.
   HealthPredicate is a `BoundaryCondition` (already exists in `tatara-process`) — a
   PromQL query, a TCP probe, an HTTP GET, or a custom Lisp predicate.

3. **Phase loop in `tatara-reconciler/src/controller.rs`**: during the
   `Running` phase, periodically evaluate the `health_check`. On failure +
   `restart_policy=OnFailure`, transition back to `Forking` to restart the
   guest. On failure + `restart_policy=Always`, idem. Existing phase
   machinery handles `Never`.

All other tatara surfaces are unchanged. The surgery is bounded — a single
PR of ~200-400 LoC.

### IX.2. Engenho components as tatara processes

Once the surgery lands, each engenho subsystem can be declared as a
tatara process:

```lisp
(defguest
  :name engenho-apiserver
  :kind (:wasm :runtime Wasmtime :component "engenho-apiserver.wasm")
  :restart-policy Always
  :health-check (:tcp :port 6443)
  :liveness-probe (:http :path "/livez" :port 6443 :period 10s))

(defguest
  :name engenho-controllers
  :kind (:wasm :runtime Wasmtime :component "engenho-controllers.wasm")
  :restart-policy Always
  :health-check (:lisp (apiserver-reachable? :port 6443))
  :depends-on (engenho-apiserver))
```

This makes engenho's *own supervision tree* a tatara declaration —
turtles all the way down. Per the CSE thesis: the substrate that supervises
engenho is the same substrate engenho supervises. Crash recovery,
upgrade-in-place, attestation chain — all carried by tatara, not
reimplemented in engenho.

### IX.3. Engenho as a tatara binary

The shipping engenho binary statically links `tatara-engine` and
`tatara-reconciler`. The binary's main loop is:

```rust
fn main() -> Result<()> {
    let spec = engenho_cli::parse_args()?;          // Lisp form → EngenhoClusterSpec
    let processes = engenho_lowering::lower(spec)?; // → Vec<ProcessSpec>
    tatara_reconciler::run(processes)               // tatara takes over
}
```

Engenho's job is to *describe* the cluster as typed processes; tatara's
job is to run them. Engenho contributes the domain (k8s + caixa);
tatara contributes the runtime (supervision + health + restart +
attestation).

### IX.4. ProcessSpec for k8s components

Every engenho subsystem maps to a `ProcessSpec` (the typed IR in
`tatara-process/src/crd.rs`) along the 6-axis lattice:

| Subsystem | Horizon | Structure | Substrate | Coordination | Trust | Intelligence |
|---|---|---|---|---|---|---|
| apiserver | Unbounded | Flat | Compute | Concurrent | Internal | Monotone |
| datastore | Unbounded | Hierarchical | Storage | Sequential | Internal | Monotone |
| controllers | Unbounded | Hierarchical | Compute | Concurrent | Internal | Adaptive |
| scheduler | Unbounded | Flat | Compute | Sequential | Internal | Adaptive |
| kubelet | Unbounded | Hierarchical | Compute | Concurrent | Internal | Monotone |
| kube-proxy | Unbounded | Flat | Network | Concurrent | Internal | Monotone |
| CNI | Bounded | Flat | Network | Sequential | Internal | Monotone |
| DNS | Unbounded | Flat | Network | Concurrent | External | Monotone |

The lattice classification is consumed by the reconciler to choose phase
transitions (e.g., Adaptive subsystems re-enter Pending on spec change;
Monotone subsystems hot-reload).

---

## X. Phased path-down

Per Operating Principle #0: phases serve the destination. Each milestone
ships a tangible, testable, deployable artifact AND advances the
substrate. No phase is allowed to ship without its full L0-L4 test surface
(L5-L6 phase-in as features land).

### M0.0 — Generation pipeline (the load-bearing primitive)

**Scope.** `engenho-types` only. `forge-gen` ingest of upstream OpenAPI v3
(pinned to v1.32). Generated source for ALL ~150 core+stable kinds.
Round-trip tests against upstream OpenAPI schemas.

**Test gates.** L0 (compiles). L1 (per-kind unit tests for
serialize/deserialize/JSON-Schema round-trip). L3 (OpenAPI v3 round-trip
conformance against upstream `api/openapi-spec/v3/`).

**Determinism gate.** `forge-gen --check` returns 0; regenerated source
== tree source byte-for-byte.

**Compounding unlock.** Every subsequent milestone consumes engenho-types
directly; no hand-authored types.

**Estimated effort.** 3-4 weeks. Most of the work is in forge-gen extension
(it already does this for Terraform, MCP, Crossplane — adding a
KubeResource backend is a sibling).

### M0.1 — Datastore + apiserver registry (kubectl handshake)

**Scope.** `engenho-datastore` (etcd-v3 over fjall) + `engenho-apiserver`
discovery / OpenAPIv3 / core+apps+rbac CRUD / watch / SSA / JSONPatch /
SMP. JWT-SA authn. RBAC authz. No admission webhooks. No CRDs. No
controller-manager. No scheduler. No kubelet.

**Test gates.** L0-L3 fully green. L4 covers: `kubectl version` /
`kubectl get` / `kubectl apply -f` / `kubectl auth can-i`. `kubectl get`
returns empty lists for every kind. `kubectl apply -f pod.yaml` succeeds
and `kubectl get pod` returns the pod (which never runs — that's M0.4).

**Bootstrap consumer.** `engenho-test` in-process apiserver: every pleme-io
repo that today imports `kube` for type structs can additionally import
`engenho-test::Cluster` for a real in-process apiserver.

**Compounding unlock.** Every controller, every admission webhook, every
CRD, every kubectl plugin in the pleme-io fleet can now develop against
engenho. The watch / SSA / JSONPatch / SMP machinery is the bedrock.

**Estimated effort.** 10-14 weeks. Watch dispatch + SSA + SMP are the
hard pieces (~3 weeks each); the rest is mechanical.

### M0.2 — Controller-manager + scheduler (workloads scheduled)

**Scope.** 18-controller built-in set + 11-plugin scheduler. Leader
election via Lease. No CRDs yet. No kubelet — Pods get bound to nodes but
nothing runs them.

**Test gates.** L0-L4 green. L4 covers: apply Deployment → ReplicaSet
materializes → Pods materialize → Pods get `nodeName`. Apply Job → Pod
materializes and reaches a typed pending state. Apply Namespace + RBAC
binding → RBAC works.

**Compounding unlock.** Engenho is now a fully-fledged API server with a
control plane — even without a runtime, it can be used to develop
admission webhooks, custom controllers, and CRDs. kubectl works for
everything except the actual workloads.

**Estimated effort.** 8-10 weeks. Each controller is ~500-1000 LoC under
the macro; the scheduler is ~3-4k LoC total.

### M0.3 — Kubelet + CRI (workloads RUN)

**Scope.** `engenho-kubelet` against containerd CRI. Evented PLEG.
Pod-worker. hostPath + Secret (via cofre) + ConfigMap + projected SA-token
volumes. Probes. Logs. `/exec` SPDY.

**Test gates.** L0-L4 green. L4 e2e: apply Deployment → Pod actually runs
to Ready. `kubectl logs` works. `kubectl exec` works. `kubectl
port-forward` works.

**Compounding unlock.** **This is the milestone where engenho becomes a
useful cluster.** Single-node, single-runtime, but functionally a
Kubernetes cluster.

**Estimated effort.** 10-14 weeks. The `/exec` SPDY shim is the gnarliest
piece (~2 weeks). Pod-worker state machine + volume drivers + probes ~6
weeks. CRI client wiring + Evented PLEG ~3 weeks.

### M0.4 — Networking (multi-pod, single-node)

**Scope.** `engenho-cni` (bridge+ipam+portmap+loopback) + `engenho-kubeproxy`
(nftables) + `engenho-dns` (hickory-dns) + `engenho-localpath`.

**Test gates.** L0-L5 green. L5 covers: two Pods on the same node talk to
each other via ClusterIP. NodePort works from the host. PVC binds to a
local-path PV. Pod resolves `kubernetes.default.svc.cluster.local`.

**Compounding unlock.** Engenho is now a complete single-node Kubernetes
distribution. Ready for homelab use on `rio`.

**Estimated effort.** 6-8 weeks. CNI is ~2k LoC; kube-proxy is ~5k; DNS
is ~3k.

### M0.5 — Multi-node

**Scope.** VXLAN overlay extension to engenho-cni. ServiceLB
(klipper-equivalent). CSR bootstrap flow for joining nodes. `engenho join`
CLI verb.

**Test gates.** L5 multi-node tests. 3-node bootstrap, Pod-to-Pod across
nodes via VXLAN, LoadBalancer-type Service.

**Compounding unlock.** Engenho works as a small cluster (3-5 nodes).

**Estimated effort.** 4-6 weeks.

### M1 — HA + native Caixa

**Scope.** openraft + fjall HA datastore mode (single-node spec still
works; HA mode is opt-in). `engenho-caixa` controllers consuming CaixaSpec
CRD. Caixa-native lowering: Biblioteca / Binario / Servico / Supervisor
emit typed Deployments/Jobs/Pods directly.

**Test gates.** L5 HA tests: kill leader, follower takes over within 30s.
L4 caixa tests: apply CaixaSpec → engenho-caixa emits Deployment.

**Compounding unlock.** Engenho is HA-capable. Caixa workloads bypass
Helm. The fleet's caixa-helm + HelmRelease pipeline becomes optional for
engenho clusters.

**Estimated effort.** 12-16 weeks.

### M2 — Typescape-native

**Scope.** Controllers expressed as tatara programs running in tatara's
`defguest` daemon mode. Generator pipeline (forge-gen) extended to emit
controller skeletons from CRD specs. CRDs declared via `#[derive(TataraDomain)]`
become K8s CRDs automatically.

**Test gates.** Existing L0-L5 surfaces unchanged. Net-new tests for the
controller-as-program runtime.

**Compounding unlock.** The substrate's typed-AST surface and engenho's
controller surface merge. Adding a custom CRD is a single Rust struct
with one derive.

**Estimated effort.** 8-12 weeks.

### M3 — Mesh + compliance + Gateway

**Scope.** `engenho-mesh` Aplicacao native enforcement. `engenho-gateway`
pingora-based Gateway API. `engenho-attest` sekiban admission webhook
in-process. Kensa compliance profiles loaded at bootstrap.

**Test gates.** L5 mesh tests: two Aplicacaos with conflicting :contratos
→ admission denial. L5 Gateway tests: HTTPRoute terminates TLS, routes
to Service. L5 compliance tests: FedRAMP-Moderate profile denies a
non-compliant Pod.

**Compounding unlock.** Engenho is the first Kubernetes distribution that
ships compliance-by-construction. The compliance proof chain (tameshi +
sekiban + kensa) is end-to-end.

**Estimated effort.** 12-16 weeks.

### M4 — CNCF Certified

**Scope.** Polish. Sonobuoy `--mode=certified-conformance` passes with
zero skips on a 3-node engenho cluster. PR to cncf/k8s-conformance.

**Test gates.** L6 fully green. Submission accepted.

**Compounding unlock.** Engenho becomes the first non-Go CNCF Certified
Kubernetes distribution. Marketable.

**Estimated effort.** 4-8 weeks of polish post-M3.

### M5 — Programs as RuntimeClass

**Scope.** runwasi-based RuntimeClass for tlisp programs. Pods can declare
`runtimeClassName: tlisp` and run as WASI modules.

**Test gates.** L4 tests: apply Pod with runtimeClassName=tlisp → tlisp
program runs in wasmtime, sees the cluster's typed env.

**Compounding unlock.** Pleme-io programs run as first-class K8s
workloads. The fleet's `programs/` repo deploys directly to engenho.

**Estimated effort.** 4-6 weeks.

### Total cumulative estimate

M0.0 → M5: roughly 18-24 person-months. For one operator + agents per
CSE's "team-equivalent throughput" thesis, plausible in 12-15 calendar
months with full focus, longer if interleaved with fleet work. Time is
the path-down variable; destination (§II-IX) is locked.

---

## XI. Naming rationale

### XI.1. Why "engenho"

Brazilian-Portuguese per [`THEORY.md` §II.3](./THEORY.md). The
naming-evocation set is "enclosed spaces, flows, growth, craft." Engenho
hits all four:

- **Enclosed space** — an engenho is a fenced productive estate.
- **Flow** — cane → mill → boiler → distillery → warehouse.
- **Growth** — the cane field grows, the workforce grows, the engenho grows.
- **Craft** — *engenho* secondarily means *ingenuity / wit / cleverness*
  in Portuguese; "tem muito engenho" = "is very clever."

The lexicon already in use:

- `magma` — molten executive force for IaC.
- `terreno` — the territory / land where workloads are declared.
- `forja` — the forge that shapes CI artifacts.
- `feira` — the marketplace where caixa is composed.
- `caixa` — the typed SDLC primitive.
- `lacre` — the seal that attests compliance.
- `cofre` — the vault that materializes secrets.
- `samba` — the rate-limited consumer pattern.

Engenho slots cleanly: terreno is *where* workloads are declared; engenho
is *what runs the land*. Magma shapes earth (cloud); engenho shapes
container runtime. Forja shapes CI artifacts; engenho shapes runtime
artifacts. Feira composes caixa; engenho hosts caixa at runtime.

### XI.1.a Why engenho is NOT magma — explicit boundary

Magma and engenho are sibling primitives at distinct substrate layers.
Both are Rust-native typed executors written in the same idiom; they
manage disjoint domains and compose exactly once:

| Axis | `magma` (Pillar 5 execution) | `engenho` (Pillar 7 runtime) |
|---|---|---|
| Substrate domain | Cloud resources (EC2, RDS, S3, VPC, Lambda, IAM, Cloudflare zones, Akeyless gateways) | Container workloads (Pods, Deployments, Services, Ingress, Secrets, ConfigMaps) |
| Wire protocol | Terraform `tfplugin5/6` gRPC | etcd v3 gRPC + CRI v1 + CNI v1 + kubectl REST + Gateway API |
| State format | `terraform.tfstate` v4 (JSON, S3-backed) | etcd-v3 MVCC keyspace over fjall (embedded) or openraft (HA) |
| Input | Pangea-rendered Terraform JSON | Kubernetes manifests (kubectl/Helm/Flux/Caixa-emitted) |
| Output | API calls to cloud control planes | Linux processes / cgroups / network namespaces on a node |
| Replaces | `opentofu` / `terraform` binary | `k3s` / `k0s` / `microk8s` binary |
| Consumer pattern | `pangea-architectures/workspaces/*` calls `magma plan/apply` | kubectl / FluxCD / Crossplane connect to engenho's apiserver |
| Test surface | tfprotov5/6 conformance, OpenTofu provider matrix | CNCF Kubernetes Software Conformance (Sonobuoy) |

The single point of composition: **magma provisions the cloud node
(EC2 instance / Hetzner server / GCP VM) that engenho runs on; engenho
hosts the Pods that consume the RDS / S3 / IAM-role / SecretsManager
resources magma provisioned.** Magma terminates at the OS install;
engenho takes over at first boot. The cluster's Pangea workspace
declares both — magma renders the AWS half, engenho renders the cluster
half — and the operator applies them as one composed plan via the
existing pangea CLI.

The fleet's homelab clusters (rio, home-edge) are the degenerate case:
no magma involved, just engenho on bare-metal NixOS. The fleet's cloud
clusters (nexus-staging, lilitu-edge) are the canonical case: magma
provisions the infra, engenho hosts the workloads.

Both primitives share substrate plumbing:
- `shikumi` for typed config
- `shigoto` for typed work graphs (magma's apply-engine; engenho's
  reconciliation loops)
- `tameshi` + `lacre` + `cartorio` + `tabeliao` for attestation
- `cofre` for secret materialization
- `forge-gen` for code generation (magma generates provider bindings
  from `tfplugin5/6` schemas; engenho generates resource types from
  Kubernetes OpenAPI v3)
- `nix-ast` for emitted Nix
- The substrate's `rust-tool-release` builder for the shipping binary

Anti-pattern: building engenho as "magma for Kubernetes" or hosting
engenho inside magma. The protocols, state formats, test surfaces, and
consumer expectations diverge so completely that conflating them would
double the surface area and halve the precision of each. Two primitives,
one idiom.

### XI.2. Rejected alternatives

- **rebanho** (herd/flock) — too pastoral; doesn't evoke machinery.
- **maestro** — Italian / borrowed; not idiomatically Brazilian.
- **fazenda** (farm) — too broad; engenho is the productive subset of fazenda.
- **moenda** (milling apparatus) — too specific; engenho includes mill + boiler + still.
- **regente** (regent / conductor) — courtly; doesn't evoke construction.
- **viveiro** (nursery) — implies growth but not running workloads.

### XI.3. Linguistic note

Pronounced /ẽˈʒẽɲu/ (en-ZHE-nyoo). The "engenho" / "Engenho" capitalization
follows the prevailing Portuguese-loan-word convention (lowercase as a
common noun; capitalized when referring to the project).

---

## XII. Open questions

These are intentionally unresolved at v1; they need operator decision or
prior-art investigation before M0.1 starts.

### XII.1. Storage backend at HA tier — fjall vs redb vs hybrid

§II.1 names fjall (LSM, log-structured, written in Rust, v3.0 stable
2026). redb (B-tree, LMDB-inspired) is a sibling option. fjall is better
for write-heavy workloads (watch traffic creates ~10× more writes than
reads); redb is better for read-heavy with mostly-static keys. Engenho's
workload is write-heavy on writes-to-objects but read-heavy on watch
streams — both are plausible. Decision: benchmark both on the engenho-
datastore workload before M0.1 lock-in.

### XII.2. CRI client — tonic vs containerd-client crate

The `containerd-client` crate exists and wraps tonic. tonic-direct gives
us full control of the gRPC client; containerd-client gives us the
already-vendored protobuf definitions. Cost/benefit unclear. Decision:
spike both in M0.3 prep.

### XII.3. kube-proxy — nftables-only vs eBPF-optional

§II.1 names nftables-only via rustables/nftnl-rs. KEP-3866 declares
nftables GA in 1.33 and the recommended default. Cilium-style eBPF is
faster but harder to write, and aya-rs is the Rust framework. Decision:
ship nftables-only at M0.4; eBPF backend is an M2 optional plugin.

### XII.4. /exec SPDY vs WebSocket v5

Kubernetes 1.32 supports both SPDY (legacy) and WebSocket v5 (newer) for
exec/attach/portforward upgrades. WebSocket v5 is the recommended path.
Decision: implement WebSocket v5 first; SPDY fallback only if a deployed
kubectl version absolutely needs it.

### XII.5. Audit log — first-class or aux?

K8s audit logs are a major compliance artifact. Engenho should emit them
as a first-class typed stream (per Pillar 10) — but where? Decision:
emit as tameshi-attested NDJSON to a local file by default; pluggable
sinks via shinryu for centralized observability.

### XII.6. Tatara surgery — separate PR or in-tree?

§IX.1 names the tatara surgery as a small PR. Should engenho live in the
tatara tree (one repo) or as a separate repo (consumes tatara)?
Recommendation: separate repo. Tatara stays foundational; engenho stays
Tier 2 per the naming convention. The surgery lands as a tatara PR,
engenho consumes it via Cargo.

### XII.7. Conformance target — v1.32 vs newest stable

Kubernetes minor cadence is ~4 months. By M4 release (12-15 months from
M0.0), upstream stable will be ~v1.36. Decision: lock to v1.32 surface
for M0-M3; bump to N-2 at M4 prep based on CNCF certification windows.

### XII.8. Repo location — pleme-io/engenho or akeylesslabs/engenho?

Engenho is fleet substrate, not customer-facing — pleme-io/engenho is the
default. Akeyless integration (if any) goes in a sibling repo.

---

## Appendix A — Relationship to existing primitives

| Existing primitive | Relationship to engenho |
|---|---|
| `tatara` | engenho is a tatara binary; tatara supervises engenho's subsystems via defguest daemon mode (§IX). Small surgery in tatara unlocks engenho. |
| `tatara-kube` | DEPRECATED. Was Nix-eval-driven SSA against external clusters. Engenho IS the cluster — orthogonal. |
| `tatara-reconciler` | FluxCD-adjacent reconciler. Will continue to coexist with engenho — tatara-reconciler manages foreign clusters; engenho IS a managed cluster. |
| `caixa` | engenho-caixa natively reconciles CaixaSpec CRDs without Helm intermediation (§VIII). caixa-helm + caixa-flux remain available for foreign clusters. |
| `feira` | `feira build` emits CaixaSpec CRDs that engenho-caixa consumes. `feira deploy` to an engenho cluster bypasses Helm. |
| `arch-synthesizer::k8s` | The hand-authored 15-primitive subset becomes a re-export of engenho-types. Composition logic stays in arch-synthesizer; raw types move to engenho-types. |
| `shigoto` | Every engenho controller and reconciliation loop is a shigoto::Dag. Bootstrap consumer for engenho's typed work surface. |
| `shikumi` | All engenho config (engenho.lisp, per-component YAML overrides) parsed via shikumi. |
| `cofre` | engenho-kubelet has a cofre client; Secret objects in engenho carry references, not plaintext. |
| `tameshi` / `sekiban` / `kensa` | engenho-attest is the in-process admission webhook running sekiban + kensa; every artifact carries a tameshi receipt. |
| `magma` | Sibling primitive at a different layer. Magma realizes Pangea (cloud); engenho realizes caixa (runtime). |
| `pleme-actions` | engenho's release pipeline uses the standard pleme-action set (release-rust-tool, publish-cratesio, etc.). |
| `helm-compliance-overlays` | Kensa profiles consumed by engenho-attest are the same overlays. |
| `forge-gen` | The generation backbone for engenho-types. Same forge-gen powers magma's typed Terraform protocol bindings. |
| `nix-ast` | engenho's `flake.nix` is typed-AST-emitted, not string-concatenated. |
| `programs` | tlisp WASI runtime becomes an engenho RuntimeClass (§VIII.4). |
| `terreno` | Declaration-layer sibling. terreno declares the cluster shape; engenho runs the shape. |
| `pangea-operator` | Out-of-cluster operator that drives infrastructure. Can drive engenho clusters as one of its targets (M3+). |

---

## Appendix B — Risks, red flags, and external prior art

### B.1. The rusternetes question

`github.com/calfonso/rusternetes` is the only known pure-Rust Kubernetes
reimplementation in the wild. Single committer, latest commit 2026-05-18.
Claims 216k LoC across 10 crates, 3,100+ tests, 94.1% Sonobuoy
conformance on v1.35. **Treat as inspiration / file-structure reference
only.** Red flags: single contributor, no team / blog / talk / production
users, jumped from ~9% to 94% in 63 test rounds, depends on bollard
(Docker socket) instead of CRI, README has zero "experimental" warnings.
The shape is a credible LLM-generated artifact; engenho should not
depend on its code, but its module layout is worth studying.

### B.2. Watch dispatch — where every clone fails

Revision-coherent listwatch, broadcast trees, compact_revision semantics
— this is the single hardest piece. Plan for a 3-4 week spike with
proptest fuzzing against the upstream Go etcd watch implementation.
Failure here means listwatch consumers (every controller, every operator,
kubectl -w) silently misbehave.

### B.3. SSA non-determinism

Upstream Go's strategic-merge-patch has known order-dependent edge cases
(map iteration leaks into output). Engenho's BTreeMap-everywhere SSA is
deterministic by construction; cost is a small allocator perf delta. Worth
it.

### B.4. CRI implementation surface

containerd is Go. Calling it via tonic is fine but introduces a Go
dependency in the deployment surface. M0-M2 ship containerd; M3+ optional
path is youki+oci-distribution+a Rust shim. The Rust shim itself is a
~40k-LoC project (see §V of this doc's cross-references). Out of scope
for engenho M0-M2; revisit M3+.

### B.5. eBPF code is hard

`aya-rs` is a great framework but writing eBPF is its own discipline. The
optional eBPF kube-proxy backend (M2) is a stretch goal; nftables is the
production path.

### B.6. Conformance regression

The upstream conformance suite changes per minor. Tracking N-2 means
engenho is always 8-12 months behind upstream features. Pleme-io's
typescape can leapfrog this by adding features (compliance, attestation,
caixa-native) that upstream doesn't have — but the conformance baseline
must keep moving.

### B.7. Crash-loop on cluster boot

Engenho's first M0.4 boot on a real node is the highest-risk operational
moment. Pre-bake an `engenho doctor` command that runs the L4 integration
suite in-process against `/proc` and `/sys` before binding any external
port. Catch the missing-cgroup-controller case at boot, not at the first
Pod.

---

*Destination locked. Path-down begins at M0.0.*
