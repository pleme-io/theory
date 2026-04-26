# Closing the runtime loop — elastic WASM/Lisp/tatara on Kubernetes

> **Frame.** The trilogy
> ([`WASM-STACK.md`](WASM-STACK.md) +
> [`WASM-PATTERNS.md`](WASM-PATTERNS.md) +
> [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) +
> [`WASM-PACKAGING.md`](WASM-PACKAGING.md))
> declares the architecture. This document closes the three loops the
> trilogy left open:
>
> 1. **Full breathability** across every tier of the runtime, with
>    Helm support that exercises every pleme-lib invariant.
> 2. **Container images** built via the canonical
>    [`substrate` `tool-image-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/build/rust/tool-image-flake.nix)
>    pattern + `forge` for publication.
> 3. **The convergence controller** ([`tatara-reconciler`](https://github.com/pleme-io/tatara/tree/main/tatara-reconciler))
>    re-implemented as a tatara-lisp program on the WASM runtime,
>    closing the recursive loop: the platform that runs Lisp programs
>    is itself running a Lisp program.

---

## I. Elastic breathability across every tier

The current `lareira-wasm-platform` chart has two tiers:
operator (always-on, leader-elected) + engine pods (dispatched per
ComputeUnit). The fully elastic shape adds two more for the function
shape ([`WASM-PACKAGING.md` §IV.1](WASM-PACKAGING.md)) and the
service shape:

```
┌──────────────────────────────────────────────────────────────────┐
│ TIER 0 — always-on, lightweight (~200 MiB resident at idle)      │
│   wasm-operator (leader-elected)                                 │
│   keda-http interceptor (already in lareira-keda-http)           │
└──────────────────────────────────────────────────────────────────┘
                              ▼ schedules
┌──────────────────────────────────────────────────────────────────┐
│ TIER 1 — scale-to-zero per ComputeUnit                           │
│                                                                    │
│   program shape   → Pod (run-once, exit)                          │
│   job shape       → CronJob → Pod                                 │
│   function shape  → Deployment scaled by NATS/Kafka/SQS depth     │
│   service shape   → Deployment + Service + HTTPScaledObject       │
│   controller shape → Deployment + lease (scaled by event rate)    │
│                                                                    │
│   Each backed by a wasm-engine pod loading the WASM module        │
│   from the cluster's content-addressed store.                     │
└──────────────────────────────────────────────────────────────────┘
                              ▼ pulls modules from
┌──────────────────────────────────────────────────────────────────┐
│ TIER 2 — module store (PVC, RWX, breathable=true labelled)       │
│                                                                    │
│   blake3-keyed cache. Resized by pleme-storage-elastic at 80%.   │
│   First fetch from github:* URL → blake3 → cached forever.        │
└──────────────────────────────────────────────────────────────────┘
                              ▼ writes events to
┌──────────────────────────────────────────────────────────────────┐
│ TIER 3 — observability (already present)                         │
│   vmsingle, victoria-logs, vector — not specific to WASM         │
└──────────────────────────────────────────────────────────────────┘
```

### I.1 Tier-1 elasticity per shape

The chart's `values.yaml` exposes per-shape elasticity defaults that
operators rarely override:

```yaml
# helmworks/charts/lareira-wasm-platform/values.yaml — proposed extension

elasticity:
  # All scale-to-zero shapes inherit these unless overridden per-shape
  default:
    cooldownPeriod: 600        # 10 min
    pollingInterval: 15
    targetIdleReplicas: 0      # how many to keep warm; 0 = full scale-to-zero

  # Per-shape policy — overrides default per ComputeUnit shape.
  shape:
    program:
      ttlSecondsAfterFinished: 60
    job:
      successfulJobsHistoryLimit: 3
      failedJobsHistoryLimit: 5

    function:                  # NEW — async event-driven
      defaultBatchSize: 10
      maxReplicas: 20          # absolute ceiling per CU
      cooldownPeriod: 60       # tear down quickly; cold start is cheap
      kedaTrigger: nats        # nats | kafka | sqs | redis-stream | rabbitmq

    service:                   # warmed serverless
      cooldownPeriod: 600      # 10 min — feels always-on for typical use
      maxReplicas: 5
      keepWarmPattern: ""      # if non-empty, regex of paths that pin minReplicas=1
                                # for canonical hot-path routes (e.g. /healthz).

    controller:
      leaseAware: true
      maxReplicas: 1           # leader-elected; 2+ for HA

  # Provider plugin cache — one PVC per cluster shared across all
  # wasm-engine pods. Avoids re-downloading WASM modules on cold-starts.
  cache:
    enabled: true
    storageClass: local-path
    size: 10Gi
    accessMode: ReadWriteMany
    annotations:
      pleme.pleme.io/breathable: "true"   # auto-resized by pleme-storage-elastic
```

### I.2 Helm-side full pleme-lib invariants

Every chart in the WASM stack ships with the pleme-lib defaults
([`pleme-lib`](https://github.com/pleme-io/helmworks/tree/main/charts/pleme-lib)
in helmworks):

| Invariant | Where |
|---|---|
| `enabled: false` master gate | already set on every lareira-* chart |
| ServiceMonitor + PrometheusRule wired | already done |
| NetworkPolicy `enabled: false` (opt-in) | already done |
| PodDisruptionBudget with sane defaults | already done |
| Resource budgets (requests + limits) | already done |
| Pod security context (runAsNonRoot, readOnlyRootFilesystem) | already done |
| Liveness + readiness + startup probes | partial — needs explicit per-tier spec |
| Topology-spread / anti-affinity for HA | partial — wasm-operator ready, engine pods need it |
| breathable=true PVC labels | already done on store + cache PVCs |

The two gaps are tracked under
[`tatara-lisp-controllers/lib/`](https://github.com/pleme-io/tatara-lisp-controllers/tree/main/lib)
as macro responsibilities — `defrule-driven-controller`
auto-emits topology-spread + anti-affinity for `:replicas > 1`, and
the engine-pod template gains `startupProbe` + `livenessProbe`
defaults from the operator.

## II. Container images via the canonical pattern

Every WASM-stack image goes through
[`substrate/lib/build/rust/tool-image-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/build/rust/tool-image-flake.nix).
The pattern composes:

- `crate2nix` for hermetic Cargo builds (no `cargo build` outside Nix sandbox).
- `dockerTools.buildLayeredImage` distroless OCI images, ~30-50 MiB total.
- `forge` for multi-arch publish to `ghcr.io/pleme-io/<tool>:<tag>`.
- BLAKE3 attestation chain via `tameshi` at publish time.

### II.1 Migration plan for the WASM-stack images

The four images named in
[`WASM-STACK.md` §VII](WASM-STACK.md):

| Image | Source | Migration |
|---|---|---|
| `wasm-operator` | `pleme-io/wasm-platform/wasm-operator/` | Add `image` attr to `wasm-platform/flake.nix` via `tool-image-flake` |
| `wasm-engine` | `pleme-io/wasm-platform/wasm-engine/` | Same |
| `tatara-lisp-script` | `pleme-io/tatara-lisp/tatara-lisp-script/` | Add `image` attr to `tatara-lisp/flake.nix` via `tool-image-flake` |
| `openclaw-artifact-registry` | `pleme-io/openclaw-artifact-registry/` | Same |

Each is 4 lines of Nix added to the flake's `outputs`:

```nix
# wasm-platform/flake.nix — proposed addition
imageOps = (import "${substrate}/lib/build/rust/tool-image-flake.nix" {
  inherit nixpkgs crate2nix flake-utils forge;
}) {
  toolName = "wasm-operator";
  src = self;
  repo = "pleme-io/wasm-platform";
  tag = "0.1.0";
};
# Then merge imageOps.packages.${system}.image into outputs.packages.
```

`nix run .#release` becomes the canonical publish command across
the fleet. No bash, no `dockerfile`, no `nix-shell` cargo invocation.

### II.2 Why this matters for the WASM operator specifically

The operator pulls WASM modules from `github:` URLs at runtime. Its
*own* image is built the same way every other pleme-io tool's image
is built — recursively, the operator's bootstrap follows the rule
it itself enforces.

```
Substrate's tool-image-flake.nix
  → builds the wasm-operator image
  → pushed to ghcr.io/pleme-io/wasm-operator:0.1.0
  → kubernetes pulls it
  → wasm-operator boots
  → reads ComputeUnit CRs that point at github:pleme-io/programs/<...>
  → fetches those .tlisp programs
  → compiles them to WASM
  → runs them in wasm-engine pods (also built via tool-image-flake)
```

The bootstrap chain is one substrate flake macro invocation per
image; everything else is content-addressed Lisp.

## III. The convergence controller in tatara-lisp

`tatara-reconciler` exists today as Rust:

- 8-phase Unix-process state machine for `Process` CRs.
- Composes three-pillar BLAKE3 attestation (artifact + control + intent).
- Emits FluxCD Kustomization / HelmRelease CRs for downstream
  controllers to apply.
- Runs as a long-lived K8s controller pod.

This is **textbook `defcontroller` shape** under the new architecture.
Re-implementing it as a tatara-lisp program eliminates ~1500 lines of
Rust boilerplate (state machine, CRD wiring, status propagation,
attestation chain composition) by leaning on:

- `defrule-driven-controller` macro (auto-generates the CRD, watches,
  reconcile loop, status propagation).
- The 8-phase state machine becomes a single `defstate-machine` macro
  call.
- BLAKE3 attestation already lives in the `pleme/tameshi` typed-domain.
- FluxCD CR emission is just `(k8s/apply-resource ...)`.

### III.1 The shape after migration

```lisp
;; pleme-io/programs/convergence-controller/main.tlisp
;;
;; The convergence controller. Replaces tatara/tatara-reconciler.

(use-library github:pleme-io/tatara-lisp-controllers/lib/defrule-driven-controller.tatara)
(use-library github:pleme-io/tatara-lisp-controllers/lib/defstate-machine.tatara)
(use-domain  pleme/k8s)
(use-domain  pleme/tameshi)
(use-domain  pleme/fluxcd)

(defstate-machine process-lifecycle
  :states [:Pending :Planning :Validating :Applying :Verifying
           :Attesting :Succeeded :Failed]
  :transitions
    {[:Pending     :Planning]   plan-process
     [:Planning    :Validating] validate-plan
     [:Validating  :Applying]   apply-via-flux
     [:Applying    :Verifying]  poll-flux-status
     [:Verifying   :Attesting]  compute-attestation
     [:Attesting   :Succeeded]  finalize-success
     [:Validating  :Failed]     fail-validation
     [:Applying    :Failed]     fail-apply
     [:Verifying   :Failed]     fail-verify}

  :on-error  (transition-to :Failed))

(defrule-driven-controller convergence-controller
  :version       "v0.1.0"
  :config-schema ProcessSpec
  :rule-schema   ProcessRule
  :watches       [(:tatara.pleme.io/v1alpha1 :processes)
                  (:tatara.pleme.io/v1alpha1 :processtables)
                  (:kustomize.toolkit.fluxcd.io/v1 :kustomizations)
                  (:helm.toolkit.fluxcd.io/v2 :helmreleases)]

  :reconcile
    (let* [process (current-resource)
           current-state (status-field process :phase :Pending)
           next-state (advance-state-machine process-lifecycle
                                              current-state
                                              process)]
      (when (not= current-state next-state)
        (k8s/patch-status process {:phase next-state})
        (emit-event :PhaseTransition
                    {:process (k8s/name process)
                     :from current-state
                     :to next-state})))

  :status-fields  [:phase :flux-status :attestation-blake3 :error])

;; Step implementations — each is a small helper that returns the next
;; state OR :Failed. Together ~150 lines.

(defn plan-process [process]
  ...)

(defn apply-via-flux [process]
  (let* [plan (resolve-plan process)]
    (fluxcd/apply-kustomization plan)
    :Verifying))

(defn poll-flux-status [process]
  (cond
    (fluxcd/healthy? (k8s/process-flux-name process)) :Attesting
    (fluxcd/failed?  (k8s/process-flux-name process)) :Failed
    :else                                              :Verifying))   ; stay

(defn compute-attestation [process]
  (let* [chain (tameshi/three-pillar-blake3
                 :artifact   (fluxcd/applied-artifact process)
                 :control    (k8s/process-control-state process)
                 :intent     (k8s/spec-snapshot process))]
    (k8s/patch-status process {:attestation-blake3 chain})
    :Succeeded))

;; Schemas — Tatara-domain auto-generates the CRD + admission validator.

(defschema ProcessSpec
  :pid          :integer
  :children     (default (list-of :string) [])
  :substrate    :substrate-ref
  :compliance   :compliance-policy
  :attestation  :attestation-policy)

(defschema ProcessRule
  :phase-on-error  (default (enum :retry :pause :fail) :retry)
  :timeout         (default :duration "10m")
  :grace-period    (default :duration "30s"))
```

~250 lines of tatara-lisp replaces ~1500 lines of Rust. The CRD is
generated, the informers are generated, the state machine has a
typed transition table. **Fewer bugs, faster iteration, hot-replaceable
on a `nix run` per the [`WASM-PACKAGING.md`](WASM-PACKAGING.md)
flow.**

### III.2 Migration phasing

This is non-trivial. The Rust controller has been hardened. We don't
big-bang-rewrite; we run both side-by-side until parity is proven:

- **Phase α** (now): the Lisp scaffold above lives at
  `pleme-io/programs/convergence-controller/` (locally created in this
  session). Reviewed but not deployed.
- **Phase β**: `tatara-reconciler` Rust controller observes a new
  CR field `spec.runtime: rust|lisp`. Operators can opt-in per
  `Process` CR. The Lisp controller picks up only those marked `lisp`.
- **Phase γ**: when the Lisp controller has reconciled at parity for
  ~30 days across the test fleet, default the field to `lisp`.
- **Phase δ**: deprecate the Rust controller. The chart's
  `lareira-tatara-reconciler` drops the Rust image and points at
  `github:pleme-io/programs/convergence-controller/main.tlisp`.

The end state: even the platform's reconciler is just another
WASM-runtime program. The Rust code that bootstraps the platform
shrinks to just `wasm-operator` + `wasm-engine` + the
content-addressed store. **Everything else is Lisp.**

### III.3 Why this is the right move now

The trilogy ([`WASM-STACK.md`](WASM-STACK.md) +
[`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) +
[`WASM-PACKAGING.md`](WASM-PACKAGING.md)) only fully demonstrates the
substrate when *the substrate runs itself.* `tatara-reconciler` is
the perfect first dogfood:

- It's a pure controller (no exotic resource needs).
- It already has a typed CRD shape that maps cleanly onto
  `defrule-driven-controller`.
- Its complexity is in the state machine, which is exactly where
  Lisp's `defstate-machine` macro shines.
- It's load-bearing — running it on the WASM runtime *is* the
  trustworthiness proof.

## IV. Closing the recursive loop

```
       wasm-operator       (Rust, built via substrate/tool-image-flake)
            │
            └── runs WASM modules ──┐
                                    │
                          ┌─────────▼─────────┐
                          │   wasm-engine pod  │
                          │  loads .wasm bytes │
                          └─────────┬─────────┘
                                    │
                                    ▼
                          ┌─────────────────────┐
                          │ convergence-controller.tlisp │
                          │   (compiled to WASM)         │
                          └──────────┬──────────────────┘
                                     │
                                     ▼
                       ┌──────────────────────────────────┐
                       │ reconciles Process CRs via FluxCD │
                       │   ↑                               │
                       │   └─── observes its own image ────│  ← the operator's
                       │       (wasm-operator @ ghcr.io)   │   image is itself
                       │       (verifies attestation)      │   a Process CR
                       └──────────────────────────────────┘
```

The operator's own deployment is a `Process` CR managed by the Lisp
convergence controller running on the operator. **The runtime is
self-hosting at the operational layer.** Rust handles the substrate
(memory safety, capability enforcement, image build); Lisp handles
*everything else*.

That's the architecture this session has been converging on. Phases
above name the path; the substrate is in place.

## V. Where the code lives after this lands

```
pleme-io/wasm-platform/                substrate (Rust) — the runtime
  + tool-image-flake → ghcr.io/pleme-io/wasm-operator
  + tool-image-flake → ghcr.io/pleme-io/wasm-engine

pleme-io/tatara-lisp/                  authoring (Rust) — the scripting language
  + new tatara-lisp-source crate (Phase B)
  + tool-image-flake → ghcr.io/pleme-io/tatara-lisp-script

pleme-io/tatara-lisp-controllers/       library (Lisp) — controller macros
  lib/                                   8 macros (already scaffolded)
  examples/                              dns-reconciler (already scaffolded)

pleme-io/programs/                      programs (Lisp) — runs on the runtime
  pvc-autoresizer/                       job
  dns-reconciler/                        controller
  github-webhook-flux/                   service
  fleet-attestation-sweep/               program (one-shot)
  thumbnail-fn/                          function (event-driven)
  convergence-controller/                NEW — replaces tatara-reconciler
                                          (closes the recursive loop)

pleme-io/helmworks/                     packaging (Helm)
  charts/lareira-wasm-platform/          deploys the runtime
  charts/lareira-tatara-stack/           umbrella
  + Phase A elasticity values landed in this session
  + Phase B image refs become real once tool-image-flake adds attrs

pleme-io/k8s/clusters/<name>/infrastructure/wasm-stack/   per-cluster opt-in
```

Six repos (one new — `programs`) participate. None are forks; each
gains additive surface. The end state is a substrate where adding
new platform behavior is a `nix run` on the programs repo.

## VI. See also

- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49 cookbook patterns
- [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) — Lisp + YAML authoring
- [`WASM-PACKAGING.md`](WASM-PACKAGING.md) — URL grammar + cache
- [`BREATHABILITY.md`](BREATHABILITY.md) — fleet-wide use-causes-spin-up
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`THEORY.md` Pillar 1 + Pillar 12](THEORY.md) — language + generation
- [`tatara/tatara-reconciler`](https://github.com/pleme-io/tatara/tree/main/tatara-reconciler) — the controller this document migrates
- [`substrate/lib/build/rust/tool-image-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/build/rust/tool-image-flake.nix) — canonical OCI build
