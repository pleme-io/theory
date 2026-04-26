# The pleme-io meta-framework — Helm × tatara-lisp × Kubernetes × WASM/WASI

> **Frame.** Six theory documents — [`THEORY.md`](THEORY.md),
> [`BREATHABILITY.md`](BREATHABILITY.md),
> [`SCRIPTING.md`](SCRIPTING.md),
> [`WASM-STACK.md`](WASM-STACK.md),
> [`WASM-PATTERNS.md`](WASM-PATTERNS.md),
> [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md),
> [`WASM-PACKAGING.md`](WASM-PACKAGING.md),
> [`WASM-RUNTIME-COMPLETE.md`](WASM-RUNTIME-COMPLETE.md) — declare the
> architecture. **This document is the meta-framework that ties them
> together** — a typed decision tree for *where a given piece of
> compute belongs, how it's packaged, how it's deployed, how it
> evolves.* The audience is anyone (operator, AI agent, future
> contributor) who's about to build something and needs to know
> *which abstraction layer it goes in.*

---

## I. The four-layer compute hierarchy

```
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 4 — DEPLOYMENT BUNDLES                  (Helm umbrella charts) │
│   FluxCD applies; one chart per cluster; opinionated defaults.       │
└─────────────────────────────────────────────────────────────────────┘
                                 ▲
                                 │ depends on
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 3 — PROGRAM CHARTS                      (Helm consumer charts) │
│   One per program; ~30 lines of values; renders ComputeUnit + sidecars│
└─────────────────────────────────────────────────────────────────────┘
                                 ▲
                                 │ depends on
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 2 — TYPED RESOURCES                              (CR shapes)   │
│   ComputeUnit, PangeaArchitecture, Process, etc.                     │
│   Operator-readable, declarative, capability-bounded.                │
└─────────────────────────────────────────────────────────────────────┘
                                 ▲
                                 │ instantiated by
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 1 — RUNTIME ARTIFACTS                  (WASM bytes / OCI images)│
│   Content-addressed (BLAKE3), built once, run many times.            │
└─────────────────────────────────────────────────────────────────────┘
                                 ▲
                                 │ compiled from
┌─────────────────────────────────────────────────────────────────────┐
│ LAYER 0 — SOURCE                          (tatara-lisp / Rust / Go)  │
│   Authored by humans (or LLMs); committed to git; URL-addressable.   │
└─────────────────────────────────────────────────────────────────────┘
```

Each layer has a single canonical pleme-io repo where it lives. The
**boundaries between layers are typed** — you can only cross a layer
boundary through an explicit transformation that the system tracks.

## II. The decision tree — *where does THIS compute go?*

When you have a piece of work to do — an alert, a reconciler, a
batch job, a webhook handler, a one-shot ops script — walk this
tree:

```
Is this a one-shot operation an operator will invoke once?
├── YES → ComputeUnit shape: program (oneShot)
│        Repo: pleme-io/programs/<name>/
│        Layer 4: NOT NEEDED. apply the CR via kubectl OR generate
│                 from a tatara-lisp `(deploy ...)` form.
│
└── NO → Is this a periodic batch operation?
    ├── YES → ComputeUnit shape: job (cron)
    │        Repo: pleme-io/programs/<name>/
    │        Layer 4: helmworks/charts/lareira-<name> (consumer chart)
    │
    └── NO → Is this an event-driven async handler?
        ├── YES → ComputeUnit shape: function
        │        Sources: NATS / Kafka / SQS / Redis-Streams / RabbitMQ
        │        Repo: pleme-io/programs/<name>/
        │        Layer 4: lareira-<name> consumer chart
        │
        └── NO → Is this a long-running HTTP/gRPC service?
            ├── YES → ComputeUnit shape: service
            │        Repo: pleme-io/programs/<name>/
            │        Layer 4: lareira-<name> consumer chart
            │        Breathability: KEDA HTTPScaledObject (cooldown 10min default)
            │
            └── NO → Is this a CRD reconciler / controller?
                ├── YES → ComputeUnit shape: controller
                │        Authoring: defrule-driven-controller (or sibling)
                │        Repo: pleme-io/programs/<name>/ (controller)
                │              + paired CRD via defschema
                │        Layer 4: lareira-<name> consumer chart
                │        (Optionally policy CR rendered from values.policy.spec)
                │
                └── NO → Is this a stateful service that can't be a WASM module?
                    ├── YES (e.g. a database) → wrap as a regular K8s Deployment
                    │        Repo: helmworks/charts/lareira-<name>
                    │        Authoring: pleme-statefulset / pleme-microservice
                    │                   library charts (NOT pleme-computeunit)
                    │        This is the existing lareira-* shape from before
                    │        the WASM stack landed.
                    │
                    └── NO → reconsider. Either it fits one of the above
                              shapes, or it's a primitive infrastructure
                              concern (DNS zone, IAM role) that goes through
                              Pangea (pangea-architectures), not into the
                              cluster as a workload.
```

## III. The repo-routing map

```
┌────────────────────────────────────────────────────────────────────┐
│ SOURCE LAYER                                                       │
│   pleme-io/programs                Layer 0: tatara-lisp programs  │
│   pleme-io/tatara-lisp-controllers Layer 0: tatara-lisp libraries │
│   pleme-io/tatara-lisp             Layer 0: the language          │
│   pleme-io/wasm-platform           Layer 0: the runtime (Rust)    │
│   pleme-io/<rust-service>          Layer 0: Rust services         │
│   pleme-io/pangea-architectures    Layer 0: Pangea/Tofu IaC       │
└────────────────────────────────────────────────────────────────────┘
                                   ▼
┌────────────────────────────────────────────────────────────────────┐
│ ARTIFACT LAYER                                                     │
│   ghcr.io/pleme-io/wasm-operator   Layer 1: runtime image         │
│   ghcr.io/pleme-io/wasm-engine     Layer 1: runtime image         │
│   ghcr.io/pleme-io/<service>       Layer 1: service images        │
│   github:pleme-io/programs/...     Layer 1: tatara-lisp packages  │
│      → blake3-keyed in-cluster module store                       │
│   pangea state s3                  Layer 1: rendered Tofu plans   │
└────────────────────────────────────────────────────────────────────┘
                                   ▼
┌────────────────────────────────────────────────────────────────────┐
│ TYPED RESOURCE LAYER                                               │
│   wasm.pleme.io/ComputeUnit                Layer 2                │
│   wasm.pleme.io/WasmModule                 Layer 2                │
│   <controller-name>.pleme.io/<Kind>        Layer 2 (per controller)│
│   pangea.pleme.io/PangeaArchitecture       Layer 2                │
│   pangea.pleme.io/InfrastructureTemplate   Layer 2                │
│   tatara.pleme.io/Process                  Layer 2                │
└────────────────────────────────────────────────────────────────────┘
                                   ▼
┌────────────────────────────────────────────────────────────────────┐
│ HELM PACKAGING LAYER                                               │
│   helmworks/charts/pleme-lib              Layer 3 (library)       │
│   helmworks/charts/pleme-computeunit      Layer 3 (library) NEW   │
│   helmworks/charts/pleme-statefulset      Layer 3 (library)       │
│   helmworks/charts/pleme-microservice     Layer 3 (library)       │
│   helmworks/charts/pleme-cronjob          Layer 3 (library)       │
│   helmworks/charts/pleme-operator         Layer 3 (library)       │
│   helmworks/charts/pleme-worker           Layer 3 (library)       │
│   helmworks/charts/lareira-<name>         Layer 4 (consumer)      │
└────────────────────────────────────────────────────────────────────┘
                                   ▼
┌────────────────────────────────────────────────────────────────────┐
│ DEPLOYMENT LAYER                                                   │
│   pleme-io/k8s/clusters/<name>/infrastructure/   FluxCD HelmRelease│
│   pleme-io/k8s/clusters/<name>/apps/             FluxCD HelmRelease│
│   pleme-io/tatara-infra/                         tatara reconciler │
│   pleme-io/pangea-architectures/workspaces/      Pangea CR / Tofu  │
└────────────────────────────────────────────────────────────────────┘
```

A new piece of compute always lands in the leftmost column it
matches; layer-3 + layer-4 wrapping is generated when applicable.
The flow is **one direction only** — Layer 0 changes propagate up;
operators never edit Layer 4 directly except for cluster-specific
overrides (small, declarative).

## IV. The packaging matrix

Cross-tabulate "what kind of work" × "which Helm library chart":

| Compute kind | Library chart | Wraps | Notes |
|---|---|---|---|
| Stateless HTTP service (Rust/Go container) | `pleme-microservice` | Deployment + Service + HPA + PDB | Pre-WASM legacy shape |
| Stateful service (Postgres, NATS) | `pleme-statefulset` | StatefulSet + Service + headless Service | DB-class |
| Always-on operator (Rust controller) | `pleme-operator` | Deployment + RBAC + CRDs + ServiceMonitor | Pre-WASM legacy |
| Background worker (Rust container) | `pleme-worker` | Deployment + ScaledObject | Pre-WASM legacy |
| Periodic CronJob (Rust/bash) | `pleme-cronjob` | CronJob | Pre-WASM legacy |
| **WASM/WASI program (any shape)** | **`pleme-computeunit`** | **ComputeUnit + per-shape sidecars** | **NEW — preferred for new work** |

**Rule of thumb**: any new in-cluster work goes through
`pleme-computeunit` unless there's an explicit reason it can't be
expressed as WASM/WASI (e.g., GPU compute, native dynamic linking,
existing third-party Helm chart we wrap rather than fork).

The pre-WASM library charts stay as they are — they handle the
existing fleet's container-based services. The migration path from
`pleme-microservice` → `pleme-computeunit` is per-service, weighed
against payoff.

## V. Composition over time — the layered evolution

Same workload, evolved across ~1 year of pleme-io maturity:

```
Year 0: bash CronJob in a YAML file
            ▼
Year 1: pleme-cronjob chart wrapping a Rust binary in a container
            ▼
Year 2: pleme-computeunit chart wrapping a tatara-lisp program
        compiled to WASM, fetched from a github:URL, capability-bounded
        with typed schemas
```

Each generation is more compact, more reproducible, more typed,
breathable, and hot-replaceable. The cluster's resident footprint
strictly decreases (containers → WASM). The number of LoC strictly
decreases (Rust → tatara-lisp + macros).

**Consequence**: the meta-framework is itself breathable. New
patterns land in higher layers; old patterns deprecate downward
into legacy charts. The same `lareira-<name>` chart name persists;
its dependency moves from `pleme-microservice` to `pleme-computeunit`
when the migration completes. **Operator-side YAML doesn't change
— the chart is the migration boundary.**

## VI. Cross-layer invariants

These hold across every workload, regardless of which compute kind it is:

### VI.1 Default-OFF

Every chart ships `enabled: false` (or per-component equivalent).
Helm renders nothing until the cluster operator opts in. Per
[`BREATHABILITY.md` §VII.7](BREATHABILITY.md).

### VI.2 Helm-first

Every in-cluster workload goes through Helm, with the right library
chart. Raw `HelmRelease` blocks with inline upstream values are
drift. Per [`BREATHABILITY.md` §VII.7](BREATHABILITY.md).

### VI.3 Capability-bounded

Every workload declares its capabilities explicitly. The runtime
refuses any host call without a matching token. Per
[`WASM-STACK.md` §V](WASM-STACK.md).

### VI.4 Breathable when possible

Every workload declares its breathability tier (always-on /
scale-to-zero / scale-to-zero-stateful / elastic-storage). Per
[`BREATHABILITY.md` §II](BREATHABILITY.md).

### VI.5 Observable by default

Every workload ships ServiceMonitor + PrometheusRule unless the
chart overrides — handled by the library charts so consumer charts
never have to think about it.

### VI.6 Tatara-lisp-first authoring

Any new program goes in tatara-lisp unless one of the explicit
exemptions applies (Rust for the runtime substrate; existing legacy
binaries during migration). Per [`SCRIPTING.md`](SCRIPTING.md).

### VI.7 URL-addressable sources

Every program is referenced by a Nix-flake-style URL
(`github:owner/repo/path?ref=tag` or equivalent), content-addressed
via BLAKE3 in the cache. Per [`WASM-PACKAGING.md`](WASM-PACKAGING.md).

### VI.8 Tameshi-attestable

Every artifact passes through the three-pillar BLAKE3 attestation
chain (artifact + control + intent). Per
[`THEORY.md` §V](THEORY.md).

## VII. The promotion ladder for a new program

Step-by-step, from notion to running in production:

```
1. Author the .tlisp source in pleme-io/programs/<name>/main.tlisp.
   ─→ Layer 0 source.

2. Test locally:    nix run pleme-io/tatara-lisp#script -- ./main.tlisp

3. Compile to WASM: nix build .#<name>
   ─→ Layer 1 artifact.

4. Publish:         git push pleme-io/programs main, tag the release.
   ─→ Source is now github:pleme-io/programs/<name>/main.tlisp?ref=<tag>

5. Wrap as a Helm consumer chart in helmworks/charts/lareira-<name>/.
   30 lines: Chart.yaml + values.yaml depending on pleme-computeunit.
   ─→ Layer 3 packaging.

6. Test the chart:   helm template ./charts/lareira-<name> --set ...
   ─→ Verifies the ComputeUnit + sidecars render.

7. Add a HelmRelease in pleme-io/k8s/clusters/<name>/<dir>/release.yaml
   with `suspend: true`.
   ─→ Layer 4 deployment, gated.

8. Verify: kustomize build clusters/<name> succeeds.

9. Operator: flip suspend → false. FluxCD reconciles; wasm-operator
   fetches the URL, compiles + caches, dispatches.

10. Hot-replacement: bump `?ref=<tag>` in the chart; FluxCD picks
    up the new chart version; wasm-operator hot-replaces (Pattern #48).
    No pod restart, no operator-side change beyond the version bump.
```

Steps 1-4 are author-side; 5-6 are packager-side; 7-9 are
operator-side. Each step is independently reviewable; each
introduces typed checks.

## VIII. Composition of programs — meta-pattern

A single ComputeUnit composing other ComputeUnits forms a saga
([`WASM-PATTERNS.md` Pattern #31](WASM-PATTERNS.md)). The
saga-controller dispatches sub-modules by emitting their CRs; each
sub-CR is a normal layer-2 resource. Recursive composition without
recursive package management — the cluster is flat.

```
┌─────────────────────────────────────────────────────────────┐
│ saga ComputeUnit                                            │
│ trigger: watch saga.pleme.io/Saga                           │
│   spec.steps: [github:.../step-1.tlisp,                     │
│                github:.../step-2.tlisp,                     │
│                github:.../step-3.tlisp]                     │
└──────────────────┬──────────────────────────────────────────┘
                   │ dispatches as needed
                   ▼
              ┌─────────┐         ┌─────────┐         ┌─────────┐
              │ step-1  │────────▶│ step-2  │────────▶│ step-3  │
              │  CU     │         │  CU     │         │  CU     │
              └─────────┘         └─────────┘         └─────────┘
              (program)            (program)            (program)
```

Every step is itself a `ComputeUnit`. The saga is just *another*
ComputeUnit watching `Saga` CRs. **No special saga runtime; the
WASM/WASI runtime + a Lisp macro do the whole thing.**

## IX. Rendering invariants the framework should enforce

For each layer, a renderer (`pangea-core`, `helm template`,
`tatara-lisp-script`, `forge-gen`) checks:

| Layer | Invariant | Enforcer |
|---|---|---|
| 0 (source) | passes `cargo clippy --pedantic` / `tatara-lisp-script --typecheck` | CI |
| 0 (source) | imports go through `use-domain` / `use-library` (typed) | tatara-lisp-script |
| 1 (artifact) | content-addressed (BLAKE3); attestation chain present | tameshi |
| 2 (resource) | matches CRD schema; capability list non-empty for non-program shapes | wasm-operator admission webhook |
| 3 (chart) | depends on a pleme-* library chart; `enabled: false` default | helm-first validator (TODO, [`BREATHABILITY.md` §VII.7](BREATHABILITY.md)) |
| 4 (deployment) | `suspend: true` on first commit; cluster-specific overrides minimal | review |

Each invariant is a typed check; the system fails fast at the
matching boundary. No untyped escape hatches.

## X. Where the framework sits relative to existing pleme-io theory

```
              Pillar 1 (Language)         Pillar 12 (Generation)
              Pillar 11 (JIT/Breathable)  Pillar 8 (Nix images)
                            ▲   ▲   ▲   ▲
                            │   │   │   │
                            └───┼───┴───┘
                                ▼
                         (this document)
                                ▼
        ┌───────────────┬───────────────┬───────────────┬───────────────┐
        │ BREATHABILITY │ SCRIPTING      │ WASM-STACK     │ ... 5 others  │
        │      §        │      §         │      §         │      §        │
        └───────────────┴───────────────┴───────────────┴───────────────┘
                                ▼
                      concrete repos + charts + programs
```

This document is the canonical *index* of those eight existing
docs — it defers each detail to the right doc but specifies the
contract between them. When someone wants to add a new compute
shape, they consult this document first and follow the layered
references.

## XI. Open questions for the next iteration

- **Metadata across layers**: a layer-3 chart's values inform a
  layer-2 ComputeUnit's spec inform a layer-1 module's runtime
  config. Today the propagation is each chart's responsibility.
  Could be lifted to a typed *context* the rendered runs in.
- **Cross-cluster propagation**: a program published to one cluster
  could auto-propagate to a fleet via a `FleetProgram` CR. Today
  each cluster has its own HelmRelease; a fleet-level abstraction
  (per [`WASM-PACKAGING.md` §V](WASM-PACKAGING.md) end state)
  would collapse this.
- **Domain expansion**: each `#[derive(TataraDomain)]` Rust crate
  becomes a Lisp authoring surface; the *taxonomy* of available
  domains is currently informal. A `pleme-domain-registry` repo
  could index them.
- **AI-assisted authoring**: an LLM agent given the meta-framework
  + cookbook could auto-generate a `lareira-<name>` consumer chart
  + program scaffold from a 1-paragraph description. Useful for
  patterns 35-38 in the cookbook (LLM integration patterns) — but
  also for the meta-framework itself (the agent uses the framework
  to author *new* framework members).

## XII. See also

- [`THEORY.md`](THEORY.md) — the 12 pillars
- [`BREATHABILITY.md`](BREATHABILITY.md) — fleet breathability invariants
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49-pattern cookbook
- [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) — authoring
- [`WASM-PACKAGING.md`](WASM-PACKAGING.md) — URL grammar + cache
- [`WASM-RUNTIME-COMPLETE.md`](WASM-RUNTIME-COMPLETE.md) — closing the runtime loop
- This is the eighth — the connecting tissue.
