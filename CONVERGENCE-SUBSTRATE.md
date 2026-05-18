# Convergence Substrate

# The universal reconciler — pangea-operator as a relentless state-convergence engine for any declarative API

> **Frame.** Terraform/OpenTofu encodes the diff-then-apply pattern
> for cloud IaC. Kubernetes encodes it for cluster state.
> The pattern is universal: anywhere you have `(read current state,
> diff against desired state, apply diff)` semantics, you have a
> reconciler. magma gives us the in-process substrate; this doc
> describes how pangea-operator becomes the universal convergence
> engine that runs reconcilers across the entire declarative
> ecosystem — not just terraform-shaped state.

---

## I. The pattern is universal

Every system with declarative state lives behind the same three
operations:

```
read_state()   →  current_state : opaque shape
diff(desired, current)  →  plan : { creates, updates, deletes }
apply(plan)    →  outcome { applied[], failed[] }
```

Terraform is one instance. Kubernetes is another. The same shape
appears in dozens of systems we already touch every day:

| Domain                  | Read state                 | Apply diff                       |
|-------------------------|----------------------------|----------------------------------|
| **Cloud IaC**           | tofu refresh / providers   | tofu apply (or magma)            |
| **Kubernetes**          | kubectl get                | kubectl apply                    |
| **GitHub**              | gh repo view               | gh repo edit, branch protection  |
| **DNS**                 | Cloudflare/Route53 list    | record create/update             |
| **Vault / Akeyless**    | list policies/roles        | put policy / role                |
| **Helm**                | helm get values            | helm upgrade                     |
| **Database schemas**    | information_schema         | ALTER / CREATE                   |
| **PostgreSQL grants**   | pg_catalog                 | GRANT / REVOKE                   |
| **Snowflake datasets**  | SHOW TABLES                | CREATE OR REPLACE                |
| **Slack channels**      | conversations.list         | conversations.invite             |
| **PagerDuty schedules** | schedules API              | schedule update                  |
| **Datadog monitors**    | monitors list              | monitor create/update            |
| **Grafana dashboards**  | dashboard search           | dashboard upsert                 |
| **Linear / Jira**       | list projects/teams        | project update                   |
| **Auth0 / Okta**        | applications/groups        | application update               |
| **Stripe products**     | list products              | product update                   |
| **Service mesh (Istio)** | get VirtualService        | apply VirtualService             |
| **API gateway (Kong)**  | routes list                | route create                     |
| **Helmworks releases**  | get release                | upgrade release                  |
| **Filesystem state**    | stat / ls                  | write / chmod (NixOS-style)      |
| **Cron schedules**      | crontab -l                 | crontab install                  |
| **Argo Workflows**      | get workflow               | submit workflow                  |
| **Browser state**       | cookies / localStorage     | set cookie / localStorage (E2E)  |

The patterns differ — REST, gRPC, SQL, file I/O, kube-rs — but the
shape is identical. The operator-side seam should be one typed trait
that hosts all of them.

---

## II. The substrate

### II.1. The trait

```rust
#[async_trait]
pub trait Reconciler: Send + Sync {
    /// Stable name for the reconciler kind ("terraform", "github", "dns").
    fn kind(&self) -> &'static str;

    /// Read the current observed state. Opaque JSON shape; each
    /// impl owns its own typed projection internally.
    async fn read_state(&self) -> Result<Value, ReconcilerError>;

    /// Compute the typed Plan from desired config + current state.
    /// Pure function — no I/O. Determinism is a trait law.
    fn compute_plan(
        &self,
        config: &Value,
        state:  &Value,
    ) -> Result<Plan, ReconcilerError>;

    /// Apply a plan. Returns a typed Outcome with applied[]/failed[].
    async fn apply(&self, plan: &Plan) -> Result<Outcome, ReconcilerError>;

    /// Detect drift = compute_plan(config, read_state()).
    /// Default impl provided.
    async fn detect_drift(&self, config: &Value) -> Result<Plan, ReconcilerError> {
        let state = self.read_state().await?;
        self.compute_plan(config, &state)
    }
}
```

Universal `Plan`, `Outcome`, `Change`, `Action`, `Severity` types
live in `magma-converge`. They're shaped to be JSON-friendly so any
reconciler can plug in.

### II.2. The trait laws

Every implementation must obey:

1. **`read_state` is referentially transparent** modulo external
   change. Two reads of an unmodified system return the same JSON.
2. **`compute_plan` is deterministic.** Same `(config, state)` →
   same `Plan` (and hence same BLAKE3 plan_id).
3. **Empty plans are no-ops.** `apply(noop_plan)` doesn't touch
   external state; `read_state` afterwards is unchanged.
4. **Apply converges.** After `apply(plan)`, `read_state` reflects
   the changes; `compute_plan(config, read_state())` is empty (or
   the diff is well-justified by external drift).

These are testable via proptest. Every magma-shipped reconciler has
a `trait_laws_obeyed` suite that runs against it.

### II.3. The engine

```rust
pub struct ConvergeEngine {
    reconcilers: HashMap<String, Arc<dyn Reconciler>>,
}

impl ConvergeEngine {
    pub async fn reconcile(&self, kind: &str, config: &Value)
        -> Result<Outcome, EngineError>
    { ... }

    pub async fn drift_sweep(&self) -> Vec<DriftReport>
    { ... }
}
```

The engine is what pangea-operator instantiates. Each CR specifies
`kind: terraform` / `kind: github` / `kind: dns` / etc.; the engine
routes to the right reconciler.

---

## III. Concrete reconcilers

### III.1. The four that ship today

| Reconciler                | Purpose                                                  | Status      |
|---------------------------|----------------------------------------------------------|-------------|
| **`TerraformReconciler`** | Wraps magma-plan + magma-apply (the existing path)       | shipping    |
| **`InMemoryKvReconciler`** | KV-style state, mock for tests + a reference impl       | shipping    |
| **`GithubRepoReconciler`** | GitHub repo settings + branch protection (mock client)  | shipping    |
| **`DnsRecordReconciler`**  | DNS records (provider-agnostic; list-then-diff)         | shipping    |

Each demonstrates a DIFFERENT API style:

- **Terraform** — provider-protocol-based (gRPC plugins, typed states)
- **InMemoryKv** — pure functions over a HashMap (the testbed)
- **GitHub** — REST + paginated CRUD
- **DNS** — list/upsert/delete with composite primary keys

Together these four prove the trait surface fits the four major API
shapes the operator will encounter.

### III.2. The next wave (M0.x candidates)

Each follows the same pattern; the listed ones are pre-mapped:

- `VaultPolicyReconciler` — Vault/Akeyless policies + roles
- `HelmReleaseReconciler` — Helm releases (chart + values vs deployed)
- `K8sNativeReconciler` — kube-rs based; reconciles any kind given a CRD/group
- `SlackChannelReconciler` — channels + members + integrations
- `DatadogMonitorReconciler` — monitors + SLOs + dashboards
- `GrafanaDashboardReconciler` — dashboards + folders + alert rules
- `PgSchemaReconciler` — extensions, roles, grants, schemas
- `StripeProductReconciler` — products, prices, webhooks
- `Auth0AppReconciler` — applications + groups + rules
- `IstioReconciler` — service mesh routing rules
- `KongReconciler` — API gateway routes/plugins/consumers
- `PagerDutyScheduleReconciler` — schedules + escalation policies
- `LinearProjectReconciler` — projects + teams + custom fields
- `ArgoWorkflowReconciler` — Argo Workflows (declarative DAGs)
- `NixFileReconciler` — declarative filesystem (NixOS-style)
- `BrowserStateReconciler` — cookies/localStorage (E2E test infra)
- `CronReconciler` — operator's own scheduled tasks
- `EmailRuleReconciler` — mailbox filters / forwarding
- `MusicLibraryReconciler` — Spotify playlists / last.fm tags (for fun)

The dream: any system you can describe declaratively gets a typed
reconciler. The substrate is shared.

### III.3. The composition: chains across reconciler kinds

`PangeaChain` (M0.13 CRD) already declares typed flows across
workspaces. With the convergence substrate, a chain doesn't have to
be terraform-only:

```yaml
apiVersion: pangea.io/v1
kind: PangeaChain
metadata:
  name: prod-deploy
spec:
  steps:
    - kind: terraform        # provision AWS infra
      ref: prod-vpc-tf
    - kind: helm             # deploy app via helm
      ref: prod-app
      depends_on: [prod-vpc-tf]
    - kind: dns              # update routing
      ref: prod-routing
      depends_on: [prod-app]
    - kind: github           # create release record
      ref: app-release
      depends_on: [prod-app]
    - kind: slack            # announce
      ref: ops-notification
      depends_on: [prod-routing, app-release]
```

One typed chain. One pod. One BLAKE3 attestation chain across all 5
reconcilers. No subprocess. No cross-system race conditions because
each step's outputs feed downstream inputs via in-heap values.

---

## IV. Compounding patterns the substrate unlocks

### IV.1. Drift detection as a primitive (`magma-drift`)

A typed `Plan` is just data. `magma-drift`:

- Classifies each `Change` by severity (`Cosmetic` / `Functional`
  / `Critical`)
- Applies typed policies (`AutoCorrect` / `Alert` / `RequireApproval`)
- Emits typed `DriftEvent` for downstream consumers (K8s events,
  prometheus metrics, audit log)
- Computes BLAKE3 hashes over the drift signature so consecutive
  observations of the same drift can be deduped

Every reconciler kind gets drift detection for free.

### IV.2. Speculative execution (`magma-spec`)

Magma's plan is a pure function. So:

- `plan_against_synthetic(config, fake_state)` — what would the
  plan look like if state were X?
- `compare_plans(a, b)` — diff two plans (environmental
  comparison: dev vs prod)
- `what_if(state_mutation)` — apply a hypothetical mutation, then
  plan
- `forecast(history, future_t)` — given drift history, project
  future plans

These power IDE-level UX: "show me what would change", "preview
this PR's effect", "compare against last week's state".

### IV.3. Plan-as-data pipeline (`magma-stream`)

A typed plan is JSON-shaped. Therefore:

- Stream plans into a graph database for impact analysis
- Run plans through a policy engine (OPA, Cedar) BEFORE apply
- Feed plans into LLM reasoners for natural-language summaries
- Hash plans into a Merkle tree for cryptographic audit
- Pipe plans into K8s events for real-time observability
- Compare plans across environments (drift across dev/staging/prod)

### IV.4. Reverse-direction reconciliation (`magma-discover`)

Instead of "declare, apply", run the loop backwards:

- Scan existing AWS account → emit Pangea Ruby (or YAML config)
- Detect undeclared resources, surface as "untracked"
- Generate import statements for adoption
- Continuous discovery (new resource appears → operator notifies)

Every reconciler kind can support discovery if it can list
resources. The `Reconciler` trait gets an optional
`discover() -> Vec<Value>` method.

### IV.5. State-machine semantics over apply (`magma-fsm`)

Each apply transitions between phases. With magma's typed control:

```
Idle → Planning → Approving → Applying → Verifying → Stable
                          ↓
                       Failed → Retrying → Stable
                          ↓
                       Failed (permanent)
```

Each transition is typed; transitions have time bounds (stuck >5m →
escalate); the state machine is resumable across operator restarts
via BLAKE3 plan_id in CR status.

### IV.6. Cross-cluster federation

Multiple pangea-operator instances coordinating via:

- A global state registry (each cluster has a region in the
  registry)
- Cross-cluster typed references (cluster A's VPC output consumed
  by cluster B's peering)
- Federation control plane brokering cross-cluster plans

This is how a multi-region fleet becomes a single typed system.

### IV.7. AI-assisted reconciliation

With typed plans + magma in-process:

- LLM-generated suggestions ("this is security-relevant because…")
- Anomaly detection ("this plan looks unusual vs historical")
- Auto-generated runbooks for failed applies
- Natural-language → Pangea Ruby (already prototyped via magnus)

### IV.8. Reconciliation as substrate for higher-order systems

- **Cost optimization** — plan diffs become cost diffs; optimize
- **Compliance enforcement** — every plan checked against typed
  policies
- **Security posture management** — drift in SG/IAM/encryption →
  alert
- **Capacity planning** — forecast cluster size based on apply
  history
- **Closed-loop autoscaling** — operator reads metrics → generates
  plan → applies (sub-second cycle because magma is in-process)
- **Disaster recovery** — typed state snapshots, BLAKE3-attested
  re-applies, cross-region failover

---

## V. The vision in one sentence

> **pangea-operator becomes the universal control plane: any system
> with declarative state gets a typed Reconciler, every change is
> BLAKE3-attested, every apply is in-process, every cross-system
> dependency is a typed edge in a flow, and every drift is
> classified, policy-evaluated, and either auto-corrected,
> alerted, or queued for approval — at K8s reconcile cadence,
> indefinitely, on one pod, with one binary, with one substrate.**

The substrate is magma. The engine is the operator. The reconcilers
are pluggable. The pattern is universal.

---

## VI. Phases shipping in this round

| Crate                | Scope                                                | Status      |
|----------------------|------------------------------------------------------|-------------|
| `magma-converge`     | `Reconciler` trait + 4 concrete impls + engine       | shipping    |
| `magma-drift`        | Typed drift classification + policies                | shipping    |
| `magma-spec`         | Speculative execution + plan diffing                 | shipping    |
| `magma-discover`     | Reverse reconciliation (cloud → typed state)         | planned     |
| `magma-stream`       | Plan-as-data pipeline (events, audit, metrics)       | planned     |
| `magma-fsm`          | Explicit state machine over apply lifecycle          | planned     |

Each ships with proptest invariants exercising the trait laws.
Together they make the convergence substrate real.

---

## VII. Test discipline

The trait has laws. Every reconciler proves them via proptest:

1. **`read_state_is_referentially_transparent`** — two reads of
   the same unmodified system return equal JSON.
2. **`compute_plan_is_deterministic`** — `compute_plan(c, s)`
   produces equal plans (modulo `created_at`) and equal `plan_id`
   on repeated invocation.
3. **`empty_plan_is_a_noop_when_applied`** — `apply(empty_plan)`
   doesn't mutate observed state.
4. **`apply_converges`** — for any `(config, state)`,
   `apply(compute_plan(c, s)).then(read_state())` produces a
   state where `compute_plan(c, read_state())` has no changes.
5. **`plan_id_changes_with_config`** — different configs against
   the same state produce different `plan_id`s (when actually
   different).

These are property tests, not unit tests. Each impl runs ≥48 cases
per property. A reconciler that fails any law isn't a reconciler.

---

## VIII. Anti-patterns this surface forbids

- **Hand-rolled diff/apply loops in controllers** — use a
  `Reconciler` impl; if a new shape doesn't fit, extend the trait.
- **State stored outside the engine** — every reconciler's state
  is accessible via `read_state`; nothing hides observable state.
- **Side effects in `compute_plan`** — `compute_plan` is pure;
  side effects belong to `apply`.
- **Manual cross-system coordination** — use `PangeaChain` to
  declare the DAG; the engine handles ordering + retries.
- **Untyped error propagation** — every reconciler emits typed
  `ReconcilerError`; controllers route based on error class.
- **Non-deterministic plans** — `compute_plan` MUST be
  deterministic. Use `chrono::Utc::now()` only for the `created_at`
  field, never for content.
- **Bypassing drift policies** — every apply goes through the
  drift-classification pipeline; high-severity changes can't
  silently land.

---

## IX. How this doc evolves

- Each shipped reconciler kind moves into §III.1 with a commit ref.
- Each compounding pattern in §IV migrates into its own RFC once
  designed.
- The `Vision` in §V is canonical; the implementation phases below
  it are the path. The destination doesn't move; the phases do.

---

## X. Why this is the right substrate-level move

Three reasons:

1. **It compounds.** Every reconciler added widens the operator's
   reach with zero additional substrate work. The same engine, the
   same drift detection, the same typed flow, the same BLAKE3
   attestation — all of it shared. Per Compounding Directive
   Principle 1: solve once, in one place, at one time.

2. **It's load-bearing.** The operator already does this implicitly
   for terraform. Making the pattern explicit + typed means the
   next 20 domains become 20 thin reconciler impls, not 20
   bespoke controllers each reinventing diff/apply.

3. **It generalizes magma's win.** magma's in-process plan/apply
   is currently terraform-shaped. With the convergence substrate,
   magma's in-process advantage applies to ANY declarative API.
   The operator becomes 100x faster at GitHub reconciles, DNS
   reconciles, Slack reconciles — all because none of them require
   forking a subprocess.

The substrate is the unfair advantage. Per CSE: the description IS
the algebra; the operator is the renderer.
