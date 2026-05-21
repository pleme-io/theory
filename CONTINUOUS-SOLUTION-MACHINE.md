# The Viggy Method — Continuous Solution Machine

# Kubernetes as the universal fixed-point operator for typed business state

> **The Viggy Method** is the pleme-io default development mode for
> runtime. Name etymology: from Portuguese *vigência* — "in force /
> currently valid" — the legal/contractual sense of a promise that is
> *currently and continuously enforced*. Every typed promessa runs
> under Viggy: declared once, observed every tick, drift classified,
> remediation typed, attestation chained, indefinitely, with
> cryptographic receipts.
>
> Software is a declaration, a proof, a rendering, a convergence, and a
> **continuously-renewed attestation**. The cluster IS the business
> algebra; controllers are the operators; the reconciliation log IS
> the audit trail. Every business concern reducible to `(desired,
> observed, action-to-close-gap)` becomes a typed CRD plus a Rust
> controller. The substrate carries the load; the operator declares;
> promises become continuously-attested theorems.
>
> The new typed surface is tiny. Almost everything is composition of
> primitives we already standardize on. **Viggy is the fleet-side peer
> of Blackmatter Development (SDLC) and Constructive Substrate
> Engineering (typescape)**: same compounding discipline, applied to
> the running cluster.

Companion docs:

- [`THEORY.md`](./THEORY.md) — unified frame; §I.1 (Five Beats), §IV.7
  (this theorem), §VIII.9 (fractal pattern)
- [`CONVERGENCE-SUBSTRATE.md`](./CONVERGENCE-SUBSTRATE.md) —
  universal `Reconciler` engine (pangea-operator, 23 declarative APIs)
- [`LISP-YAML-CONTROLLERS.md`](./LISP-YAML-CONTROLLERS.md) — canonical
  controller authoring shape (tatara-lisp + YAML + WASM)
- [`ENGENHO.md`](./ENGENHO.md) — Rust-native K8s; the principal runtime
- [`MAGMA.md`](./MAGMA.md) — Rust-native IaC executor; the cloud half
- [`SAGUAO.md`](./SAGUAO.md) — identity peer controller layer
- [`SHIGOTO.md`](./SHIGOTO.md) — typed job system; each controller's
  internal Dag
- [`docs/cofre.md`](../docs/cofre.md) — typed secret materialization

---

## Part I — The Frame

### I.1 The thesis

Kubernetes is not a container orchestrator. Kubernetes is the canonical
realization of a **universal fixed-point operator** for typed business
state. The reconciliation loop is the universal pattern: a controller
runs `(desired, observed) → action-to-close-gap` forever, the action
changes the observed state, the next tick observes the new state, the
system **continuously solves** the problem of "make observed match
declared."

Any business concern reducible to that shape — uptime, cost,
compliance, customer KPIs, security posture — is a controller in
disguise. Today most of these live in dashboards, runbooks,
spreadsheets, on-call rotations. They belong on the cluster as typed
CRDs with controllers reconciling them indefinitely. **The cluster IS
the business algebra.** Each typed CRD adds a class of business
problems the cluster now solves forever. The operator's job is to
**declare**; the cluster's job is to **continuously solve**.

This widens Pillar 7 (Kubernetes control) from "deploys workloads" to
"runs the continuous-solution machine" and extends Pillar 12
(generation over composition) from build-time to runtime. No new
pillars; existing twelve hold; their interpretation widens.

### I.2 The Five Beats

THEORY.md §I.1 names software as a four-beat cycle (Declare / Prove /
Render / Converge). This document makes explicit the **fifth beat**
that was implicit:

1. **Declare** desired state in types.
2. **Prove** the declaration is well-formed.
3. **Render** the declaration into an executable artifact.
4. **Converge** the running world toward declared state.
5. **Attest continuously** that convergence is being maintained.

All five are constructive, checkpointed, typed. Beats 1–3 run once per
declaration change. Beat 4 begins at deploy. Beat 5 is **proof-of-life**
of beat 4 — a typed receipt every reconcile tick, BLAKE3-chained,
Ed25519-signed. Beat 5 distinguishes a *deployed* platform (once-correct)
from a *running* platform (continuously-correct, with cryptographic
receipts).

### I.3 Two equivalent formalizations

The continuous-solution machine has two mathematically equivalent
views. pleme-io commits to both interchangeably:

| | Fixed-point view | State-machine view |
|---|---|---|
| Frame | `f(x_n, d) → x_{n+1}` converges to `x*` (Banach + Knaster-Tarski) | typed state machine with **time as a dimension**; trajectory through state space; **eventually consistent** |
| Operator-ergonomic for | "will this converge?" / "what's the proof?" | "what state are we in now?" / "how did we get here?" |
| State | abstract operator argument | typed tuple `(PromessaSpec, Snapshot, Drift, Severity, Decision, ActionOutcome)` at tick T — *the OutcomeReceipt IS the state* |
| Transition | one fixed-point iteration | typed `TypedAction` applied (or `AnomalyEmission` emitted) |
| Solution | the fixed point `x*` | the **trajectory** = the entire OutcomeChain |
| Accepting state | `f(x*, d) = x*` | `severity = Cosmetic` ∧ `decision = Noop` |
| Sub-state-machines | nested operators | EscalationLadder sub-trajectory recorded in AnomalyChain |

Both views are typed, checkpointed, replayable. Operators reasoning
about a promessa switch fluently between them.

### I.4 The three commitments

The Five Beats produce three commitments. They are the **default
operating mode** of pleme-io going forward:

1. **Every reducible operational concern is a controller.** Runbooks,
   manual remediations, cron-wrapped bash, dashboard-only monitoring
   with human triage are anti-patterns. The ★★ CONTINUOUS CONVERGENCE
   directive (Part VII.1) enforces this.

2. **Every operational promise is a typed value with continuously-renewed
   attestation.** SLAs, cost budgets, compliance postures, customer
   KPIs, security postures are typed `(defpromessa …)` values; the
   OutcomeChain is the proof of holding. The ★★ PROVABLE OUTCOMES
   directive (Part VII.2) enforces this.

3. **Every anomaly is a typed event with typed remediation.** Drift,
   SLO breaches, posture regressions are typed `AnomalyEmission`s
   routed through declared `RemediationPolicy` values (NoOp / Alert /
   AutoCorrect / RequireApproval / Escalate). Anomaly response is
   reconciliation, not improvisation.

The framework is universal. **Exceptions are typed waivers, not
informal carve-outs** (Part IV.3).

---

## Part II — The Pattern + The Substrate That Already Carries It

### II.1 The Seven-Beat Convergence Tick

The runtime micro-loop every PromessaController executes:

```
Observe → Diff → Classify → Decide → Act → Attest → Tick
```

Implemented as a **seven-node shigoto Dag** per controller (depth 7,
budgets per-beat). Pure functions: Diff, Classify, Decide. I/O effects:
Observe, Act, Attest. The tick repeats indefinitely at the spec's
declared `reconcile_every` interval.

This is the *body* of the eight-phase universal loop's eighth phase
(RECONVERGE) executed as the steady-state mode of operation. The first
seven phases (DECLARE through VERIFY) run once at admission; the
seven-beat tick is what runs forever.

Trait laws (peer of CONVERGENCE-SUBSTRATE §II.2): Observe is
referentially transparent; Diff/Classify/Decide are pure and
deterministic; Act is idempotent on `Noop`; Attest is deterministic up
to signature. All five property-tested.

### II.2 The fractal controller pattern

The seven-beat shape is universal. pleme-io has the same shape at
multiple layers. Kubernetes is the **principal** instance; everything
else is a **peer** with the same shape at a different state space:

| Layer | Desired | Observed | Diff | Act |
|---|---|---|---|---|
| **engenho (workloads)** | Deployment / StatefulSet spec | Pod state via kube-watch | Reconciler diff | kube-apply |
| **magma (cloud IaC)** | Pangea slice | Cloud provider state | magma-plan | magma-apply |
| **pangea-operator (universal)** | Any declarative API config | Provider read_state | `Reconciler::compute_plan` | `Reconciler::apply` |
| **FluxCD (GitOps)** | Git tree | Cluster state | Kustomize diff | kube-apply |
| **caixa (SDLC)** | Caixa Servico spec | HelmRelease + pods | caixa-mesh diff | helmrelease upsert |
| **saguao (identity)** | `(defcrachá …)` | Authentik + RBAC | crachá-controller diff | passaporte + vigia upsert |
| **cofre (secrets)** | `SecretRef` | Backend secret state | cofre diff | backend write |
| **mado (terminal UI)** | engawa render graph + UI spec | Frame buffer + input | engawa diff | wgpu draw call |
| **blackmatter (workstation)** | HM module config | activation result | nix-darwin diff | switch-to-configuration |
| **tatara-reconciler** | tatara service spec via NATS | service observed state | tatara-reconciler diff | service-action emit |
| **LLM-as-controller** | conversation goal | conversation + tool results | model inference | tool call |
| **promessa** (new) | `(defpromessa …)` | shinryu observation | promessa-diff (severity-classified) | act via principal-layer controller |

Same shape, different state spaces. mado / blackmatter / agents are
not strictly Kubernetes — but they share the seven-beat shape; the
substrate discipline transfers. promessa is a **new peer** sitting
*above* all the others, declaring outcomes that compose across layers.

The fractal pattern gives the operator a **uniform mental model**:
every system in pleme-io is, in steady state, executing the seven-beat
tick on some typed desired state.

### II.3 The substrate we already have

Compact survey — one line per primitive. Everything below is a *given*
the continuous-solution layer consumes:

| Primitive | What it provides | Doc |
|---|---|---|
| **engenho** | Rust-native K8s; CRDs `#[derive(KubeResource, TataraDomain)]` | ENGENHO.md |
| **magma** | Rust-native IaC executor (Terraform-compatible) | MAGMA.md |
| **pangea-operator** | Universal `Reconciler` engine (23 declarative APIs catalogued) | CONVERGENCE-SUBSTRATE.md |
| **FluxCD** | GitOps reconciliation; `k8s/clusters/<cluster>/` IS the source of truth | THEORY.md §VII.2 |
| **caixa** | Typed SDLC primitive; per-Servico `:limits`/`:behavior`/`:upgrade-from` | CAIXA-SDLC.md |
| **saguao** | Fleet identity / authz / portal as peer controller layer | SAGUAO.md |
| **cofre** | Typed secret materialization across Akeyless / SOPS | docs/cofre.md |
| **tameshi** + sekiban + kensa + inshou + kanshi | Three-pillar attestation, admission gate, compliance engine, Nix integrity, eBPF runtime verifier | THEORY.md §V.3 |
| **shinryu** | Universal observability SQL surface (logs, metrics, traces unified) | system MCP |
| **shigoto** | Typed job-system primitive; every controller's internal Dag | SHIGOTO.md |
| **LISP-YAML-CONTROLLERS** | Canonical controller authoring shape (tatara-lisp + YAML + WASM) | LISP-YAML-CONTROLLERS.md |
| **TataraDomain** | Six-line contract for new typed authoring surfaces | tatara/docs/rust-lisp.md |
| **arch-synthesizer** | The typescape; every primitive type + proof | THEORY.md §III.1 |
| **denshin** | Typed NATS surface; the cross-controller event bus | denshin |
| **slack-forge** | Typed alert sinks | slack-forge |

**The continuous-solution layer reimplements none of these.** Every
new primitive in Part III is either a typed slot shape on top of an
existing surface or a thin new leaf type extending an existing chain.

### II.4 The three gaps

What this substrate **does not yet have**:

1. **A typed primitive for business outcomes.** Caixa declares
   workloads. Pangea declares cloud resources. Saguao declares
   identity. Cofre declares secrets. Nothing declares *outcomes* —
   SLAs, cost budgets, compliance postures, customer KPIs, security
   postures — as typed values with reconciling semantics. **promessa**
   closes this gap.

2. **A typed primitive for cross-controller anomalies.** magma-drift
   classifies cloud drift internally; nothing classifies cross-controller
   anomalies uniformly or routes them through typed RemediationPolicies
   + EscalationLadders. **anomalia** closes this gap.

3. **A continuous-outcome attestation chain.** tameshi attests
   *artifacts* (static three-pillar BLAKE3 + Ed25519). Nothing attests
   *held outcomes* across time. **OutcomeChain** (and its peer
   **AnomalyChain**) close this gap.

### II.5 The meshing matrix — what's new vs. exploited

The single most important table in this document. Each new primitive's
**new typed surface is tiny**; the **borrowed substrate is huge**:

| New primitive | Exploits (borrowed verbatim) | New (≤ a few hundred LoC) |
|---|---|---|
| **promessa** | TataraDomain (authoring), typescape (validation), caixa-publish (rendering), FluxCD (admission), engenho (controller registration via kube-rs informer), sekiban (admission gate) | The typed slot shape — scope, target, observation source, remediation policy, escalation ladder, dependencies |
| **anomalia** | TataraDomain, magma-drift Severity enum (re-exported), denshin (NATS subject typing), saguao audit log shape | The typed emission shape + the routing decision tree (RemediationPolicy variants) |
| **PromessaController trait** | `Reconciler` trait (universal action dispatch), shigoto Job DAG (the seven beats), kube-rs informer, tameshi signing pipeline | The trait's specialized Observe/Diff/Classify/Decide structure + the five trait laws |
| **AnomalyController** | denshin subscribe, RemediationPolicy applier, EscalationLadder runner, sekiban policy integration | The routing logic + a sliding-bloom dedup |
| **RemediationPolicy** | magma-drift's `AutoCorrect`/`Alert`/`RequireApproval` (re-exported), `Reconciler::apply` dispatch, FluxCD commits (via `GithubRepoReconciler`), crachá/cofre/sekiban patches | Adds `Escalate(ladder)` + `Compose(Vec<…>)` variants |
| **EscalationLadder** | PagerDuty integration, shikumi typed schemas, retry-policy patterns | Typed step sequence + timeout-promotion logic |
| **OutcomeChain** | tameshi's BLAKE3 + Ed25519 + HeartbeatChain pipeline (verbatim), cofre key materialization, MinIO receipt-sink discipline (tameshi's existing pattern) | The leaf type (per-tick reconciliation receipt) + the BLAKE3-of-spec genesis |
| **AnomalyChain** | Same pipeline as OutcomeChain | The leaf type (per-anomaly receipt) + dedup behavior |
| **PromessaLattice** | The lattice trait shape from compliance lattice (THEORY §III.3) | meet/join/leq specialized for promessa kinds |
| **Seven-Beat Tick** | shigoto Job DAG (just 7 typed Jobs in a line) | The seven-named-beat structure + the trait laws over it |
| **`kensa verify outcome-chain`** | kensa CLI infrastructure, tameshi chain walker, compliance projection | A new subcommand that walks the OutcomeChain |
| **MCP tools** | convergence-controller's MCP server | A tool set (`promessa.*`, `anomaly.*`) |

**Total new LoC budget: ~5,000 lines across 6 new crates.** Borrowed
substrate accounts for >95% of the implementation. This is what
"maximally leveraging generation tools of all kinds" looks like in
practice.

### II.6 What to exploit aggressively

Concrete reuse hooks to dig into immediately:

- **engenho's CRD machinery** — PromessaCR, AnomalyCR, PromessaDependencyCR
  are just kinds; controllers are kube-rs informers + the seven-beat
  impl. No new admission engine.
- **`Reconciler::apply` dispatch** — every action is either a
  `FluxCommit` (via `GithubRepoReconciler`) or one of the 22 other
  catalogued reconcilers. No new I/O code paths.
- **tameshi's signing + chain pipeline** — verbatim reuse with new
  leaf types. The chain genesis is `blake3(canonical(PromessaSpec))`.
- **shigoto Scheduler** — the seven-beat tick IS a Job DAG of depth 7.
  Budgets/retries/gates inherited; nothing new on the scheduling side.
- **cofre signing-key materialization** — PromessaControllers sign via
  per-cluster cofre-resolved Ed25519 key. Zero new secret discipline.
- **shinryu queries** — every observation is a named-param SQL query
  resolving to a typed `Snapshot`. No bespoke metrics scraping.
- **LISP-YAML-CONTROLLERS authoring** — `(defpromessa …)` source +
  rendered YAML CR + Rust controller is the same tier-2-or-tier-3 shape
  (LISP-YAML-CONTROLLERS §V.2–V.3) we already use for DNS reconcilers
  etc.
- **caixa's `:limits`** — observable via shinryu; SLA promessas cite
  them as floors. The mesh is bidirectional.
- **saguao's audit log** — observation source for security promessas
  (who accessed what cross-cluster).
- **compliance lattice** — Compliance PromessaController consumes the
  existing meet/join machinery; doesn't reimplement.
- **arch-synthesizer typescape** — PromessaSpec is just another typed
  value in the typescape; validation pipeline unchanged.

The pattern: **every reuse hook is a place we ship faster.** The new
typed surface is bounded to what genuinely doesn't exist yet.

---

## Part III — The New Typed Primitives

### III.1 promessa — typed business outcome

A **promessa** (Portuguese for "promise") is a typed value declaring a
business-level outcome the cluster must continuously hold. Authored as
`(defpromessa …)`; rendered to a YAML CR; reconciled by a
PromessaController; continuously attested via OutcomeChain.

```lisp
(defpromessa lilitu-prod-sla
  :namespace        lilitu-prod
  :scope            (services [api-v1 api-v2 api-v3])
  :target           (sla :availability "99.99%" :latency-p99 "200ms")
  :window           "30d"
  :observation      (shinryu :query "..." :materialize SlaSnapshot)
  :reconcile-every  "60s"
  :remediation      { :on-cosmetic   alert
                      :on-functional (auto-correct :action (flux-commit ...))
                      :on-critical   escalate }
  :escalation       (ladder
                      :step1 (controller :timeout "5m")
                      :step2 (on-call    :timeout "15m" :who pagerduty)
                      :step3 (manager    :timeout "1h"))
  :dependencies     [{ :to lilitu-prod-cost-budget :op meet }])
```

Rust shape (key signatures only; full enum variants in `promessa-types/`):

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
#[tatara(keyword = "defpromessa")]
pub struct PromessaSpec {
    pub name:           String,
    pub namespace:      String,
    pub scope:          PromessaScope,
    pub target:         PromessaTarget,   // 5 kinds; see below
    pub window:         Duration,
    pub observation:    ObservationSource,
    pub reconcile_every: Duration,
    pub remediation:    RemediationPolicyMap,
    pub escalation:     EscalationLadder,
    pub dependencies:   Vec<PromessaDependency>,
}

pub enum PromessaTarget {
    Sla(SlaTarget),                 // availability, latency-pXX, error-rate
    CostBudget(CostBudgetTarget),   // max-spend, period, on-overspend
    Compliance(ComplianceTarget),   // baseline, controls, on-violation
    CustomerKpi(CustomerKpiTarget), // metric, minimum, data-source
    Security(SecurityTarget),       // max-cve-age per severity, banned-packages
    Custom(CustomTarget),
}

pub enum ObservationSource {
    Shinryu          { query: String, bindings: …, materialize: TypeName },
    KubeApi          { gvr: …, selector: …, projection: JsonPath },
    HttpEndpoint     { url: Url, auth: AuthBinding, parse_as: ContentType },
    TameshiHeartbeat { chain_id: …, layer_type: LayerType },
    CompositeOf(Vec<ObservationSource>),
}
```

**Five canonical kinds:**

| Kind | Declares | Observed via | Acts via |
|---|---|---|---|
| **SLA** | availability / latency / error-rate over rolling window | shinryu over metrics + traces | FluxCD scale-up commit; Istio traffic-shift; alert |
| **CostBudget** | max spend per period per scope | shinryu over CUR + billing exporter | FluxCD scale-down commit; magma right-size; non-critical eviction |
| **Compliance** | continuous compliance against baseline | kensa projection + sekiban admission + tameshi chain | Reconciler apply; crachá quarantine; sekiban policy patch |
| **CustomerKpi** | KPI minimum (NPS, CSAT, retention) | HttpEndpoint to analytics pipeline | feature-flag rollback via FluxCD; PRD-level approval queue |
| **Security** | max CVE age per severity; banned packages; runtime posture | shinryu over CVE feed + kanshi eBPF + image-scan results | FluxCD image-pin; crachá quarantine; sekiban deploy block |

Adding a kind = `#[register_target_controller(target_kind = "…")]` on
a `TargetController` impl. Six lines of ceremony.

### III.2 anomalia + RemediationPolicy + EscalationLadder

An **anomalia** is a typed value emitted by any controller, describing
an observed deviation. AnomalyController routes it.

```rust
pub struct AnomalyEmission {
    pub source:    EmittedBy,          // PromessaName / ReconcilerKind / PeerControllerId
    pub kind:      AnomalyKind,        // DriftDetected / SloBreached / PostureRegression / BudgetExceeded / CveDetected / Custom
    pub severity:  Severity,           // Cosmetic / Functional / Critical (re-exports magma-drift)
    pub context:   serde_json::Value,  // typed per kind (schema in source's spec)
}

pub enum RemediationPolicy {
    NoOp,                                          // emit receipt only
    Alert           { sink: AlertSinkRef },        // ntfy / PagerDuty / slack-forge
    AutoCorrect     { action: TypedAction },       // apply via Reconciler/FluxCommit/magma/cofre/crachá
    RequireApproval { approver_group: ApproverGroupRef },  // GitHub Issue; block
    Escalate        { ladder: EscalationLadderRef },        // typed step sequence
    Compose(Vec<RemediationPolicy>),
}

pub enum TypedAction {
    FluxCommit        { path: PathBuf, patch: JsonPatch },   // preferred per ★★ GITOPS-NATIVE
    ReconcilerApply   { kind: String, config: Value },
    MagmaApply        { workspace: WorkspaceRef, plan_id: BlakeHash },
    CofreRotate       { secret_ref: SecretRef },
    CrachaPatch       { policy: AccessPolicyRef, patch: JsonPatch },
    Custom            { kind: String, payload: Value },
}

pub struct EscalationLadder {
    pub name:  String,
    pub steps: Vec<EscalationStep>,    // Controller / OnCall / Manager / Exec / Custom with typed timeouts
}
```

The AnomalyController is **one instance per cluster** (single-replica
for ordering), subscribed to `anomaly.<cluster>.>` via denshin.
Repeated emissions of the same anomaly dedupe via sliding bloom;
Critical never dedupes.

### III.3 OutcomeChain + AnomalyChain

**OutcomeChain** is a typed, BLAKE3-chained, Ed25519-signed sequence
of `OutcomeReceipt`s, one per PromessaController reconcile tick.
**Peer of tameshi's HeartbeatChain** — same pipeline, different leaf
type:

| | HeartbeatChain (tameshi) | OutcomeChain (this doc) |
|---|---|---|
| Leaf | static construction event | reconciliation tick |
| Cadence | per build/deploy/admission | per spec-declared interval |
| Proves | "this artifact was built from these inputs" | "this promessa was held across this window" |

```rust
pub struct OutcomeReceipt {
    pub promessa_name:    String,
    pub spec_hash:        BlakeHash,    // BLAKE3 of canonical PromessaSpec
    pub tick_index:       u64,
    pub tick_ts:          DateTime<Utc>,
    pub snapshot:         Snapshot,     // typed observed-state
    pub drift:            Drift,        // typed gap (may be Empty)
    pub severity:         Severity,
    pub decision:         Decision,
    pub outcome:          ActionOutcome,
    pub prev_receipt_hash: BlakeHash,   // chain link
    pub receipt_hash:      BlakeHash,   // BLAKE3 of canonical bytes
    pub signature:         Ed25519Sig,
    pub signer:            SignerId,
}
```

The chain genesis is `prev_receipt_hash = blake3(canonical(PromessaSpec))`
— the chain has a typed origin tying tick 1 to the spec it was declared
under. Mutating any receipt breaks the chain; replay verifies the
entire history.

**AnomalyChain** is the per-cluster peer for anomaly emissions and
their resolutions. Same pipeline, leaf type carries `(emission,
resolved_policy, outcome, escalation_path)`. Post-incident reviews
become `kensa replay anomaly-chain --window <incident>`.

`kensa` learns two new verbs:

```
kensa verify outcome-chain --promessa <name> --window <…> --baseline <…>
  → typed OutcomeVerificationReport (auditor's deliverable)
kensa replay anomaly-chain --cluster <…> --window <incident>
  → typed PostIncidentReport
```

Both reports are typed Rust values — never PDFs. The chain IS the
proof.

### III.4 PromessaController + AnomalyController traits

```rust
#[async_trait]
pub trait PromessaController: Send + Sync {
    fn kind(&self) -> &'static str;

    async fn observe(&self, spec: &PromessaSpec) -> Result<Snapshot, …>;
    fn diff(&self, spec: &PromessaSpec, observed: &Snapshot) -> Drift;
    fn classify(&self, drift: &Drift) -> Severity;
    fn decide(&self, spec: &PromessaSpec, sev: Severity, drift: &Drift) -> Decision;
    async fn act(&self, decision: &Decision) -> Result<ActionOutcome, …>;
    async fn attest(&self, …) -> Result<OutcomeReceipt, …>;

    /// The seven-beat tick. Default impl provided.
    async fn tick(&self, spec: &PromessaSpec) -> Result<TickResult, …> {
        let observed = self.observe(spec).await?;
        let drift    = self.diff(spec, &observed);
        let severity = self.classify(&drift);
        let decision = self.decide(spec, severity, &drift);
        let outcome  = self.act(&decision).await?;
        let receipt  = self.attest(spec, &observed, &drift, severity, &decision, &outcome).await?;
        Ok(TickResult { observed, drift, severity, decision, outcome, receipt })
    }
}
```

**Five trait laws** (all property-tested via proptest):

1. `observe` is referentially transparent modulo external mutation.
2. `diff`, `classify`, `decide` are pure and deterministic.
3. `act` is idempotent on `Decision::Noop`.
4. `tick` is convergent: post-act diff yields Noop or strictly smaller
   magnitude.
5. `attest` is deterministic up to signature bytes.

`AnomalyController` mirrors the shape: `receive → resolve_policy →
apply_policy → attest`. Single-replica per cluster.

### III.5 PromessaLattice + PromessaDependency

Cross-promessa composition mirrors the compliance lattice (THEORY §III.3):

```rust
pub trait PromessaLatticeOp {
    fn meet(a: &PromessaSpec, b: &PromessaSpec) -> Result<PromessaSpec, …>;
    fn join(a: &PromessaSpec, b: &PromessaSpec) -> Result<PromessaSpec, …>;
    fn leq(a: &PromessaSpec, b: &PromessaSpec) -> bool;
}

pub struct PromessaDependency {
    pub to:      PromessaName,
    pub op:      DependencyOp,    // Meet | Join | Observe
    pub timeout: Duration,
}
```

Meet/join defined only for *same-kind* promessas (SLA-meet-SLA,
Compliance-meet-Compliance). Cross-kind composition is via
`Meet`/`Join` dependency edges, enforced by the per-cluster
`PromessaDependencyController`: a promessa's `act` blocks if its
`Meet` dependency is in Functional+ severity, and the blocked
controller emits `DependencyBlocked` instead of mutating state.

---

## Part IV — The New Normal

### IV.1 The cognitive shift

| Before (per-problem reasoning) | After (convergence-first reasoning) |
|---|---|
| "How do we deploy this?" | "What's the typed desired state?" |
| "How do we monitor it?" | "What's the typed ObservationSource?" |
| "What do we do when X breaks?" | "What's the typed RemediationPolicy + EscalationLadder?" |
| "Who's on-call?" | "Which EscalationLadder step fires; AnomalyController routes." |
| "When's the next compliance review?" | "Continuous — `kensa verify outcome-chain` is the review." |
| "What's our SLA story?" | "Read the SLA promessa's OutcomeChain over the requested window." |
| "Was this incident our fault?" | "Walk the AnomalyChain via `kensa replay`." |
| "How do we report to the auditor?" | "Hand the auditor the public key + `kensa verify outcome-chain`." |

The shift: from **imperative reasoning** ("what do we do?") to
**declarative reasoning** ("what's the typed answer the cluster is
holding right now?"). The operational questions are answered by
reconciliation chains; the operator's role collapses to authoring
declarations.

### IV.2 What becomes the open discussion

**No longer open** (these were one-time decisions; the answer is now
canonical):

- "Should we adopt continuous convergence?" — adopted.
- "Should this be a runbook or a controller?" — controller.
- "Should we track this in a dashboard or as a promessa?" — promessa.
- "Manual or auto-remediation?" — declared in the policy at authoring
  time.
- "How will we report to compliance?" — `kensa verify outcome-chain`.

**Newly open** (the substantive discussion topics going forward):

- "What's the typed desired state for this outcome?"
- "What's the right ObservationSource?"
- "How do we classify Severity for this drift kind?"
- "What's the cheapest auto-correct that preserves correctness?"
- "Where in the EscalationLadder does this need to escalate?"
- "Are these two promessas in a Meet or Join lattice relation?"
- "Should this be federated across clusters?"
- "Do we need a new PromessaTarget kind?"
- "Is this work a typed exception to the framework, and what's the
  typed reason?"

The intellectual surface contracts around the *substrate* and expands
around the *declarations*. Less time on plumbing; more time refining
the typed shape of what we want held.

### IV.3 Typed exception markers

Exceptions are typed waivers, not informal carve-outs. Two new markers
join the existing set (`skip-blackmatter:`, `skip-shigoto:`,
`skip-saguao:`, etc.):

- `skip-continuous-convergence: <typed-reason>` — no operational
  concern reducible to `(desired, observed, action)`.
- `skip-provable-outcomes: <typed-reason>` — no operational promise to
  attest.

**Acceptable typed reasons:** "pure library; no runtime state to
reconcile" / "build-time tool; no persistent state" / "third-party
vendor SDK; we don't own the operational surface" / "generated-code
repo; the surface lives downstream" / "pre-substrate-bootstrap;
required to install the substrate itself."

**Not acceptable:** "we don't have time" / "this is legacy" / "the
customer doesn't need it" / "we'll do it later." Per Operating
Principle #0 (path of least resistance is a cardinal sin): if a
workload could plausibly be a controller, it is a controller. The
conversation is about *what kind*, not *whether*.

### IV.4 The operator's three verbs

Under the new normal, the operator has exactly three verbs:

1. **Declare** — author a new promessa, anomalia, or substrate
   extension.
2. **Audit** — query chains, verify outcomes, replay anomalies.
3. **Override** — typed break-glass when reconciliation cannot
   resolve; document; **promote to a substrate improvement.**

The first is creative (extends the algebra). The second is analytical
(reads the algebra). The third is exceptional (and signals a missing
typed primitive — every override is data toward a substrate gap).
Everything else the operator used to do — deploying, scaling,
patching, monitoring, alerting, rotating, right-sizing — is
*reconciliation*, and reconciliation is the cluster's job.

---

## Part V — Worked Example: SLA Promessa Traced Through the Full Pipeline

One canonical worked example. The point is to show **every existing
primitive the new promessa exploits**, not to enumerate features.

### V.1 The declaration

```lisp
(defpromessa lilitu-prod-sla
  :namespace        lilitu-prod
  :scope            (services [api-v1 api-v2 api-v3])
  :target           (sla :availability "99.99%" :latency-p99 "200ms")
  :window           "30d"
  :observation      (shinryu
                      :query "SELECT avg(availability), quantile(latency_ms, 0.99)
                              FROM events WHERE service IN (?services)
                                AND ts > now() - INTERVAL '30 days'"
                      :bindings { :services [api-v1 api-v2 api-v3] }
                      :materialize SlaSnapshot)
  :reconcile-every  "60s"
  :remediation      { :on-cosmetic   alert
                      :on-functional (auto-correct
                                       :action (flux-commit
                                                 :path "k8s/clusters/lilitu/apps/api/values.yaml"
                                                 :patch (json-patch :replicaCount (+ current 1))))
                      :on-critical   escalate }
  :escalation       (ladder
                      :step1 (controller :timeout "5m")
                      :step2 (on-call    :timeout "15m" :who lilitu-prod-pager))
  :dependencies     [{ :to lilitu-prod-cost-budget :op meet }])
```

### V.2 What's exploited at each stage

| Stage | What carries the load |
|---|---|
| **Author** | tatara-lisp + `#[derive(TataraDomain)]` (Pillar 1) |
| **Validate** | arch-synthesizer typescape + Dry::Struct-equivalent proptests (Pillar 6) |
| **Render** | caixa-publish + repo-forge boilerplate; YAML CR generated, committed under `k8s/clusters/lilitu/promessas/` (Pillar 9 + ★★ GITOPS-NATIVE) |
| **Admit** | FluxCD applies; sekiban Phase 2 signature check (Pillar 7 + Pillar 10) |
| **Register** | engenho kube-rs informer + `#[register_target_controller(target_kind = "sla")]` |
| **Tick** | shigoto 7-node Dag; budgets/retries/gates inherited |
| **Observe** | shinryu SQL with typed bindings → `SlaSnapshot` materialization |
| **Diff** | `SlaController::diff` pure; produces typed `SlaDrift` with `availability_delta`, `p99_delta` |
| **Classify** | `SlaController::classify`: p99 > 1.5× target = Functional; availability < target − 0.01% = Critical |
| **Decide** | Look up `:remediation` policy; produce `Decision::AutoCorrect(FluxCommit{…})` |
| **Dependency check** | PromessaDependencyController consulted: is `lilitu-prod-cost-budget` in Cosmetic+? If yes proceed; else emit `DependencyBlocked` anomaly |
| **Act** | `TypedAction::FluxCommit` dispatches to `GithubRepoReconciler` (one of the 23 catalogued reconcilers); commit lands; FluxCD applies; engenho rolls pods |
| **Attest** | OutcomeReceipt built; signed via cofre-resolved Ed25519 key; chained to prev; written to per-cluster MinIO; published via denshin to subscribed sinks (audit bucket, Datadog as alongside-sink, compliance UI) |
| **Tick again** | 60s later |
| **Audit (later)** | `kensa verify outcome-chain --promessa lilitu-prod-sla --window 30d --baseline pci-dss-4.0` walks the chain and emits typed `OutcomeVerificationReport` |
| **Postmortem (if escalated)** | `kensa replay anomaly-chain --window <incident>` walks the AnomalyChain, renders typed `PostIncidentReport` |

**The novel code in this entire flow is the SlaController's
diff/classify methods (~150 LoC) and the typed slot shapes
(~300 LoC).** Everything else is `Reconciler::apply`,
`shigoto::Scheduler::tick`, `tameshi::sign`, `cofre::resolve`,
`shinryu::query`, `engenho::admit`, `kensa::verify`, `FluxCD apply` —
all already shipping.

### V.3 Tick trace (after a load spike)

```
T+00:00  observe → SlaSnapshot { availability=0.9991, p99=242ms }
         diff    → SlaDrift { p99_delta=+42ms }
         classify→ Functional
         decide  → AutoCorrect(FluxCommit { replicaCount: 5 → 6 })
         dep_chk → cost-budget at Cosmetic; proceed
         act     → commit pushed; FluxCD reconciling
         attest  → OutcomeReceipt { tick=1247, sev=Functional, dec=AutoCorrect, out=Committed }

T+01:00  observe → SlaSnapshot { availability=0.9994, p99=215ms }
         diff    → SlaDrift { p99_delta=+15ms }
         classify→ Cosmetic
         decide  → Alert
         attest  → OutcomeReceipt { tick=1248, sev=Cosmetic, dec=Alert }

T+05:00  observe → SlaSnapshot { availability=0.9999, p99=187ms }
         diff    → Empty; classify→ Cosmetic; decide→ Noop
         attest  → OutcomeReceipt { tick=1252, sev=Cosmetic, dec=Noop }
```

Five ticks, ~5 minutes, auto-corrected breach, full attested record.
No human involved. The OutcomeChain is the entire story.

---

## Part VI — Anti-Patterns

| Anti-pattern | Continuous-solution-machine replacement |
|---|---|
| Runbook PDFs / Notion docs | Controller running the seven-beat tick |
| Cron-wrapped bash (cleanup, rotation, right-sizing) | Promessa with `Custom` target + scheduled action |
| Manual `kubectl scale` for load | SLA promessa auto-correct |
| Manual cloud-resource right-sizing | CostBudget promessa auto-correct |
| Quarterly compliance review via screenshots | Compliance promessa + `kensa verify outcome-chain` |
| SLA tracked only in a Datadog dashboard | SLA promessa with OutcomeChain (Datadog is a sink, not system of record) |
| Cost in spreadsheets | CostBudget promessa with OutcomeChain |
| NPS only in Mixpanel | CustomerKpi promessa with OutcomeChain |
| CVE response via manual triage | Security promessa with auto-correct image-pin |
| On-call humans triaging every alert | AnomalyController routing per declared policy |
| Postmortems in Notion | `kensa replay anomaly-chain` → typed PostIncidentReport |
| "We'll fix it next sprint" for ongoing concerns | Promessa with declared remediation — fix is reconciliation |
| Compliance attestation as a one-time deploy artifact | Continuous attestation via OutcomeChain |
| Slack incident channels as the audit trail | AnomalyChain as audit trail; Slack is a sink |
| Imperative `kubectl edit` | Per ★★ GITOPS-NATIVE: commit; controller reconciles |
| Contractual SLAs with no enforcement | Promessa CR with the SLO as `:target` |

The common property: every anti-pattern is **uncontrolled, untyped,
unattested, or human-dependent**. The replacement makes it controlled,
typed, attested, and human-independent unless the declared policy
explicitly says otherwise.

---

## Part VII — The Two New ★★ Directives

These land in `pleme-io/CLAUDE.md` (full text). The summary here is
the canonical reference.

### VII.1 ★★ CONTINUOUS CONVERGENCE — controllers, not runbooks

Every operational concern reducible to `(desired-state, observed-state,
action-to-close-gap)` is a typed CRD plus a Rust controller on
engenho/magma/pangea-operator. Runbooks, cron-bash, dashboard-only
monitoring, on-call triage of routine events are anti-patterns.

This is Pillar 12 extended to runtime: hand-execution is the fallback;
generated controllers running indefinitely is the default. Pillar 7
widens from "deploys workloads" to "runs the continuous-solution
machine."

**Operates above ★★ GITOPS-NATIVE.** GitOps-native governs *how*
state mutates (commits, not commands); continuous-convergence governs
*whether the substrate ever asks a human to mutate state at all*.

Per-repo waiver: `skip-continuous-convergence: <typed-reason>`.

### VII.2 ★★ PROVABLE OUTCOMES — every promise is a continuously-attested theorem

Every operational promise is a typed `(defpromessa …)` reconciled by a
PromessaController and continuously attested via OutcomeChain.
Promises are typed values the cluster proves it is holding tick by
tick — never human assertions, dashboard configurations, or spreadsheet
entries.

Compliance audits, SLA reports, incident postmortems, posture reviews
become **queries against the OutcomeChain and AnomalyChain**. The
substrate is the system of record; everything else is a view.

This is THEORY.md §I.1's **fifth beat** operationalized: the knowable
platform's static proofs (build-time) join continuously-renewed
dynamic proofs (runtime).

Per-repo waiver: `skip-provable-outcomes: <typed-reason>`.

### VII.3 Sharpening of THEORY §I.1 — Five Beats

THEORY.md §I.1 sharpens from four beats to five. The fifth — continuous
attestation — was implicit; now explicit. The sharpening is canonical;
this document is its source.

---

## Part VIII — Implementation Roadmap

Bounded; total ~5,000 new LoC across 6 crates. Most work is
composition; novel code is the typed slot shapes + diff/classify methods
per kind + the trait-laws proptest harness.

| M | Scope | Acceptance |
|---|---|---|
| **M0** | `promessa-types` crate; PromessaSpec + 5 PromessaTarget kinds + ObservationSource + RemediationPolicy + EscalationLadder + PromessaDependency; `#[derive(TataraDomain)]`; spec round-trip proptest | `(defpromessa …)` parses to PromessaSpec; round-trips YAML ↔ SExpr byte-identically |
| **M1** | `engenho-promessa-controllers`; PromessaController trait + seven-beat default impl + SlaController + shinryu ObservationSource resolver + FluxCommit TypedAction; trait_laws proptest | SLA promessa admitted, controller ticks, OutcomeReceipt stream visible |
| **M2** | `anomalia-types` + `engenho-anomaly-controller`; AlertSinkRef resolver; GitHub Issue approval queue; EscalationLadder firing | SlaController emits Critical; AnomalyController routes to PagerDuty; ladder fires on timeout |
| **M3** | Remaining 4 TargetControllers (CostBudget, Compliance, CustomerKpi, Security); per-kind trait_laws | Each kind ships a reference promessa; all tick without errors |
| **M4** | `outcome-chain` + `anomaly-chain` crates; cofre-key signing; MinIO sink; `kensa verify outcome-chain` + `kensa replay …` subcommands; sekiban admission rule | `kensa verify outcome-chain --baseline pci-dss-4.0` returns typed report |
| **M5** | `promessa-lattice` + `PromessaDependencyController`; `Meet`/`Join` blocking | SLA + cost-budget meet — SLA refuses scale-up when budget breached |
| **M6** | `promessa-federation`; cross-cluster denshin; shinryu federated queries | Global SLA promessa observes federated metrics, coordinates executors |
| **M7** | MCP tool set (`promessa.*`, `anomaly.*`); replay UI (Yew, like varanda); compliance dashboard generation | Operator drives every primitive move from MCP |

Roadmap order maps to the three commitments (I.4): M1 + M3 produce
controllers (commitment 1); M4 produces the attestation chain
(commitment 2); M2 produces typed anomaly routing (commitment 3).

---

## Part IX — Relationship to Existing Canon

This document **adds no new pillar**. It widens Pillar 7 and extends
Pillar 12 to runtime. The relationship:

| Existing canon | Relationship |
|---|---|
| THEORY.md §I.1 (four beats) | **Sharpened** to five beats; fifth was implicit, now explicit |
| THEORY.md §IV.2 (controllers as fixed-point operators) | **Specialized** by PromessaController + AnomalyController |
| THEORY.md §IV.3 (eight-phase loop) | Phase 8 (RECONVERGE) is the **steady-state mode**; first 7 phases run once at admission |
| THEORY.md §V.1 (knowable platform) | **Extended to runtime** — static proofs joined by continuously-renewed dynamic proofs |
| THEORY.md §V.3 (three-pillar attestation) | **Joined by OutcomeChain** — same BLAKE3+Ed25519 pipeline, dynamic leaf type |
| THEORY.md §V.4 (two-phase signature) | **Promoted to continuous** — Phase 2 signatures refresh every tick |
| THEORY.md §VIII.8 ("every business is a convergence declaration") | **Made literal** — every business outcome is a promessa |
| CONVERGENCE-SUBSTRATE.md | **Layer below** — `Reconciler` trait is what PromessaControllers dispatch to |
| LISP-YAML-CONTROLLERS.md | **Authoring pattern reused** — `(defpromessa …)` source + YAML CR + Rust controller |
| ENGENHO.md | **Substrate** — PromessaCR, AnomalyCR live as typed CRDs |
| MAGMA.md | **Cloud action engine** — actions mutating cloud state route through magma |
| SAGUAO.md | **Identity peer** — security promessas observe and patch crachá |
| docs/cofre.md | **Secret peer** + signing-key materialization |
| tameshi / sekiban / kensa | **Attestation extended** — OutcomeChain peers HeartbeatChain; kensa gains chain verbs |
| SHIGOTO.md | **Controller internals** — the seven-beat tick IS a shigoto Dag |
| compliance lattice (THEORY §III.3) | **Consumed by Compliance promessas; mirrored by PromessaLattice** |
| Pillar 7 (Kubernetes control) | **Widened** to "the continuous-solution machine" |
| Pillar 12 (generation over composition) | **Extended to runtime** — generation governs build-time; continuous-convergence governs runtime |
| ★★ GITOPS-NATIVE directive | **Cited from new directives** — Act prefers FluxCommit |
| ★★ TYPED EMISSION directive | **Cited** — every serialization through typed AST; no `format!()` |
| ★★ Shigoto directive | **Consumed** — the seven-beat tick is a shigoto Dag |

**No duplication. Pure consumption + extension.** The continuous-solution
machine sits as a layer that consumes every existing primitive and
closes the gap between the static substrate and continuous business
operation.

---

## Part X — Closing — Viggy

Pleme-io is a software system, not a program. It compounds its ability,
quality, and value over time. **The Viggy Method is that compounding
applied to runtime.** Every new typed CRD + controller widens the
substrate's continuous problem-solving algebra; every new OutcomeChain
widens the attestation surface; every new RemediationPolicy +
EscalationLadder widens automated-response.

Promises become continuously-attested theorems. SLAs, cost budgets,
compliance postures, customer KPIs, security postures — each is a
typed value the cluster proves it is holding, tick by tick,
indefinitely, with cryptographic receipts. **The substrate IS the
system of record; everything else is a view.**

The author's responsibility is the declaration. The substrate's
responsibility is everything else. **We solve once; the cluster solves
forever.**

```
╔═══════════════════════════════════════════════════════════════════════╗
║                                                                       ║
║   T H E   V I G G Y   M E T H O D                                     ║
║   the cluster as continuous solution machine                          ║
║                                                                       ║
║   The cluster is the algebra; controllers are the operators;          ║
║   promessa is the typed promise; OutcomeChain is the continuous proof;║
║   anomalia is the typed anomaly; AnomalyChain is the response record. ║
║                                                                       ║
║   The operator declares — the cluster continuously solves.            ║
║                                                                       ║
╚═══════════════════════════════════════════════════════════════════════╝
```

---

> Rust owns types. Lisp owns flow. shikumi owns config. forge-gen owns
> APIs. Pangea owns infrastructure. arch-synthesizer owns the
> typescape. Nix owns images. Helm + Kustomize + FluxCD own deploys.
> substrate owns the SDLC. tameshi owns the static proof chain. JIT
> Infrastructure owns compute. **The Viggy Method owns the continuous
> solution — promessa declares the typed outcome, OutcomeChain attests
> it tick by tick, anomalia + AnomalyChain type the response.**
