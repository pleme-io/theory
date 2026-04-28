# Mesh composition — caixa Servicos into Aplicacaos

> **Frame.** [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) describes one
> Servico per ComputeUnit. [`INSPIRATIONS.md`](./INSPIRATIONS.md)
> catalogs the patterns we should absorb. [`RUNTIME-PATTERNS.md`](./RUNTIME-PATTERNS.md)
> says what becomes possible. **This doc is the next layer up:** how
> *many* Servicos compose into a single typed Application — a mesh —
> and how to standardize that composition ruthlessly so meshes built
> on this substrate are normalized by construction.
>
> The Lisp/wasm/typed substrate makes one thing dramatically easier
> than every prior composition stack: *the application graph is a
> compile-time typed value*. We name it; we type-check it; we render
> it to whatever runtime layer makes sense. Below are the
> composition primitives we should adopt, the prior art they build
> on, and the typed shape of `:kind Aplicacao` — the fourth caixa
> kind that turns a fleet of Servicos into one declarative unit.

---

## I. The composition problem

A single Servico does one thing well — caixa.lisp + servicos/* +
source. But real applications are *meshes*: a checkout flow is
catalog + cart + checkout + payment + notification + email; a
data platform is ingest + dedup + enrich + index + query. None of
those nodes is the "app" — the *app is the typed graph*.

Today's pleme-io substrate handles each Servico in isolation:

- caixa-publish.yml ships one image
- feira deploy adds one programs.yaml entry
- wasm-operator reconciles one ComputeUnit at a time
- KEDA scales each Servico independently

There's no first-class concept for the *graph*. An "application" is
a folk convention: "all the Servicos with the `app: checkout` label."
That's:

1. **Untyped** — labels are strings; one typo and the graph drops a node.
2. **Implicit** — the graph isn't visible to the operator, the
   reconciler, or the auditor. Nobody can ask "what *is* the
   checkout app?"
3. **Unstandardized** — every team picks their own ad-hoc convention
   for inter-Servico contracts (REST vs gRPC vs NATS vs raw HTTP).
4. **Unauditable** — there's no way to enforce "the checkout app
   must satisfy these mesh-level invariants" at the substrate level.

The substrate has the bones to fix all four: typed primitives, lacre
closures, capability tokens. We just need a typed wrapper over a *graph*
of Servicos.

This document lays out:

1. What modern composition systems do, and what to take from each
2. The proposed typed shape: `(defcaixa :kind Aplicacao …)`
3. Standardization rules every Aplicacao satisfies
4. The implementation roadmap

---

## II. Prior art — what to absorb

### II.1. Erlang/OTP applications + distributed applications

OTP solved this for telecom switches in 1990. An *application* is a
named bundle of:

- a **start callback** (`Mod:start/2` — boots the supervisor tree)
- a **stop callback** (`Mod:stop/1`)
- a **dependency list** (`{applications, [kernel, sasl, mnesia, …]}`)
- a **supervisor tree** (the running graph of processes)
- a **distributed configuration** (which nodes can host this app)

Distributed OTP applications add *takeover/failover*: the app runs on
*one* node of the cluster at a time; if that node dies, another node
takes over. The cluster gossips to elect a new host. This is a typed
HA primitive: declare the candidate node list; the runtime owns
election + recovery.

What we should take:

- **Application as a named, typed bundle** of supervised processes.
- **Dependencies as part of the manifest** — the OTP `.app` file lists
  every other application this one needs to start first. That's the
  graph, declared.
- **Distributed app semantics** — typed takeover/failover at the
  application level, not just the process level.

What we improve:

- OTP applications have **no inter-app contract types** — node A's
  `gen_server:call` to node B is dynamically typed. We add WIT-typed
  inter-Servico contracts.
- OTP distributed apps run on **one node at a time**. We add the
  Akka/Orleans option of *sharded* per-tenant placement.

[Erlang/OTP design principles](https://www.erlang.org/doc/system/design_principles.html), [Distributed OTP applications](https://learnyousomeerlang.com/distributed-otp-applications)

### II.2. Service meshes — Istio Ambient, Linkerd, Cilium

Three modern shapes, three different answers to "how do Pods talk":

#### Istio Ambient (2024+)

[Istio Ambient Mesh](https://istio.io/latest/docs/ambient/) ditches
sidecars: one *ztunnel* per node handles L4 (mTLS, identity); an
optional *waypoint proxy* per namespace handles L7 (HTTP routing,
authorization, retries). The data plane is split: kernel-level
networking + per-namespace L7 proxies. Lower overhead, simpler
debugging, sidecarless.

Per [a 2025 academic paper](https://www.youngju.dev/blog/culture/2026-04-13-service-mesh-istio-linkerd-complete-guide-2025.en):
"Istio Ambient showed the best latency performance […] outperforming
both Linkerd and Istio sidecar mode" at 12,800 RPS.

What to take:
- **Identity at the kernel** (mTLS by SPIFFE ID, not IP) — pod
  identity = caixa lacre closure root. Already typed.
- **Split data plane** — L4 is per-node, L7 is per-application
  scope. The Aplicacao caixa becomes the natural waypoint.
- **mTLS by default** — no per-Servico opt-in.

#### Linkerd 2.19 (Oct 2025)

[Linkerd's Rust proxy](https://www.buoyant.io/linkerd-vs-istio) is
the smallest data-plane in the mesh ecosystem. 2.19 shipped post-quantum
mTLS via ML-KEM-768 — first mesh to do so. Per-Servico CPU/memory
overhead is single-digit MB.

What to take:
- **Operator simplicity > feature surface** — Linkerd's design
  prioritizes "you can debug it." Caixa's mesh layer should be the
  same: typed, introspectable, no Envoy-grade complexity.
- **Post-quantum identity** — match by adopting tameshi's BLAKE3
  chain as the canonical pod identity (already content-addressed).

#### Cilium service mesh (eBPF, sidecar-free)

[Cilium](https://docs.cilium.io/en/stable/network/servicemesh/index.html)
runs L7 inspection in the **Linux kernel** via eBPF programs. No
sidecars at all. mTLS via SPIFFE identities; L7 policy via eBPF;
load balancing via XDP. Per
[2026 commentary](https://www.rack2cloud.com/service-mesh-vs-ebpf-kubernetes-cilium-vs-calico/):
"if you're evaluating a service mesh from scratch and your CNI is
Cilium, the answer is that you may not need Istio at all."

This is the most promising direction for caixa: the mesh is a
*property of the kernel*, not a separate control plane. Caixa's
typed identity (lacre closure root) becomes the SPIFFE ID; Cilium's
eBPF enforces L7 policy.

What to take:
- **The mesh is the kernel** — no sidecars, no per-pod overhead.
- **Identity-based authorization** — every caixa Servico gets a
  cryptographic identity rooted in its lacre BLAKE3 closure; Cilium
  enforces L7 policy by identity, not IP.
- **Hubble observability** — every inter-Servico call is traced for
  free in the kernel; feed into tameshi's event chain.

### II.3. GraphQL federation — typed schema composition

[Apollo Federation](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/federation)
solves a different composition problem at the API layer: many
GraphQL services (subgraphs) compose into one (supergraph) by *typed
schema merging*. Composition is:

1. Each subgraph declares its types + fields.
2. The composition step runs at build time, not runtime.
3. Conflicts (two subgraphs claiming the same `User.email`) are
   compile errors.
4. The output is a unified schema — "the supergraph" — that the
   gateway uses to plan queries.

Per the 2026 [GraphQL Composite Schema Working Group](https://graphql.org/learn/federation/):
the spec is being standardized across Apollo, ChilliCream, Graphile,
Hasura, Netflix, and The Guild. Federation is a real cross-vendor
standard.

What to take:
- **Composition as a typed build step**, not a runtime negotiation.
- **Conflicts are errors at compile time**, not warnings at runtime.
- **The output of composition is itself a typed value** (the
  supergraph) — for caixa, this is the resolved Aplicacao.

What's specific to GraphQL:
- Federation is *schema*-level. caixa adds *capability*-level
  (which Servico can talk to which) and *resource*-level (mesh-wide
  rate limits, breakers).

### II.4. Akka cluster + Orleans virtual actors

Akka and Orleans solve *placement* + *sharding* for actor-shaped
systems:

- Akka **Cluster Sharding**: entities (one per ID) distribute across
  cluster nodes by hash. Adding a node triggers automatic rebalance.
- Akka **Persistence + Event Sourcing**: state is the fold of an
  event log; durability and rebalance both fall out of replay.
- Orleans **virtual actors**: never explicitly created/destroyed;
  activated on demand, deactivated when idle. Single-threaded per
  grain.

What to take:
- **Sharded placement at app level** — an Aplicacao can declare
  "this Servico is sharded by request key." Already in
  [`RUNTIME-PATTERNS.md`](./RUNTIME-PATTERNS.md) §G2.
- **Virtual activation by default** — every Servico is KEDA-driven
  scale-to-zero today; the Aplicacao formalizes "every node is
  always *available*, possibly unactivated."

### II.5. Nix flake composition

The most local prior art. A Nix `flake.nix`:

- declares **inputs** (other flakes, pinned by hash)
- exposes **outputs** (packages, devShells, modules, …)
- composes via **input passthrough** (one flake imports another)

The composition graph is a typed value. Hash-pinning makes builds
deterministic. Outputs of one flake are inputs to another.

What to take:
- **Composition is hash-addressed** — caixa-lacre already does this
  for Servicos. The Aplicacao just composes lacre closures.
- **Outputs are typed values** — every caixa Aplicacao is itself a
  caixa, can be a dep of another Aplicacao, etc. (Recursive
  composition; same pattern as Nix flake-of-flakes.)

---

## III. The synthesis — `:kind Aplicacao`

A typed graph of caixa Servicos with explicit inter-Servico
contracts, mesh policies, and placement strategy. Brazilian-Portuguese
naming consistent with the rest of the substrate (Aplicacao =
"application").

### III.1. Author surface

```lisp
(defcaixa
  :nome           "checkout"
  :versao         "0.1.0"
  :kind           Aplicacao
  :edicao         "2026"
  :descricao      "Checkout flow — catalog → cart → payment → fulfillment."
  :repositorio    "github:pleme-io/checkout"
  :licenca        "MIT"
  :autores        ("pleme-io")
  :etiquetas      ("checkout" "ecommerce" "aplicacao")

  ;; The Servicos that make up this app, with version constraints
  ;; (resolved through the same lacre pipeline as :deps).
  :membros        ((:caixa "catalog"      :versao "^0.1")
                   (:caixa "cart"         :versao "^0.1")
                   (:caixa "payment"      :versao "^0.2")
                   (:caixa "fulfillment"  :versao "^0.1")
                   (:caixa "notification" :versao "^0.1"))

  ;; Typed inter-Servico contracts. Each declares "Servico A calls
  ;; Servico B with this WIT-typed message shape." The build refuses
  ;; if A's import doesn't match B's export.
  :contratos      ((:de "cart"         :para "catalog"
                    :wit "wasi:http/proxy"
                    :endpoint "/products/:id")
                   (:de "cart"         :para "payment"
                    :wit "wasi:http/proxy"
                    :endpoint "/charge")
                   (:de "payment"      :para "fulfillment"
                    :wit "wasi:keyvalue/store"
                    :slot "checkout/$orderId")
                   (:de "fulfillment"  :para "notification"
                    :wit "nats:pub-sub"
                    :subject "rio.events.order.shipped"))

  ;; Mesh policies — apply to every contrato unless overridden.
  ;; Subset of Cilium L7 policy + Istio retry/timeout/breaker.
  :politicas      ((:timeout "30s")
                   (:retries 3)
                   (:circuit-breaker (:max-failures 5 :window "60s"))
                   (:mtls-required t)
                   (:rate-limit (:per-caller "100/s")))

  ;; Distributed-app placement strategy. Three options:
  ;;   single-node      — only one cluster runs the app; takeover on death (OTP-style)
  ;;   replicated       — every cluster runs an instance (active-active)
  ;;   sharded          — entities distribute by hash (Akka cluster sharding)
  :placement      (:estrategia replicated
                   :clusters   ("rio" "mar" "plo")
                   :affinity   "data-locality")

  ;; The graph's typed entry-point — what an external caller sees.
  ;; Generates the Cilium L7 policy + Istio gateway / VirtualService
  ;; for outside traffic.
  :entrada        (:host "checkout.quero.cloud"
                   :para "cart"
                   :paths ("/api/cart" "/api/products")))
```

That's the typed shape. Every field is required by the type system;
omissions are compile errors. Authoring an Aplicacao is *exactly* as
much surface as authoring a Servico — same `(defcaixa …)` form, same
`feira` verbs, same caixa-publish CI workflow.

### III.2. What renders from `:kind Aplicacao`

caixa-helm + caixa-flux + a new caixa-mesh crate emit:

1. **One programs.yaml entry per `:membros`** — the existing fleet
   manifest pattern. Nothing new on this axis.
2. **One Cilium `CiliumNetworkPolicy` per `:contratos`** — typed L7
   allow-list; rejected traffic is logged through Hubble.
3. **One `CiliumClusterwideEnvoyConfig` per `:politicas`** — the
   timeout/retry/breaker policy applies across the contracts.
4. **One `Gateway` + `HTTPRoute` per `:entrada`** — external ingress;
   K8s Gateway API standard.
5. **One Aplicacao CR** (`mesh.pleme.io/v1alpha1/Aplicacao`) — the
   typed composition itself, watched by an `app-operator` (a
   companion to wasm-operator) that owns the cross-Servico
   reconciliation (placement strategy, distributed-app takeover).

Per-Servico ComputeUnits don't change shape. The mesh layer is
*additive* — existing Servicos compose into Aplicacaos without
modification, and the substrate's typed boundaries (lacre closure,
capability tokens) become the mesh's identity primitives directly.

### III.3. The compile-time guarantees

Because every layer is typed:

- **Contract drift is a build error.** If `cart`'s caixa.lisp changes
  the response shape of `/products/:id`, the checkout Aplicacao
  fails to build; the dev sees the error before deploy.
- **Capability leaks are build errors.** A `:contratos` entry
  requires the source Servico to have `:capabilities` matching the
  WIT. A Servico without `http-out:catalog` can't be the source of an
  HTTP `:contrato` to `catalog`.
- **Cluster-locality violations are build errors.** A `:placement
  :sharded` Aplicacao with a member that depends on cluster-singleton
  state (e.g. an Mnesia-style node-local DB) fails to compose.
- **Cycles are build errors.** No circular `:contratos`. Either
  break with an event-sourced indirection or use NATS pub-sub
  (which is acyclic by construction).

These guarantees come from making the graph a typed value at compile
time. That's what makes the typed substrate *qualitatively
different* from labels-and-strings mesh frameworks.

### III.4. The runtime guarantees

At runtime, the substrate's existing primitives compose:

- **Identity** — every Servico's pod identity is its lacre closure
  root (BLAKE3 hash). Cilium's SPIFFE-style identity binding uses
  this directly. Two pods with the same lacre = same identity =
  same policy applies. No spoofing.
- **mTLS** — Cilium provides it for free given identity; the
  Aplicacao's `:politicas :mtls-required t` gates it on at the
  Aplicacao layer.
- **Tracing** — Hubble (Cilium's observability layer) exports every
  inter-Servico call as a typed event into tameshi's event chain.
- **Failure containment** — each Servico's wasm sandbox + supervisor
  tree (M2.4) bounds the blast radius of a bug; the Aplicacao adds
  policy-level circuit-breakers for cascades.

---

## IV. Standardization rules — what every Aplicacao must declare

Ruthless standardization comes from *required* slots. Every
`:kind Aplicacao` caixa must declare:

| slot | purpose | layout invariant |
|---|---|---|
| `:nome` | typed identifier | same DNS-1123 rule as Servico |
| `:versao` | semver pin | parsed by caixa-version |
| `:membros` | the typed graph nodes | ≥1 entry; each must resolve via lacre |
| `:contratos` | typed inter-Servico edges | each :de + :para must be in :membros; :wit must reference a registered WIT world |
| `:politicas` | mesh-level defaults | at minimum :timeout + :mtls-required |
| `:placement` | distribution strategy | one of {single-node, replicated, sharded} |
| `:entrada` | external entry point | optional but recommended; required for public Aplicacaos |

The build refuses any Aplicacao that omits a required slot. cse-lint
gets a new check (`aplicacao-completeness`) that rejects PRs which
land partial Aplicacaos.

What this gives us:

- **Every Aplicacao in the fleet has the same shape**, so an operator
  walking from one Aplicacao to another sees the same surface.
- **Every Aplicacao is auditable** by the same tooling — feira app
  graph, feira app deploy, cse-lint repo, all work uniformly.
- **Every Aplicacao is composable**: an Aplicacao can be a member of
  another (recursive composition).

---

## V. Compounding properties

What composes once we have `:kind Aplicacao`:

### Application templates

A team can publish a *template* Aplicacao that other teams
instantiate with their own Servicos. E.g. `pleme-template-saas-app`
declares the canonical "auth + tenancy + audit + billing" mesh shape;
new teams `feira app from-template pleme-template-saas-app` and fill
in their domain Servicos.

This is Helm's umbrella-chart pattern with typed contracts instead
of YAML values.

### Cross-cluster federation

`:placement :replicated :clusters ("rio" "mar")` deploys the
Aplicacao to every named cluster. The substrate handles cross-cluster
Servico discovery (Cilium ClusterMesh + caixa-resolver peer-cluster
hierarchy). The author writes `:replicated` and forgets about
multi-region.

### Adaptive Service Graph Compression (Unison)

Once `:contratos` are typed, the Adaptive compression pattern from
[`INSPIRATIONS.md` §IV.2](./INSPIRATIONS.md#iv2-adaptive-service-graph-compression)
kicks in: the runtime profiles call frequencies between Servicos and
co-locates frequent callers automatically. The Aplicacao remains
declaratively the same; the *placement* becomes runtime-optimized.

### Saga / event-sourced Aplicacaos

Mesh-level event sourcing falls out of declared `:contratos`:
every `:wit "nats:pub-sub"` edge is a typed event stream that
tameshi's BLAKE3 chain can replay. An entire Aplicacao becomes
time-travel-debuggable end-to-end.

### App-level governance via cse-lint

The fleet-wide CSE invariants extend to mesh shape:

- Every Aplicacao declares `:placement` (no implicit cluster choice).
- Every Aplicacao declares `:politicas :mtls-required t` (no plaintext intra-mesh).
- Every Aplicacao declares `:politicas :timeout` (no infinite blocking).
- Every public Aplicacao declares `:entrada :host` (no missing ingress).

cse-lint checks them on every commit.

---

## VI. Implementation roadmap

This builds on M2's foundation. Concrete deliverables:

### M3.x (mid-year): typed Aplicacao primitives

| item | scope | tests | hours |
|---|---|---|---|
| `caixa-core::AplicacaoSpec` (membros + contratos + politicas + placement + entrada) | ~250 LoC + `CaixaKind::Aplicacao` | 15 | 3 |
| `caixa-core::WitContract` typed contract | ~150 LoC | 10 | 2 |
| Layout invariants for `:kind Aplicacao` | ~80 LoC in layout.rs | 8 | 1 |
| `caixa-mesh` new crate (renderer) | ~400 LoC: emit Cilium policies + Gateway + HTTPRoute | 15 | 4 |
| `feira app graph` — print typed graph | ~150 LoC in caixa-feira | 5 | 1.5 |
| `feira app deploy` | ~150 LoC; wires to caixa-flux | 8 | 2 |
| cse-lint `aplicacao-completeness` check | ~120 LoC | 6 | 1.5 |

Critical path: AplicacaoSpec types → caixa-mesh renderer → feira app
verbs → cse-lint check. ~1300 LoC + 67 tests across the milestone.

### M4.x (late year): runtime support

| item | scope |
|---|---|
| `app-operator` — companion CRD for distributed-app placement (takeover, sharding) | new wasm-operator-shaped Rust crate |
| Cilium NetworkPolicy + EnvoyConfig integration | new lareira-charts library chart `pleme-aplicacao` |
| Cross-cluster `:replicated` placement via FluxCD | extend caixa-flux with cluster-fanout rendering |
| Akka-style cluster sharding for `:sharded` | wasm-operator hash-shard reconciler |

### M5 (next year): the substrate self-hosts

The substrate's own infrastructure becomes an Aplicacao:

```lisp
(defcaixa
  :nome    "tatara-stack"
  :kind    Aplicacao
  :membros ((:caixa "wasm-operator"  :versao "^0.1")
            (:caixa "wasm-engine"    :versao "^0.1")
            (:caixa "caixa-operator" :versao "^0.1")
            (:caixa "caixa-resolver" :versao "^0.1")
            (:caixa "lareira-fleet-programs" :versao "^0.1"))
  :placement (:estrategia replicated :clusters ("rio" "mar" "plo")))
```

The recursive loop: the substrate that runs Aplicacaos is itself an
Aplicacao. `feira app deploy --cluster rio --apply` deploys *the
substrate itself*. Disaster recovery becomes "redeploy the
tatara-stack Aplicacao to a fresh cluster."

---

## VII. The bigger picture

The point of typing the mesh is the same as typing the package: bad
states should be unrepresentable at compile time. Today's mesh
ecosystems treat composition as a runtime concern — sidecars
negotiate, control planes reconcile, errors surface in dashboards.
The substrate inverts this: composition is a typed value at
authoring time; the runtime is what makes the typed value live.

What we get when this lands:

- **Whole-application authoring** — one `(defcaixa :kind Aplicacao)`
  describes the entire mesh.
- **Whole-application deployment** — one `feira app deploy` ships
  every Servico, contract, and policy atomically.
- **Whole-application typing** — contract drift, capability leak,
  placement violation are all build-time errors.
- **Whole-application introspection** — `feira app graph` shows the
  typed graph; cse-lint enforces shape; tameshi attests.
- **Whole-application composition** — Aplicacaos are themselves
  caixas, dependable from other Aplicacaos.

The *ruthless standardization* the user asked for emerges from the
type system. There's no convention-over-configuration debate to have;
the slots are required, the contracts are checked, the policies are
enforced. Every Aplicacao in the fleet has the same shape because
the build refuses anything else.

This is the typed-mesh primitive the substrate has been building
toward. It composes with everything M2 just landed (limits,
behaviors, upgrade-from, supervisor trees), with everything in the
M3 absorption queue (Unison Remote, Adaptive compression, Pony
capabilities), and with everything below the Servico layer (lacre
identity, capability tokens, wasm sandboxing). The typed Aplicacao
is what turns "many caixas" into "one application," and what turns
"one application" into "the whole substrate, recursively self-hosted."

---

## VIII. Sources

- [Erlang/OTP design principles](https://www.erlang.org/doc/system/design_principles.html)
- [Distributed OTP applications](https://learnyousomeerlang.com/distributed-otp-applications)
- [Istio Ambient Mesh](https://istio.io/latest/docs/ambient/) — sidecarless data plane
- [Linkerd vs Istio comparison (2026)](https://www.buoyant.io/linkerd-vs-istio)
- [Cilium service mesh — eBPF identity-based networking](https://docs.cilium.io/en/stable/network/servicemesh/index.html)
- [Sidecar-Free Service Mesh: Understanding Cilium's eBPF Architecture](https://blog.aicademy.ac/cilium-ebpf-sidecar-free-service-mesh)
- [Apollo Federation — typed schema composition](https://www.apollographql.com/docs/graphos/schema-design/federated-schemas/federation)
- [GraphQL Federation specification (Composite Schema WG)](https://graphql.org/learn/federation/)
- [Akka Cluster Sharding](https://doc.akka.io/libraries/akka-core/current/typed/cluster-sharding.html)
- [Akka Persistence + Event Sourcing](https://doc.akka.io/libraries/akka-core/current/typed/persistence.html)
- [SPIFFE identity spec](https://spiffe.io/) — workload identity primitives
- [K8s Gateway API](https://gateway-api.sigs.k8s.io/) — the canonical ingress API

The substrate's intellectual debt to each is acknowledged. The
synthesis we're proposing — typed Aplicacao caixas + Cilium-style
identity + Federation-style typed composition + OTP-style typed
applications — is novel as a stack, not novel in any individual
piece. Pillar-12 (generation over composition) shows up here too:
we *generate* the runtime mesh from one typed authoring surface,
rather than *composing* it from a stack of point tools.
