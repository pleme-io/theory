# VIGGY-LEGOS — the atomic state-imposition substrate

> **Thesis.** A solution under the Viggy Method is not a monolithic
> program — it is a *composition of atomic legos that bind together*
> through typed seams to make Kubernetes (and every peer controller
> layer) **continuously chase a declared business outcome over time**.
> Every lego is typed, swappable, composable, attested. The substrate's
> job is to make the right legos available; the operator's job is to
> declare which legos compose; the cluster's job is to continuously
> impose the composed state on the running world.
>
> This document inventories the legos we have, names the legs that
> compose them into a complete state-imposition pipeline, audits which
> legs are fully standardized vs. drafted vs. paper-only, and identifies
> the remaining gaps as substrate-improvement targets.

Companion docs:

- [`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md) — the Viggy framework spec
- [`VIGGY-AUTHORING.md`](./VIGGY-AUTHORING.md) — the typed authoring discipline
- [`CONVERGENCE-SUBSTRATE.md`](./CONVERGENCE-SUBSTRATE.md) — the universal Reconciler engine
- [`PANGEA-OPERATOR.md`](./PANGEA-OPERATOR.md) — IaC controller for cloud resources
- [`ENGENHO.md`](./ENGENHO.md) — Rust-native K8s; principal substrate
- [`THEORY.md`](./THEORY.md) §IV.7, §VIII.9 — the frame

---

## Part I — The Frame

### I.1 What "state imposition" means under Viggy

To **impose state** on the running world is to:

1. Declare typed desired state
2. Observe current state via a typed source
3. Compute the typed diff
4. Decide on a typed action
5. Apply the action via a typed dispatcher
6. Attest the result via a typed receipt
7. Repeat indefinitely

Every step is **typed**, **checkpointed**, and **swappable** within
the layer it occupies. The substrate's responsibility is providing
the typed seams; the controllers' responsibility is filling them.

### I.2 The lego principle

A lego in this document = **one atomic, typed, swappable unit of a
state-imposition pipeline**. Three properties:

1. **Atomic** — does exactly one thing; has one trait surface
2. **Typed** — input and output are Rust types (or TataraDomain-derived
   typed values); no untyped strings, no untyped JSON blobs
3. **Swappable** — implements a known trait or pattern; can be replaced
   by another impl of the same trait without changing surrounding code

Composability follows: legos that satisfy adjacent trait seams snap
together; legos that don't share seams must be bridged by a typed
adapter — which itself is a lego.

### I.3 Why "legos" as the framing

The user's framing — "standardized flexible atomic legos and pieces" —
captures three discipline requirements that other framings miss:

- **Standardized**: trait surfaces are canonical; one shape per
  concern (per ★★ macros-everywhere).
- **Flexible**: every lego is swappable for any other impl of the same
  trait (per Pillar 12 — generation over composition).
- **Atomic**: legos can't be subdivided without losing typing
  guarantees (per Pillar 10 — proof discipline).

The legos *snap together* via typed seams. The Viggy Method is the
operator-side discipline that **specifies which legos compose** to
solve a business problem; this document is the substrate-side
discipline that **specifies what legos exist and how they snap**.

---

## Part II — The Ten Legs of State Imposition

Every Viggy state-imposition pipeline composes ten typed legs. Each
leg has a canonical trait surface + a shipping (or drafted)
implementation per layer.

### II.1 Declaration leg

**Purpose:** capture the typed desired state.

| Mechanism | Implementation | Status |
|---|---|---|
| `#[derive(TataraDomain)]` Rust struct + `(defX …)` Lisp form | tatara-lisp + tatara-derive | shipping |
| Rendered YAML CR via caixa-publish | caixa-flux / caixa-helm renderers | shipping |
| Direct YAML authoring (operator-facing override) | any text editor | shipping |

**Canonical seam:** SExpr IR → typed Rust value → BLAKE3 content-addressable.

### II.2 Admission leg

**Purpose:** gate invalid declarations at the cluster boundary.

| Mechanism | Implementation | Status |
|---|---|---|
| K8s CRD schema validation | engenho API server (or upstream kube-apiserver) | shipping |
| Phase-2 signature gate | sekiban admission webhook + tameshi BLAKE3 chain | shipping |
| Compliance pre-flight check | kensa CertificationArtifact verification | shipping |
| Nix integrity gate (pre-activation) | inshou | shipping |

**Canonical seam:** AdmissionReview request → typed validation → allow/deny.

### II.3 Registration leg

**Purpose:** wire the declared CR to its controller.

| Mechanism | Implementation | Status |
|---|---|---|
| kube-rs informer + Reconciler trait registration | tatara-reconciler, pangea-operator, caixa-operator, wasm-operator, convergence-controller, shinka, kenshi, crachá-controller | shipping |
| Engenho-native controller framework | engenho | draft (planned per ENGENHO.md) |
| WASM-runtime controller (lisp-yaml-controllers pattern) | wasm-operator + tatara-lisp-controllers | shipping |

**Canonical seam:** CR `metadata.uid` → controller instance → reconcile loop.

### II.4 Tick leg

**Purpose:** advance the controller's state machine one step.

| Mechanism | Implementation | Status |
|---|---|---|
| Seven-Beat Convergence Tick (Viggy) | **PROPOSED via shigoto Dag of 7 nodes** | **gap — Viggy M1** |
| Eight-phase Unix lifecycle (tatara-reconciler) | `phase_machine.rs` (Pending → Forking → Execing → Running → Attested → Reconverging → Exiting → Failed → Zombie → Reaped) | shipping |
| Pangea reconcile state machine | Pending → Synthesizing → Planning → Awaiting → Applying → Converged → Drifted | shipping (in pangea-operator) |
| Generic shigoto Dag of N typed Job nodes | shigoto-scheduler InProcessScheduler | shipping (consumed by tend) |

**Canonical seam:** Tick boundary → typed `TickResult` → next observation.

### II.5 Observation leg

**Purpose:** read the current state of the running world.

| Mechanism | Implementation | Status |
|---|---|---|
| **shinryu** — universal SQL over events / metrics / logs / traces | shinryu-mcp | shipping |
| K8s API watch + projection (kube-rs informer) | tatara-reconciler, all kube-watch controllers | shipping |
| External HTTP endpoint (analytics, billing, CVE feeds) | per-promessa `HttpEndpoint` ObservationSource | **gap — Viggy M3** |
| tameshi HeartbeatChain leaf reads | tameshi consumer crate | shipping |
| Pangea tofu plan output (drift snapshot) | pangea-operator | shipping |
| CompositeOf (composition of multiple sources) | **proposed in Viggy ObservationSource enum** | **gap — Viggy M1+** |

**Canonical seam:** ObservationSource → typed Snapshot value.

### II.6 Diff / Classify / Decide leg

**Purpose:** pure computation from (spec, snapshot) to (severity,
decision).

| Mechanism | Implementation | Status |
|---|---|---|
| TargetController per-kind diff/classify/decide (Viggy) | **PROPOSED** | **gap — Viggy M1–M3** |
| magma-drift typed drift classification (cosmetic/functional/critical) | magma-converge | shipping |
| pangea-operator drift detection + reactive policy | pangea-operator (per-resource policy engine, settling tracker) | shipping |
| Custom controller-specific diff loops | each in-cluster controller | shipping (heterogeneous) |

**Canonical seam:** `diff(Spec, Snapshot) → Drift` (pure) → `classify(Drift) → Severity` (pure) → `decide(Spec, Severity, Drift) → Decision` (pure).

### II.7 Action leg (TypedAction dispatch)

**Purpose:** apply the typed decision; mutate the running world.

| TypedAction | Backing Controller / Reconciler | Status |
|---|---|---|
| `FluxCommit { path, patch }` | **GithubRepoReconciler** (CONVERGENCE-SUBSTRATE) + FluxCD source/kustomize/helm controllers | shipping |
| `ReconcilerApply { kind, config }` | One of 23 catalogued Reconcilers via pangea-operator universal engine | 4 shipping (terraform / kv / github / dns) / 19 planned |
| `MagmaApply { workspace, plan_id }` | magma (terraform protocol v5/v6 + state v4) | shipping (in pangea-operator); standalone magma draft |
| `CofreRotate { secret_ref }` | cofre CLI | shipping (CLI only; controller-side **gap**) |
| `CrachaPatch { policy, patch }` | crachá-controller | draft (Phase 5) |
| `Custom { kind, payload }` | Registered substrate extension | escape hatch only |

**Canonical seam:** TypedAction → I/O effect → typed ActionOutcome.

### II.8 Attestation leg

**Purpose:** seal the result with a typed receipt.

| Mechanism | Implementation | Status |
|---|---|---|
| Static three-pillar attestation (artifact ⊕ control ⊕ intent → BLAKE3 → Ed25519) | tameshi HeartbeatChain | shipping |
| Phase-1 signature (origin only) | tameshi-derive on rendered artifacts | shipping |
| Phase-2 signature (compliance-verified, admission-gated) | sekiban + kensa pipeline | shipping |
| **OutcomeChain leaf (per-tick reconciliation receipt)** | **PROPOSED — peer of HeartbeatChain** | **gap — Viggy M4** |
| **AnomalyChain leaf (per-anomaly response receipt)** | **PROPOSED — peer of HeartbeatChain** | **gap — Viggy M4** |
| Cycle receipts (pangea-operator per-reconcile receipts) | pangea-operator status.lastCycle | shipping |

**Canonical seam:** receipt struct → canonical bytes → BLAKE3 → Ed25519 → chain.

### II.9 Anomaly leg

**Purpose:** route typed anomalies to typed remediations.

| Mechanism | Implementation | Status |
|---|---|---|
| Typed AnomalyEmission via denshin NATS subject | **PROPOSED** | **gap — Viggy M2** |
| Typed AnomalyController (single-replica per cluster) | **PROPOSED** | **gap — Viggy M2** |
| RemediationPolicy enum (NoOp / Alert / AutoCorrect / RequireApproval / Escalate / Compose) | **PROPOSED** | **gap — Viggy M2** |
| EscalationLadder (Controller → OnCall → Manager → Exec) | **PROPOSED** | **gap — Viggy M2** |
| Alertmanager + AlertmanagerConfig | upstream + helmworks observability | shipping (untyped); to be wrapped |
| PagerDuty integration | TBD; Viggy uses as a sink | shipping (external) |

**Canonical seam:** `AnomalyEmission` → RemediationPolicy → TypedAction or EscalationLadder.

### II.10 Audit leg

**Purpose:** query the attestation chains for proof.

| Mechanism | Implementation | Status |
|---|---|---|
| `kensa verify <static-chain>` | kensa | shipping |
| `kensa verify outcome-chain --baseline <…>` | **PROPOSED — new kensa verb** | **gap — Viggy M4** |
| `kensa replay anomaly-chain --window <…>` | **PROPOSED — new kensa verb** | **gap — Viggy M4** |
| `promessa get / history` MCP | **PROPOSED** | **gap — Viggy M7** |
| `anomaly history / escalations` MCP | **PROPOSED** | **gap — Viggy M7** |
| Replay UI under varanda | **PROPOSED** | **gap — Viggy M7** |

**Canonical seam:** chain walker → typed report value (`OutcomeVerificationReport`, `PostIncidentReport`).

---

## Part III — The Controller Universe

Every controller in pleme-io that runs a continuous reconcile loop.
Layer = the substrate stratum it occupies; Strategy per THEORY.md
§IV.2.

### III.1 Layer 1 — In-cluster controllers

| Controller | Repo | What it reconciles | Strategy | Trigger | Output | Status |
|---|---|---|---|---|---|---|
| **FluxCD** (source / kustomize / helm controllers) | upstream | Git → cluster state via HelmRelease / Kustomization / GitRepository / OCIRepository | Declarative | git commit + interval | applied K8s resources | shipping (canonical) |
| **tatara-reconciler** | `tatara/tatara-reconciler` | `Process` CRs through 8-phase Unix lifecycle | DiffAndPatch | kube-watch | emitted Flux Kustomization / HelmRelease CRs + BLAKE3 attestations | shipping |
| **tatara-pool-reconciler** | `tatara/tatara-pool-reconciler` | `EphemeralPool` + `EphemeralAllocation` (warm pre-attested pods) | EventDriven | kube-watch (Deployment image change) | allocated Process CRs ready for requestors | shipping |
| **wasm-operator** | `wasm-operator/` | `ComputeUnit` (wasm.pleme.io) → Job/CronJob/Deployment + KEDA ScaledObject | Declarative | kube-watch | K8s workloads + KEDA scaling | shipping |
| **pangea-operator** | `pangea-operator/` | `InfrastructureTemplate` / `InfrastructureFlow` / `ComplianceBinding` / `ComplianceSchedule` CRs → tofu plan/apply | FullRebuild | kube-watch + interval | Terraform state + signed cycle receipts + reactive policy events | shipping |
| **convergence-controller** | `convergence-controller/` | `ConvergenceProcess` CRDs → cluster lifecycle (full business provisioning) | FullRebuild | kube-watch + signals | cloud resources + cluster GitOps repos + Grafana access | shipping |
| **caixa-operator** | `caixa/caixa-operator` | `Caixa`, `Lacre`, `CaixaBuild` CRs → rendered charts + OCI images + mesh policies + FluxCD manifests | Declarative | kube-watch | OCI images + `lareira-<nome>` Helm charts + Cilium NPs + Gateway/HTTPRoute | shipping |
| **shinka** | `shinka/` | `DatabaseMigration` CRDs → migration Job orchestration with CloudNativePG health checks | EventDriven | kube-watch (Deployment image change) | K8s Job + status transitions | shipping |
| **kenshi** | `kenshi/` | `BuildPipeline`, `TestGate`, `ScheduledBuild`, `DriftWatcher`, `DeploymentPipeline` CRDs → Nix build pipelines, test gates, promotions | EventDriven | webhook + cron + kube-watch | OCI images + test reports + promotion commits + Discord notifications | shipping |
| **crachá-controller** | `cracha/cracha-controller` | `AccessPolicy` CRDs → in-memory authz index for saguão fleet identity | Declarative | kube-watch | gRPC `Authorize` API + REST `/accessible-services` | draft (Phase 5) |
| **vigia** | `vigia/` | nginx forward-auth subrequest → OIDC JWT validation + crachá `Authorize()` | EventDriven (per request) | webhook | 200 allow / 401 redirect / 403 deny + 5m cache | scaffold (Phase 7) |
| **sekiban** | `sekiban/` | K8s API mutations via ValidatingAdmissionWebhook → verify OCI digests, SignatureGate, Certification, CompliancePolicy CRDs | Declarative | webhook (api-server admission) | admission allow/deny + attestation annotation enforcement | shipping |
| **kanshi** | `kanshi/` | kernel `execve`/`open`/`mmap`/`mprotect` syscalls → BLAKE3 hash verification against BPF allow map | Declarative (kernel LSM) | kernel hooks | EPERM on unauthorized execution + HeartbeatChain audit | shipping |
| **kensa** | `kensa/` | Plugin registry (kernel/nix/oci/k8s) → NIST 800-53 + OSCAL assessment | Declarative | compile-time / on-demand | OSCAL 1.1.2 assessment JSON | shipping |
| **inshou** | `inshou/` | Nix-built closure + CertificationArtifact → pre-rebuild gate | Declarative | pre-rebuild hook | exit 0 allow / exit 1 reject | shipping |
| **tameshi-watch** | `tameshi-watch/` | RSS/NVD/GitHub CVE feeds → NATS event emission | Daemon | RSS/API poll | CVE feed to NATS for subscribers | shipping |
| **convergence-signal** | `convergence-signal/` | Event stream (git, NATS, webhooks) → signal dispatch | EventDriven | external signals | convergence process triggers | shipping |

### III.2 Layer 2 — Out-of-cluster reconcilers (universal `Reconciler` engine)

Per CONVERGENCE-SUBSTRATE.md §III. The pangea-operator is the engine
that hosts these.

| Reconciler kind | What it reconciles | Status |
|---|---|---|
| **TerraformReconciler** | wraps magma-plan + magma-apply | shipping |
| **InMemoryKvReconciler** | KV store (mock + reference impl) | shipping |
| **GithubRepoReconciler** | repo settings + branch protection (the FluxCommit backend) | shipping |
| **DnsRecordReconciler** | DNS records (provider-agnostic) | shipping |
| VaultPolicyReconciler | Vault/Akeyless policies + roles | planned |
| HelmReleaseReconciler | Helm chart + values vs deployed | planned |
| K8sNativeReconciler | any K8s kind via kube-rs informer | planned |
| SlackChannelReconciler | channels + members + integrations | planned |
| DatadogMonitorReconciler | monitors + SLOs + dashboards | planned |
| GrafanaDashboardReconciler | dashboards + folders + alert rules | planned |
| PgSchemaReconciler | extensions + roles + grants + schemas | planned |
| StripeProductReconciler | products + prices + webhooks | planned |
| Auth0AppReconciler | applications + groups + rules | planned |
| IstioReconciler | service mesh routing | planned |
| KongReconciler | API gateway routes + plugins | planned |
| PagerDutyScheduleReconciler | schedules + escalation policies | planned |
| LinearProjectReconciler | projects + teams + fields | planned |
| ArgoWorkflowReconciler | declarative DAG workflows | planned |
| NixFileReconciler | declarative filesystem | planned |
| BrowserStateReconciler | cookies + localStorage (E2E infra) | planned |
| CronReconciler | scheduled tasks owned by the operator | planned |
| EmailRuleReconciler | mailbox filters / forwarding | planned |
| MusicLibraryReconciler | Spotify / last.fm (for fun) | planned |

### III.3 Layer 3 — Renderers / transpilers (build-time, not steady-state)

These don't reconcile; they emit. Listed for completeness; they
*compose with* controllers.

| Renderer | Repo | Emits |
|---|---|---|
| caixa-helm | `caixa/caixa-helm` | Servico → `lareira-<nome>` Helm chart |
| caixa-flux | `caixa/caixa-flux` | Servico → GitRepository/HelmRelease/Kustomization CRs |
| caixa-mesh | `caixa/caixa-mesh` | Aplicacao → Cilium NetworkPolicies + Gateway/HTTPRoute |
| compliance-forge | `compliance-forge/` | Pangea spec → compliance controls |
| crossplane-forge | `crossplane-forge/` | provider OpenAPI → Crossplane provider repo |
| iac-forge | `iac-forge/` | TOML resource specs → IacResource IR → backend impl |
| forge-gen | `forge-gen/` | OpenAPI → SDKs + MCP + IaC + completions + docs |
| repo-forge | `repo-forge/` | archetype → repo scaffold |
| inspec-rspec | `inspec-rspec/` | InSpec profile → RSpec suite |

### III.4 Layer 4 — Chart deployment (helmworks)

`helmworks/` is the deployment surface for every shipping controller.
The `lareira-*` umbrella pattern wraps a controller's bundle into a
deployable Helm chart, consuming the `pleme-lib` library for
ServiceMonitor / PrometheusRule / mandatory observability per
Pillar 11.

Notable `lareira-*` umbrellas:

- `lareira-convergence-controller` — convergence-controller chart
- `lareira-pangea-operator` — pangea-operator chart (per §VIII of PANGEA-OPERATOR.md)
- `lareira-fleet-attestation-sweep` — fleet-wide attestation sweep
- `lareira-fleet-programs` — fleet program catalog
- `lareira-cartorio` / `lareira-lacre` — compliant-artifact-provability stack
- `lareira-dns-reconciler` — DNS reconciler chart (consumes the Layer-2 Reconciler)
- `lareira-keda` / `lareira-keda-http` — KEDA scaling
- `lareira-kyverno` / `lareira-falco` / `lareira-kubescape` — admission + runtime security
- `lareira-cert-manager` / `lareira-external-secrets` — supporting infra
- `lareira-ingress-nginx` — ingress
- Application charts: `lareira-immich`, `lareira-jellyfin`, `lareira-home-assistant`, etc.

**The pattern:** each chart depends on `pleme-lib`; each is rendered
by caixa-helm OR hand-authored under the helmworks library
convention; each is FluxCD-deployed via a HelmRelease in
`k8s/clusters/<cluster>/`.

### III.5 Layer 5 — Crossplane providers

Per BLACKMATTER.md and the iac-forge generation chain:

| Provider | Status |
|---|---|
| crossplane-akeyless | shipping |
| crossplane-datadog | drafted |
| crossplane-splunk | drafted |
| crossplane-aws (typescape projection) | partial — see iac-forge sync |
| crossplane-cloudflare | drafted |
| crossplane-gcp | drafted |
| crossplane-azure | drafted |
| crossplane-hcloud | drafted |
| crossplane-kubernetes | drafted |

All 9 providers are typed-emitted from `iac-forge --backend crossplane`.
None duplicates work in any other layer; they're an alternative
*delivery* of the same typed Reconciler model (Crossplane being
upstream's reconciler engine).

### III.6 Layer 6 — Signals + observation

| Primitive | Role |
|---|---|
| **shinryu** | Universal SQL surface over events / metrics / logs / traces |
| **convergence-signal** | Event broadcaster (git commits / NATS / webhooks → convergence-controller signals) |
| **denshin** | Typed NATS surface (cross-controller event bus) |
| **tameshi HeartbeatChain** | Static attestation event stream |
| **OutcomeChain** (Viggy) | **PROPOSED** — continuous attestation event stream |
| **AnomalyChain** (Viggy) | **PROPOSED** — continuous anomaly response stream |

### III.7 Layer 7 — Peer controllers (fractal pattern)

Same seven-beat shape, different state spaces (per THEORY.md §VIII.9):

| Peer | Where it runs | What it reconciles |
|---|---|---|
| **mado** | operator laptop | engawa render graph + UI spec → wgpu draw calls |
| **blackmatter** | operator workstation | HM module config → switch-to-configuration activation |
| **engenho-local** | operator laptop (planned) | local engenho mini-cluster state |
| **LLM-as-controller** | MCP session | conversation goal → tool calls |
| **tatara-reconciler (laptop variant)** | operator-side tatara daemon (per ENGENHO §788) | local Process CRDs |

### III.8 Layer 8 — Viggy business-outcome (NEW)

The new typed surface that sits above Layers 1–7:

| Primitive | Role | Status |
|---|---|---|
| **promessa** | Typed business-outcome declaration | **gap — Viggy M0** |
| **anomalia** | Typed anomaly emission | **gap — Viggy M2** |
| **PromessaController** | Runs Seven-Beat Convergence Tick on engenho | **gap — Viggy M1** |
| **AnomalyController** | Routes anomalies via RemediationPolicy | **gap — Viggy M2** |
| **PromessaDependencyController** | Enforces Meet/Join/Observe edges | **gap — Viggy M5** |
| **PromessaLivenessController** | Detects wedged PromessaControllers | **gap — Viggy M2 or M3** |
| **TargetController** (per kind) | Specialized diff/classify/decide per kind | **gap — Viggy M1–M3** |
| **OutcomeChain pipeline** | BLAKE3+Ed25519 per-tick receipts | **gap — Viggy M4** |
| **AnomalyChain pipeline** | per-emission receipts | **gap — Viggy M4** |

---

## Part IV — The TypedAction Dispatch Matrix

The center of the substrate-binding story. **Which TypedAction routes
to which controller / reconciler:**

```
Promessa.act() → TypedAction
                    │
        ┌───────────┼───────────┬──────────┬──────────┬──────────┐
        ▼           ▼           ▼          ▼          ▼          ▼
   FluxCommit  Reconciler   MagmaApply  CofreRotate CrachaPatch Custom
        │       Apply           │           │          │          │
        ▼           ▼           ▼           ▼          ▼          ▼
  GithubRepo  pangea-      pangea-       cofre     crachá-    registered
  Reconciler  operator     operator      CLI →     controller substrate
   → commit   (one of 23   (terraform     backend   → patch    extension
   to k8s/    catalogued)  provider)      write     AccessPolicy
   clusters/      │                                       CR
        │         ▼
        ▼     for each kind:
   FluxCD     terraform / kv / github / dns (shipping)
   reconciles vault / helm / k8s-native / slack / datadog /
        ▼     grafana / pg-schema / stripe / auth0 / istio /
   engenho    kong / pagerduty / linear / argo-workflow /
   applies    nix-file / browser-state / cron / email-rule /
              music (planned)
```

**Key insight: every typed action terminates in either FluxCD,
pangea-operator's universal Reconciler engine, cofre, or
crachá-controller.** The dispatch graph is finite, typed, and
exhaustive.

---

## Part V — Standardization Audit

### V.1 Fully standardized (shipping, canonical, used in production)

| Lego | Standardized as |
|---|---|
| TataraDomain derive + `(defX …)` | one shape per typed domain (★★ macros-everywhere) |
| FluxCD as commit/apply | ★★ GITOPS-NATIVE; one HelmRelease/Kustomization CRD shape |
| tameshi BLAKE3+Ed25519 chain | one signing pipeline; cofre key materialization |
| sekiban admission gate | one ValidatingAdmissionWebhook shape |
| helmworks `lareira-*` umbrella + `pleme-lib` library | every controller's deployment manifest |
| shigoto Dag for work graphs | ★★ Shigoto directive |
| shinryu for observation | canonical SQL surface |
| caixa Servico + caixa-helm/flux/mesh renderers | ★★ Caixa canonical SDLC primitive |
| pangea-operator + InfrastructureTemplate CR | canonical IaC controller |
| tatara-reconciler + Process CRD | canonical Unix-process cluster lifecycle |
| Reconciler trait (CONVERGENCE-SUBSTRATE §II.1) | universal action dispatch surface |
| iac-forge + 9 Crossplane providers | typed delivery alternative |
| Pillar 11 alert layer (mandatory ServiceMonitor + PrometheusRule via pleme-lib) | every chart |
| repo-forge archetypes | every repo scaffold |

### V.2 Partially standardized (drafted, in progress)

| Lego | Status |
|---|---|
| crachá-controller (saguao authz) | Phase 5 |
| vigia forward-auth (saguao data plane) | Phase 7 scaffold |
| engenho Rust-native K8s | M0–M5 roadmap |
| magma standalone (extract from pangea-operator) | M0–M5 roadmap |
| 19 remaining Reconciler kinds (CONVERGENCE-SUBSTRATE §III.2) | planned; each is ~30 LOC + thin chart |
| cofre on-cluster controller (cofre is CLI-only today) | future — would enable typed action dispatch from cluster |
| OutcomeChain + AnomalyChain (Viggy attestation peers) | M4 roadmap |
| Promessa* + Anomaly* controllers (Viggy core) | M0–M2 roadmap |

### V.3 Identified gaps (substrate-improvement tickets)

These are the **substrate gaps** that survive after applying the
directive-derived auto-answer principle (VIGGY-AUTHORING.md §15.7).
Each is a substrate-improvement ticket waiting to be filed.

1. **PromessaController + TargetController kinds.** The center of
   Viggy. M1 of the roadmap. ~5,000 LoC across 6 crates per
   CONTINUOUS-SOLUTION-MACHINE.md §XIV.
2. **AnomalyController + denshin subjects.** The cluster-wide anomaly
   router. M2.
3. **OutcomeChain + AnomalyChain pipelines.** Peer of HeartbeatChain
   with new leaf types. M4.
4. **The 19 pending Reconciler kinds.** Each enables a new
   `ReconcilerApply` target for Viggy `TypedAction`. Drip-feed:
   prioritize Vault (security promessas), Helm (composition with
   other Viggy promessas), K8sNative (catch-all), Datadog/Grafana
   (observability backend).
5. **cofre on-cluster controller.** Today cofre is a CLI; Viggy
   `CofreRotate` TypedAction would benefit from a cofre-controller
   running on engenho.
6. **engenho native API server.** Per ENGENHO.md M4 (CNCF
   Certified Kubernetes Software Conformance on v1.34 zero skips).
   The principal substrate.
7. **PromessaLattice meet/join evaluator.** Same-kind lattice ops as
   typed views; M5.
8. **PromessaDependencyController.** Cross-kind Meet/Join/Observe
   enforcement; M5.
9. **kensa `verify outcome-chain` + `replay anomaly-chain` verbs.**
   The audit surface; M4.
10. **MCP tool set** (`promessa.*`, `anomaly.*`). Operator surface;
    M7.

---

## Part VI — Composition Rules — How Legos Snap Together

### VI.1 The reuse-first hierarchy (recap from VIGGY-AUTHORING §3 + §16)

When materializing a solution, prefer the highest-step option that
satisfies the typed seam:

| Step | Action | Substrate cost |
|---|---|---|
| 1 | Cite an existing typed primitive (caixa Servico, sekiban policy, kensa control, cofre rotation, saguao crachá, FluxCD HelmRelease, magma resource) | zero |
| 2 | Wrap an existing primitive with a promessa | ~5 lines YAML |
| 3 | Compose existing TargetControllers via PromessaLattice / PromessaDependency | ~10 lines YAML |
| 4 | Author a new promessa using existing TargetController kind | ~20 lines `.tatara` |
| 5 | Extend an existing TargetController (new field) | substrate change |
| 6 | Author a new TargetController kind (`Custom`) | ~150 LoC + macro |
| 7 | Author a new TypedAction variant | substrate change |

### VI.2 Per-leg substitution invariants

For each of the ten legs (Part II), the substrate guarantees:

| Leg | Invariant |
|---|---|
| Declaration | Same typed IR (SExpr) regardless of source language |
| Admission | Same Phase-2 signature gate regardless of CR kind |
| Registration | Same kube-rs informer pattern regardless of controller |
| Tick | Same `TickResult` typed output regardless of internal phase shape |
| Observation | Same `Snapshot` typed output regardless of ObservationSource variant |
| Diff/Classify/Decide | Same purity laws regardless of TargetController |
| Action | Same `ActionOutcome` typed result regardless of TypedAction variant |
| Attestation | Same BLAKE3+Ed25519 pipeline regardless of receipt type |
| Anomaly | Same denshin subject + RemediationPolicy resolution regardless of source |
| Audit | Same chain-walker shape regardless of chain type |

**Per-leg swap is a one-implementation change with no surrounding
impact.** This is the lego property.

### VI.3 The full pipeline view — a solution traced through all 10 legs

Worked example: lilitu prod SLA promessa.

```
Leg 1 (Declaration):   (defpromessa lilitu-prod-sla …)  in promessas/*.tatara
                                │
Leg 2 (Admission):     sekiban verifies Phase-2 signature on PromessaCR YAML
                                │
Leg 3 (Registration):  engenho kube-rs informer registers SlaController instance
                                │
Leg 4 (Tick):          shigoto Dag (7 nodes) advances each `reconcile_every`=60s
                                │
Leg 5 (Observation):   shinryu SQL query returns SlaSnapshot
                                │
Leg 6 (Diff/Class/Dec): SlaController::diff → SlaDrift; ::classify → Functional;
                       ::decide → AutoCorrect(FluxCommit{...})
                                │
Leg 7 (Action):        FluxCommit dispatched to GithubRepoReconciler →
                       commit to k8s/clusters/lilitu/apps/api/values.yaml →
                       FluxCD source/kustomize/helm-controllers reconcile →
                       engenho applies → pods roll
                                │
Leg 8 (Attestation):   OutcomeReceipt built; tameshi pipeline signs;
                       MinIO sink writes; chain link established
                                │
Leg 9 (Anomaly):       (Cosmetic on next tick → no anomaly fired)
                                │
Leg 10 (Audit):        Operator runs `kensa verify outcome-chain --window 30d`
                       → typed OutcomeVerificationReport
```

**Every leg uses an already-shipping primitive (or, for Legs 4, 5
composite, 6, 8, 9, 10 — a planned-substrate primitive).** The new
authoring effort is ~20 lines of `.tatara`. The substrate produces
every downstream artifact.

---

## Part VII — The Next Legs Needed (Priorities)

Ordered by leverage (per Compounding Principle #1 — solve once):

1. **PromessaController + first TargetController kind (SLA).** Closes
   Legs 3, 4, 6 for the first kind. Unlocks the first real promessa.
   Estimated ~600 LoC novel code. M0+M1 of the roadmap.

2. **OutcomeChain pipeline.** Closes Leg 8. Reuses tameshi entirely;
   the new code is just the leaf type + MinIO sink wiring + chain
   genesis logic. ~300 LoC. M4 — but could land earlier as the
   attestation surface tests well in isolation.

3. **AnomalyController + denshin subject typing.** Closes Leg 9.
   ~400 LoC. M2.

4. **The remaining 4 canonical TargetController kinds** (CostBudget,
   Compliance, CustomerKpi, Security). Each is ~150 LoC + macro
   `trait_laws_obeyed!`. M3 — but split across teams; each is
   independent.

5. **VaultPolicyReconciler + HelmReleaseReconciler + K8sNativeReconciler.**
   Closes the most-used `ReconcilerApply` targets. ~30 LoC + thin
   chart each. Can land alongside any TargetController that needs
   them.

6. **kensa chain verbs** (`verify outcome-chain`, `replay
   anomaly-chain`). Closes Leg 10. ~500 LoC; reuses kensa CLI
   infrastructure. M4.

7. **PromessaDependencyController + PromessaLattice.** Closes
   cross-promessa composition. M5.

8. **MCP tool set** (`promessa.*`, `anomaly.*`). Closes the operator
   surface. M7.

---

## Part VIII — Closing — The Algebra of Legos

The substrate already provides ~85% of the legos needed for full
Viggy state imposition. The remaining 15% — the Viggy-specific layer
(PromessaController, AnomalyController, OutcomeChain, AnomalyChain,
the 5 TargetController kinds, the missing Reconcilers) — is bounded:
**~5,000 LoC across 6 new crates per CONTINUOUS-SOLUTION-MACHINE.md
§XIV**. Every other leg snaps to a primitive that ships today.

**The algebraic property:** legos that satisfy adjacent typed seams
compose without per-pair adapter code. Adding a new TypedAction
variant doesn't require touching the diff/classify/decide
implementations. Adding a new TargetController kind doesn't require
touching FluxCD or sekiban. Adding a new Reconciler doesn't require
touching the Viggy core. Each leg evolves independently.

**The state-imposition property:** Kubernetes becomes a continuous
solution machine *by composing these legos*. Every PromessaCR
admitted to engenho is a typed declaration the cluster now chases
forever. Every TypedAction dispatched is the cluster mutating the
running world toward that declaration. Every OutcomeReceipt signed
is a cryptographic proof that the cluster IS chasing it. Every
anomaly emitted is a typed reaction. Every audit chain replayed is
the substrate accounting for its own behavior across time.

**The compounding property:** each new typed lego widens the class
of business outcomes the cluster can chase. The substrate grows
monotonically; the operator declares; the cluster solves.

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║   T E N   L E G S                                                      ║
║   Declaration → Admission → Registration → Tick → Observation →       ║
║   Diff/Classify/Decide → Action → Attestation → Anomaly → Audit       ║
║                                                                       ║
║   Each typed; each swappable; each atomic.                            ║
║   The substrate snaps them together — the operator declares,          ║
║   the cluster solves.                                                 ║
║                                                                       ║
║                            V I G G Y   L E G O S                      ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

> *The cluster is the algebra. The legos are typed. The seams are
> standardized. The author writes one (defpromessa …); the substrate
> composes every leg of the imposition pipeline. **Viggy is the
> practice; legos are the alphabet.***
