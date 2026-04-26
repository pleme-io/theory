# WASM/WASI on Kubernetes — programs, jobs, services, controllers

> **Frame.** [Pillar 1](THEORY.md#pillar-1-language) makes Rust + tatara-lisp +
> WASM/WASI the language stack. This document packages that stack
> *for Kubernetes* — one Helm umbrella that turns any pleme-io cluster
> into a runtime where every reusable unit of work (a one-shot program,
> a periodic job, a long-running service, a CRD controller) is
> authored once as a WASM module and dispatched via a typed
> `ComputeUnit` resource.
>
> **Centre of gravity:** the [`wasm-platform`](https://github.com/pleme-io/wasm-platform)
> Rust workspace ([`compute-theory.md`](https://github.com/pleme-io/wasm-platform/blob/main/docs/compute-theory.md)),
> packaged as `lareira-wasm-platform`. Everything else in this doc is
> *opt-in sugar* on top.

---

## I. The four program shapes

```
                            ┌──────────────────────────────────┐
                            │   ComputeUnit CR (typed)         │
                            │   spec:                           │
                            │     module:    blake3:abc123…    │
                            │     trigger:   one of {…}         │
                            │     capabilities: […]             │
                            └──────────────────┬───────────────┘
                                               │
                            ┌──────────────────┼───────────────┐
                            ▼                  ▼               ▼
                       ┌────────┐         ┌────────┐      ┌────────┐
                       │ program│         │  job   │      │service │      ┌────────────┐
                       │ (once) │         │ (cron) │      │(always)│      │ controller │
                       └────────┘         └────────┘      └────────┘      └────────────┘
                       trigger:           trigger:        trigger:        trigger:
                         oneShot           cron:           service:         watch:
                                            "0 5 * * *"     port: 8080       group:
                                                                              kind:
```

Same `ComputeUnit` resource serves all four; `spec.trigger` is a
sum-type that selects the dispatch shape:

| Shape | spec.trigger | Lifecycle | Wake | Sleep |
|---|---|---|---|---|
| **program** | `oneShot: { args: [...] }` | run-and-exit | `kubectl apply` | when complete |
| **job** | `cron: "0 5 * * *"` | scheduled batch | scheduler tick | when complete |
| **service** | `service: { port: 8080, hosts: [...] }` | long-running | first request (KEDA HTTP) | cooldown |
| **controller** | `watch: { group: "...", kind: "..." }` | reconcile-loop | first watched-resource change | when watch closes |

The **controller** shape is what makes this stack genuinely powerful:
authoring a CRD reconciler stops being "implement kube-rs in Rust + ship
a Docker image" and becomes "write a WASM module + apply a 12-line
ComputeUnit". The wasm-operator handles informer setup, capability-
delegated K8s API access, and lease-based leader election; the
user's WASM module receives typed events and emits typed patches.

## II. Authoring paths

Any language that compiles to `wasm32-wasi` is welcome. Pleme-io
provides four first-class paths:

```
┌────────────────────────────────────────────────────────────────────┐
│ author writes:                       compiles to              ships│
├────────────────────────────────────────────────────────────────────┤
│ tatara-lisp (defprogram …)           tatara-lisp-script   →   WASM │
│ Rust       (kube-rs + wit-bindgen)   cargo + cargo-component  WASM │
│ Go         (kubernetes-sigs go-wit)  TinyGo + WASI            WASM │
│ Python     (componentize-py)         componentize-py          WASM │
└────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
                            optional staging:
              ┌───────────────────────┴──────────────────┐
              │ commit to git OR push to OCI registry   │
              │ (any OCI registry; openclaw is one)     │
              └─────────────────────┬───────────────────┘
                                    ▼
                        kubectl apply ComputeUnit
                                    ▼
                         wasm-operator picks it up
```

Tatara-lisp gets the highest bandwidth in [`SCRIPTING.md`](SCRIPTING.md)
because it composes typed pleme-io domains for free. Rust + Go +
Python are valid for cases where existing libraries pull. The
runtime doesn't care; ComputeUnit is a typed contract over WASM bytes.

## III. The Helm packaging

One foundational chart, plus optional add-ons:

```
helmworks/charts/
├── lareira-wasm-platform     ← THE CHART. wasm-operator + engine + store.
├── lareira-tatara-runtime    optional: Lisp evaluator that compiles tatara-lisp
│                             to WASM and submits ComputeUnits.
├── lareira-openclaw          optional: artifact cache + attestation service
│                             when you want signed/audited program shipping
│                             instead of plain OCI registry pulls.
└── lareira-tatara-stack      umbrella that turns all three on at once.
```

**The minimum.** A cluster that wants WASM/WASI as a runtime needs
ONLY `lareira-wasm-platform`. Authors push WASM modules to any OCI
registry (ghcr.io, the in-cluster OCI registry, a tarball mounted via
ConfigMap — anything the operator can fetch by content hash). The
operator pulls and runs. Openclaw is **not in the critical path**.

**The convenience.** Add `lareira-tatara-runtime` if your authors are
publishing tatara-lisp scripts and you want auto-compilation +
auto-submit of `ComputeUnit` CRs. Add `lareira-openclaw` if you want
attestation-gated artifacts (every program signed with the BLAKE3 +
tameshi chain before the operator accepts it). Each is independent.

## IV. Examples per shape

### IV.1 Program (one-shot)

```yaml
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata:
  name: regenerate-fleet-attestations
  namespace: tatara-programs
spec:
  module:
    source: oci://ghcr.io/pleme-io/programs:fleet-attest-v1.2.0
    blake3: abc123…              # optional pin
  trigger:
    oneShot: { args: ["--fleet=plo,rio,seph"] }
  capabilities:
    - kube-secret-read@flux-system/sops-age
    - http-out:registry.pleme.io
  resources:
    requests: { cpu: 100m, memory: 256Mi }
```

`kubectl apply` → wasm-engine pod runs once → exits → CR.status.phase=Succeeded.

### IV.2 Job (periodic)

```yaml
spec:
  module:
    source: oci://ghcr.io/pleme-io/programs:pvc-autoresizer-v0.1.0
  trigger:
    cron: "*/5 * * * *"
  capabilities:
    - kube-pvc-list
    - kube-pvc-patch
    - prom-query@vmsingle-vm.monitoring.svc:8429
```

Replaces the Rust binary I just wrote in
`helmworks/charts/pleme-storage-elastic/image/` — same logic,
authored as WASM, no per-script container image.

### IV.3 Service (HTTP listener)

```yaml
spec:
  module:
    source: oci://ghcr.io/pleme-io/programs:thumbnail-resizer-v3.1.0
  trigger:
    service:
      port: 8080
      paths: ["/v1/resize"]
      hosts: [thumbnails.rio.svc.cluster.local]
  capabilities:
    - http-in:0.0.0.0:8080
    - kube-configmap-read@thumbnails/config
  breathability:
    enabled: true
    minReplicas: 0
    cooldownPeriod: 600
```

KEDA HTTP add-on scales the wasm-engine pod 0→N→0 on inbound traffic.
Replaces "ship a Go HTTP service in a 200 MiB container" with
"ship a 2 MiB WASM module".

### IV.4 Controller (CRD reconciler)

```yaml
spec:
  module:
    source: oci://ghcr.io/pleme-io/programs:vault-replicator-controller-v0.5.0
  trigger:
    watch:
      group: "secrets.pleme.io"
      version: "v1"
      kind: "VaultMirror"
      namespaces: []        # all
  capabilities:
    - kube-cr-watch@secrets.pleme.io/vaultmirrors
    - kube-secret-read@vault-system/*
    - kube-secret-write@*       # subject to the operator's RBAC
    - http-out:vault.example.com
  resources:
    requests: { cpu: 50m, memory: 128Mi }
```

Authoring a controller stops being a multi-day Rust + kube-rs +
operator-sdk project; it's a WASM module + a typed CR.

The wasm-operator handles:
- Informer setup against the watched group/kind.
- Backoff + leader election across replicas.
- Capability-delegated K8s API access (the WASM module receives a
  typed `kube` host import; it can only call verbs it has tokens for).
- Lifecycle: scale to zero on watch idle, wake on first event.

## V. Capability-based security (already in `wasm-types`)

Every ComputeUnit declares an explicit capability set. The
wasm-engine refuses any host call without a matching token. Common
capabilities:

| Capability | Grants |
|---|---|
| `kube-pod-list` | `pods.list` API verb (cluster-wide unless namespace-suffixed) |
| `kube-cr-watch@<group>/<kind>` | watch a specific CRD |
| `kube-secret-read@<ns>/<name>` | read one Secret (no wildcard by default) |
| `http-out:<host>` | egress HTTP/HTTPS to a host |
| `http-in:<addr>:<port>` | listen on an address (only valid for service shape) |
| `prom-query@<service>` | scalar/instant queries against a Prometheus endpoint |
| `fs:<path>:<rw>` | read/write filesystem at a tmpfs-mounted path |
| `cap-net-bind` | privileged port binding |
| `cap-time` | wall-clock time (denied to deterministic builds by default) |

The token list is enforced at the engine boundary — a malicious WASM
module cannot exfiltrate a secret it didn't get a token for.
Foundational reference: Dennis & Van Horn, "Programming Semantics for
Multiprogrammed Computations" (ACM 1966).

## VI. Why this is breathable by construction

Each shape has natural breathability:

- **program**: lifecycle is "run + exit". Resident at idle: 0.
- **job**: cron schedule. Resident at idle: 0; pod is created on schedule.
- **service**: KEDA HTTP scales 0→N→0. Resident at idle: 0 (plus interceptor pod).
- **controller**: scale-to-zero on watch idle, wake on first event. Resident at idle: 0.

The wasm-engine's cold-start is ~3 seconds (vs ~30s for a fresh
container). That makes WASM/WASI workloads the *cheapest* tier of
breathable workload in the fleet.

## VII. Phasing

### Phase A (designed + scaffolded — this session)

- ✅ This design doc + `helmworks/charts/lareira-wasm-platform`
  (operator + engine + store + RBAC + ServiceMonitor + alerts).
- ✅ Optional `helmworks/charts/lareira-tatara-runtime` (lisp evaluator).
- ✅ Optional `helmworks/charts/lareira-openclaw` (attestation registry).
- ✅ `helmworks/charts/lareira-tatara-stack` (umbrella).

### Phase B (action items)

| # | Action | Where |
|---|---|---|
| 1 | Add `image` attribute to `wasm-platform`'s flake.nix (operator + engine images) | wasm-platform |
| 2 | Add `image` attribute to `tatara-lisp`'s flake (script binary in distroless) | tatara-lisp |
| 3 | Add `image` attribute to `openclaw-artifact-registry`'s flake | openclaw-artifact-registry |
| 4 | Publish all three to ghcr.io/pleme-io via the fleet publisher | helmworks (or whatever ships) |
| 5 | Land `clusters/rio/infrastructure/wasm-platform/` HelmRelease (suspend: true) | pleme-io/k8s |

### Phase C (next iteration)

- Convert the `pleme-storage-elastic` Rust watcher to a `pvc-autoresizer.tatara`
  program. First production-grade WASM job.
- Convert one CRD controller (e.g. one of `pangea-operator`'s lighter
  reconcilers) to a WASM controller. First production-grade WASM controller.
- Wire `tatara` (the convergence computer) to dispatch jobs via the
  `wasm-operator` backend by adding `ComputeBackend::Wasm` to its
  Driver enum. Already supported by `wasm-types::ComputeBackend`.
- Author starter tatara-lisp programs in `tatara-infra` for fleet ops:
  `chart-publisher.tatara`, `flux-reconciler.tatara`,
  `fleet-attestation-sweep.tatara`.

## VIII. Why this isn't reinventing wheels

| External | Pleme-io equivalent |
|---|---|
| wasmCloud / lattice | `wasm-platform` (the operator + engine + capability model) |
| Spin / SpinKube | `wasm-engine` (Wasmtime-driven runtime) |
| Krustlet | Not needed — `wasm-engine` runs as Pod containers, not as a kubelet shim |
| OCI Wasm Artifacts (`oci://…:wasm-content`) | Native — `wasm-store` is content-addressed; openclaw is the optional attested cache |
| KEDA HTTP add-on | Reused as-is for service-shape breathability |
| operator-sdk / kube-rs | The user-side template — `wasm-operator` handles informers; users write reconciliation logic in WASM |

Each piece is best-in-class in its niche. We don't fork; we package.

## IX. See also

- [`THEORY.md` Pillar 1](THEORY.md#pillar-1-language)
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as the authoring layer
- [`BREATHABILITY.md`](BREATHABILITY.md) — fleet-wide use-causes-spin-up
- [`wasm-platform/docs/compute-theory.md`](https://github.com/pleme-io/wasm-platform/blob/main/docs/compute-theory.md) — the typed-`ComputeUnit` rationale
- [`wasm-platform/docs/infrastructure-theory.md`](https://github.com/pleme-io/wasm-platform/blob/main/docs/infrastructure-theory.md) — the 8-slice CIA frame
- [`tatara-lisp`](https://github.com/pleme-io/tatara-lisp), [`tatara`](https://github.com/pleme-io/tatara), [`openclaw-artifact-registry`](https://github.com/pleme-io/openclaw-artifact-registry) — upstream repos
