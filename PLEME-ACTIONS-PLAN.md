# pleme-actions implementation plan

> Companion to [`CONSTRUCTIVE-ACTIONS.md`](./CONSTRUCTIVE-ACTIONS.md). That
> document specifies the pattern; this one tracks what needs to be built to
> make it real. Single source of truth for project status.

## Phase ladder

Typescape-render-first: the `Action` domain in arch-synthesizer leads,
everything else renders from it.

| # | Phase | Duration | Status | Outcome |
|---|---|---|---|---|
| 0 | Lock down the design | 1 hour | ✅ done 2026-04-27 | CONSTRUCTIVE-ACTIONS.md + this plan + pleme-io/CLAUDE.md section + pleme-actions skill |
| 1 | `Action` domain in arch-synthesizer | 1 day | pending | `arch-synthesizer/src/action_domain.rs` — typed Rust struct + `#[derive(TataraDomain)]` auto-generates the `(defaction …)` Lisp parser + render-target attrs |
| 2 | `pleme-actions-shared` library crate | 0.5 day | pending (parallel to 1) | Rust crate with `INPUT_<name>` env-var parsing, step-summary builders, attestation hooks. Published to crates.io. |
| 3 | Substrate primitive | 1 day | pending | `substrate/lib/rust-action-release-flake.nix` — extends `rust-tool-release-flake.nix` with action.yml emission + binary-download glue |
| 4 | Rendering pipeline + repo-forge `rust-action` archetype | 1 day | pending | `arch-synthesizer/src/action_domain/render.rs` projects an `Action` value to the full file set; `repo-forge/archetypes/rust-action/` provides the template surface; `repo-forge new --archetype rust-action` produces a working repo |
| 5 | First action: `terragrunt-apply` | 1.5 days | pending | `(defaction "terragrunt-apply" …)` in `pleme-actions-catalog.lisp` → render → fill in operator-authored `src/main.rs` → tag `v0.1.0` |
| 6 | Two more actions: `k8s-pause-verify` + `helmworks-render-check` | 1 day each | pending | Pattern proven across 3 distinct shapes (TF, K8s, Helm) |
| 7 | Migrate first downstream consumer | 0.5 day | pending | Any pleme-io-fleet workflow consuming terragrunt + paused-K8s-deploy drops from ~200 lines of inline YAML to ~30 by consuming the published actions |
| 8 | `pleme-actions` index repo | 0.5 day | pending | README hub, auto-generated version compat matrix from the typescape, contributing guide |
| 9 | First public release announcement | 0.5 day | pending | Optional — once 5+ actions stable, surface to broader pleme-io ecosystem |

**Total estimate:** ~8 days of focused work, spread across multiple
sessions. Phases 1, 2 are independent and can run in parallel; phase 3
depends on phase 2's interface; phase 4 depends on phases 1+3; phase 5
needs all four prior phases.

---

## Phase 1 — `Action` domain in arch-synthesizer

**Files:**
- `arch-synthesizer/src/action_domain.rs` — typed Rust struct + helpers
- `arch-synthesizer/src/action_domain/render.rs` — render targets
- `arch-synthesizer/src/lib.rs` — register the new domain

**The typed primitive** (per `theory/THEORY.md` §II.4 Rust + Lisp pattern):

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
#[tatara(keyword = "defaction")]
pub struct Action {
    pub name: String,
    pub description: String,
    pub inputs: Vec<ActionInput>,
    pub outputs: Vec<ActionOutput>,
    pub behavior: ActionBehavior,
    pub semver_compat: SemverPolicy,
    pub attestation: AttestationPolicy,
}
```

`#[derive(TataraDomain)]` auto-generates the `(defaction …)` Lisp parser.
Round-trip Rust ↔ Lisp is by-construction isomorphic (no information loss
in either direction).

**Acceptance criteria:**
- `cargo test -p arch-synthesizer action_domain` passes — round-trip tests
  for several `(defaction …)` examples
- `arch_synthesizer::action_domain::render::all(&Action {…})` returns a
  `BTreeMap<PathBuf, String>` covering every per-action repo artifact
- A sample `(defaction "hello-world" …)` form parses into the right
  typed value
- Domain is registered in the typescape's universe and visible to
  arch-synthesizer's introspection (`SystemTypescape::action_domain()`)

**Dependencies:** existing arch-synthesizer + tatara-lisp infrastructure.

---

## Phase 2 — `pleme-actions-shared` library crate

**Files:**
- `pleme-io/pleme-actions-shared/Cargo.toml`
- `pleme-io/pleme-actions-shared/src/lib.rs` — public surface
- `pleme-io/pleme-actions-shared/src/input.rs` — `Input<T: Deserialize>` reads from `INPUT_<NAME>` env vars
- `pleme-io/pleme-actions-shared/src/output.rs` — `Output<T: Serialize>` writes to `$GITHUB_OUTPUT`
- `pleme-io/pleme-actions-shared/src/summary.rs` — `StepSummary` builder writing to `$GITHUB_STEP_SUMMARY`
- `pleme-io/pleme-actions-shared/src/error.rs` — `ActionError` matching GitHub's `::error::` / `::warning::` annotation format
- `pleme-io/pleme-actions-shared/src/log.rs` — structured logging that respects `RUNNER_DEBUG=1`
- `pleme-io/pleme-actions-shared/src/test_helpers.rs` — `MockEnv` + `with_simulated_inputs(…)` for action tests

Published to crates.io as `pleme-actions-shared` so per-action repos can
consume via crates.io rather than git deps (faster builds, version pinning
through Cargo.lock).

**Acceptance criteria:**
- Unit tests cover: input parsing happy path + missing required + wrong type
- Cross-platform binaries pass smoke tests on all 4 targets
- Documented public API on docs.rs
- `cargo publish --dry-run` succeeds against crates.io

**Dependencies:** none (parallel with Phase 1). Phase 4 needs both.

---

## Phase 3 — Substrate primitive

**File:** `pleme-io/substrate/lib/rust-action-release-flake.nix`

Extends `rust-tool-release-flake.nix` with two extra outputs:

1. The standard cross-platform binary release (existing behavior).
2. An `action.yml` rendered from a typed Nix attrset describing the
   action's inputs/outputs/runs config. The action.yml uses
   `runs.using: composite` and emits steps that:
   - `curl -fsSL` the binary from the matching GH release based on the
     `${{ runner.os }}` and `${{ runner.arch }}` of the consumer
   - `chmod +x` it
   - Invoke it with all `${{ inputs.* }}` passed as env vars (per the
     `yaml.github-actions.security.run-shell-injection` hardening pattern)

**Acceptance criteria:**
- `nix run .#release` on a sample action repo produces both the GH release
  and the action.yml file at the repo root
- The composite action.yml is valid per `actionlint`
- Consumers using `uses: pleme-io/sample-action@v1` see the binary download
  + execution work on Linux x86_64, Linux aarch64, macOS x86_64, macOS aarch64
- Attestation hashes from forge propagate into the GH release notes

**Dependencies:** Phase 2 (the substrate emits the action.yml that calls
into `pleme-actions-shared`-conforming binaries; needs the shared crate's
input/output convention to be stable).

---

## Phase 4 — Rendering pipeline + repo-forge `rust-action` archetype

**Files:**
- `arch-synthesizer/src/action_domain/render.rs` — completes the render targets started in Phase 1 with the actual file content (boilerplate sections per render target)
- `pleme-io/repo-forge/archetypes.lisp` — add `(defarchetype "rust-action" …)` form referencing the Action domain
- `pleme-io/repo-forge/archetypes/rust-action/` — template directory with:
  - `Cargo.toml.template`
  - `flake.nix.template` (consumes `rust-action-release-flake.nix` from Phase 3)
  - `action.yml.template` (driven by the Action domain's render output)
  - `.github/workflows/release.yml.template`
  - `.github/workflows/ci.yml.template`
  - `README.md.template`
  - `LICENSE` (MIT, fixed)
  - `.envrc`, `.gitignore` (standard pleme-io)

The archetype's template files reference variables that the Action
domain's render layer fills in. Per repo-forge's principle 3 (file
taxonomy), `Cargo.toml`, `flake.nix`, action.yml, and the workflows are
**boilerplate** (fully derivable from the typed Action value); `src/`
and `tests/` are **authored** (operator-owned).

**Acceptance criteria:**
- `repo-forge new --archetype rust-action --var name=sample-action --path /tmp/sample-action`
  produces a working repo with the right content for a stub Action declaration
- `cd /tmp/sample-action && nix flake check` succeeds
- `repo-forge migrate /tmp/sample-action` is idempotent (no diff after one apply)
- Drift detection (modify a managed file, run check) flags the modification
- Modifying the catalog's `(defaction …)` form + re-running migrate updates
  the boilerplate to match (proves the typescape ↔ filesystem round-trip)

**Dependencies:** Phases 1 + 3.

---

## Phase 5 — First action: `terragrunt-apply` (was Phase 4)

**Files:**
- `pleme-io/pleme-actions-shared/Cargo.toml`
- `pleme-io/pleme-actions-shared/src/lib.rs`
- `pleme-io/pleme-actions-shared/src/input.rs` — typed `Input<T: Deserialize>` reads from the workflow's env vars (GitHub Actions sets `INPUT_<NAME>` for each input)
- `pleme-io/pleme-actions-shared/src/output.rs` — typed `Output<T: Serialize>` writes to `$GITHUB_OUTPUT`
- `pleme-io/pleme-actions-shared/src/summary.rs` — `StepSummary::open()` / `.section(…)` / `.finalize()` builders writing to `$GITHUB_STEP_SUMMARY`
- `pleme-io/pleme-actions-shared/src/error.rs` — `ActionError` with `::error()` / `::warning()` / `::notice()` matching GitHub's annotation format
- `pleme-io/pleme-actions-shared/src/log.rs` — structured logging that respects `RUNNER_DEBUG=1`

Published to crates.io as `pleme-actions-shared` so per-action repos can
consume via crates.io rather than git deps (faster builds, version pinning).

**Acceptance criteria:**
- Unit tests cover: input parsing happy path + missing required + wrong type
- A trivial sample action consuming the shared crate compiles + runs in a
  GitHub-hosted runner with the expected behavior
- Cross-platform binaries pass smoke tests on all 4 targets

**Dependencies:** none (parallel with phase 1+2). The shared crate informs
what the archetype's template Cargo.toml depends on, so authoring this
first slightly de-risks phase 2.

---

## Phase 4 — First action: `terragrunt-apply`

**Repo:** `pleme-io/terragrunt-apply`

**Inputs:**
- `working-directory` (string, required): leaf directory to apply from
- `action` (enum: `plan` | `apply` | `destroy`, default `plan`): mode
- `auto-approve` (bool, default `true` for apply/destroy, ignored for plan)
- `terragrunt-version` (string, default `0.71.5`): version to install
- `tofu-version` (string, default `1.10.6`): OpenTofu version to install
- `tf-vars` (map<string, string>, default `{}`): passed as `TF_VAR_*` env vars
- `aws-region` (string, optional): if set, runs `aws sts get-caller-identity` first as a sanity check
- `update-kubeconfig` (string, optional): EKS cluster name; if set, runs `aws eks update-kubeconfig`

**Outputs:**
- `plan-summary` (string): plan additions/changes/destroys count, formatted for `$GITHUB_STEP_SUMMARY`
- `state-version` (string): tofu state version after the operation
- `applied-resources` (json array): list of resource addresses that changed (apply only)

**Behavior:**
- Installs OpenTofu + Terragrunt at pinned versions
- Pre-flight: validates TF_VAR_* secrets are present (no leak)
- Optional: AWS identity check, kubeconfig update
- Runs `terragrunt --non-interactive <action>` in the leaf
- Captures structured output, writes to step summary, exposes via outputs

**Acceptance criteria:**
- `cargo test` covers the happy path + each input validation
- The action binary runs locally outside CI (operator can use it for
  ad-hoc applies)
- `uses: pleme-io/terragrunt-apply@v1` in a test workflow produces
  equivalent behavior to a hand-rolled inline shell step against the same
  terragrunt leaf

**Dependencies:** Phases 1, 2, 3, 4.

---

## Phase 6 — `k8s-pause-verify` + `helmworks-render-check` (was Phase 5)

**`k8s-pause-verify`**

Inputs: namespace, runner-set-label, expected-pod-count (default 0), kubectl-context (optional).

Outputs: pod-count, listener-status, configmap-paused-flag.

Behavior: queries the cluster via kubectl, asserts pause invariants, fails
with structured error if violated.

**`helmworks-render-check`**

Inputs: chart (e.g. `pleme-arc-runner-pool`), version, values-file, expected-resources (list of `Kind/name`), expected-violations (list of contract names).

Outputs: rendered-yaml-path, resource-count, contract-pass-bool.

Behavior: pulls chart from OCI, renders with values, asserts expected
resources + contract violations match expectations.

**Acceptance criteria for both:** same as terragrunt-apply, plus:
- The pattern of "thin Rust binary + composite action.yml" is now exercised
  on three distinct domains (TF, K8s, Helm)
- The shared crate `pleme-actions-shared` covers all three without
  domain-specific bloat
- Versions ship as `v0.1.0` initially, with a clear path to `v1.0.0` once
  a real consumer's stability needs lock the API

**Dependencies:** Phase 5 proves the pattern; these confirm it generalizes.

---

## Phase 7 — Migrate the first downstream consumer (was Phase 6)

Any existing pleme-io-fleet workflow that runs `terragrunt apply` + a
paused-K8s-deploy verification step is a candidate. The first migration is
chosen by whichever workflow happens to need the next maintenance touch
after Phase 5 lands.

Typical migration shape — ~200 lines of inline YAML drops to ~30 by
replacing inline terragrunt + kubectl steps with:

```yaml
- uses: pleme-io/terragrunt-apply@v1
  with:
    working-directory: <leaf-path>
    action: ${{ inputs.action }}
    aws-region: <region>
    tf-vars: |
      <key>=${{ secrets.<secret-name> }}
- uses: pleme-io/k8s-pause-verify@v1
  if: inputs.action == 'apply'
  with:
    namespace: <pool-namespace>
    runner-set-label: <pool-label>
    expected-pod-count: 0
```

**Acceptance criteria:**
- Workflow line count drops by >80%
- Behavior parity with the pre-migration workflow (verified by re-running
  the consumer's existing test plan)
- The consumer pins to a specific action major version (`@v1`) — proves
  the API contract is stable enough for production use

**Dependencies:** Phases 5, 6.

---

## Phase 8 — `pleme-actions` index repo (was Phase 7)

**Repo:** `pleme-io/pleme-actions`

Pure docs. Contents:
- `README.md`: catalog of published actions with one-line descriptions, links to each repo
- `VERSIONS.md`: cross-action version compatibility matrix
- `CONTRIBUTING.md`: how to add a new action (links to repo-forge skill + this plan)
- `ATTESTATION.md`: how to verify a published action's binary against the source commit

No code. The repo's job is consumer discoverability.

**Acceptance criteria:**
- The README serves as the entry point a potential consumer reads first
- Each published action repo's README links back to the index
- The version compat matrix is auto-generated (or at least auto-checkable)
  from the catalog

**Dependencies:** Phases 5–7 (need 3+ published actions to be meaningfully
worth indexing).

---

## Phase 9 — First public release announcement (optional, was Phase 8)

If we want broader pleme-io ecosystem adoption (rio's CI, customer onboards,
public OSS), write a short blog post on `pleme-io.github.io` (or wherever
public-facing content lives) describing the pattern, linking to the index
repo, and inviting consumers.

Skip if the pattern stays internal-only.

---

## Dependencies + critical path

```
Phase 0
  │
  ├──→ Phase 1 (Action domain)         ──┐
  │                                      ├──→ Phase 4 (rendering + archetype) ──→ Phase 5 (terragrunt-apply) ──→ Phase 7 (migrate consumer)
  └──→ Phase 2 (shared crate) ──→ Phase 3 (substrate) ──┘                                                       │
                                                                                                                ├──→ Phase 6 (other 2 actions) ──→ Phase 8 (index)
                                                                                                                                              └──→ Phase 9 (announce)
```

Phases 1 and 2 are independent and can run in parallel. Phase 3 depends
on Phase 2 (the substrate emits action.yml that calls binaries conforming
to the shared crate's input/output convention). Phase 4 needs both 1 and
3. Phase 5 (the first published action) gates everything downstream.

---

## Risks + mitigations

| Risk | Mitigation |
|---|---|
| Cross-platform binary download flaky on some runners | Pre-test on all 4 OS/arch combos as part of Phase 1 acceptance criteria |
| GitHub Actions API changes the input passing convention | Inputs come from the `INPUT_<NAME>` env vars, which is a documented contract. We pin our toolkit shim to a known shape. |
| Catalog drift across N repos | repo-forge migrate is idempotent + drift-detecting. Re-running across the catalog snaps everything back to spec. |
| Per-action versioning gets complicated | Major-version stability is a hard guarantee. We tolerate occasional minor-version churn during early-life (`v0.x.y`) but lock down at `v1.0.0`. |
| Consumers pin to `@main` and break on every commit | Document hard rule against `@main`; add a CI check on consumer repos that flags it. |

---

## Status updates

This file is updated after every phase milestone. Cross-link to relevant
PRs, commits, and decision records as they land.
