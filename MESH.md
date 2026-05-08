# Mesh — typed service mesh primitive (pleme-io shape)

> **Frame.** [`THEORY.md`](./THEORY.md) names the substrate's
> compounding thesis (one rule, mechanical re-derivation across
> environments). [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) places
> one Servico per ComputeUnit. [`MESH-COMPOSITION.md`](./MESH-COMPOSITION.md)
> already composes Servicos into typed Aplicacaos *within* a cluster
> via `:contratos` (typed inter-Servico edges) and Cilium NetworkPolicy.
> [`SAGUAO.md`](./SAGUAO.md) handles fleet-wide identity for *people*.
>
> **This doc is the missing layer:** how Servicos in an Aplicacao
> *talk* to each other — mTLS, retries, circuit-breakers, identity-
> aware policy, L7 routing, observability — at fleet scale, expressed
> as one typed primitive that renders to every concrete mesh runtime.
>
> Aplicacao's `:contratos` slot today says *who can talk to whom and
> with what WIT shape*. It does NOT say *how the bytes get there*. A
> mesh is the typed answer to the *how*.
>
> Working name: `(defmesh …)` — a fifth peer to `:kind` Servico /
> Biblioteca / Binario / Supervisor / Aplicacao. Hanabi is the
> L7/BFF-aware **sub-component** of the data plane, not the whole
> mesh.

---

## I. Destination first (principle #0)

> *"Path of least resistance is a cardinal sin. The first design pass
> is what is the absolute-best long-term answer regardless of how
> long it takes."*  — `pleme-io/CLAUDE.md`

The destination is **a typed mesh primitive that subsumes every
service-mesh problem the fleet will ever face — single rule, multiple
backends — with hanabi composed in for L7-aware behavior**. Every
intermediate step is a phase toward that endpoint, not an alternative
to it.

When complete, the operator writes:

```lisp
(defmesh openclaw-mesh
  :aplicacao   openclaw
  :identity    spiffe                       ; or :authentik-mtls / :pki-only
  :data-plane  :proxy linkerd-style-rust
                :l7    hanabi               ; ←  hanabi as sub-component
  :mtls        :strict
  :retries     {:default 3 :budget 0.2}
  :timeouts    {:default 30s :slow 5m}
  :circuit-breaker {:max-requests 100 :max-failures 10 :reset 30s}
  :rate-limit  {:per-source 1000/s}
  :observability {:traces vector :metrics victoria :logs victoria}
  :policy      :contratos          ; inherit from Aplicacao :contratos
  :gateway     :entrada            ; inherit from Aplicacao :entrada
  :saguao      true)               ; route through fleet IdP for human edges
```

…and the renderer mechanically emits *for the chosen target backend*:

| Backend | Renderer emits |
|---|---|
| **Pure k8s + Cilium** (today) | sidecar-injector MutatingWebhook, init-container iptables/eBPF redirector, CSI mount of SPIFFE certs, CNAME-style ServiceAccount→SPIFFE-ID, vector sidecars, hanabi sidecar with the merged BFF config, CiliumNetworkPolicy from `:contratos`, ServiceMonitor scrapes |
| **Linkerd backend** | Linkerd CRDs (`Server`, `ServerAuthorization`, `HTTPRoute`), TrafficPolicy, mTLS via Linkerd's identity, hanabi composed as L7 sidecar |
| **Istio backend** | `PeerAuthentication`, `AuthorizationPolicy`, `VirtualService`, `DestinationRule`, hanabi as Envoy WASM filter or external-auth ext |
| **Cilium ServiceMesh** | `CiliumEnvoyConfig`, eBPF interception, SPIFFE through Cilium's mTLS, hanabi as L7 filter chain |
| **Native (no mesh)** | warning at synth time, fall back to the proxy in hanabi only — no mTLS, no sidecar, just BFF-shaped routing |

One typespec; many concrete deployables. **The point is not to ship
a mesh. The point is to make every mesh choice the fleet ever needs
to make a typed value with mechanical re-derivation.**

---

## II. The mesh problem

### II.1. What an Aplicacao does NOT today have

`MESH-COMPOSITION.md` defines `:contratos` as the typed inter-Servico
edge:

```
{:de catalog :para cart :wit /cart/contracts/add-item :endpoint http://...}
```

That's the *interface contract*. It's mapped to a Cilium
NetworkPolicy that allows traffic on that port from that source. But:

- **No identity verification** — anyone in the source ns can hit
  the destination port; "is this really catalog?" is a network claim,
  not a cryptographic one.
- **No mTLS** — bytes are plaintext on the wire (or TLS via per-app
  configuration, which is per-app boilerplate).
- **No retry / circuit-breaker / timeout** — each Servico hand-codes
  these (or doesn't) inside its tokio client.
- **No L7 awareness** — the policy is L4 (IP+port). HTTP path
  authorization, GraphQL operation gating, gRPC method ACLs, all
  belong to L7.
- **No traffic shaping** — canary, A/B, weighted routing, all
  manual.
- **No observability uniformity** — each Servico emits its own
  traces / metrics in its own shape.
- **No identity-aware human edges** — `:entrada` (Aplicacao gateway)
  has Host/Path routing but no integration with [`SAGUAO.md`](./SAGUAO.md)'s
  passaporte/crachá identity flow at the *internal* L7 boundary.

A typed mesh is what closes all of those, *uniformly*, *as one rule*.

### II.2. Why not just adopt Istio / Linkerd / Cilium directly

**Three reasons.**

1. **Substrate value is rendering, not running.** The pleme-io thesis
   (`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`) is that the typespec is
   the executable proof — adopting any one of those meshes wholesale
   means the operator is still writing Istio CRDs / Linkerd
   ServerAuthorization / Cilium L7 policies *by hand*. Same as a
   typescape leaking into raw Terraform. The substrate's job is to
   own the typed surface and emit those CRDs from a single
   declaration.

2. **One rule across multiple environments.** The fleet runs k3s
   today, EKS tomorrow, podman+nomad on rio after that, browser
   wasm later. Linkerd is k8s-only; Istio is k8s-only; Cilium is
   k8s-only. The substrate's `defmesh` lowers to *all* of them and
   to a `proxy-only-no-mesh` backend for podman/nomad/wasm —
   meaning the same `:contratos` graph compiled in dev (no mesh,
   just hanabi+iptables) ships to staging (Linkerd) ships to prod
   (Cilium ServiceMesh) ships to a future air-gapped DoD enclave
   (Istio) — *with the operator changing zero LOC*.

3. **L7-aware features belong to hanabi, not the proxy.** Every
   off-the-shelf mesh's L7 layer is Envoy-or-Linkerd-proxy with
   WASM extensions. Hanabi already has CSP/CORS/HSTS/rate-limit/
   GraphQL-aware proxy/federation/session/health-check. Re-doing
   those as Envoy WASM filters duplicates effort. Composing hanabi
   in as the L7 sub-component lets us *reuse* hanabi's existing
   surface. Pillar 12: generation over composition; here,
   composition of an existing, validated, typed sub-component.

### II.3. What hanabi is, and is not, today

| Hanabi has | Hanabi lacks (mesh-shaped gaps) |
|---|---|
| Axum-based HTTP server | Sidecar injection / iptables redirect |
| GraphQL/proxy + federation | mTLS via SPIFFE/SPIRE (only TLS as terminator) |
| Static-file serving + SPA fallback | Identity-aware policy (SPIFFE-ID → action) |
| CSP / CORS / HSTS / X-Frame-Options | Per-edge retry + circuit-breaker (only timeouts) |
| Per-route rate limiting | xDS-style dynamic config; restart-to-reconfigure |
| Vector/StatsD metrics emission | Distributed-trace context propagation (W3C/Zipkin) |
| Health endpoints | gRPC L7 awareness (HTTP only) |
| Bug-report endpoint, env.js, config.yaml | Connection pooling against many upstreams |
| Single-upstream BFF mode | Service discovery (only static URLs) |

The mesh primitive supplies the "lacks" column via a *separate*
proxy component; hanabi is reused for the "has" column.

---

## III. Existing meshes — absorb / adapt / reject

(Format follows [`INSPIRATIONS.md`](./INSPIRATIONS.md): each entry
states what we take verbatim, what we adapt, what we reject, and why.)

### III.1. Linkerd

- **Absorb verbatim.** Rust-based proxy. Single-binary data plane.
  Identity model: trust anchor + intermediate CA + per-pod cert,
  rotation via Kubernetes ServiceAccount Tokens (TokenRequest API)
  → SPIFFE SVID. Sidecar pattern with init-container iptables.
  TrafficPolicy CRD shape (`Server`, `ServerAuthorization`,
  `HTTPRoute`) is clean.
- **Adapt.** Wrap Linkerd's `linkerd2-proxy` (Rust) as a Rust caixa
  Biblioteca; consume from a new `:kind ProxyBiblioteca` (or extend
  Servico with a `:role :sidecar` slot). Replace Linkerd's CRDs with
  rendered output of `(defmesh …)`.
- **Reject.** Linkerd's installation surface (`linkerd install`,
  Helm). The substrate owns the install via `(defmesh …)` rendering.

### III.2. Istio

- **Absorb verbatim.** Envoy as the L7 proxy reference for what's
  possible (WASM filters, ext-authz, RDS/CDS/EDS xDS protocols).
  PeerAuthentication / AuthorizationPolicy CRD shapes.
- **Adapt.** xDS as a *typed protocol* — define `(defxds-source …)`
  TataraDomain that emits an xDS server; both Linkerd and Envoy
  proxies can consume it. Render `defmesh` to AuthorizationPolicy
  for the "Istio backend" target.
- **Reject.** Istio's control plane (istiod). The substrate is the
  control plane; istiod's job is partially renderer (we have that),
  partially live xDS server (typed and rendered, not adopted).

### III.3. Cilium ServiceMesh

- **Absorb verbatim.** eBPF interception (no sidecar). SPIFFE-style
  identity via Kubernetes ServiceAccount. CiliumNetworkPolicy already
  used in `MESH-COMPOSITION.md`'s `:contratos` rendering.
- **Adapt.** When the cluster runs Cilium (as rio does), prefer the
  eBPF datapath for L4; render `defmesh` to `CiliumEnvoyConfig` for
  L7 with hanabi as the upstream Envoy cluster.
- **Reject.** Cilium ServiceMesh's L7 (Envoy embedded) — replace
  with hanabi for the L7 sub-component in pleme-io-flavored
  deployments. Operator can flip a slot back to native Envoy via
  `:l7 envoy` if they want.

### III.4. NGINX / Traefik / HAProxy as edge proxy

- **Reject.** Already-rejected upstream edge (cf. lilitu-pattern,
  hanabi already replaced nginx). Don't reintroduce as mesh data
  plane.

### III.5. Consul Connect

- **Adapt.** Connect's Service Identity (SPIFFE-ID per service) +
  Intentions (typed who-can-talk-to-whom) is conceptually the cleanest
  identity model. Steal the *concept*; reject the implementation
  (HCL/Consul registry lock-in).

### III.6. SPIFFE/SPIRE

- **Absorb verbatim.** SPIFFE-ID format
  (`spiffe://trust-domain/ns/<ns>/sa/<sa>`), SVID issuance via x509,
  workload-API for SVID retrieval. SPIRE-server as the issuer
  reference.
- **Adapt.** Implement a Rust SPIRE-equivalent caixa
  (`shikiriya`?) — workload-API + SVID issuer + trust-domain
  fanout. Optional backend (operator can wire SPIRE proper or our
  Rust impl).

---

## IV. Architecture — five layers

```
                ┌────────────────────────────────────────────────┐
                │  L0  Identity                                  │
                │      SPIFFE-ID per ServiceAccount,             │
                │      SVID issuance, trust domain               │
                ├────────────────────────────────────────────────┤
                │  L1  Data plane (proxy)                        │
                │      Rust sidecar, mTLS terminator,            │
                │      retries / circuit-breaker / timeout       │
                │      sidecar OR eBPF (operator choice)         │
                ├────────────────────────────────────────────────┤
                │  L2  L7 (hanabi sub-component)                 │
                │      HTTP/GraphQL/gRPC awareness, CSP/CORS,    │
                │      rate-limit, BFF, federation, env.js       │
                ├────────────────────────────────────────────────┤
                │  L3  Control plane                             │
                │      Renderer (substrate), live xDS server,    │
                │      sidecar injector, policy admission        │
                ├────────────────────────────────────────────────┤
                │  L4  Observability                             │
                │      vector sidecar, OTel/W3C trace ctx,       │
                │      victoria sinks (homelab) / DD (SaaS)      │
                └────────────────────────────────────────────────┘
                  ↑
         (defmesh …)  authored once
```

### IV.1. L0 — Identity

- One trust domain per fleet (`spiffe://pleme.io`).
- One SVID per ServiceAccount per namespace per cluster.
- Issuer: a typed `(defidentity-issuer spire …)` caixa Servico OR
  a `:spire` slot on `(defmesh …)` that says "use SPIRE chart, here
  are the trust roots."
- SVID rotation: Kubernetes TokenRequest API → SPIRE workload-API
  → mounted at `/var/run/secrets/spiffe.io/`.
- Reuse [`SAGUAO.md`](./SAGUAO.md)'s passaporte for *human* identity
  edges — when traffic enters via `:entrada` it terminates a
  passaporte session, then mTLS-as-a-service-identity for the
  internal hops. Two distinct identity domains, single typed
  declaration.

### IV.2. L1 — Data plane (proxy)

- Default: a new Rust caixa called `proxy` (working name) — modeled
  on `linkerd2-proxy`, ideally vendoring it as a library and
  wrapping the substrate-shaped configuration surface.
- Pattern: sidecar in every Servico's pod (init-container iptables
  redirect to the proxy), OR eBPF interception when the cluster
  runs Cilium.
- Responsibilities: mTLS termination, retry, circuit-breaker,
  timeout, connection pooling, request-level metrics, distributed
  trace propagation.
- L7 work is *delegated* upward to L2 (hanabi).
- Configuration: dynamic via xDS from L3, OR static via mounted
  config (operator choice; static is simpler, dynamic is richer).

### IV.3. L2 — L7 (hanabi)

Hanabi is composed *into the data plane sidecar* as a second
container. Pod shape:

```
Pod
├── proxy          (Rust, mTLS, retries, observability)
├── hanabi         (L7: HTTP/GraphQL/gRPC, CSP/CORS, rate-limit, BFF)
└── <user Servico> (the actual workload, e.g. cartorio)
```

- Proxy terminates mTLS, hands plaintext to hanabi over loopback.
- Hanabi applies L7 rules (rate-limit, CSP, GraphQL filtering,
  BFF), forwards to user Servico.
- Reverse path: user Servico response → hanabi (response headers,
  CSP, observability) → proxy (re-mTLS) → wire.
- New hanabi mode: `:role :sidecar` (vs `:role :standalone`). In
  sidecar mode hanabi listens on loopback only, reads policy from
  a typed config block injected by the renderer, emits metrics to
  vector co-sidecar.
- Hanabi gains *no* mesh-specific code — only its existing surface
  (CSP/CORS/rate-limit/BFF) gets exercised in a new place.

### IV.4. L3 — Control plane

Three components, all typed:

1. **Renderer** (already exists conceptually — extend `arch-synthesizer`).
   `(defmesh …)` → K8s manifests for the chosen backend. Static
   config; ships at deploy time.

2. **Sidecar injector** — MutatingAdmissionWebhook that adds the
   proxy + hanabi sidecars to any Pod whose namespace has the
   mesh enabled. Itself a caixa Servico, deployable per-cluster.

3. **xDS server** (optional, dynamic-config tier). For canary,
   weighted routing, runtime policy changes. Itself a caixa
   Servico, consumes a Kubernetes-side typed source of truth
   (CRDs emitted by the renderer), serves xDS to the proxies.

### IV.5. L4 — Observability

- vector co-sidecar in every meshed pod (already standard
  per [`OBSERVABILITY.md`](./OBSERVABILITY.md) frame).
- W3C trace-context (`traceparent`) propagation by proxy.
- OTel sink to vmagent (homelab) / Datadog (SaaS) per existing
  observability variants.
- Mesh-emitted metrics: per-edge latency, retry rate, circuit-
  breaker state, mTLS handshake counters, identity validation
  failures.

---

## V. Hanabi as L7 sub-component — concrete shape

### V.1. New `:role :sidecar` mode

Hanabi today expects:
- A `static_dir` (filesystem)
- An `http_port` to listen on
- Optionally a BFF upstream

Sidecar mode adds:
- `:upstream` slot — the user Servico's loopback URL (e.g.
  `http://127.0.0.1:8080`).
- `:downstream-listen` slot — loopback port the proxy hands
  traffic on (e.g. `127.0.0.1:15001`).
- `:policy-source` slot — file path or socket where the renderer
  drops the typed policy (CSP rules, rate-limit per-route, etc).
- `:no-static` flag — disable static-file serving; sidecar mode is
  pure pass-through with L7 rules applied.

These are *additions*, not replacements. Standalone hanabi continues
working unchanged.

### V.2. What hanabi gains, what stays out

Gains:
- `:role :sidecar` (new)
- `:upstream-loopback` slot (new)
- W3C `traceparent` propagation (new — small change)
- xDS subscription mode (later, for dynamic policy)

Stays out (these belong to L1 proxy, not hanabi):
- mTLS (the proxy terminates it, hanabi sees plaintext)
- Connection pooling (proxy's job)
- Retry budget (proxy's job)
- Circuit-breaker (proxy's job)

This delineation is the entire architectural argument:
**proxy = transport, hanabi = semantics**.

### V.3. The composition story for each existing hanabi feature

| Hanabi feature | Mesh role |
|---|---|
| GraphQL federation / proxy | L7 routing for graphql edges; replaces Envoy WASM grpc_to_graphql equivalents |
| CSP / CORS / HSTS | Applied at the egress edge (entering the cluster from `:entrada`) and at every internal hop |
| Rate-limit (per-route, redis-backed) | Token-bucket per-edge in mesh; standardizes the same code path everywhere |
| Static + SPA fallback | Edge-only; not used in sidecar mode |
| BFF mode (proxy GraphQL → backend) | Repurposed: in sidecar mode, BFF=true means hanabi runs L7 routing logic for graphql |
| env.js runtime config | Edge-only; not used in sidecar mode |
| Health endpoints | Reused as-is (sidecar exposes /health) |

---

## VI. The `(defmesh …)` form

```lisp
(defmesh openclaw-mesh
  ;; Required: which Aplicacao is being meshed.
  :aplicacao   openclaw

  ;; Required: identity backend.
  :identity    {:kind :spiffe :issuer :spire :trust-domain "pleme.io"}

  ;; Required: data plane.
  :data-plane  {:proxy linkerd-proxy
                :sidecar-mode :auto       ; :auto | :sidecar | :ebpf
                :l7    hanabi}            ; ←  hanabi as sub-component

  ;; mTLS posture.
  :mtls        :strict                    ; :off | :permissive | :strict

  ;; Retry / timeout / circuit-breaker. Per-edge override possible
  ;; via Aplicacao :contratos[*]:policy.
  :defaults    {:retries 3
                :retry-budget 0.2
                :timeout 30s
                :slow-timeout 5m
                :circuit-breaker {:max-requests 100
                                  :max-failures 10
                                  :reset 30s}}

  ;; Rate limit.
  :rate-limit  {:per-source 1000/s :per-route {/api 100/s}}

  ;; Observability.
  :observability {:traces :otel :metrics :prometheus :logs :json}

  ;; Inherit policy from Aplicacao (don't restate :contratos).
  :policy      :contratos

  ;; Inherit gateway from Aplicacao.
  :gateway     :entrada

  ;; Saguão integration: human edges (entering :entrada) flow through
  ;; passaporte; SPIFFE identity for everything else.
  :saguao      true)
```

`#[derive(TataraDomain)]` on the matching Rust struct makes this
form a typed AST node. The Rust struct flows into
`arch-synthesizer/src/mesh.rs` (new), which:

1. Validates type-level invariants (retry-budget ∈ [0,1], strict
   mTLS requires identity not :off, etc.).
2. Walks the chosen `:data-plane.proxy` to its renderer.
3. Composes hanabi's policy block from `:rate-limit`, `:saguao`,
   and per-edge overrides from `(defaplicacao … :contratos …)`.
4. Emits the K8s manifests for the chosen backend (sidecar
   injector, MutatingWebhook, hanabi sidecar config, identity
   issuer, observability sinks).
5. Optionally emits an xDS server caixa if dynamic config is
   requested.

---

## VII. Phasing — M0 → M5+

| Phase | Goal | Concrete deliverable | Gate |
|---|---|---|---|
| **M0** | Design (this doc) + canonical example sketch | `theory/MESH.md` (this) ; `(defmesh …)` form spec ; choose one demo Aplicacao (openclaw is the natural candidate) | Operator review of the destination + form |
| **M1** | Identity baseline | SPIFFE-ID per ServiceAccount, SPIRE-server caixa Servico (or wrap upstream), SVID mounted in pod ; Rust workload-API client crate | mTLS handshake between two demo Servicos using SVIDs ; CI test |
| **M2** | Sidecar injector + proxy MVP | Rust proxy caixa (vendor `linkerd2-proxy` to start) ; injector caixa with MutatingWebhook ; iptables init-container ; static config | Two demo Servicos auto-meshed; intercepted traffic round-trips with mTLS |
| **M3** | Hanabi `:role :sidecar` mode | Hanabi gains `:role`, `:upstream-loopback`, `:policy-source` slots ; renderer composes hanabi as second sidecar with the merged policy | One demo edge: GraphQL request enters proxy → hanabi (rate-limit + CSP) → user Servico ; trace shows all three hops |
| **M4** | TataraDomain + renderer | `(defmesh …)` form ; Rust struct + derive ; arch-synthesizer/mesh.rs ; renders sidecar-injector + proxy + hanabi sidecar from one declaration | One Lisp form deploys mesh on openclaw |
| **M5** | xDS server (dynamic config) | Rust xDS-server caixa Servico ; CRDs for runtime policy ; proxies subscribe ; hot-reload | Canary 5%→50%→100% on a demo edge with no Pod restart |
| **M6** | Multi-backend renderer | Add Linkerd, Istio, Cilium ServiceMesh as renderer targets ; same `(defmesh …)` form lowers to all four | Same demo Aplicacao deploys identically on Linkerd-mode and Cilium-mode clusters |
| **M7+** | Production hardening | Failure-mode testing, large-cluster scale, doc-complete, blog post, fleet adoption beyond openclaw | Mesh adopted by ≥3 Aplicacaos in the fleet |

**Estimated calendar**: M0 done with this doc. M1–M3 are
~6–10 weeks of focused work. M4 is ~3–4 weeks once M3 lands.
M5 is ~6–8 weeks. M6 is ~6 weeks per backend. M7+ is operational.

Total to **GA on one backend** (k3s + sidecar + hanabi-L7): ~4–5
months of agent-augmented work. **GA on three backends**: ~9–12
months.

---

## VIII. Open questions

1. **Vendor `linkerd2-proxy` or write a thinner Rust proxy from
   scratch?** Vendoring is faster to M1, slower to maintain; from-
   scratch is more substrate-aligned but costs a quarter.

2. **eBPF or sidecar as default datapath?** Sidecar is universal;
   eBPF requires Cilium and a recent kernel. Decision likely:
   sidecar default, eBPF when cluster signals support.

3. **xDS or static config?** Static is simpler and covers ~90% of
   needs. Dynamic xDS is M5; defer until the demand is real.

4. **SPIRE: vendor or wrap?** Spire-server in Go is well-tested;
   wrapping is fast to M1. A pure-Rust SPIRE-equivalent (`shikiriya`?)
   is a new caixa in its own right — defer to M3+.

5. **How does `(defmesh …)` interact with `(defaplicacao … :contratos
   …)`?** Two viable shapes: (a) mesh is *separate* declaration that
   *references* an Aplicacao; (b) mesh is a slot on Aplicacao itself.
   (a) keeps Aplicacao single-purpose; (b) is more concise. Lean
   toward (a) — cleaner separation of concerns.

6. **Hanabi's `:role :sidecar` mode — same crate or new sub-crate?**
   Same crate, gated behind a runtime mode flag. The L7 surface is
   95% shared; sidecar mode is a configuration of standalone, not a
   reimplementation.

7. **Multi-cluster mesh (federation across rio + mar + plo)?**
   Out of scope for M0–M5. The `:trust-domain` slot is forward-
   compatible (one domain spans clusters), but cross-cluster
   discovery / xDS sharding is its own multi-quarter project.

8. **Saguão composition specifics.** When traffic enters via
   `:entrada`, passaporte session validates the human; mTLS-as-
   service-identity for internal hops. But: what's the typed bridge
   between human session and SPIFFE? Likely a per-Servico
   `:downstream-identity-as` slot that says "for downstream edges,
   identify as ServiceAccount X regardless of inbound human". M4
   work.

---

## IX. What we are NOT doing

- **Replacing Istio/Linkerd/Cilium for non-pleme-io tenants.** The
  mesh is for Aplicacaos built on the substrate; standard upstream
  meshes remain valid for non-substrate workloads on the same
  cluster.

- **Running on non-k8s in M0–M6.** podman+nomad / wasm / Lambda
  backends are renderer targets, not implementation work. The
  proxy and injector are k8s-shaped first.

- **Replacing hanabi everywhere.** Hanabi continues as the standalone
  edge BFF for openclaw-web et al. *Sidecar mode* is an additional
  posture, not a replacement.

- **Owning a CA or KMS.** SPIFFE-ID issuance uses an existing CA
  (cluster-issuer or external CA Bundle); pleme-io does not become
  a public CA in this project.

- **L4-only mesh.** Pure-L4 (mTLS + identity, no L7 awareness) is
  Cilium ServiceMesh's lane and is fine for non-substrate workloads.
  pleme-io's mesh is L4+L7 by construction (hanabi gives the L7
  free).

---

## X. Why this is worth the detour

> *"pleme-io exists to compound its ability, quality, and value over
> time. Promises become theorems."*  — `pleme-io/CLAUDE.md`

Today, every Aplicacao the fleet runs has *implicit* mesh-shaped
concerns — retries, mTLS, observability, identity, rate-limit. They
get hand-rolled per-Servico in slightly different ways. Each
divergence is debt the operator carries.

A typed `(defmesh …)` with hanabi composed in:

- Turns "is this edge mTLS'd?" from a runtime check into a
  compile-time theorem.
- Reduces "wire up rate-limit on this Servico" from per-app code to
  one slot on one declaration.
- Makes "switch from k3s to EKS" trigger zero mesh-related work
  beyond `(defmesh … :backend cilium)` instead of `:backend k8s+sidecar`.
- Lets the fleet ship *uniform* observability — every edge emits
  the same shape of trace/metric/log.
- Makes the existing hanabi work compound: every L7 feature added
  to hanabi (CSP, CORS, rate-limit, GraphQL federation) becomes
  available *fleet-wide* through the mesh primitive without per-
  consumer integration.

It's a multi-month detour. It bends the substrate's compounding
curve.

---

## XI. Next concrete step (when M0 review lands)

Choose **openclaw** as the M1–M4 demo Aplicacao. It has the
right shape:
- Three Servicos that talk to each other (cartorio, lacre,
  publisher-pki).
- Public surface (openclaw-web) that needs L7 (CSP/CORS/rate-limit
  — already configured in hanabi for this demo).
- Compliance surface that benefits from mTLS-as-identity (so
  cartorio can verify "this admit really is from a publisher
  Servico, not just from anyone in the namespace").
- Already running on pleme-dev with hanabi as edge — the same
  binary becomes the L7 sidecar. Validation feedback loop ~minutes.

Start at M1: SPIFFE-ID for each openclaw ServiceAccount, mounted
SVID, one demo edge round-trips through mTLS. Concrete output:
a Lisp form that deploys SPIFFE-IDed cartorio + lacre with mTLS
between them, renderable to k3s today.
