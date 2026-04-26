# Fleet declaration — pull programs from git, run them, reach perfect state

> **Frame.** [`META-FRAMEWORK.md`](META-FRAMEWORK.md) names the four
> compute layers; [`WASM-PACKAGING.md`](WASM-PACKAGING.md) makes every
> program URL-addressable. **This document closes the user-facing
> loop:** declare every program a cluster runs in *one* file, FluxCD
> reconciles the cluster's `ComputeUnit` set to match, and the
> wasm-operator pulls programs from git on demand. **A cluster's
> entire program inventory is a single Helm release.**

---

## I. The promise

> Pull programs from git, run them on the cluster, reach perfect state
> with a consistent system.

What "perfect state" means concretely:

- For every entry in the cluster's program list, a matching
  `ComputeUnit` CR exists in the right namespace.
- Each `ComputeUnit`'s `module.source` points at the declared git URL
  (and version tag).
- Each `ComputeUnit`'s `.status.phase` is one of {`Running`,
  `Succeeded` (for one-shots), `Pending` (for cron between runs)} —
  never stuck or erroring.
- Programs that were declared but no longer needed are deleted.
- Programs whose declared version changed have been hot-replaced
  ([Pattern #48](WASM-PATTERNS.md#48-hot-replacement)).

When all five conditions hold, the cluster is in perfect state.

## II. The declaration shape

A single Helm release with `lareira-fleet-programs` as the chart and
a list of programs in its values:

```yaml
# clusters/<name>/programs/release.yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: <cluster>-fleet-programs
  namespace: tatara-system
spec:
  chart:
    spec:
      chart: lareira-fleet-programs
      version: "0.1.x"
      sourceRef:
        kind: HelmRepository
        name: pleme-charts
        namespace: flux-system
  values:
    enabled: true

    programs:
      # Add an entry → cluster runs it.
      # Remove an entry → cluster stops running it.
      # Bump ?ref=<tag> → cluster hot-replaces.

      - name: pvc-autoresizer
        module:
          source: "github:pleme-io/programs/pvc-autoresizer/main.tlisp?ref=v0.1.0"
        trigger:
          cron: "*/5 * * * *"
        capabilities:
          - kube-pvc-list
          - kube-pvc-patch
        config:
          triggerAt: 0.80

      - name: dns-reconciler
        module:
          source: "github:pleme-io/programs/dns-reconciler/main.tlisp?ref=v0.2.1"
        trigger:
          watch: { crd: dns.pleme.io/DnsReconciler }
        capabilities:
          - kube-cr-watch@dns.pleme.io/dnsreconcilers
          - http-out:api.cloudflare.com

      # ... N more entries
```

That's the whole interface. A cluster's program inventory is a YAML
list, edited via PR.

## III. The reconciliation pipeline

```
1. operator: edit clusters/<name>/programs/release.yaml; PR + merge
                       │
                       ▼ FluxCD source-controller polls (or webhook)
2. FluxCD: detects new commit; pulls the chart's rendered manifests
                       │
                       ▼
3. Helm renders the lareira-fleet-programs chart with the values:
     - One ComputeUnit per entry
     - One Service + HTTPScaledObject per service-shape entry
     - One KEDA ScaledObject per function-shape entry
     - One PrometheusRule per entry that defines rules
                       │
                       ▼
4. helm-controller applies via SSA (server-side apply)
     - New entries → new CRs created
     - Modified entries → CRs patched in-place
     - Removed entries → CRs deleted (Helm release tracks ownership)
                       │
                       ▼
5. wasm-operator observes the CRs:
     For each ComputeUnit:
       a. resolve module.source URL → BLAKE3 hash → cache lookup
          - cache miss: fetch tarball / file from github + verify
          - cache hit: reuse cached compiled WASM
       b. compile to WASM (one-time per BLAKE3)
       c. dispatch per spec.trigger:
          - oneShot   → Pod (run-and-exit)
          - cron      → CronJob → Pod
          - service   → Deployment + Service routing
          - function  → Deployment scaled by KEDA event source
          - watch     → Deployment + lease-elected reconciler loop
                       │
                       ▼
6. each engine pod loads the WASM module + capability tokens
     - host calls only succeed for declared capabilities
     - lifecycle: per-shape (see WASM-PACKAGING.md §IV)
                       │
                       ▼
7. perfect state reached when all 5 conditions from §I hold
```

Steps 1-2 are git-driven (operator side). Steps 3-4 are
Helm-driven (CI/automation). Step 5+ are runtime — they happen
continuously. The cluster moves toward perfect state on every
reconciliation tick (FluxCD default 5min, faster with webhooks via
[`github-webhook-flux`](https://github.com/pleme-io/programs/tree/main/github-webhook-flux)).

## IV. Drift detection + correction

The wasm-operator's reconcile loop is the canonical drift detector.
For every ComputeUnit it manages, on every tick:

1. **Module drift**: compare `spec.module.source` (declared) against
   the cached BLAKE3 the engine pod is running. If different, do
   hot-replacement (Pattern #48).
2. **Capability drift**: compare `spec.capabilities` against the
   active token set. If different, restart the pod with new tokens
   (programs are stateless except for state stored in CRs/secrets).
3. **Trigger drift**: e.g. cron changed, service hosts changed,
   event source changed. Reconcile the Pod / Deployment / KEDA
   ScaledObject accordingly.
4. **Phase drift**: a controller-shape pod that's been
   `lease-renewed` but isn't producing events for >threshold may
   indicate a wedged informer; emit `WedgedController` event,
   restart pod.

Each drift class has a typed correction. Operator never has to
`kubectl edit` to fix it; the operator emits a typed event the
cluster's alert layer consumes.

## V. Adding a new program

```
1. Author <new-program>/main.tlisp in pleme-io/programs.
2. Test:    nix run pleme-io/tatara-lisp#script -- ./main.tlisp
3. Push:    git tag v0.1.0; git push --tags
4. Edit:    clusters/<name>/programs/release.yaml
            programs:
              - name: <new-program>
                module:
                  source: github:pleme-io/programs/<new-program>/main.tlisp?ref=v0.1.0
                trigger: { cron: "*/10 * * * *" }   # or whatever shape
                capabilities: [ ... ]
5. PR + merge.
6. FluxCD picks up the chart change within 5min (or instantly via webhook).
7. wasm-operator fetches + compiles + dispatches.
8. perfect state restored with the new program included.
```

Total operator work: 5 lines of YAML in the cluster's overlay.

## VI. Removing a program

Delete the entry from `programs:`. PR + merge. FluxCD's helm-controller
sees the resource is no longer rendered → deletes the CR. wasm-operator
sees the CR deleted → reaps the pod / Service / ScaledObject.

Total operator work: deleting 5 lines.

## VII. Bumping a program version

```diff
- module: { source: github:pleme-io/programs/dns-reconciler/main.tlisp?ref=v0.1.0 }
+ module: { source: github:pleme-io/programs/dns-reconciler/main.tlisp?ref=v0.2.0 }
```

PR + merge. FluxCD reconciles → ComputeUnit's `spec.module.source`
changes → operator hot-replaces (Pattern #48). Observed events
confirm the version transitioned without dropping in-flight requests.

Total operator work: changing one tag.

## VIII. Multi-cluster propagation

The same `lareira-fleet-programs` chart is consumed per-cluster.
Each cluster has its own values file with its own program list:

```
pleme-io/k8s/
├── clusters/
│   ├── plo/programs/release.yaml          plo's program inventory
│   ├── rio/programs/release.yaml          rio's program inventory
│   ├── seph/programs/release.yaml         seph's program inventory
│   └── ...
```

Each cluster's program list is independent. Want to roll out a new
program to all clusters? Edit each cluster's file (or use a YAML
overlay tool — kustomize, jsonnet, tatara-lisp) to share the entry.

A future "fleet-wide propagation" pattern would be a single
`FleetProgram` CR observed by every cluster's wasm-operator —
deferred to [`WASM-PACKAGING.md` §V](WASM-PACKAGING.md) phase E.

## IX. Validation at every step

The chart fails fast at `helm template` time on:

- entry without `name`
- entry without `module.source` or `module.oci`
- entry with zero or multiple trigger shapes
- entry with unknown event-source URL prefix

The wasm-operator fails fast at admission time on:

- ComputeUnit referencing a non-existent github URL
- Capability list referencing a Secret the operator can't read
- Trigger configuration the runtime can't dispatch

Failures bubble up as typed events. Operators never see a silently
broken state.

## X. Comparison to alternatives

| Approach | Adding a program | Removing | Multi-cluster | LoC per program |
|---|---|---|---|---|
| Per-program HelmRelease | Create new YAML | Delete YAML | Per-cluster create | ~30 + per-cluster wiring |
| Kustomize overlays | Multi-file edit | Multi-file delete | Per-cluster overlay | ~20 + overlay |
| ExternalSecrets / Argo CD App-of-Apps | Argo Application CR | Delete CR | Argo target list | ~50 |
| **fleet-programs chart (this)** | **5-line YAML append** | **5-line YAML delete** | **per-cluster values file** | **5-15 per program** |

The fleet-programs chart is the lowest-friction shape that still
preserves Helm's declarative + content-addressed properties.

## XI. Composition with other lareira-* charts

The fleet-programs chart deploys *applications*. The cluster's
*platform* still uses per-component lareira-* charts (cert-manager,
ingress-nginx, KEDA, vm-stack, etc.) per the meta-framework
([`META-FRAMEWORK.md`](META-FRAMEWORK.md)).

The two layers stack naturally:

```
clusters/<name>/infrastructure/   ← lareira-* per-component charts (platform)
clusters/<name>/programs/         ← lareira-fleet-programs (applications)
```

The first layer is *what the cluster needs to function*. The second
is *what the cluster does*. Both layers are pure declarative YAML;
both reach perfect state via FluxCD.

## XII. The end state

```
              cluster
                │
                ▼
┌──────────────────────────────────────────────────┐
│ INFRASTRUCTURE (per-component lareira-* charts)  │
│  cert-manager, ingress-nginx, keda, vm-stack,     │
│  authentik, ...                                   │
└──────────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────┐
│ WASM RUNTIME (lareira-tatara-stack)              │
│  wasm-operator, wasm-engine, content-addressed    │
│  module store                                     │
└──────────────────────────────────────────────────┘
                │
                ▼
┌──────────────────────────────────────────────────┐
│ PROGRAMS (lareira-fleet-programs — single release)│
│   one entry per program, each pointing at         │
│   github:owner/repo/path?ref=tag                  │
│   wasm-operator fetches, compiles, dispatches     │
└──────────────────────────────────────────────────┘
                │
                ▼
        perfect state, continuous
```

A cluster's full configuration is **two things**: the infrastructure
charts (the platform) and the fleet-programs values (the application
inventory). Both committed to git, both reconciled by FluxCD.

To rebuild a cluster from scratch: install FluxCD + apply the
cluster's overlay → all the lareira-* charts deploy → all the
programs in the fleet-programs values get fetched and run. **No
operator interaction beyond the initial bootstrap.**

## XIII. See also

- [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — the typed compute hierarchy
- [`WASM-PACKAGING.md`](WASM-PACKAGING.md) — module URL grammar
- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`BREATHABILITY.md`](BREATHABILITY.md) — fleet breathability
- [`lareira-fleet-programs` chart](https://github.com/pleme-io/helmworks/tree/main/charts/lareira-fleet-programs)
