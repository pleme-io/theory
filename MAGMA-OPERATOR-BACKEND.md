# Magma as a pangea-operator IaC backend

> **Frame.** pangea-operator already has the seam designed for this
> swap (`IacExecutor` trait, `StateBackend` trait). This doc captures
> the destination — what we get when magma becomes the operator's
> in-process executor — and the phased path to get there. The
> existing `TofuExecutor` is **not** removed; both backends ship side
> by side, selected per-CR or via a helm toggle. The operator becomes
> the executor, not a launcher of one.

---

## I. Frame

pangea-operator's reconcile loop is, today, a three-process tree:

```
operator pod  →  fork+exec tofu  →  fork+exec provider plugins
```

Every reconcile spawns tofu (~50–200 ms cold-start, ~10–30 MB RSS),
which itself shells out N provider plugins, each with its own gRPC
plane. Inputs and outputs serialize through the file system:
Pangea-rendered `main.tf.json`, `terraform.tfstate`, `tfplan` binary
file, JSON output streams. Errors come back as parsed stderr.

This works. It's also wasteful in three places that compound:

1. **Serialization.** Every reconcile crosses three serialization
   boundaries (Pangea → tf.json → tfstate parse → JSON output parse).
   Each is a place data can drift, each costs CPU+heap, each makes
   reconcile latency a function of state-file size.
2. **Subprocess overhead.** Fork+exec is the dominant cost for small
   workspaces (sub-second reconciles).
3. **Loss of typing.** The operator can't make typed reconciliation
   decisions ("the only change is a tag — skip apply") because it
   only sees tofu's stdout, not the typed plan.

magma — pleme-io's Rust-native, Pangea-Ruby-first IaC executor —
collapses all three. magma already:

* Loads Pangea-rendered Terraform JSON via `magma_config::Config::from_json`.
* Plans via `magma_plan::plan` (pure Rust fn returning typed `Plan`).
* Applies via `magma_apply::run_plan` (pure async Rust fn returning typed `Outcome`).
* Reads/writes state via the `magma_backend::Backend` trait.
* Runs multi-workspace flows via `magma_flow::run` (M0.9 shipped).

So: if the operator embeds magma as a library and implements one
`magma_backend::Backend` impl over the operator's existing
`StateBackend` trait, the entire reconcile pipeline becomes a single
in-process call chain. No subprocess. No disk intermediary
(beyond optional plan-file checkpoints for restart safety). No JSON
serialization roundtrip.

---

## II. Destination — two backends side by side

```
                                  pangea-operator pod
                                  ────────────────────
                                                     ┌──── TofuExecutor ──→ fork+exec tofu (today)
                                                     │     (file-based handoff to/from disk)
   K8s CR ──→ controller ──→ IacExecutor (trait) ───┤
                                                     │
                                                     └──── MagmaExecutor ──→ magma_plan::plan(...)
                                                           (in-process)      magma_apply::run_plan(...)
                                                                             magma_flow::run(...)
                                                                             ↓
                                                                             OperatorStateBackend
                                                                             (impl magma_backend::Backend
                                                                              over StateBackend trait)
                                                                             ↓
                                                                             PostgresStateBackend
                                                                             (operator's existing storage)
```

Two `IacExecutor` impls coexist. Selection is per-CR via
`Pangea.spec.executor: tofu | magma` (default: cluster-wide setting),
or cluster-wide via the helm `useMagmaExecutor` toggle (default: false
until burn-in proves parity). The operator never loses the ability to
fall back to tofu — both implementations are exercised by CI on every
PR.

### The four phases collapse to one process

```
TODAY (3 processes, 4 serializations)                 MAGMA-AS-BACKEND (1 process, 0 serializations)
─────────────────────────────────────────             ──────────────────────────────────────────────
Pangea CRD                                            Pangea CRD
  ↓                                                     ↓
pangea-ruby-eval (magnus, in-pod)                     pangea-ruby-eval (magnus, in-pod)
  ↓ pangea_types::Value                                 ↓ pangea_types::Value
serialize → main.tf.json on disk                      to_tf_json() → serde_json::Value in heap
  ↓                                                     ↓
fork+exec `tofu plan` (subprocess)                    magma_config::Config::from_json(value)
  ↓ tofu reads main.tf.json                             ↓
parse stderr/stdout/exit code                         magma_plan::plan(&cfg, &state) → typed Plan
  ↓                                                     ↓
fork+exec `tofu apply` (subprocess)                   magma_apply::run_plan(&plan, &mut state)
  ↓ reads + writes terraform.tfstate                    ↓ writes through OperatorStateBackend
parse stderr/stdout/exit code                         Outcome { plan_id, applied, failed, ... }
  ↓                                                     ↓
status: "Applied 3 changes"                           status.plan_id: 0xBLAKE3...
                                                      status.changes_applied: 3 (typed)
                                                      events: typed magma::ApplyEvent stream
```

Two process boundaries collapse into zero. Two serialization
roundtrips collapse into one (memory → memory). Typed reconciliation
decisions become available everywhere in the controller.

---

## III. Crate composition

The work touches three places:

### III.1. magma (no new crate; reuses existing surface)

magma's library surface is already shaped for in-process consumption:

* `magma::config::Config::from_json(serde_json::Value) -> Config`
* `magma::plan::plan(&cfg, &state) -> Plan`
* `magma::apply::run_plan(&plan, &mut state) -> Outcome`
* `magma::backend::Backend` (async trait)
* `magma::types::{State, Plan, ResourceChange, ...}`
* `magma_flow::run(&FlowFile) -> AggregateReport`

The operator just consumes these. No magma-side changes are
load-bearing for M0.10–M0.13.

### III.2. pangea-operator additions

* **`executor/magma.rs`** (new file) — feature-gated `executor_magma`.
  Three types:
  * `OperatorStateBackend<S: StateBackend>` — implements
    `magma_backend::Backend` over the operator's existing
    `StateBackend` trait. State reads/writes route through the
    operator's PG store (or `InMemoryStateBackend` for tests).
  * `MagmaExecutor` — implements the operator's `IacExecutor` trait
    using magma's in-process Rust APIs.
  * `MagmaExecutorConfig` — typed config: which workspace dir
    convention to use, whether to write a plan checkpoint to disk for
    restart safety, attestation policy.

* **`executor/mod.rs`** gains a `MagmaExecutor` re-export under the
  `executor_magma` feature, parallel to `TofuExecutor`.

* **`Cargo.toml`** gains `executor_magma` feature with deps on `magma`,
  `magma-plan`, `magma-apply`, `magma-state`, `magma-config`,
  `magma-flow`, `magma-backend`, `magma-types`, `magma-pangea`.

* **`controller/mod.rs`** picks the executor at startup time based on
  the helm-provided env var `PANGEA_EXECUTOR=tofu|magma` (default
  `tofu`). Per-CR override via `Pangea.spec.executor` lands in M0.12.

* **CR spec gains `executor: String`** field (optional, default
  inherited from operator-wide setting).

### III.3. theory updates

This doc + a §VIII update to `PANGEA-MAGMA-ORCHESTRATION.md`
recording the operator phase.

---

## IV. The state-backend bridge

magma defines its own `Backend` trait:

```rust
#[async_trait]
pub trait Backend: Send + Sync {
    async fn read_state(&self) -> Result<State, BackendError>;
    async fn write_state(&self, state: &State) -> Result<(), BackendError>;
    async fn lock(&self) -> Result<LockHandle, BackendError>;
    // ...
}
```

The operator's existing `StateBackend` trait is shaped slightly
differently (named keys, raw bytes). The bridge:

```rust
pub struct OperatorStateBackend<S: StateBackend> {
    inner:           Arc<S>,
    schema_name:     String,
    template_name:   String,
    state_name:      String,
}

#[async_trait]
impl<S: StateBackend> magma_backend::Backend for OperatorStateBackend<S> {
    async fn read_state(&self) -> Result<State, BackendError> {
        let parsed = self.inner
            .get_parsed_state(&self.schema_name, &self.template_name, &self.state_name)
            .await
            .map_err(into_magma_backend_err)?;
        match parsed {
            Some(terraform_state) => Ok(terraform_state_to_magma_state(terraform_state)),
            None => Ok(magma_state::empty_state()),
        }
    }
    async fn write_state(&self, state: &State) -> Result<(), BackendError> {
        let bytes = serde_json::to_vec_pretty(state)
            .map_err(|e| BackendError::Other(e.into()))?;
        self.inner
            .save_state(&self.schema_name, &self.template_name, &self.state_name, &bytes)
            .await
            .map_err(into_magma_backend_err)?;
        Ok(())
    }
    // lock: delegated to operator's existing pg_try_advisory_lock path
}
```

The reverse-direction conversion
`terraform_state_to_magma_state(TerraformState) -> magma_types::State`
is the load-bearing piece. magma's `State` uses typed
`ProviderReference { source, name, alias }` whereas tofu's serialized
state uses the string-encoded form `provider["registry.terraform.io/hashicorp/aws"]`.
We need a tested converter both directions (the inverse is also
needed for write-back, since the operator's existing readers expect
the tofu-shaped state for compatibility with reads from outside the
operator).

---

## V. Phases

| Phase | Status | Scope |
|---|---|---|
| **M0.10** | shipping | Bridge: `OperatorStateBackend` + state-shape converter + `MagmaExecutor` skeleton. Feature-gated `executor_magma`; default off. Unit tests with `InMemoryStateBackend`. |
| **M0.11** | next | `MagmaExecutor` full `IacExecutor` impl: init (no-op), plan (renders to typed `Plan`, writes plan checkpoint to workspace dir for restart safety), apply (reads checkpoint, runs `magma_apply::run_plan`), destroy (synthesize delete-all plan), show_plan, output, refresh, import. Integration test against the operator's existing reconcile spec suite using a synthetic Pangea-rendered workspace. |
| **M0.12** | next | Per-CR executor selection via `Pangea.spec.executor`. Helm chart adds `useMagmaExecutor` toggle (default false). The controller picks the executor per CR; both impls live in the same operator binary. |
| **M0.13** | next | Typed Chain CRD (`PangeaChain`). Operator's controller for that CR drives `magma_flow::run` in-pod. Cross-workspace output passing happens in heap, no PG roundtrip per consumer. |
| **M0.14** | later | Attestation surface — BLAKE3 plan_id becomes typed `Pangea.status.planId`. Apply receipts emit typed K8s Events. External auditors verify by hashing the rendered tf.json. |
| **M0.15** | later | Dual-run burn-in. Helm flag `dualRunBurnIn: true` runs both backends on every reconcile (tofu for prod, magma for dry-run); operator emits a typed diff event when receipts diverge. Once parity is proven (week of clean diffs), flip default to magma. |

---

## VI. Selection model — both backends, always

Three layers of selection:

1. **Per-CR** (highest priority): `Pangea.spec.executor: magma`
2. **Operator-wide** (env var): `PANGEA_EXECUTOR=magma` (helm-derived)
3. **Default**: `tofu` (until M0.15 flip)

Every controller method resolves the executor at the moment of use
via `controller.executor_for(cr).await` — a typed `Arc<dyn
IacExecutor>` that may be a `TofuExecutor` or `MagmaExecutor`
instance. Both are cached by the controller. Both are exercised by
CI on every PR.

This means:

* Operators that need tofu's mature provider ecosystem keep it.
* Operators that need magma's typed surface opt in per-CR.
* Cluster operators run mixed fleets — magma for new workloads,
  tofu for legacy.
* Fallback is one-line — flip the spec field.
* CI catches drift between the two impls (typed Plan comparison
  via the burn-in phase).

---

## VII. Tradeoffs reframed as features

The "risks" of in-process execution become controls when you own the
process model:

| "Risk" | Reframe | How we exploit it |
|---|---|---|
| Operator panics now affect apply | The operator is now the executor — panics ARE applies | Per-reconcile `tokio::spawn` with panic catcher. Panic in reconcile-N doesn't affect reconcile-M (different task). Apply panics become typed K8s Events. Restart picks up via BLAKE3 plan-id replay. |
| Operator memory ceiling | RAM use becomes predictable + measurable | Apply allocates `state.resources.len() * ~16KB` typed structs. K8s pod requests/limits scale linearly. Streaming apply (M0.16) collapses peak RAM to streaming buffer size. |
| Restart semantics | Plan/apply atomicity becomes a typed invariant | Plan emits BLAKE3 plan_id → stored in CR status BEFORE apply. Restart mid-apply → controller sees stored plan_id, re-derives same plan, resumes apply (apply is idempotent against same plan). |
| Magnus + magma in same process | Both linked once, full pleme-io substrate available in-pod | Operator gains direct access to magma_attest BLAKE3 chains, magma_state typed inspection, magma_mcp for in-cluster tool exposure. The operator IS the substrate. |

---

## VIII. Anti-patterns this surface forbids

* **Shelling out to `magma` binary from the operator.** The operator
  embeds magma as a library; the binary is for operator-CLI use only.
* **Serializing the typed `Plan` to disk for handoff between
  controller methods.** Use `Arc<Plan>` for in-process handoff.
  Disk plan-files are restart-safety checkpoints only.
* **Adding new tofu-specific assumptions to the controller.** Every
  reconciliation decision must work under both backends. Add typed
  surface to `IacExecutor` (the trait) if a new capability is needed.
* **Removing `TofuExecutor`.** Both backends ship indefinitely;
  operators choose. Magma's adoption is per-CR opt-in, not a
  cluster-wide migration.
* **Diverging behavior between the two executors.** The burn-in
  phase (M0.15) catches typed-Plan divergence; both must agree.

---

## IX. Open questions

1. **State byte-equality between tofu and magma writes.** magma's
   typed `State` round-trips through serde; tofu's serialized state
   uses string-encoded provider references + ordering conventions.
   M0.11 must include a typed converter that produces byte-equal
   tofu-readable state files for backwards compat (so operators can
   read magma-applied state with tofu later).
2. **In-cluster magma binary footprint.** Operator pod RSS today
   ~50–80 MB. Adding magma (with tonic/rustls/magma-providers) likely
   pushes to ~150 MB. Measurable; comfortably within helm pod limits;
   worth recording in burn-in.
3. **Provider plugin parity.** magma today speaks tfplugin5/6 to
   real provider binaries (same as tofu). The operator path doesn't
   need this for in-cluster reconciles of pleme-io's own Pangea
   primitives, but cloud provider work still needs the plugin
   protocol. M0.11 ships with magma's existing plugin path.
4. **Attestation propagation.** Should plan_ids attest into FluxCD
   GitOps state? Likely yes; lands in M0.14 with a typed pleme-io/k8s
   admission webhook that records plan_ids in audit logs.

---

## X. How this doc evolves

* Each phase moves from "next" → "shipping" → "shipped" with the
  commit reference recorded in §V.
* Open questions migrate to dedicated RFCs as they get answered.
* The dual-backend selection model in §VI is canonical for every
  similar pleme-io component: if there are two valid executor
  shapes, both ship side by side, both are CI-exercised, the
  fall-forward path is per-CR.
