# Pangea ↔ magma orchestration — typed functions, fluid composition,
# state-organization migration

> **Frame.** [`THEORY.md`](./THEORY.md) §I (Pillars) names Pangea Ruby
> as the IaC declaration layer (Pillar 5) and magma as the typed
> execution runtime ([`MAGMA.md`](./MAGMA.md)).
> [`MAGMA.md` §II.9](./MAGMA.md#ii9-in-memory-pipelines--shigoto-work-graph)
> introduces magma's `Workspace` + `WorkspaceChain` typed primitives —
> the in-memory work-graph below the Pangea Ruby layer.
> [`MAGMA.md` §II.10](./MAGMA.md#ii10-substrate-for-pangea-api--the-future-product-layer)
> names pangea-api as the future product consuming magma as a library.
>
> **This doc is the typed surface where Pangea meets magma.** Pangea
> stops being a renderer that emits Terraform JSON and starts being a
> typed orchestration substrate: every operation an operator wants to
> express — *deploy this; split that workspace into two; move this VPC
> from state-A to state-B; reconcile A→B→C in topological order;
> migrate the entire seph cluster's state-organization without recreate
> — is a typed function call against a typed primitive.
>
> **Status.** Draft v1. Pre-implementation for everything beyond the
> bootstrap (`Pangea::Backend` + `Pangea::Magma` modules in pangea-core
> exist today; the surface in this doc extends them). Names every
> primitive *before* the code; the Ruby implementation that follows
> compiles against this surface, not the other way around.

---

## I. The frame

Pangea Ruby today is a **renderer**: `pangea synth template.rb` emits
Terraform JSON, then `pangea plan/apply` shells out to tofu (or to
magma, once the backend is selected per [`MAGMA.md`](./MAGMA.md) §II.11).
The DSL is expressive about resource shapes but silent on
*operations between workspaces* — those happen at the operator's shell
prompt, in `pangea-architectures/workspaces/<name>/flake.nix` files,
or in `nix/fleet.yaml` orchestration graphs. Cross-workspace state
flow happens via `data "terraform_remote_state"` (a disk-bound Terraform
escape hatch). Migrations between workspaces happen via `pangea
state mv` ceremony + bash glue.

**Magma's typed runtime changes the floor.** With `magma::pangea`'s
`Workspace` + `WorkspaceChain` (§II.9), the runtime understands typed
values flowing between workspaces, topologically-ordered
reconciliation, in-memory DAGs. The Ruby layer on top can — and
should — surface these capabilities as typed Pangea functions.

This doc names the typed surface that makes Pangea + magma a unified
orchestration substrate. **Once the surface exists, "deploy X
workspaces in order" stops being prose; it becomes a typed function
call against typed values.**

---

## II. The destination

### II.1. Six typed primitives

Pangea's magma orchestration surface adds **six** typed primitives on
top of the existing renderer:

```
pangea-core/lib/pangea/magma/
├── workspace.rb       # Pangea::Magma::Workspace   (declared workspace + I/O slots)
├── chain.rb           # Pangea::Magma::Chain       (typed DAG of workspaces)
├── migration.rb       # Pangea::Magma::Migration   (state-organization moves)
├── distribution.rb    # Pangea::Magma::Distribution (split/merge/collocate strategy)
├── optimization.rb    # Pangea::Magma::Optimization (instantiation hints)
└── orchestrator.rb    # Pangea::Magma::Orchestrator (the top-level: composes the above)
```

Each is a typed Ruby Struct or class with declared fields. Each maps
1:1 to a magma-side typed primitive (or to a sequence of magma
operations). Authors compose them in Pangea Ruby; pangea-core delegates
execution to magma via `Pangea::Magma.verify_workspace` /
`magma flow run` / `magma migrate` (new) / `magma state mv` /
`magma chain run` (new).

### II.2. Mapping to magma primitives

| Pangea primitive (Ruby) | magma primitive (Rust) | Concrete magma binary call |
|---|---|---|
| `Pangea::Magma::Workspace` | `magma_pangea::workspace::Workspace` | `magma plan/apply` against the workspace dir |
| `Pangea::Magma::Chain` | `magma_pangea::chain::WorkspaceChain` | `magma flow run <chain.json>` |
| `Pangea::Magma::Migration` | new — `magma_migrate::Migration` | `magma migrate <plan.json>` (new subcommand) |
| `Pangea::Magma::Distribution` | new — `magma_pangea::distribution::Distribution` | implicit via Chain construction; no separate call |
| `Pangea::Magma::Optimization` | new — `magma_pangea::optimization::Optimizer` | influences Chain.reconcile_all scheduling + caching |
| `Pangea::Magma::Orchestrator` | the umbrella — Chain + Migration + Distribution composed | top-level operator interface |

The Rust-side new crates (`magma-migrate`, `magma-pangea::distribution`,
`magma-pangea::optimization`) are M0.1 deliverables; the typed Ruby
surface declares them now so the operator surface stays stable as
implementations land.

---

## III. The typed surface

### III.1. `Pangea::Magma::Workspace`

A typed workspace declaration. Wraps a Pangea Ruby `template :name do
... end` form + declares typed I/O slots so the orchestrator can wire
cross-workspace flows.

```ruby
Pangea::Magma::Workspace.declare(
  name:       :seph_vpc,
  template:   'seph_vpc.rb',
  workspace_dir: 'workspaces/seph-vpc',
  inputs: {
    cidr_block: { type: String, default: '10.50.0.0/16' },
    region:     { type: String, default: 'us-east-1' },
  },
  outputs: {
    vpc_id:        { type: String, sensitive: false },
    public_subnet_id:  { type: String },
    private_subnet_id: { type: String },
  },
  requires: {
    feature:      :in_memory_pipeline,
    input_format: 'terraform-json',
  },
)
```

The `requires` block hooks into `Pangea::Backend.verify_compatible!`
(per [`MAGMA.md`](./MAGMA.md) §II.11) — declaring a workspace's typed
requirements fails fast at orchestration build time if the chosen
backend can't satisfy them.

### III.2. `Pangea::Magma::Chain`

Typed DAG of `Workspace`s + cross-workspace edges. Mirrors
`magma_pangea::chain::WorkspaceChain`.

```ruby
chain = Pangea::Magma::Chain.build do |c|
  c.workspace seph_vpc
  c.workspace seph_cluster
  c.workspace seph_fluxcd

  # Cross-workspace edge: vpc's vpc_id → cluster's vpc_id input
  c.edge from: seph_vpc,     output: :vpc_id,
         to:   seph_cluster, input:  :vpc_id

  # Cluster's kubeconfig → fluxcd's kubeconfig
  c.edge from: seph_cluster, output: :kubeconfig,
         to:   seph_fluxcd,  input:  :kubeconfig
end

chain.reconcile_all(external_inputs: {})   # full DAG run
chain.reconcile_subset([:seph_fluxcd], stub_upstream_outputs: { ... })  # isolation
```

Reconciliation delegates to `magma flow run` — `Pangea::Magma::Chain`
serializes the typed DAG to a flow.json the magma CLI consumes, then
parses the typed report back. The Ruby side never re-implements the
work-graph; magma owns the execution.

### III.3. `Pangea::Magma::Migration`

The load-bearing operation this doc names: **typed state-organization
migration**. Move resources from one workspace's state to another
without recreate, preserving resource identity (UUIDs, ARNs, etc.).

```ruby
# Migrate the cluster IAM role from k3s-permissions → platform-iam,
# preserving the role's ARN + tags.
migration = Pangea::Magma::Migration.declare(
  from: :k3s_permissions,
  to:   :platform_iam,
  resources: [
    { kind: :managed,
      address: 'aws_iam_role.k3s_node',
      new_address: 'aws_iam_role.platform_k3s_node' },
    { kind: :managed,
      address: 'aws_iam_role_policy_attachment.k3s_node_*',
      new_address_pattern: 'aws_iam_role_policy_attachment.platform_k3s_node_*' },
  ],
  preserve: [
    :resource_identity,   # don't destroy/recreate
    :tags,
    :dependent_resources, # resources depending on these stay attached
  ],
  dry_run: true,
)

# Returns a typed MigrationPlan; operator inspects, approves, applies.
plan = migration.plan
plan.summary    # → "1 resource moved; 4 policy attachments renamed; no recreates"
plan.apply!     # → drives `magma migrate <plan.json>` against both workspaces
```

Migration semantics:
- **`:resource_identity`** — preserve the cloud resource (no destroy).
  Implemented via `magma state mv` between workspaces' state files
  rather than terraform `taint` + plan + apply.
- **`:tags`** — preserve tag set even if Pangea Ruby renders different
  ones in the new location (tags are operator-managed metadata).
- **`:dependent_resources`** — resources NOT in the migration set that
  depend on the moved resources have their dependency references
  rewritten to point at the new location.

The Migration is a typed value; `plan.apply!` is the only side-effect
boundary. Dry runs return the typed `MigrationPlan` for inspection.

### III.4. `Pangea::Magma::Distribution`

Strategy for distributing architecture pieces across workspaces.
Operators express **what should live together** and **what should
live apart**; magma's orchestrator implements the distribution by
constructing the appropriate `WorkspaceChain`.

```ruby
distribution = Pangea::Magma::Distribution.declare(
  strategy: :tier_separation,
  tiers: {
    network:  [:seph_vpc, :seph_dns, :seph_lb],
    cluster:  [:seph_iam, :seph_k3s, :seph_kms],
    workload: [:seph_fluxcd, :seph_observability],
  },
  edges: {
    network  => cluster,        # cluster depends on network
    cluster  => workload,       # workload depends on cluster
  },
  placement: {
    network:  { region: 'us-east-1', state_backend: 's3://pleme-dev/seph-network' },
    cluster:  { region: 'us-east-1', state_backend: 's3://pleme-dev/seph-cluster' },
    workload: { region: 'us-east-1', state_backend: 's3://pleme-dev/seph-workload' },
  },
)

chain = distribution.to_chain
```

Built-in strategies:
- **`:tier_separation`** — workspaces grouped by tier (network /
  cluster / workload / observability); each tier has its own state file.
- **`:single_workspace`** — collapse everything into one workspace
  (the "monolith" mode; not recommended above ~50 resources but
  supported for small projects).
- **`:per_provider`** — one workspace per provider (aws, cloudflare,
  akeyless, etc.). Cross-provider workspaces depend on outputs.
- **`:custom`** — operator provides the partition function directly.

The strategy choice doesn't change what resources are deployed — only
where they live in state. Operators can `migrate` between strategies
via `Pangea::Magma::Migration`.

### III.5. `Pangea::Magma::Optimization`

Typed hints magma uses to optimize instantiation. Not a constraint —
a suggestion the orchestrator respects when it can.

```ruby
opt = Pangea::Magma::Optimization.declare(
  reconcile_strategy: :parallel_per_wave,  # default — Kahn's algorithm
  state_caching:      :memoize_per_chain,  # don't re-read state per workspace
  schema_caching:     :persistent,          # cache provider schemas across chain runs
  apply_concurrency:  4,                   # max parallel resource applies
  retry_policy: {
    transient_errors: { max_attempts: 3, backoff: :exponential },
    quota_errors:     { max_attempts: 1, backoff: :linear, base: 60 },
  },
)

chain.reconcile_all(optimization: opt)
```

These map to magma's `shigoto::Scheduler` config (per §II.9). The
typed surface lets operators declare orchestration policy
declaratively; magma enforces the policy at runtime.

### III.6. `Pangea::Magma::Orchestrator`

The umbrella primitive — composes everything above. The operator's
top-level interface.

```ruby
orch = Pangea::Magma::Orchestrator.new(
  distribution: distribution,
  optimization: opt,
  attestation:  { enabled: true, signing_key: cofre.ref('magma/operator-signing-key') },
)

# Common operations on the typed orchestrator:
orch.deploy!                            # plan + apply across the entire distribution
orch.deploy!(only: [:seph_vpc])         # subset deploy
orch.migrate!(migration)                # apply a typed Migration plan
orch.diff(target_distribution: other)   # what changes if we re-distribute?
orch.attestation_chain                  # the tameshi receipt chain across all workspaces
```

Orchestrator owns:
- **Plan + apply across the full distribution** — every workspace's
  Chain reconciled in topological order; one logical operation.
- **Subset operations** — deploy or migrate just a slice.
- **Cross-distribution diffs** — "what would change if we moved from
  `:tier_separation` to `:per_provider`?"
- **Attestation aggregation** — receipts from every workspace stitched
  into a single audit chain.

---

## IV. The four operations

The typed surface above unlocks **four canonical operations** that
weren't expressible before — each maps to a typed function on the
orchestrator.

### IV.1. Split

Take one workspace's resources and divide them across N new workspaces
without recreate.

```ruby
split = Pangea::Magma::Split.declare(
  source: :seph_monolith,
  targets: {
    seph_network:  { selector: ->(r) { r.type_id.start_with?('aws_vpc', 'aws_subnet') } },
    seph_cluster:  { selector: ->(r) { r.type_id.start_with?('aws_iam_role', 'aws_kms_key') } },
    seph_workload: { selector: ->(r) { r.type_id.start_with?('kubernetes_') } },
  },
)
split.apply!
```

Result: `seph_monolith`'s state file is empty; three new state files
populated. No cloud resources recreated. The operator can now manage
each tier independently.

### IV.2. Merge

Inverse: consolidate N workspaces' resources into one.

```ruby
merge = Pangea::Magma::Merge.declare(
  sources: [:seph_network, :seph_cluster, :seph_workload],
  target:  :seph_monolith,
)
merge.apply!
```

### IV.3. Migrate (state-organization)

Already declared in III.3 — move resources between workspaces
preserving identity. The most surgical of the four operations.

### IV.4. Compose

Inverse of split-at-application-time: deploy multiple workspaces as if
they were one logical unit, with cross-workspace outputs flowing
in-memory (no S3 state file roundtrips per §II.9).

```ruby
chain = Pangea::Magma::Chain.compose(
  [:seph_network, :seph_cluster, :seph_workload],
  output_propagation: :in_memory,
)
chain.reconcile_all
```

Compose ≠ Merge: compose runs N typed workspaces in coordinated
sequence; merge actually moves resources between state files. Compose
is the everyday case; merge is the rare structural change.

---

## V. Migration semantics in detail

Migrations are the load-bearing surgical operation — they preserve
cloud resource identity while changing the state organization. Five
guarantees:

| Guarantee | How magma enforces |
|---|---|
| **No recreate** | `magma migrate` uses `state mv` semantics; the cloud resource (and its ARN/UUID/CRN) is unchanged |
| **No data loss** | Validation pass before any state mutation; refuses to proceed if a managed resource would be orphaned (in neither source nor target state) |
| **Dependent updates** | Resources NOT in the migration set whose dependencies are migrated have their references rewritten — references stay valid post-migration |
| **Attestation continuity** | Pre- and post-migration tameshi receipts; the migration itself emits a typed `MigrationReceipt` linking the two |
| **Atomicity** | Either every declared resource migrates, or none does. The operator never lands in a half-migrated state |

### V.1. Migration plan shape

```ruby
plan = migration.plan

plan.actions
# [
#   <Migration::Action :move,
#     source_state: 'k3s-permissions',
#     target_state: 'platform-iam',
#     address:      'aws_iam_role.k3s_node',
#     new_address:  'aws_iam_role.platform_k3s_node',
#     identity:     'arn:aws:iam::123:role/k3s-node-role'>,
#   <Migration::Action :rewrite_reference,
#     in_workspace: 'seph_cluster',
#     reference:    '${aws_iam_role.k3s_node.arn}',
#     new_reference: '${data.terraform_remote_state.platform_iam.outputs.platform_k3s_node_arn}'>,
#   ...
# ]

plan.invariants_satisfied?  # → true if validation pass succeeds
plan.would_recreate         # → [] if no resource recreates
plan.attestation_continuity # → { pre: <hash>, post_predicted: <hash> }
```

### V.2. Cross-state-backend migrations

A migration can move resources across workspaces that live in
different state backends (different S3 buckets, different remote
state stores). The orchestrator coordinates the multi-backend
operation atomically using a two-phase commit pattern:

1. **Prepare phase** — copy source state's resource records into target
   state under the new addresses. Source state still has them too.
2. **Validate phase** — both states are consistent. If anything is
   off, abort + roll back the prepare.
3. **Commit phase** — remove resource records from source state. The
   migration is final.
4. **Rewrite phase** — update dependent workspaces' references.

Failures during commit raise typed `MigrationStallError`; the
operator can `migration.resume!` from the recorded checkpoint. No
manual `terraform state` ceremony required.

---

## VI. The tooling surface

The typed primitives above are usable from **every interface magma
exposes** (per [`MAGMA.md`](./MAGMA.md) §II.8):

| Interface | How to use Pangea::Magma::Orchestrator |
|---|---|
| **CLI** | `pangea orchestrate <distribution.rb>` (new subcommand in pangea-core) |
| **Ruby library** | direct API calls (`orch.deploy!`, `orch.migrate!`, …) |
| **MCP** | new tools: `pangea_orchestrate`, `pangea_migrate_dry_run`, `pangea_split`, `pangea_merge` |
| **magma CLI** | `magma migrate <plan.json>` consumes the typed Migration plan emitted by Ruby |
| **tatara-lisp** | `(defpangea-orchestrator …)` form compiles to the Ruby surface |

### VI.1. New magma CLI subcommands (M0.1)

| Subcommand | What it does |
|---|---|
| `magma migrate <plan.json>` | Apply a typed Migration plan against two or more state files atomically |
| `magma split <plan.json>` | Apply a typed Split plan |
| `magma merge <plan.json>` | Apply a typed Merge plan |
| `magma chain run <chain.json>` | Already exists as `magma flow run` — alias for clarity |
| `magma orch <orchestrator.json>` | Top-level — execute an entire Orchestrator declaration |

Each subcommand reads a typed JSON document (the serialized
Pangea::Magma::* value) and drives the appropriate magma library calls.

### VI.2. New magma-pangea Rust crates / modules (M0.1)

| Module | Owns |
|---|---|
| `magma_pangea::distribution` | Distribution + Tier + Placement typed primitives |
| `magma_pangea::optimization` | Optimization + RetryPolicy + Schedule typed primitives |
| `magma_migrate` (new crate) | Migration + Split + Merge + cross-state-backend coordinator |
| `magma_pangea::orchestrator` | Composition primitive — wires the rest |

Each typed Rust value is `Serialize + Deserialize` so the Ruby side's
plan-emit path is just `Pangea::Magma::Migration#to_json → magma
migrate <stdin>`.

---

## VII. Compounding implications

This typed surface compounds across the substrate:

1. **Every pangea-* gem** can declare typed migrations + distributions
   instead of operators writing bash. New gem releases unlock new
   typed primitives.
2. **pangea-architectures** stops being a flat list of workspaces and
   becomes a typed orchestration corpus — each `architectures/*.rb`
   declares its inputs/outputs/requires; chains compose them.
3. **pangea-operator** consumes `Pangea::Magma::Orchestrator` directly
   — Kubernetes CRDs reify the same typed primitives, no schema drift
   between operator-CRD-shape and operator-Ruby-shape.
4. **pangea-api** (per [`MAGMA.md`](./MAGMA.md) §II.10) — the future
   product surface — exposes Orchestrator as its public REST/GraphQL
   shape. The typed Ruby primitives are the Pangea side; the typed
   Rust primitives are the magma side; the REST schema sits between
   them.
5. **Existing pangea-architectures workspaces** auto-gain typed
   I/O slots via inference from their existing rendered JSON; a
   migration pass can promote them.

The same primitives serve the operator (CLI), the AI agent (MCP), the
product (pangea-api), and the substrate (Ruby + Rust libraries).
**One typed surface, every interface.**

---

## VIII. Phases

| Phase | Status | Scope |
|---|---|---|
| **M0.0** | ✅ shipped | `Pangea::Backend` + `Pangea::Magma` exist in pangea-core (theory/MAGMA.md §II.11). magma-arch-test crate exposes the typed harness used by `Pangea::Magma.verify_workspace`. |
| **M0.1** | ✅ shipped | `Pangea::Magma::Workspace` + `Chain` Ruby types live in pangea-core. `Chain#reconcile_all` drives `magma flow run` via Pangea::Magma::Runner. Integration spec (`magma_chain_integration_spec.rb`) verifies topo order + output propagation end-to-end. |
| **M0.2** | ✅ shipped | `Pangea::Magma::Migration` + `magma migrate` subcommand + `magma-migrate` Rust crate. Atomic two-phase commit, BLAKE3 receipts pre/post. 5 unit tests cover atomic move + validation + dry-run. |
| **M0.3** | ✅ shipped | `magma split` + `magma merge` CLI subcommands as typed wrappers over `magma migrate`. `merge` auto-enumerates source state. |
| **M0.4** | ✅ shipped | `Pangea::Magma::Optimization` typed hints (strategy / max_concurrency / retries / timeout_ms) plumb through Chain → flow.json → magma-cli's FlowFile. Honored by shigoto-wrapped apply (planned). |
| **M0.5** | ✅ shipped | `Pangea::Magma::Orchestrator` umbrella + `pangea orchestrate` CLI subcommand with `--backend`, `--only`, `--dry-run` flags. |
| **M0.6** | ✅ shipped | MCP tools `pangea_orchestrate` / `magma_migrate_dry_run` / `magma_migrate` / `magma_split` / `magma_merge`, dispatched in-process (no shelling out). Destructive ones gated by `confirm:true`. `magma capabilities` reports 17 typed MCP tools. |
| **M0.7** | ✅ shipped (minimal) | `magma flow` accepts `.lisp` / `.tatara` / `.scm` via `tatara_lite` reader. `(deforch …)` compiles to FlowFile JSON. 3 proptest invariants × 48 cases each lock in the round-trip. |
| **M0.8** | 🚧 in progress | pangea-architectures sweep — typed `Pangea::Magma::Workspace` declarations alongside templates. Five workspaces promoted (k3s-permissions, k3s-cluster, akeyless-dev-cluster, nix-builders, cloudflare-pleme × 6 templates). |
| **M0.9** | ✅ shipped | **Compounding refactor.** Extracted `magma-flow` shared crate (FlowFile / topological_order / run engine — eliminates duplication between magma-cli and magma-mcp). Extracted `Pangea::Magma::Runner` shared subprocess helper. Added `Pangea::Magma::Stack.build` high-level helper (auto-derives cross-tier edges by output↔input name matching). Added `Pangea::Magma::TestSupport` shared rspec scaffolding. Default `requires:` set in Workspace.declare. `magma_plan` MCP tool now dispatches in-process. |

Each shipped phase carries rspec + Rust integration coverage. By
M0.8 every existing pangea-architectures workspace will declare
typed inputs/outputs; pangea-architectures' rspec suite uses
`Pangea::Magma::Matchers` and `Pangea::Magma::TestSupport` for
cross-workspace assertions.

### The compounding surface today

Operators reach for one of these four authoring forms, in order of
ascending leverage:

1. **`Pangea::Magma::Workspace.declare(…)`** — typed I/O contract
   for a single state boundary. One per `.rb` template.
2. **`Pangea::Magma::Chain.build do |c| c.workspace …; c.edge … end`** —
   explicit cross-workspace wiring. Use when edges aren't pure
   name-matches.
3. **`Pangea::Magma::Stack.build(name:, tiers:, optimization:, attestation:)`** —
   high-level. Auto-derives cross-tier typed edges from upstream
   outputs ↔ downstream inputs sharing a name. The convention every
   pleme-io workspace pair already follows; Stack codifies it.
4. **`(deforch :name … :workspaces (…) :edges (…))`** — tatara-lisp
   surface. Same FlowFile JSON; alternate authoring path.

All four compile mechanically to the same canonical FlowFile shape
consumed by the single `magma_flow::run` engine. The four interfaces
(CLI, MCP, Rust library, tatara-lisp) all dispatch into the same
engine — no possibility of drift between authoring surface and
execution.

---

## IX. Anti-patterns this surface forbids

- **Manual `terraform state mv` ceremony** for cross-workspace
  migrations. Use `Pangea::Magma::Migration` — typed, dry-runnable,
  atomic, attestation-continuous.
- **Cross-workspace state via `data "terraform_remote_state"`** in
  Pangea-Ruby authoring. Use `Pangea::Magma::Chain` edges — typed-value
  flow, no S3 roundtrip per consumer (§II.9).
- **Bash glue scripts** invoking `pangea apply` in sequence across N
  workspaces. Use `Pangea::Magma::Orchestrator.deploy!` — single
  typed operation, single attestation chain.
- **Inline `nix run .#deploy-A; nix run .#deploy-B; nix run .#deploy-C`**
  in fleet.yaml. Authors a `Pangea::Magma::Chain` in Ruby; substrate's
  `pangea-arch-workspace.nix` produces ONE `.#deploy-chain` app that
  consumes the typed chain.
- **Re-implementing `WorkspaceChain` in Ruby**. Pangea-Ruby's chain is
  a typed declaration — execution delegates to `magma flow run`.
  Ruby never re-implements the work-graph.

---

## X. Open questions

1. **State-backend portability** — when migrating between S3 buckets
   or HTTP-backed states, who owns the temporary atomic-staging area?
   Likely: a `magma migrate stage` intermediate area in the source
   bucket, with the commit phase consisting of writes to both
   destinations + a delete from the staging area. Needs an RFC.
2. **Resource-graph inference** — can magma auto-infer dependencies
   when migrating between workspaces? Today the operator declares
   them. M0.x: derive from the rendered JSON's `${...}` references.
3. **Concurrent multi-orchestrator** — what happens when two
   operators run `pangea orchestrate` against the same distribution
   simultaneously? Need a typed lock (`magma_state::lock` extension
   to the orchestrator level).
4. **Backwards compat** — operators who hand-author Terraform JSON
   today. Bridge: `Pangea::Magma::Workspace.import_tf_json(path)`
   produces a typed Workspace from arbitrary rendered JSON. Workspaces
   imported this way have only inferred I/O slots.
5. **Naming for the next-tier abstraction** — beyond `Orchestrator`,
   is there a "fleet" or "deployment" primitive that composes many
   orchestrators? Likely yes; deferred to a sibling doc once `M0.5`
   lands.

---

## XI. How this doc evolves

- Update phases (§VIII) per milestone. Move "next" → "shipped" with
  the commit reference.
- Resolved open questions move to a "resolved questions" appendix —
  never delete-without-trace, so the reasoning chain is auditable.
- Cross-references to [`MAGMA.md`](./MAGMA.md) stay current — each
  primitive in §II refers back to its magma counterpart. When
  magma's primitive evolves, this doc updates in the same commit.
- pangea-core's `lib/pangea/magma/` directory matches §II.1 exactly.
  The doc names; the code implements. Drift between the two is a bug
  in the code, not the doc.
