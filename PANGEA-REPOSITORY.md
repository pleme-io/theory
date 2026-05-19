# PangeaRepository — Continuous Reconciliation Primitive

Status: ★★★ canonical (destination locked; M0 lands today)
Implementing crate: `pleme-io/magma::magma-repo`
Related: [`MAGMA-AS-PLATFORM.md`](./MAGMA-AS-PLATFORM.md), [`MAGMA.md`](./MAGMA.md), [`MAGMA-OPERATOR-BACKEND.md`](./MAGMA-OPERATOR-BACKEND.md), [`TESTING-SUBSTRATE.md`](./TESTING-SUBSTRATE.md), [`CONVERGENCE-SUBSTRATE.md`](./CONVERGENCE-SUBSTRATE.md)

## I. Thesis

**A `PangeaRepository` is a typed primitive that watches a Git source for a directory shaped like `pangea-architectures` + continuously reconciles every workspace described in its `pangea.yml`.** Pangea-operator subscribes to one or more PangeaRepository primitives; each becomes a self-driving reconcile loop that:

* Pulls the latest source from Git (clone first time, fetch+reset later)
* Parses `pangea.yml` (root + per-workspace) into typed configs
* Discovers workspaces declared by the repo shape
* Orders them by `depends_on` / `fleet.yaml` flow edges
* Runs each workspace through `MagmaExecutor` — preflight laws, drift classification, plan checkpoint, bundle attestation, audit chain
* Self-bootstraps from artifacts the repo ships (`Gemfile.lock`, `gemset.nix`, magma binary attestations) so the operator pod needs zero pre-baked Pangea state

`pangea.yml` is the single source of truth. Everything the operator needs to reconcile is encoded there + the typed primitives `magma-repo` reads from the directory structure surrounding it.

## II. Why this primitive

Today's gap: pangea-operator's MagmaExecutor knows how to reconcile **a workspace** (one rendered Terraform JSON + one state file). It doesn't know:

* How to discover what workspaces a Git repo declares
* How to fetch + bootstrap the repo on the operator pod
* How to order multi-workspace reconciles by typed `depends_on`
* How to watch the source for changes + drive a continuous loop
* How to attest the entire repo closure (not just a workspace's gem tree)

`PangeaRepository` is the missing primitive. Once it lands:

* A pangea-operator CR is just `{ git: { url, branch } }`
* The operator pulls, discovers workspaces, reconciles in dep order
* Every reconcile bundle carries `repo_attestation` (commit SHA + tree hash) alongside `gem_tree_attestation`
* Drift → auto-remediation per `DriftPolicy` runs continuously without human intervention
* New workspaces show up just by committing to the repo; deleted ones get destroyed automatically per policy

## III. The typed surface

```rust
pub struct PangeaRepository {
    pub source:      Source,                  // Local path | Git URL | GitHub repo
    pub branch:      String,                  // default "main"
    pub poll_interval: Duration,              // M0 polling; future webhook
    pub reconcile_policy: ReconcilePolicy,
    pub bootstrap:   BootstrapPolicy,         // how to set up the working dir
}

pub enum Source {
    Local { path: PathBuf },
    Git   { url: String, ref_: Reference },   // commit/tag/branch
    GitHub { owner: String, repo: String, ref_: Reference },
}

pub struct RootPangeaConfig {
    pub tags:       BTreeMap<String, String>,
    pub accounts:   BTreeMap<String, AccountConfig>,
    pub sso:        Option<SsoConfig>,
    pub state:      Option<StateBackendConfig>,
    pub cascade:    Option<CascadeConfig>,
    pub namespaces: BTreeMap<String, NamespaceConfig>,
}

pub struct WorkspacePangeaConfig {
    pub default_namespace: String,
    pub account:           Option<String>,
    pub tags:              BTreeMap<String, String>,
    pub namespaces:        BTreeMap<String, NamespaceConfig>,
    pub depends_on:        Vec<String>,
}

pub struct DiscoveredRepo {
    pub root_config:  RootPangeaConfig,
    pub workspaces:   Vec<DiscoveredWorkspace>,
    pub fleet:        Option<FleetConfig>,
    pub bootstrap:    BootstrapArtifacts,    // Gemfile.lock, gemset.nix, etc.
    pub commit_sha:   Option<String>,        // git rev-parse HEAD
    pub repo_attestation: String,            // BLAKE3 of the closure
}

pub struct DiscoveredWorkspace {
    pub name:     String,
    pub dir:      PathBuf,
    pub config:   WorkspacePangeaConfig,
    pub template: Option<PathBuf>,
}
```

The `PangeaRepoReconciler` impl:

```rust
impl Reconciler for PangeaRepoReconciler {
    fn kind(&self) -> &'static str { "pangea_repo" }

    async fn read_state(&self) -> Result<Value, ReconcilerError> {
        // Return: { commit_sha, workspaces: [name → last_bundle_id], … }
    }

    fn compute_plan(&self, config: &Value, state: &Value) -> Result<Plan, ReconcilerError> {
        // Walk workspaces, classify drift per workspace as a single Change
    }

    async fn apply(&self, plan: &Plan) -> Result<Outcome, ReconcilerError> {
        // For each Change: invoke MagmaExecutor for that workspace
    }
}
```

The reconciler plugs into the existing `magma-controller` surface — same FSM, same bundle attestation, same audit chain. The compounding effect: every operator-level primitive in the substrate (`magma-test-laws`, `magma-drift`, `magma-fsm`, `magma-stream`, `magma-bundle`) applies to a whole repository, not just a single workspace.

## IV. Bootstrap policy

`pangea.yml` declares what the operator needs to set up its working directory:

```yaml
bootstrap:
  gemfile:        Gemfile         # parsed by magma-rubygems M1
  gemfile_lock:   Gemfile.lock    # parsed + attested by magma-rubygems M0
  gemset_nix:    gemset.nix       # consumed by magma-nix M8 (future)
  magma_binary:   bin/magma       # optional; verified via tameshi attestation
  pangea_version: "0.3.0"         # required pangea-core gem version
```

Operator startup flow:

1. Watcher detects new repo (or initial setup)
2. Clone source to a tmpfs working dir
3. Parse `pangea.yml` → typed `RootPangeaConfig`
4. Read `Gemfile.lock` → compute `gem_tree_attestation` via magma-rubygems M0
5. (Future M3) Materialize gem tree in-memory from `gemset.nix`
6. (Future M5) Construct embedded CRuby env pointing at the materialized tree
7. Walk workspaces → for each, invoke `MagmaExecutor::plan` + `apply`
8. Every reconcile bundle carries: `repo_attestation` + `gem_tree_attestation` + `plan_id` + `bundle_id`
9. Loop continuously at `poll_interval`

The bootstrap closure is itself attested: the operator stores the commit SHA + the BLAKE3 of the rendered closure so two operators reconciling the same commit produce byte-identical bundles.

## V. ReconcilePolicy — continuous self-driving

```rust
pub struct ReconcilePolicy {
    /// What to do when a workspace fails plan/apply.
    pub on_failure:        FailureAction,
    /// What to do when DriftPolicy::evaluate returns RequireApproval.
    pub on_approval:       ApprovalAction,
    /// Cap on parallel workspace reconciles (per repo).
    pub max_concurrent:    usize,
    /// Soft deadline per full repo reconcile cycle.
    pub cycle_deadline:    Duration,
    /// Retry policy when a Git fetch fails.
    pub git_retry:         magma_budget::RetryPolicy,
}

pub enum FailureAction {
    /// Halt the loop until human acks.
    HaltWithAlert,
    /// Continue reconciling other workspaces; flag this one.
    SkipAndContinue,
    /// Retry per RetryPolicy.
    Retry,
}

pub enum ApprovalAction {
    /// Open a typed approval request (K8s CR or external).
    Block,
    /// Auto-approve (use only for trusted classes — Pillar 0 safety).
    AutoApprove,
}
```

Combined with the existing `magma-drift::DriftPolicy`, this gives operators four configurable behaviors:

| Drift severity | Default DriftDecision | Default action |
|---|---|---|
| Cosmetic | AutoCorrect | apply silently |
| Functional | AutoCorrectWithAlert | apply + emit K8s Event |
| Critical | RequireApproval | block per ApprovalAction |

## VI. Git + GitHub integration

`Source::Git` covers any `git clone` URL. `Source::GitHub` adds GitHub-specific niceties:

* Authenticated API calls via `GITHUB_TOKEN` (operator secret)
* Webhook receive endpoint (future) — pushes wake the reconcile loop instantly instead of waiting for `poll_interval`
* Pull request preview reconciles (future) — render plan against the PR HEAD, comment back

M0 ships polling only (every `poll_interval` seconds, fetch + reset). Webhook + PR preview are M1+.

## VII. Test surface

Per the existing testing substrate, magma-repo gains:

1. **Reconciler trait laws** — `PangeaRepoReconciler` passes `magma_test_laws::assert_all_laws` (read_state idempotent, plan deterministic, apply converges, …).
2. **Typed pangea.yml parse round-trip** — every Pangea workspace's pangea.yml parses + re-emits structurally identical.
3. **Discovery snapshot** — `discover(real_pangea_architectures_path)` returns the expected workspaces in stable order.
4. **Cross-fixture exercise** — magma-test integration runs discovery over the actual pangea-architectures clone and verifies workspace count + name set.
5. **Repo attestation determinism** — same commit → same `repo_attestation`.

## VIII. Operator integration

`pangea-operator` gains a new CRD: `PangeaRepository`. The controller:

1. Watches for `PangeaRepository` CRs
2. Constructs a `magma_repo::PangeaRepoReconciler` per CR
3. Wraps it in `BudgetedReconciler` + `ReconcileController`
4. Schedules continuous reconciles at `poll_interval`
5. Emits CR status updates after each cycle (workspace count, last bundle_ids, drift counts)
6. Surfaces `RequireApproval` decisions via K8s Events + (future) approval CRs

CR shape:

```yaml
apiVersion: pangea.pleme.io/v1
kind: PangeaRepository
metadata:
  name: pangea-architectures-prod
spec:
  source:
    github:
      owner: pleme-io
      repo:  pangea-architectures
      ref:   main
  pollInterval: 60s
  reconcilePolicy:
    onFailure: SkipAndContinue
    onApproval: Block
    maxConcurrent: 4
    cycleDeadline: 30m
  driftPolicy:
    fallback: RequireApproval
status:
  lastCommitSha: "a1b2c3..."
  lastReconciledAt: "2026-05-19T18:00:00Z"
  workspaces:
    - name: seph-vpc
      bundleId: "abc..."
      phase: Stable
    - name: cluster
      bundleId: "def..."
      phase: Stable
```

## IX. Milestones

| Milestone | What lands | Status |
|---|---|---|
| M0 | Typed pangea.yml parser + repo discovery | started today |
| M0.5 | PangeaRepoReconciler stub (Reconciler trait impl) | started today |
| M1 | Polling Git watcher (clone/fetch/reset) | ⬜ |
| M2 | ReconcilePolicy threading + per-workspace MagmaExecutor invocation | ⬜ |
| M3 | repo_attestation in Bundle (BLAKE3 of commit + tree) | ⬜ |
| M4 | PangeaRepository CRD + operator controller wiring | ⬜ |
| M5 | GitHub webhook intake (replace polling for connected repos) | ⬜ |
| M6 | PR preview reconciles (synth a plan against the PR HEAD) | ⬜ |

## X. Compounding cycle

Every existing primitive applies to a repository the moment magma-repo lands:

* `magma-test-laws::preflight` runs over every workspace in the repo
* `magma-drift::classify` routes per-workspace anomalies through one DriftPolicy
* `magma-fsm` tracks the full repo-reconcile cycle (Planning → Applying-N-workspaces → Verifying → Stable)
* `magma-bundle` emits one bundle per workspace + an aggregate per cycle
* `magma-stream` chains the audit events across the whole cycle
* `magma-rubygems` attests the gem closure for every workspace
* `magma-controller` wraps the lot with retry budget + concurrency limit

The operator becomes a **continuous self-driving reconciliation engine** for entire Pangea repositories. Per the user directive: *"a continuous process in the operator that is always trying to achieve state and automatically handle issues."*
