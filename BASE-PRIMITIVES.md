# Base primitives — the smallest set from which everything composes

> **The user, 2026-04-26**: *"we should have base primitives for so
> much like flows DAGs execution chains that are all fed by just in
> time infrastructure breathability all based on git and nix
> delivery"*
>
> This document catalogs the **smallest set of base primitives** from
> which every higher-level pattern in the pleme-io architecture
> composes. Five primitive categories. Each primitive has a real
> implementation today (or a near-future one named here). Every
> capability in the cookbook ([`WASM-PATTERNS.md`](WASM-PATTERNS.md))
> is a composition of these primitives.

---

## I. The primitive categories

Five categories, each a different *axis* of what programs do:

| Category | What it answers | Examples |
|---|---|---|
| **Compute** | What runs? | program, job, function, service, controller |
| **Composition** | How do compute units combine? | flow, DAG, chain, saga, fan-out, fan-in, retry, parallel, sequential, conditional |
| **Lifecycle** | How does a unit transition? | spawn, await, cancel, restart, hot-replace, snapshot, restore |
| **Infrastructure** | How is the substrate provisioned? | spin-up, spin-down, scale, reserve, release, breathe |
| **Delivery** | How does code reach the runtime? | pin, fetch, verify, cache, swap, rollback |

Every higher-level pattern is a tuple `(compute primitives, composition rule, lifecycle policy, infrastructure tier, delivery method)`. **No bespoke architecture; every capability decomposes into these axes.**

## II. Compute primitives — what runs

The five `ComputeUnit.spec.trigger` shapes from
[`WASM-STACK.md` §I](WASM-STACK.md). Already implemented in
`wasm-types::ComputeUnit::spec.trigger` (research confirms full
struct + ComputeBackend enum exist):

| Primitive | Trigger | Existing impl |
|---|---|---|
| **program** | `oneShot { args }` | `wasm-types::ComputeUnit` |
| **job** | `cron "..."` | same; CronJob materialization in operator |
| **function** | `event { source, batch }` | same; KEDA scaler triggers |
| **service** | `service { port, paths }` | same; KEDA HTTP scale-to-zero |
| **controller** | `watch { group, kind }` | same; reconcile loop in `wasm-operator` |

**Status: shipped.** Wasmtime instantiation works
(`wasm-engine::WasmtimeExecutor`); operator generates K8s Deployments
for Wasm/Container/CloudFunction backends.

**Gap (small):** the operator's `wasm-types::ProgramHooks` for
snapshot/restore — required by [`LIVE-DEPLOYMENT.md`](LIVE-DEPLOYMENT.md).
Not yet implemented.

## III. Composition primitives — how units combine

The shapes for combining two or more compute units. Each is itself a
compute unit (controller-shape) whose spec describes its children.

### III.1 Flow (sequential ordered execution)

```yaml
spec:
  flow:
    - { name: step-1, source: github:.../step-1.tlisp?ref=v1, on-failure: stop }
    - { name: step-2, source: github:.../step-2.tlisp?ref=v1, on-failure: compensate }
    - { name: step-3, source: github:.../step-3.tlisp?ref=v1, on-failure: retry, max: 3 }
```

Children execute in declared order; results from step N are available
to step N+1 via the saga's status. Each step is a `program` ComputeUnit
spawned in turn. The flow itself is a controller.

### III.2 DAG (typed dependency graph)

```yaml
spec:
  dag:
    nodes:
      - { id: build,   source: github:.../build.tlisp?ref=v1 }
      - { id: test,    source: github:.../test.tlisp?ref=v1 }
      - { id: scan,    source: github:.../scan.tlisp?ref=v1 }
      - { id: deploy,  source: github:.../deploy.tlisp?ref=v1 }
    edges:
      - { from: build, to: [test, scan] }    # parallel after build
      - { from: [test, scan], to: deploy }   # deploy waits for both
```

Topological execution; nodes with no remaining dependencies fire in
parallel. Used for build pipelines, multi-step deploys, multi-cluster
fan-out/in.

### III.3 Chain (data pipeline)

```yaml
spec:
  chain:
    source: { type: nats, subject: "events.x" }
    pipeline:
      - { transform: github:.../parse.tlisp?ref=v1 }
      - { transform: github:.../enrich.tlisp?ref=v1 }
      - { transform: github:.../validate.tlisp?ref=v1 }
    sink:   { type: s3, bucket: "..." }
```

Stream-shaped composition. Each transform is a function-shape
ComputeUnit; events flow through; backpressure handled by the chain
controller.

### III.4 Saga (compensating transactions)

```yaml
spec:
  saga:
    steps:
      - { do: github:.../book-flight.tlisp,  undo: github:.../cancel-flight.tlisp }
      - { do: github:.../book-hotel.tlisp,   undo: github:.../cancel-hotel.tlisp }
      - { do: github:.../book-car.tlisp,     undo: github:.../cancel-car.tlisp }
    on-step-failure: compensate-from-current-back-to-1
```

Pattern from [`WASM-PATTERNS.md` #31](WASM-PATTERNS.md). Failure
mid-saga unwinds via the `undo:` modules.

### III.5 Parallel + sequential combinators

```lisp
(parallel a b c)              ; spawn all three; await all
(sequential a b c)            ; spawn each in order; result of one feeds next
(fan-out source [a b c])      ; copy source result to each
(fan-in [a b c] sink)         ; wait for all; pass tuple to sink
(retry strategy thunk)        ; with retries, backoff, circuit-breaker
(race a b c)                  ; first to finish wins
```

These are tatara-lisp macros in `tatara-lisp-controllers/lib/` that
emit the matching DAG/chain spec. Authors compose at the Lisp level.

**Status:** designed in [`WASM-PATTERNS.md` §VII](WASM-PATTERNS.md);
not yet implemented as concrete macros in `tatara-lisp-controllers`.
Mechanical work — each macro is a small Sexp transformation.

## IV. Lifecycle primitives — how a unit transitions

Each compute unit transitions through a state machine. The primitive
verbs are the transitions:

| Primitive | Operator action | Program-side hook |
|---|---|---|
| `spawn` | Create engine pod, schedule | (none — module instantiates) |
| `await` | Wait for completion / event | (yields to scheduler) |
| `cancel` | Terminate engine pod | `(on-cancel)` if defined |
| `restart` | Reap + spawn fresh | (none) |
| `hot-replace` | Spawn new beside, snapshot, swap, reap | `(snapshot)` + `(receive-state)` |
| `snapshot` | Serialize state to JSON / Sexp | `(snapshot)` returns Value |
| `restore` | Load state into a fresh instance | `(receive-state state)` |

**Status:** spawn/cancel/restart shipped (operator does this today);
hot-replace and snapshot/restore designed in
[`LIVE-DEPLOYMENT.md`](LIVE-DEPLOYMENT.md), needs operator-side
implementation of the `wasm-types::ProgramHooks` protocol.

## V. Infrastructure primitives — how substrate is provisioned

These are the breathability verbs from
[`BREATHABILITY.md`](BREATHABILITY.md):

| Primitive | What it does |
|---|---|
| `spin-up` | Bring resources online; provision pods, PVCs, queues |
| `spin-down` | Release resources; reap pods, archive state |
| `scale` | Adjust replica count up or down |
| `reserve` | Hold capacity ahead of demand (warm pool) |
| `release` | Return reserved capacity to free pool |
| `breathe` | Continuous adjustment driven by demand signal |

**Status:** all shipped. `spin-up` is `wasm-engine` pod creation;
`spin-down` is reap. `scale` is KEDA-driven. `reserve` is the
`minReplicas` setting on a KEDA ScaledObject. `breathe` is the
default mode (`minReplicas: 0` + cooldown).

## VI. Delivery primitives — how code reaches the runtime

The packaging-side verbs from
[`TATARA-PACKAGING.md`](TATARA-PACKAGING.md):

| Primitive | What it does |
|---|---|
| `pin` | Reference content by hash (URL `?ref=<hash>`) |
| `fetch` | Resolve URL → bytes via `tatara-lisp-source` |
| `verify` | BLAKE3-check fetched bytes against pin |
| `cache` | Store at `~/.cache/tatara/sources/<blake3>/` (host) or `/var/lib/wasm-store/sources/<blake3>/` (cluster) |
| `swap` | Replace running module with newly fetched one |
| `rollback` | Re-resolve an older `?ref=<hash>` and swap |

**Status:** all shipped in `tatara-lisp-source` crate (URL grammar,
fetcher, BLAKE3, FileCache). Cluster-side reuse is mechanical
(operator imports the same crate).

## VII. The composition graph — every higher-level pattern is a tuple

Pattern shape:

```
(compute, composition, lifecycle, infrastructure, delivery)
```

For each cookbook pattern, the tuple instantiates:

| Pattern | Compute | Composition | Lifecycle | Infrastructure | Delivery |
|---|---|---|---|---|---|
| #1 PVC autoresizer | job | none | spawn | breathe (cron) | pin + fetch + cache |
| #2 DNS reconciler | controller | none | spawn + hot-replace | breathe (event) | pin + fetch + cache |
| #9 GitHub webhook | service | none | spawn + breathe | breathe (HTTP) | pin + fetch + cache |
| #31 saga reconciler | controller | saga | spawn children + compensate | breathe per step | pin + fetch + cache |
| #33 workflow DAG | controller | DAG | spawn topologically | breathe per node | pin + fetch + cache |
| #36 RAG indexer | function + service | chain (cron + service) | spawn + hot-replace | breathe | pin + fetch + cache |
| #38 agent-as-controller | controller | (LLM-driven flow) | spawn + restart on hang | breathe | pin + fetch + cache |

**Every pattern decomposes into the same five-axis tuple.** Authors
who learn the primitives can read any cookbook entry as a literal
mapping: pick a compute, pick a composition, etc.

## VIII. The composition rules

How primitives combine — five mechanical rules:

### VIII.1 Composition primitives are themselves controllers

A flow is a controller-shape ComputeUnit watching its child
ComputeUnits. A DAG is the same. A saga is the same. **The composition
primitive is a level of indirection; it doesn't introduce a new
runtime model.**

### VIII.2 Lifecycle is independent of composition

Whether you're running one program or a 50-step DAG, the lifecycle
verbs (spawn, hot-replace, snapshot) work identically. The DAG's
runtime hot-replace is just hot-replacing each running node
in-place.

### VIII.3 Infrastructure breathes per node

Each child of a composition has its own breathability tier. A DAG
might have a `small` job at one node and a `large-aws-medium`
controller at another; each scales 0→N→0 independently. The DAG
itself, being a controller, also breathes.

### VIII.4 Delivery is universal

Every node — whether a leaf compute primitive or a composition wrapper
— is referenced by the same URL grammar. **A 50-program DAG is
described by 50 URLs.** The operator resolves each independently,
caches each by BLAKE3, hot-replaces each independently when its `?ref=`
changes.

### VIII.5 Composition primitives are themselves URL-addressable

```yaml
spec:
  compose: github:pleme-io/programs/saga-bookings/main.tlisp?ref=v1
  with:
    flight-booking: github:pleme-io/programs/book-flight/main.tlisp?ref=v2
    hotel-booking:  github:pleme-io/programs/book-hotel/main.tlisp?ref=v1
    car-booking:    github:pleme-io/programs/book-car/main.tlisp?ref=v1
```

The composition ITSELF is a tatara-lisp program (`saga-bookings/main.tlisp`)
that takes the children as parameters. Authors write composition
once, parametrize across many concrete instances.

## IX. The five-axis summary

Every program in pleme-io is described by:

```
PROGRAM = (
  compute     : program | job | function | service | controller,
  composition : none | flow | dag | chain | saga | combinators,
  lifecycle   : default | hot-replace | continuation-aware,
  infra       : breathe (default) | reserve N | always-on,
  delivery    : github:owner/repo/path?ref=<hash> | oci://... | local-path
)
```

The space of expressible programs is the cartesian product of these
five axes. **The architecture is closed under this product** —
EXTENSIBILITY.md's "boundlessly extensible" claim formalized.

## X. What's missing from the foundations

Per the parallel research:

### X.1 Operator-side
1. **`wasm-types::ProgramHooks`** — the snapshot/restore protocol
   (Direction-2 lifecycle primitives). ~150 lines of Rust.
2. **`tatara-lisp-source` integration into `wasm-operator`'s reconcile loop.**
   Currently the operator uses its own loaders (OCI/Git stubs); switching
   to the standardized resolver. ~50 lines.
3. **CRD YAML export** — operator's CRDs are generated at compile-time
   via kube-derive but no `deploy/crds/*.yaml` exists. Needed for FluxCD
   distribution. ~30 minutes.

### X.2 Composition-side
1. **`tatara-lisp-controllers/lib/defflow.tlisp`** — the macro that
   emits a flow ComputeUnit from a list of step URLs.
2. Same for `defdag.tlisp`, `defchain.tlisp`, `defsaga.tlisp`,
   `defparallel.tlisp`, etc.
3. Each is ~50 lines of tatara-lisp once `tatara-lisp` user macros work
   (per the parallel research, `MacroDef` + `Expander` exist in the
   crate but evaluator integration is "Phase 2.2 scaffold" — needs
   wiring).

### X.3 Lisp-side macros
The macros in `tatara-lisp-controllers/lib/` are skeletons. Real
implementations need user-defined macro support in `tatara-lisp-eval`'s
evaluator — currently the macroexpander is exposed but not invoked at
eval time.

### X.4 Hot-replace verb
`spawn` + `cancel` work today. `hot-replace` + `snapshot` need:
- Operator: implement the algorithm from
  [`LIVE-DEPLOYMENT.md` §IV](LIVE-DEPLOYMENT.md).
- Tatara-lisp: support `(defprogram :snapshot ... :receive-state ...)`
  hooks (real macro work).

## XI. Phasing — the closing implementation arc

Phase ordering from "primitives complete":

1. **`tatara-lisp-eval` user macros** — wire macroexpander into eval loop.
   Unlocks every macro-based composition in the controllers library.
2. **Composition primitives in `tatara-lisp-controllers`** — `defflow`,
   `defdag`, `defchain`, `defsaga`, `defparallel`. Each one Lisp-level macro.
3. **`tatara-lisp-source` → wasm-operator** — standardized resolver for
   ComputeUnit module sources.
4. **CRD YAML export from `wasm-operator`** — for FluxCD distribution.
5. **`ProgramHooks` snapshot protocol** — operator + Lisp-side.
6. **Hot-replace algorithm** — operator implements the
   [`LIVE-DEPLOYMENT.md` §IV](LIVE-DEPLOYMENT.md) state machine.

Each phase enables the next; total: ~2-4 weeks of focused work to
complete the primitive set end-to-end.

## XII. The closing claim

Five categories. Eight composition primitives. Six lifecycle verbs.
Six infrastructure verbs. Six delivery verbs. **31 base primitives**
total. Every higher-level pleme-io capability is a literal tuple of
these.

The user's vision — *flows, DAGs, execution chains, fed by JIT
breathability, all on git+nix delivery* — maps exactly to this
primitive set. **No new primitives are needed** for the cookbook;
the work ahead is implementation, not design.

## XIII. See also

- [`THEORY.md`](THEORY.md) — the 12 pillars (the architecture's axioms)
- [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — 4-layer compute hierarchy
- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49 patterns (each a tuple instance)
- [`BREATHABILITY.md`](BREATHABILITY.md) — infrastructure axis
- [`TATARA-PACKAGING.md`](TATARA-PACKAGING.md) — delivery axis
- [`LIVE-DEPLOYMENT.md`](LIVE-DEPLOYMENT.md) — lifecycle axis
- [`EXTENSIBILITY.md`](EXTENSIBILITY.md) — the closing-the-loop principle
- [`RUST-LISP-EMBEDDING.md`](RUST-LISP-EMBEDDING.md) — Direction-2 hosting
- [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) — controller authoring
- [`TATARA-CODEGEN-MATRIX.md`](TATARA-CODEGEN-MATRIX.md) — every spec → domain
- [`HELLO-WORLD-LIVE.md`](HELLO-WORLD-LIVE.md) — the canonical proof
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`FLEET-DECLARATION.md`](FLEET-DECLARATION.md) — perfect-state contract

This is the **14th and final** generation-1 theory document. The
architecture's primitive set is now fully named.
