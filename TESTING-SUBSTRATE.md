# Testing Substrate

Status: ★★★ canonical
Implementing crate: [`pleme-io/magma`](https://github.com/pleme-io/magma), specifically `magma-test-laws` + `magma-arch-test`
Related: [`CONVERGENCE-SUBSTRATE.md`](./CONVERGENCE-SUBSTRATE.md), [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md), [`MAGMA.md`](./MAGMA.md), [`MAGMA-OPERATOR-BACKEND.md`](./MAGMA-OPERATOR-BACKEND.md), [`PANGEA-MAGMA-ORCHESTRATION.md`](./PANGEA-MAGMA-ORCHESTRATION.md)

## I. Thesis

**Every layer of the convergence substrate ships its own universal trait-law battery.** A new impl of any substrate primitive — a new `Reconciler`, a new `Backend`, a new typed architecture, a new workspace, a new chain of workspaces — gains the full contract test suite in one line of test code.

This is the operational realization of the Compounding Directive's "magma fully batteries loaded for the complete and total lifecycle of testing." The substrate proves its own correctness; downstream tests don't have to re-derive what "correct" means.

## II. Layered law batteries

`magma-test-laws` is the canonical home for every law battery. Each layer is gated behind a cargo feature so consumers only pay for what they use.

| Layer | Module | Feature | One-line entrypoint |
|---|---|---|---|
| Reconciler trait | (top-level) | (default) | `magma_test_laws::assert_all_laws(&r, …).await` |
| Backend trait | [`backend`](https://github.com/pleme-io/magma/blob/main/magma-test-laws/src/backend.rs) | `backend-laws` | `magma_test_laws::backend::assert_all_laws(&b).await` |
| Pangea architecture | [`architecture`](https://github.com/pleme-io/magma/blob/main/magma-test-laws/src/architecture.rs) | `architecture-laws` | `magma_test_laws::architecture::assert_all_laws(&cfg)` |
| Workspace lifecycle | [`workspace`](https://github.com/pleme-io/magma/blob/main/magma-test-laws/src/workspace.rs) | `workspace-laws` | `magma_test_laws::workspace::assert_all_laws(&cfg)` |
| Workspace chain | [`chain`](https://github.com/pleme-io/magma/blob/main/magma-test-laws/src/chain.rs) | `chain-laws` | `magma_test_laws::chain::assert_all_laws(&flow)` |
| Proptest strategies | [`strategies`](https://github.com/pleme-io/magma/blob/main/magma-test-laws/src/strategies.rs) | `strategies` | shared generators |

### Reconciler trait laws (5)

Every `Reconciler` impl in the substrate satisfies:

1. **read_state idempotent** — two reads with no intervening write return equal state.
2. **empty config produces noop plan** — `compute_plan(empty, state)` is a no-op + applying it doesn't mutate state.
3. **apply converges** — after `apply(plan)`, `detect_drift(config)` is empty.
4. **plan_id deterministic** — `compute_plan(c, s)` twice yields the same `PlanId`.
5. **plan_id differs with config** — different change shape ⇒ different `plan_id`.

Locked across all 6 shipped impls (Helm, Dns, Vault, Github, Terraform, InMemoryKv) via `magma-test-laws/tests/concrete_reconcilers_obey_laws.rs`.

### Backend trait laws (5)

Every `Backend` impl satisfies:

1. **read_state idempotent**.
2. **write → read round-trips** — lineage + serial + resource count preserved.
3. **lock → unlock round-trip** — correct-id unlock succeeds.
4. **unlock with wrong id doesn't panic** — error semantics are backend-specific.
5. **serial monotonically increases** — second write with rising serial is durably preserved.

Locked across `InMemoryBackend` + `LocalBackend` via `magma-test-laws/tests/backend_law_battery.rs`.

The Backend law battery surfaced a real bug in `LocalBackend::read_state`: calling read on a missing state file generated a fresh UUID v4 lineage every call (because `empty_state_inline()` used `Uuid::new_v4()`). Fixed by materializing empty state to disk on first read (matches OpenTofu's `tofu init` semantics).

### Architecture composition laws (5)

Every Pangea-rendered architecture satisfies, over `magma_config::Config`:

1. **Resource addresses unique** — no duplicate `(type, name)` pairs.
2. **No dangling references** — every `${type.name.attr}` interpolation resolves to a declared resource/data/module/output. HCL-builtin refs (`var.*`, `local.*`, `each.*`, `count.*`, `path.*`, `terraform.*`, `self.*`) skipped.
3. **Outputs have values** — no null outputs.
4. **terraform.required_providers present** — workspace with resources declares providers.
5. **Every resource type's provider is registered** — `aws_vpc` → `aws` must be in `required_providers`.

### Workspace lifecycle laws (5 + 1 opt-in)

Over `magma_config::Config`:

1. **plan deterministic** — same `(cfg, state)` → same `PlanId`.
2. **apply enumerates all changes** — `applied.len + failed.len == plan.change_count`.
3. **apply converges** — re-plan after apply has zero non-NoOp changes.
4. **apply bumps serial** — successful applies advance `state.serial`.
5. **destroy round-trip** — apply, then plan against empty config emits Deletes; running destroy empties state.
6. **import absorbs** (opt-in) — a state seeded with an externally-discovered resource whose address matches `cfg` plans as NoOp/Update, never Create.

### Workspace chain laws (6)

Over `magma_flow::FlowFile`:

1. **acyclic** — `topological_order` succeeds.
2. **edges reference declared workspaces** — no orphan edges.
3. **workspace names unique**.
4. **workspace dirs non-empty**.
5. **optimization knobs positive** — `max_concurrency > 0`, `timeout_ms > 0` when present.
6. **apply_order / destroy_order helpers** for inline tests.

### Proptest strategies (shared)

`magma_test_laws::strategies` provides generators consumed across the substrate test surface:

* `arb_action` — universal `Action` (Create/Update/Delete/Replace/NoOp)
* `arb_plan_id` / `arb_optional_plan_id` — typed PlanId
* `arb_plan` — 0-8 random changes over typed addresses
* `arb_phase` — all 9 FSM phases
* `arb_lifecycle_happy_walk` — 0-4 happy-path FSM transitions
* `arb_event_payload` — all 4 EventPayload variants
* `arb_event_chain(max_len)` — valid prev_hash-chained event sequence

Adding a field to `Plan` / `EventPayload` / `LifecycleState` updates ONE strategy file; every downstream proptest gets the new field's coverage for free.

## III. Bridge to real Pangea workspaces

`magma_arch_test::WorkspaceTestHarness` bridges from a workspace path (Pangea Ruby dir OR rendered TF JSON file) to the law battery in one line:

```rust
use magma_arch_test::WorkspaceTestHarness;

#[tokio::test]
async fn seph_vpc_passes_substrate_laws() {
    let harness = WorkspaceTestHarness::new("workspaces/seph-vpc");
    harness.assert_all_substrate_laws().await.unwrap();
}
```

This single call runs:

* `magma_test_laws::architecture::assert_all_laws(&cfg)` — composition invariants
* `magma_test_laws::workspace::assert_all_laws(&cfg)` — lifecycle invariants

Returns a `WorkspaceReport` (typed shape, plan_id, action histogram, provider list) for further inspection. Panics on the first law violation with a descriptive message naming the broken law.

### Cross-fixture coverage

`magma-test/tests/integration_pangea_architectures_coverage.rs` runs the bridge against every fixture in `magma-test/fixtures/pangea-architectures/` (19 architectures as of 2026-05-18). A single test, `every_fixture_passes_all_substrate_laws`, exercises **every law × every fixture** — adding a new fixture is a one-file drop, no test edits required.

## IV. Pangea-operator integration

`pangea-operator` runs the law battery as a **preflight check** before each reconcile cycle:

1. On controller startup (or CR creation), parse the CR's referenced workspace into `Config`.
2. Run `magma_test_laws::architecture::assert_all_laws(&cfg)` — fail fast if the rendered workspace is malformed (dangling refs, duplicate addresses, missing providers).
3. Run `magma_test_laws::workspace::assert_all_laws(&cfg)` — fail fast if the plan/apply/destroy round-trip isn't well-formed in memory.
4. Only then proceed to actual reconcile against the live state backend.

This shifts the failure mode from "applied broken state to cloud" to "operator refused to start reconcile cycle for malformed input." The cost is one in-memory plan-apply round-trip per CR per startup; the benefit is mechanical preflight verification of every architectural promise the workspace makes.

## V. The compounding cycle

Per the operator's directive — *"pangea-operator as programming for how that reconciliation cycle can be more efficient and effective over time given the use case"*:

```
                  Use case surfaces a missed law
                                │
                                ▼
                Add law helper to magma-test-laws
                                │
                                ▼
         Every workspace + architecture + chain now
         enforces it on every CI run + on every
         pangea-operator preflight
                                │
                                ▼
              The contract widens by one promise;
              every consumer benefits silently
                                │
                                ▼
            Reconciliation efficiency increases:
            preflight catches the failure class
            before the live backend pays for it
```

The substrate's correctness becomes monotonically stronger over time. Every new use case that surfaces a missed contract becomes one more law in the battery; the battery never shrinks.

## VI. Future expansions

Open law batteries to author:

* **Resource provider laws** — every Pangea-rendered provider gem's typed function (`synth.aws_vpc(...)`) renders deterministically + conforms to schema.
* **State migration laws** — `magma-migrate` between state shapes preserves identity (lineage stable, serial advances).
* **Cross-cluster federation laws** — multi-cluster constellations preserve global resource address uniqueness.
* **Compliance-attestation laws** — every law-battery pass produces a typed `Bundle` whose BLAKE3 attestation matches the inputs.
* **Property-based fuzzing for the law helpers themselves** — proptest-driven exploration of `assert_all_laws` over generated Configs to surface laws we haven't written yet.

## VII. Test count

Current snapshot (commit `f90dad7`, 2026-05-18):

```
magma workspace: 410 tests pass, 0 failures (with all law features enabled)
```

Test classes:

* 6 Reconciler-impl law batteries (5 laws each)
* 5 Backend trait law batteries (across 2 impls)
* 9 Architecture composition law tests (incl. negative panics)
* 10 Workspace lifecycle law tests (incl. import absorption)
* 9 Chain law tests (incl. negative panics)
* 5 strategy keep-alive proptests
* 19 Pangea-architectures fixtures × 6 cross-fixture integration tests
* (and the ~330 property-based + unit tests covering each substrate primitive's internals)

Every load-bearing promise the substrate makes is mechanically verified, not asserted on faith.
