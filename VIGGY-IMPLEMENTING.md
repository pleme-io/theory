# VIGGY-IMPLEMENTING — substrate-side discipline

> **Thesis.** Where [`VIGGY-AUTHORING.md`](./VIGGY-AUTHORING.md)
> disciplines the *operator* (who declares promessas), this document
> disciplines the *substrate engineer* (who authors the controllers,
> typed actions, observation sources, and macros that BACK promessas).
> Same compounding discipline; different audience.
>
> The substrate engineer's job is to extend the typed surface so that
> *the next promessa author writes ~20 lines of .tatara* and the
> substrate produces every downstream artifact. Pillar 12 (generation
> over composition) operationalized at the implementation layer.

Companion docs:

- [`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md) — the Viggy framework spec
- [`VIGGY-AUTHORING.md`](./VIGGY-AUTHORING.md) — operator-side discipline
- [`VIGGY-LEGOS.md`](./VIGGY-LEGOS.md) — the ten legs + controller universe
- [`CONVERGENCE-SUBSTRATE.md`](./CONVERGENCE-SUBSTRATE.md) — the universal Reconciler engine
- [`SHIGOTO.md`](./SHIGOTO.md) — the typed Job system the tick consumes

---

## Part I — Frame

### I.1 Who this document is for

The **substrate engineer**: anyone adding a new TargetController kind,
extending a TypedAction variant, adding an ObservationSource binding,
or refining the trait laws + macros that hold the substrate together.

**Not** for promessa authors (read VIGGY-AUTHORING.md instead).

### I.2 What you're extending vs. solving

The substrate engineer is **never** solving a specific business
problem in this document — that's the operator's job under
VIGGY-AUTHORING. The substrate engineer is **widening the algebra**
of what business problems can be declared. A new TargetController
kind opens a class of problems the cluster now solves forever; a new
TypedAction variant opens a class of state mutations declarable from
within Viggy.

Per Compounding Principle #1 (solve once, in one place): the
substrate engineer's commits are *load-bearing* — every promessa
author downstream consumes them.

### I.3 Directives the substrate engineer observes

Every substrate-side change is gated on:

| Directive | Check |
|---|---|
| **★★★ Compounding Directive** | New typed surface advances the substrate; no path-of-least-resistance shortcuts |
| **★★ The Viggy Method** | New surface respects CONTINUOUS CONVERGENCE + PROVABLE OUTCOMES |
| **★★ macros-everywhere** | Every recurring shape extracts to a macro; three-times rule |
| **★ NO SHELL** | Rust + tatara-lisp + Nix + YAML only |
| **★★ TYPED EMISSION** | `format!()` is banned; typed AST renderers |
| **★★ GITOPS-NATIVE** | Action surfaces prefer FluxCommit; never bypass typed dispatch |
| **★★ Shigoto** | Every work-graph internal to a controller IS a shigoto Dag |
| **Pillar 10** | Proof discipline: `trait_laws_obeyed!` + proptest + golden trajectories |
| **Pillar 12** | Generation over composition: extend via macros, not hand-writing |

Failure to observe any of these = the change is not substrate-grade
and should not merge.

---

## Part II — The substrate crate layout

The Viggy implementation lives across six new crates plus extensions
to four existing crates. Each is published independently and consumed
via Cargo workspaces.

| Crate | Owns | Status |
|---|---|---|
| **`promessa-types`** | `PromessaSpec`, `PromessaTarget` enum + 5 canonical kinds, `ObservationSource`, `RemediationPolicy`, `TypedAction`, `EscalationLadder`, `PromessaDependency`, `Snapshot`, `Drift`, `Severity`, `Decision`, `ActionOutcome` | new — M0 |
| **`anomalia-types`** | `AnomalySpec`, `AnomalyEmission`, `AnomalyKind`, re-exports `Severity` + `RemediationPolicy` | new — M2 |
| **`outcome-chain`** | `OutcomeReceipt`, BLAKE3 chaining, Ed25519 signing via cofre, MinIO sink | new — M4 |
| **`anomaly-chain`** | `AnomalyReceipt`, dedup bloom, chain peer | new — M4 |
| **`engenho-promessa-controllers`** | `PromessaController` trait + default tick impl, `TargetController` trait, per-kind impls (`SlaController`, `CostBudgetController`, …) | new — M1–M3 |
| **`engenho-anomaly-controller`** | `AnomalyController` (single-replica), denshin subscriber, RemediationPolicy dispatcher, EscalationLadder runner | new — M2 |
| **`tameshi`** | extended: re-exports OutcomeReceipt / AnomalyReceipt as new `LayerType` variants | extension — M4 |
| **`kensa`** | extended: `verify outcome-chain` + `replay anomaly-chain` subcommands | extension — M4 |
| **`sekiban`** | extended: PromessaCR admission rule (Phase-2 + OutcomeChain bucket present) | extension — M4 |
| **`pangea-operator`** | extended: 19 new Reconciler kinds per CONVERGENCE-SUBSTRATE §III.2 | rolling |

**Total novel LoC:** ~5,000 across the six new crates. Per
CONTINUOUS-SOLUTION-MACHINE.md §II.5: ≥95% of the implementation is
reuse of existing substrate.

---

## Part III — Authoring a new TargetController

The canonical substrate engineering task. **Six steps** end-to-end.
Worked example: `SlaController`.

### III.1 The six steps

1. Declare the typed `Target` shape via `#[derive(TataraDomain)]`
2. Declare default severity thresholds as `const` per-kind
3. Impl the `TargetController` trait (six methods, all pure except observe/act/attest)
4. Register via `#[register_target_controller(target_kind = "…")]` proc macro
5. Wire trait-laws proptest via `trait_laws_obeyed!(YourController)` macro
6. Ship the YAML CR schema + helmworks `lareira-engenho-promessa-controllers` chart entry

### III.2 Step 1 — Declare the target shape

```rust
// engenho-promessa-controllers/src/sla/target.rs

use blake3::Hash;
use chrono::Duration;
use promessa_types::Percentage;
use serde::{Deserialize, Serialize};
use tatara_lisp_derive::TataraDomain;

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, PartialEq)]
#[tatara(keyword = "sla")]
pub struct SlaTarget {
    pub availability: Percentage,
    pub latency_p99:  Duration,
    pub latency_p50:  Duration,
    pub error_rate:   Percentage,
}
```

The `#[tatara(keyword = "sla")]` attribute makes
`(sla :availability "99.99%" :latency-p99 "200ms" …)` author-able
inside the `:target` slot of `(defpromessa …)`. The proc macro
generates: SExpr round-trip, BLAKE3 canonical hashing, schema export
for sekiban admission, YAML serializer/deserializer.

**Six lines of ceremony.** Per the six-line contract (THEORY.md §II.1).

### III.3 Step 2 — Severity thresholds

```rust
// engenho-promessa-controllers/src/sla/thresholds.rs

pub const COSMETIC_LATENCY_RATIO: f64    = 1.2;  // observed / target
pub const FUNCTIONAL_LATENCY_RATIO: f64  = 1.5;
pub const CRITICAL_LATENCY_RATIO: f64    = 2.0;

pub const COSMETIC_AVAILABILITY_DELTA: f64    = 0.00001;  // within 0.001%
pub const FUNCTIONAL_AVAILABILITY_DELTA: f64  = 0.0001;   // within 0.01%
// Critical = below target

pub const COSMETIC_ERROR_RATE_RATIO: f64    = 1.5;
pub const FUNCTIONAL_ERROR_RATE_RATIO: f64  = 2.0;
pub const CRITICAL_ERROR_RATE_RATIO: f64    = 3.0;
```

These ARE the calibration per VIGGY-AUTHORING §4.1. Living in code as
typed `const` (not magic numbers) means: tests reference them, docs
auto-generate from them, overrides are explicit.

### III.4 Step 3 — The trait impl

The `TargetController` trait is the substrate engineer's primary
extension surface:

```rust
// engenho-promessa-controllers/src/lib.rs

#[async_trait::async_trait]
pub trait TargetController: Send + Sync + 'static {
    type Target: TataraDomain + Send + Sync;       // e.g., SlaTarget
    type Snapshot: Send + Sync + std::fmt::Debug;  // e.g., SlaSnapshot
    type Drift: Send + Sync + std::fmt::Debug;     // e.g., SlaDrift

    fn kind(&self) -> &'static str;

    async fn observe(&self, spec: &PromessaSpec<Self::Target>)
        -> Result<Self::Snapshot, ControllerError>;

    fn diff(&self, spec: &PromessaSpec<Self::Target>, observed: &Self::Snapshot)
        -> Self::Drift;

    fn classify(&self, drift: &Self::Drift) -> Severity;

    fn decide(&self, spec: &PromessaSpec<Self::Target>, severity: Severity, drift: &Self::Drift)
        -> Decision;

    async fn act(&self, decision: &Decision)
        -> Result<ActionOutcome, ControllerError>;

    async fn attest(&self, /* … */)
        -> Result<OutcomeReceipt, ControllerError>;
}
```

The `PromessaController` trait (in `promessa-types`) wraps any
`TargetController`:

```rust
pub struct PromessaController<TC: TargetController> {
    target_controller: TC,
    chain: OutcomeChain,
    scheduler: shigoto::Scheduler,
}

impl<TC: TargetController> PromessaController<TC> {
    /// The seven-beat tick. Default impl provided.
    pub async fn tick(&self, spec: &PromessaSpec<TC::Target>) -> Result<TickResult, ControllerError> {
        let observed = self.target_controller.observe(spec).await?;
        let drift    = self.target_controller.diff(spec, &observed);
        let severity = self.target_controller.classify(&drift);
        let decision = self.target_controller.decide(spec, severity, &drift);
        let outcome  = self.target_controller.act(&decision).await?;
        let receipt  = self.target_controller.attest(spec, &observed, &drift, severity, &decision, &outcome).await?;
        self.chain.append(receipt.clone()).await?;
        Ok(TickResult { observed, drift, severity, decision, outcome, receipt })
    }
}
```

**The substrate engineer implements `TargetController`, not
`PromessaController`.** The tick logic lives once in the substrate.

### III.5 Step 4 — Register via proc macro

```rust
// engenho-promessa-controllers/src/sla/mod.rs

#[register_target_controller(target_kind = "sla")]
pub struct SlaController {
    shinryu: ShinryuClient,
    flux_repo: GithubRepoReconciler,
}

#[async_trait::async_trait]
impl TargetController for SlaController {
    type Target   = SlaTarget;
    type Snapshot = SlaSnapshot;
    type Drift    = SlaDrift;

    fn kind(&self) -> &'static str { "sla" }

    async fn observe(&self, spec: &PromessaSpec<SlaTarget>) -> Result<SlaSnapshot, ControllerError> {
        let observation = spec.observation_source.as_shinryu()?;
        self.shinryu.query(&observation.query, &observation.bindings).await
            .map(|rows| SlaSnapshot::from_rows(rows))
    }

    fn diff(&self, spec: &PromessaSpec<SlaTarget>, observed: &SlaSnapshot) -> SlaDrift {
        SlaDrift {
            availability_delta: observed.availability - spec.target.availability,
            p99_ratio: observed.latency_p99.as_secs_f64() / spec.target.latency_p99.as_secs_f64(),
            p50_ratio: observed.latency_p50.as_secs_f64() / spec.target.latency_p50.as_secs_f64(),
            error_rate_ratio: observed.error_rate / spec.target.error_rate,
        }
    }

    fn classify(&self, drift: &SlaDrift) -> Severity {
        if drift.availability_delta < -COSMETIC_AVAILABILITY_DELTA { return Severity::Critical; }
        if drift.p99_ratio >= CRITICAL_LATENCY_RATIO { return Severity::Critical; }
        if drift.p99_ratio >= FUNCTIONAL_LATENCY_RATIO { return Severity::Functional; }
        if drift.p99_ratio >= COSMETIC_LATENCY_RATIO { return Severity::Cosmetic; }
        Severity::Cosmetic
    }

    fn decide(&self, spec: &PromessaSpec<SlaTarget>, severity: Severity, drift: &SlaDrift) -> Decision {
        let policy = spec.remediation.for_severity(severity);
        Decision::from_policy(policy, drift)
    }

    async fn act(&self, decision: &Decision) -> Result<ActionOutcome, ControllerError> {
        match &decision.action {
            TypedAction::FluxCommit { path, patch } => {
                self.flux_repo.compute_plan(&CommitConfig { path, patch })?;
                self.flux_repo.apply(&plan).await
            }
            // …other variants…
        }
    }

    async fn attest(&self, /* … */) -> Result<OutcomeReceipt, ControllerError> {
        OutcomeReceipt::build(/* … */).sign(self.cofre_key())
    }
}
```

The `#[register_target_controller]` macro registers the controller in
a static `TargetControllerRegistry` keyed by `target_kind`. When a
PromessaCR with `target.kind == "sla"` is admitted, engenho's
informer looks up `SlaController` in the registry and wires it.

### III.6 Step 5 — Trait-laws proptest harness

```rust
// engenho-promessa-controllers/tests/sla_trait_laws.rs

use trait_laws::trait_laws_obeyed;

trait_laws_obeyed!(SlaController);
```

**One line.** The macro expands to ten property tests:

1. `observe_referentially_transparent`
2. `diff_deterministic`
3. `classify_deterministic`
4. `classify_monotonic`
5. `decide_deterministic`
6. `act_idempotent_on_noop`
7. `tick_converges`
8. `attest_canonical`
9. `no_action_without_observation`
10. `repeated_failure_terminates`

Each property runs against `MockObservationSource` over 48+ random
configurations via `proptest`. CI gates on it. **Failure = the
controller is not a controller.** Cannot ship.

### III.7 Step 6 — YAML CR schema + helmworks chart

The `#[derive(TataraDomain)]` proc macro auto-emits the OpenAPI v3
schema for the PromessaCR. Drop it into the
`lareira-engenho-promessa-controllers` chart:

```yaml
# helmworks/charts/lareira-engenho-promessa-controllers/crds/sla.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: promessas.viggy.pleme.io
spec:
  # ...schema auto-emitted by `cargo run --bin engenho-promessa-controllers -- emit-crds`...
```

The chart consumes `pleme-lib` for ServiceMonitor + PrometheusRule
(Pillar 11 mandatory). The HelmRelease lands in
`k8s/clusters/<cluster>/infrastructure/engenho-promessa-controllers/`.

---

## Part IV — Adding a TypedAction variant

The escape hatch — used **only** when none of the existing variants
suffice. Per the three-times rule + R-5.3 in VIGGY-AUTHORING.

### IV.1 When you need a new variant

If three or more PromessaControllers reach for `Custom` with the same
payload shape, extract a new variant. **One use = stay in `Custom`.
Two uses = note the pattern. Three = extract.**

### IV.2 Adding the enum variant

```rust
// promessa-types/src/typed_action.rs

#[derive(Serialize, Deserialize, Debug, Clone)]
pub enum TypedAction {
    FluxCommit        { path: PathBuf, patch: JsonPatch },
    ReconcilerApply   { kind: String, config: serde_json::Value },
    MagmaApply        { workspace: WorkspaceRef, plan_id: BlakeHash },
    CofreRotate       { secret_ref: SecretRef },
    CrachaPatch       { policy: AccessPolicyRef, patch: JsonPatch },
    + YourNewVariant  { /* typed fields */ },     // ← here
    Compose(Vec<TypedAction>),
    Custom            { kind: String, payload: serde_json::Value },
}
```

### IV.3 Wiring the dispatch

Extend the central dispatcher in `engenho-promessa-controllers`:

```rust
// engenho-promessa-controllers/src/dispatch.rs

pub async fn dispatch(action: &TypedAction, ctx: &DispatchContext) -> Result<ActionOutcome, ControllerError> {
    match action {
        TypedAction::FluxCommit { path, patch } => ctx.flux_repo.apply_commit(path, patch).await,
        // …
        TypedAction::YourNewVariant { field1, field2 } => {
            // dispatch to the backing Reconciler / substrate
            ctx.your_new_reconciler.apply(field1, field2).await
        }
        // …
    }
}
```

### IV.4 Backing Reconciler

If the new variant requires a new substrate, author it as a Reconciler
under `pangea-operator` per CONVERGENCE-SUBSTRATE §III. The
Reconciler trait laws apply.

---

## Part V — Extending ObservationSource

When a new observation surface emerges (e.g., a new metrics
provider, a new compliance source), extend `ObservationSource`.

### V.1 When to extend vs. compose

Use `CompositeOf(Vec<ObservationSource>)` first — most "new"
observations are joins of existing sources. Only add a new variant
when the source is **structurally new** (a new protocol, a new
authentication shape, a new query language).

### V.2 The variant + resolver

```rust
// promessa-types/src/observation_source.rs

#[derive(Serialize, Deserialize, Debug, Clone)]
pub enum ObservationSource {
    Shinryu          { query: String, bindings: Bindings, materialize: TypeName },
    KubeApi          { gvr: GroupVersionResource, selector: LabelSelector, projection: JsonPath },
    HttpEndpoint     { url: Url, auth: AuthBinding, parse_as: ContentType },
    TameshiHeartbeat { chain_id: ChainId, layer_type: LayerType },
    + YourNewSource  { /* typed fields */ },     // ← here
    CompositeOf(Vec<ObservationSource>),
}

// engenho-promessa-controllers/src/observation/resolver.rs

pub async fn resolve(source: &ObservationSource, ctx: &ResolverContext) -> Result<Snapshot, ControllerError> {
    match source {
        ObservationSource::Shinryu { … } => ctx.shinryu.query(…).await,
        // …
        ObservationSource::YourNewSource { … } => ctx.your_new_resolver.resolve(…).await,
    }
}
```

---

## Part VI — The trait-laws proptest harness

The substrate's proof discipline (Pillar 10). Every TargetController
ships with this; **CI gates on it.**

### VI.1 The ten invariants (recap)

| # | Invariant | What it proves |
|---|---|---|
| 1 | `observe_referentially_transparent` | observe() on a static world produces equal Snapshots |
| 2 | `diff_deterministic` | same (spec, snapshot) → same Drift → same hash |
| 3 | `classify_deterministic` | same Drift → same Severity |
| 4 | `classify_monotonic` | strictly larger drift produces ≥-severe Severity |
| 5 | `decide_deterministic` | same (spec, severity, drift) → same Decision |
| 6 | `act_idempotent_on_noop` | Decision::Noop doesn't mutate the world |
| 7 | `tick_converges` | post-act diff = Noop OR strictly smaller magnitude |
| 8 | `attest_canonical` | same args → same receipt_hash modulo signature |
| 9 | `no_action_without_observation` | failed observation → no AutoCorrect |
| 10 | `repeated_failure_terminates` | no infinite silent retries |

### VI.2 The macro expansion

`trait_laws_obeyed!(SlaController)` expands to a `#[cfg(test)] mod
trait_laws { … }` containing ten property tests, each running 48+
random configs via `proptest`. The macro lives in
`engenho-promessa-controllers::trait_laws`.

### VI.3 MockObservationSource

Lives in `promessa-types-test` crate. Returns scripted
`Snapshot` values for test inputs. **One mock fleet-wide** — per
★★ macros-everywhere; never write a custom mock.

### VI.4 Golden trajectory tests

For each TargetController:

```rust
// engenho-promessa-controllers/tests/golden/sla_trajectory.rs

#[test]
fn sla_trajectory_matches_golden() {
    let spec = PromessaSpec::from_tatara(include_str!("../fixtures/sla.tatara"));
    let mock = MockObservationSource::from_json(include_str!("../fixtures/sla-observations.json"));
    let chain = run_n_ticks(&SlaController::new(mock), &spec, 10);
    let chain_root = chain.blake3_root_excluding_signatures();
    assert_eq!(chain_root.to_hex(), GOLDEN_SLA_TRAJECTORY_ROOT);
}
```

Golden trajectory roots live in `tests/golden/<kind>.root.txt`.
Behavior regressions silently change classification logic or
decision shape; the golden root catches them.

---

## Part VII — YAML CR rendering pipeline

How `(defpromessa …)` becomes a deployable YAML.

### VII.1 The flow

```
.tatara source
    │
    │ tatara-lisp compile_from_sexp
    ▼
PromessaSpec<TargetKind>   (typed Rust value)
    │
    │ #[derive(TataraDomain)] auto-emits Serialize impl
    │ + canonical SExpr serialization for hashing
    ▼
PromessaCR YAML            (committed under k8s/clusters/<cluster>/promessas/)
    │
    │ FluxCD source/kustomize controllers
    ▼
engenho admission           (sekiban Phase-2 + OutcomeChain bucket presence)
    │
    ▼
kube-rs informer
    │
    ▼
PromessaController instance (from registry by target.kind)
    │
    ▼
seven-beat tick begins
```

### VII.2 caixa-publish integration

Per Pillar 9 (SDLC), every repo with `promessas/*.tatara` gets the
canonical `caixa-publish` workflow (`.github/workflows/caixa-publish.yml`),
which:

1. Runs `feira render --kind promessa` per source file
2. Validates schema against sekiban admission rules at PR time
3. Emits rendered YAML CRs to `k8s/clusters/<cluster>/promessas/*.yaml`
4. Commits + opens PR (or commits directly per repo policy)

No hand-edited YAML. **The `.tatara` is the source of truth.**

---

## Part VIII — Admission wiring (sekiban)

The substrate engineer extends sekiban to handle PromessaCRs.

### VIII.1 Required admission rules

```rust
// sekiban/src/rules/promessa.rs

pub struct PromessaAdmissionRule;

#[async_trait]
impl AdmissionRule for PromessaAdmissionRule {
    fn applies_to(&self, gvk: &GroupVersionKind) -> bool {
        gvk.group == "viggy.pleme.io" && gvk.kind == "Promessa"
    }

    async fn validate(&self, req: &AdmissionRequest) -> AdmissionResult {
        let cr: PromessaCR = req.deserialize_object()?;

        // Rule 1: Phase-2 signature gate (THEORY.md §V.4)
        verify_phase2_signature(&cr)?;

        // Rule 2: target.kind is registered
        TargetControllerRegistry::lookup(&cr.spec.target.kind)
            .ok_or(AdmissionError::UnknownTargetKind(cr.spec.target.kind))?;

        // Rule 3: OutcomeChain bucket exists if requireOutcomeChain (default true)
        if cr.spec.attestation.require_outcome_chain {
            verify_outcome_chain_bucket_writable(&cr.metadata.cluster)?;
        }

        // Rule 4: dependencies don't form a cycle (consult PromessaDependencyController)
        PromessaDependencyController::check_acyclic(&cr.spec.dependencies)?;

        // Rule 5: severity overrides are tighter than defaults (or have :loosening-waiver)
        validate_severity_overrides(&cr.spec.severity_override, &cr.spec.target.kind)?;

        Ok(AdmissionResult::Allow)
    }
}
```

Register the rule in sekiban's `RuleRegistry`. The PromessaCR
admission lives alongside the existing OCI-digest / SignatureGate
/ CompliancePolicy rules.

---

## Part IX — Attestation wiring (tameshi + OutcomeChain)

### IX.1 OutcomeReceipt as a tameshi LayerType

```rust
// tameshi/src/layer_type.rs

#[derive(Serialize, Deserialize, Debug, Clone, PartialEq, Eq, Hash)]
pub enum LayerType {
    // …existing variants…
    Nix,
    Oci,
    Helm,
    Tofu,
    Kubernetes,
    Kindling,
    Tatara,
    FluxCD,
    ArgoCD,
    Akeyless,
    AkeylessTarget,
    // …three ML/math variants…
    + OutcomeReceipt,   // ← new (M4)
    + AnomalyReceipt,   // ← new (M4)
}
```

### IX.2 The chain pipeline

The `outcome-chain` crate consumes tameshi's existing signing
pipeline verbatim — only the leaf type is new:

```rust
// outcome-chain/src/lib.rs

pub struct OutcomeChain {
    bucket: MinioBucket,
    signer: Ed25519Signer,          // resolved via cofre
    last_receipt_hash: BlakeHash,
}

impl OutcomeChain {
    pub fn genesis_from_spec(spec_hash: BlakeHash) -> Self {
        Self {
            last_receipt_hash: spec_hash,
            // …
        }
    }

    pub async fn append(&mut self, receipt: OutcomeReceipt) -> Result<OutcomeReceipt, ChainError> {
        let canonical = receipt.canonical_bytes()?;
        let receipt_hash = blake3::hash(&canonical);
        let signed = SignedOutcomeReceipt {
            inner: receipt,
            prev_receipt_hash: self.last_receipt_hash,
            receipt_hash,
            signature: self.signer.sign(&canonical),
        };
        self.bucket.put(&format!("{}.json", signed.receipt_hash.to_hex()), &signed).await?;
        self.last_receipt_hash = receipt_hash;
        Ok(signed.into_inner())
    }
}
```

### IX.3 cofre key materialization

The Ed25519 signing key per cluster:

```yaml
# In the cluster's k8s manifests
apiVersion: cofre.pleme.io/v1
kind: SecretRef
metadata:
  name: outcome-chain-signing-key
  namespace: engenho-promessa-controllers
spec:
  backend: akeyless
  path: /pleme-io/<cluster>/outcome-chain-ed25519-key
  envVar: OUTCOME_CHAIN_KEY
```

The controller's pod loads the key at startup via the standard
ExternalSecret → core/Secret → env path (per pangea-operator §III).
**Zero plaintext exposure.**

### IX.4 MinIO sink discipline

Per-cluster bucket: `s3://outcome-chains-<cluster>/<promessa>/<receipt-hash>.json`.
Lifecycle: 7 years post-promessa-retirement (per VIGGY-AUTHORING
§8.4). Same shape as tameshi's existing receipt-sink discipline.

---

## Part X — MCP tool authoring

The operator-facing MCP surface (Mode A–E of `/viggy`).

### X.1 The promessa.* tools

```rust
// convergence-controller/mcp/promessa.rs

#[mcp_tool(name = "promessa.list")]
pub async fn promessa_list(req: ListRequest) -> Result<ListResponse, McpError> { … }

#[mcp_tool(name = "promessa.get")]
pub async fn promessa_get(req: GetRequest) -> Result<GetResponse, McpError> { … }

#[mcp_tool(name = "promessa.history")]
pub async fn promessa_history(req: HistoryRequest) -> Result<HistoryResponse, McpError> { … }

#[mcp_tool(name = "promessa.verify")]
pub async fn promessa_verify(req: VerifyRequest) -> Result<OutcomeVerificationReport, McpError> { … }

#[mcp_tool(name = "promessa.replay")]
pub async fn promessa_replay(req: ReplayRequest) -> Result<OutcomeReplayReport, McpError> { … }

#[mcp_tool(name = "promessa.lattice")]
pub async fn promessa_lattice(req: LatticeRequest) -> Result<LatticeView, McpError> { … }
```

### X.2 The anomaly.* tools

```rust
#[mcp_tool(name = "anomaly.list")]            { … }
#[mcp_tool(name = "anomaly.history")]         { … }
#[mcp_tool(name = "anomaly.escalations")]     { … }
#[mcp_tool(name = "anomaly.replay")]          { … }
```

### X.3 Registration

The `#[mcp_tool]` macro registers the tool into convergence-controller's
MCP server (per THEORY.md §VII.5). The tool definitions auto-emit
OpenAPI schemas for the operator's LLM router.

---

## Part XI — Testing via kind cluster + kenshi ephemeral preview

Pre-merge gate per VIGGY-AUTHORING §10.4.

### XI.1 The kind-based integration

```rust
// engenho-promessa-controllers/tests/integration/kind_sla.rs

#[tokio::test]
async fn sla_in_kind_cluster() {
    let kind = KindCluster::up("viggy-sla-test").await?;
    kind.apply_engenho_promessa_controllers_chart().await?;

    let promessa_cr = include_str!("fixtures/sla-test.yaml");
    kind.kubectl_apply(promessa_cr).await?;

    let mock_obs = MockObservationSource::stream(/* …10 snapshots… */);
    kind.configure_mock_observation(&mock_obs).await?;

    // Run 100 ticks
    let receipts = kind.collect_outcome_receipts("lilitu-prod-sla", 100).await?;
    assert_eq!(receipts.len(), 100);

    // AnomalyChain integrity
    let anomaly_chain = kind.fetch_anomaly_chain().await?;
    assert!(anomaly_chain.verify_integrity().is_ok());

    kind.down().await?;
}
```

CI runs every integration test. **No merge if any fails.**

### XI.2 kenshi ephemeral preview

For human pre-merge review of a new promessa (not for substrate
engineering):

```bash
$ kenshi env up viggy-preview-lilitu-prod-sla
$ kubectl apply -f promessas/lilitu-prod-sla.yaml
$ kensa replay outcome-chain --promessa lilitu-prod-sla --window 1h --mock
$ kenshi env down viggy-preview-lilitu-prod-sla
```

---

## Part XII — Deployment via helmworks

### XII.1 The chart

`helmworks/charts/lareira-engenho-promessa-controllers/`:

```
Chart.yaml                # depends on pleme-lib + pleme-operator
values.yaml               # kind controller registrations, replicas, log level
templates/
  deployment.yaml         # via pleme-operator library template
  rbac.yaml               # cluster-scoped (manages PromessaCR in any namespace)
  crds/                   # PromessaCR + AnomalyCR + PromessaDependencyCR auto-emitted
  servicemonitor.yaml     # Pillar 11 mandatory
  prometheusrule.yaml     # Pillar 11 mandatory
  externalsecret.yaml     # cofre OutcomeChain signing key
```

### XII.2 Per-cluster HelmRelease

Land in `k8s/clusters/<cluster>/infrastructure/engenho-promessa-controllers/release.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: engenho-promessa-controllers
  namespace: viggy-system
spec:
  chart:
    spec:
      chart: lareira-engenho-promessa-controllers
      sourceRef:
        kind: HelmRepository
        name: helmworks
  values:
    enabledKinds: ["sla", "cost-budget", "compliance", "customer-kpi", "security"]
    outcomeChain:
      bucket: outcome-chains-${cluster}
```

---

## Part XIII — The macros catalog

The substrate's standardization surface. Every recurring shape in
Viggy implementation extracts to one of these macros.

| Macro | Crate | Expands to |
|---|---|---|
| `#[derive(TataraDomain)]` | `tatara-derive` | SExpr round-trip + BLAKE3 + YAML serde + OpenAPI schema export |
| `#[register_target_controller(target_kind = "…")]` | `engenho-promessa-controllers-derive` | inserts the controller into the static `TargetControllerRegistry` |
| `trait_laws_obeyed!(MyController)` | `engenho-promessa-controllers::trait_laws` | the 10-invariant proptest harness |
| `#[mcp_tool(name = "…")]` | `mcp-derive` | tool registration + OpenAPI emission for the LLM router |
| `#[derive(TameshiLayer)]` | `tameshi-derive` | new LayerType variant + canonical serialization |
| `#[register_reconciler(kind = "…")]` | `convergence-substrate-derive` | inserts into universal Reconciler engine |
| `golden_trajectory!(controller, fixture, root)` | `engenho-promessa-controllers::testing` | the trajectory regression test |

**Three-uses-then-extract** for any new recurring shape. If you find
yourself writing the same boilerplate twice, the third instance
should be a macro extraction PR.

---

## Part XIV — Anti-patterns (substrate engineering)

| Anti-pattern | Why forbidden | Directive |
|---|---|---|
| `format!()` to compose Nix / YAML / Go / Rust source | typed AST renderer required | ★★ TYPED EMISSION |
| Hand-writing CRD schemas | `#[derive(TataraDomain)]` auto-emits | Pillar 12 |
| Bespoke proptest harness per controller | `trait_laws_obeyed!` macro is canonical | ★★ macros-everywhere |
| Per-controller observation client | `ObservationSource` enum + central resolver | Compounding #1 |
| Direct kube-rs informer without going through engenho's PromessaController scaffold | engenho IS the controller harness | Compounding #1 |
| Adding a new severity tier (4th level) | three are canonical fleet-wide | ★★ macros-everywhere |
| Hand-rolled retry loop in a controller | shigoto Dag + RetryPolicy | ★★ Shigoto |
| Inline cloud-API call (HTTP / SDK) | route through Reconciler trait | Compounding #1 |
| Skipping the kind-cluster integration test | mandatory CI gate | Pillar 10 |
| `#[allow(dead_code)]` on a TargetController trait method | trait must be fully implemented | Pillar 10 |
| Mutable shared state in a TargetController impl | functions must be pure (Diff/Classify/Decide) | trait law |

---

## Part XV — Closing

VIGGY-IMPLEMENTING is the substrate-side discipline. Six steps to
author a TargetController; the macros do the rest. Every recurring
shape extracts to a macro. Every change observes the directives.
**The substrate engineer widens the algebra; the operator declares;
the cluster solves.**

---

> *The author writes one (defpromessa …). The substrate engineer
> ensures every leg snaps. The macros do the work. **Generation over
> composition, every time.***
