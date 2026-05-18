# magma — the Rust-native OpenTofu-compatible IaC executor

> **Frame.** [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> says every recurring shape becomes a typed substrate primitive, proven by
> construction. [`THEORY.md`](./THEORY.md) §I.1 names Rust + tatara-lisp +
> WASM/WASI as Pillar 1 and Pangea Ruby → Terraform JSON as Pillar 5.
> [`TERRENO.md`](./TERRENO.md) and [`PANGEA-OPERATOR.md`](./PANGEA-OPERATOR.md)
> own the *declaration* layer of Pillar 5 (how infrastructure is shaped in
> Pangea Ruby and how the operator consumes it). [`CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md`](./CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md)
> already says the same constructive rule for the K8s-control-plane half of
> Pillar 7. **This doc owns the missing half of Pillar 5: the _execution_
> layer.** The Rust binary that consumes Pangea's rendered Terraform JSON
> (or any HCL2) and realizes it on cloud substrate via the Terraform
> provider protocol — fully compatible with the existing v5/v6 provider
> ecosystem, fully pleme-io-idiomatic internally.
>
> The doc is normative. Every pleme-io workspace that today shells out to
> `tofu`/`terraform` migrates to `magma`. Third-party providers (OpenTofu /
> Terraform protocol v5/v6) load and execute unchanged — the compatibility
> contract is byte-exact at the surfaces listed in §II.2. `skip-magma:` as
> the first line of a workspace's `pangea.yml` defers migration; the only
> sanctioned long-term deviation is `akeylesslabs/*` Terragrunt workspaces
> (pleme-io/CLAUDE.md ★ Pangea-always exception).
>
> **Status.** Draft v1. Pre-implementation. Destination locked first per
> pleme-io/CLAUDE.md Operating Principle #0 ("write the destination before
> the plan"); the M0–M6 phases in §VI are the path-down, not the goal —
> time-pressure deviation never collapses §II–V. Bootstrap consumer is
> `pangea-architectures/workspaces/*` (the existing pleme-io workspaces that
> shell out to tofu today); promotion to fleet default in M6 is gated on
> bit-exact state-file round-trip + plan-diff parity against `tofu plan` on
> the full pleme-io workspace corpus.
>
> **Name.** Brazilian-Portuguese per the [`THEORY.md` §II naming
> convention](./THEORY.md). Pangea (the supercontinent) declares the shape
> of the world; magma is the molten executive force beneath the crust that
> *realizes* that shape on the physical substrate. The metaphor maps 1:1
> onto the architecture — declaration up top in Pangea Ruby, execution
> below in magma, cloud providers as the substrate that takes shape. Pairs
> naturally with `forja` (forge shapes metal in CI/CD; magma shapes earth
> in IaC). See §V for the full naming rationale.

---

## I. The repeating pattern

Today every pleme-io workspace looks like this:

```
Pangea Ruby DSL ─► Terraform JSON ─► [opentofu binary] ─► cloud
                                         │
                                         ├─ resolve providers from registry
                                         ├─ speak tfplugin5/6 gRPC
                                         ├─ build resource DAG
                                         ├─ diff state vs config
                                         ├─ apply in dep order
                                         └─ rewrite state + lock file
```

We render a typed plan and then hand it to a 1.3 MLOC Go binary we don't
control. Eight failure modes from that trust:

| # | Failure mode | Today | What pleme-io loses |
|---|---|---|---|
| 1 | Plan-diff non-determinism | tofu quirks leak into every pleme-io workspace | proof discipline (Pillar 10) |
| 2 | State-file races | tofu locking is best-effort + DynamoDB | typed concurrency control |
| 3 | Provider crash recovery | tofu retries opaquely | typed retry policy (shigoto) |
| 4 | Function-library drift | OpenTofu vs Terraform diverge silently | runtime-semantics drift |
| 5 | HCL parse edge cases | tofu's parser has its own bugs | typed AST guarantees |
| 6 | Lock-file rewriting | tofu mutates state + lock out-of-band | typed audit trail |
| 7 | Telemetry / observability | tofu's stdout is unstructured prose | pleme-io's typed event stream |
| 8 | Attestation | tofu plans aren't BLAKE3-signed | tameshi integration |

The OpenTofu binary is a Go process that handles all eight ad-hoc. Naming
the executor as a typed pleme-io primitive lets the substrate carry these
concerns *once*. Same shape as `shigoto` (work-graph), `nix-ast` (Nix
emission), `crossplane-forge` (Go provider emission): every recurring
shape becomes a typed primitive, owned in one place, consumed everywhere.

The pattern is universal — Pulumi has a Go engine; cdktf has a synth +
shell-out shape; OpenTofu is the canonical instance. None expose the
execution layer as a typed primitive an operator can extend. Magma does.

---

## II. The destination

### II.1. One typed primitive — the magma crate workspace

```
pleme-io/magma                       (Cargo workspace, substrate's rust-workspace-release)
├── magma-types        — Plan, Resource, ResourceAddress, Action, Diff,
│                       State, Lock, ProviderSchema, …
├── magma-pangea       — Pangea Ruby DSL in-process evaluator (CRuby via
│                       magnus + pangea-ruby-eval) + Terraform JSON
│                       reader. Returns typed values; never touches disk
│                       in the in-memory chaining path (§II.9). The
│                       canonical M0 input layer.
├── magma-config       — Terraform JSON → magma-types::Config + small
│                       `${resource.attribute}` interpolation resolver
│                       (the narrow subset of HCL-expression evaluation
│                       that survives Pangea's render). No HCL parser.
├── magma-protocol     — tfplugin5.proto + tfplugin6.proto bindings
│                       (generated via prost-build; no hand-written gRPC)
├── magma-plugin       — go-plugin handshake, mTLS bootstrap, stdio framing,
│                       gRPC client lifecycle, sub-process management
├── magma-providers    — Provider discovery, download, lock-file, registry
│                       (OpenTofu + Terraform registry protocols)
├── magma-state        — terraform.tfstate v4 read/write, state migrations,
│                       state locking, sensitive-value redaction
├── magma-backend      — Backend trait + impls (local / s3 / http / consul /
│                       kubernetes / postgres / azurerm / gcs / remote)
├── magma-graph        — Resource DAG construction (petgraph) + wave
│                       planning — companion to shigoto::Dag at the
│                       operation level
├── magma-plan         — Plan algorithm: Config × State → []Action.
│                       Surfaces as a typed `shigoto::Job` (§II.9).
├── magma-apply        — Apply engine: drives provider RPC, updates state.
│                       Each resource change is a `shigoto::Job`; the
│                       resource graph is a `shigoto::Dag`; retries /
│                       budgets / audit / FSM-resumption-after-crash
│                       come from shigoto, not re-implemented here.
├── magma-attest       — BLAKE3 over (config + state + plan); tameshi
│                       receipt emission; tabeliao registration
├── magma-mcp          — MCP server (JSON-RPC 2.0 over stdin/stdout).
│                       Tools generated mechanically from magma-types
│                       schemas. Destructive operations gated behind
│                       explicit `confirm: true` parameter. Per §II.8
│                       interface 3.
├── magma-cli          — Binary; drop-in compat for `terraform`/`tofu`
│                       (argv[0] sensing, TF_* env vars, exit codes) +
│                       native magma subcommands + `magma mcp` launcher.
├── magma-test         — Compatibility test harness (Tier 1/2/3 × levels
│                       1–5 per §II.6) + mock provider binary for
│                       offline tests
└── magma              — Umbrella crate, re-exports the public surface.
                          Consumers pull `magma::*`, never sub-crates.
```

Pure-Rust above the protocol layer. The only `unsafe` is in `magma-plugin`
around the gRPC + TLS bootstrap (auditable, contained) and in
`magma-pangea`'s magnus `embed::init` (CRuby VM bootstrap; one call per
process). The only IO above `magma-protocol` and `magma-backend` lives
in consumers — types, parser, planner, state, apply engine are all
pure-Rust against typed inputs.

**No HCL parser.** Magma reads two input shapes only: Pangea Ruby
(in-process via magnus, the canonical M0 path) and Terraform JSON (the
fallback for non-Ruby environments). HCL2 syntax + the ~150-function
HCL library + HCL module sources are intentionally out of scope per
§IX anti-patterns — Pangea Ruby evaluates all HCL-equivalent
expressions at render time, leaving magma with flat resource definitions
plus the narrow `${aws_vpc.main.id}` style references resolved by
`magma-config`'s interpolation resolver.

The crate boundary is enforced: `magma-plan` depends on `magma-types`
+ `magma-config` + `magma-graph` + `magma-state`, but not on
`magma-apply` or `magma-cli`. This makes the planner unit-testable
against typed inputs without spawning subprocesses, and keeps the CLI
a thin façade over the typed core.

### II.2. The compatibility contract — what magma matches byte-for-byte

Byte-exactness is the gate. Every existing provider, every existing
state file, every existing module, every existing lock file must work
without modification. This is what makes magma drop-in for `tofu`/
`terraform`:

| # | Surface | Format | Source of truth in OpenTofu |
|---|---|---|---|
| 1 | Provider protocol v5 | gRPC; `tfplugin5.proto` | `internal/tfplugin5/` |
| 2 | Provider protocol v6 | gRPC; `tfplugin6.proto` | `internal/tfplugin6/` |
| 3 | go-plugin handshake | magic cookies + mTLS + stdio frame | `internal/plugin/` + `internal/plugin6/` |
| 4 | State file | JSON, schema version 4 | `internal/states/statefile/` |
| 5 | Lock file | HCL, `.terraform.lock.hcl` (emit-only via typed `HclValue` AST; **not parsed**) | `internal/depsfile/` |
| 6 | Pangea Ruby DSL (in-process) | CRuby via magnus + `pangea-ruby-eval` → typed `LoadedWorkspace` | pangea-operator's `pangea-ruby-eval` crate (the canonical pleme-io pattern) |
| 7 | Terraform JSON input | `*.tf.json` → typed `Config` via `magma-config` | OpenTofu `internal/configs/configload/` (JSON path only) |
| 8 | CLI surface | `init/plan/apply/destroy/state/import/workspace/output/show/refresh/taint/force-unlock/get/fmt/validate/console` + native additions (`mcp`, `daemon`, `watch`, `attest`, `config`, `flow`) | `cmd/opentofu/` |
| 9 | Backend protocols | s3/http/consul/k8s/postgres/azurerm/gcs/local/remote | `internal/backend/remote-state/*` |
| 10 | Provider registries | OpenTofu registry + Terraform registry | `internal/getproviders/` |
| 11 | Variable precedence | env > -var > -var-file > auto.tfvars > defaults | `internal/configs/named_values.go` |
| 12 | Plan JSON output | `magma show -json` | `internal/command/jsonplan/` |
| 13 | State JSON output | `magma show -json` for state | `internal/command/jsonstate/` |
| 14 | Sensitive / ephemeral / write-only marks | marks on cty values | `internal/lang/marks/` |
| 15 | TF environment-variable inventory | `TF_VAR_*`, `TF_DATA_DIR`, `TF_LOG`, `TF_LOG_PATH`, `TF_CLI_ARGS_*`, `TF_INPUT`, `TF_IN_AUTOMATION`, `TF_TOKEN_*`, `TF_PLUGIN_CACHE_DIR`, `TF_REGISTRY_DISCOVERY_RETRY`, `TF_REGISTRY_CLIENT_TIMEOUT`, `TF_WORKSPACE`, `TF_IGNORE` | OpenTofu `internal/command/cliconfig/` |

Rows **deliberately removed** vs the v1 spec:

| Removed | Reason |
|---|---|
| HCL2 syntax (full spec) | Magma never reads `.tf` files — Pangea Ruby + Terraform JSON only. See §IX anti-patterns. |
| Function library (150+ built-ins, exact semantics) | Same — no HCL evaluator means no HCL function library. The narrow `${...}` reference resolver in `magma-config` is the only HCL-expression-adjacent surface. |
| Module sources (local/registry/git/http/s3) | Pangea Ruby owns module composition; magma never sees module sources. The `module {}` BLOCK in rendered JSON is read for graph + output tracking, but not the SOURCE. |

Every line item gets a proptest harness in `magma-test` that drives
identical input through `magma` and `tofu`, asserts byte-equal output.
The first time `magma` and `tofu` disagree on any of these, we stop
shipping until we know which side is correct — and bug the divergent
side with evidence.

### II.3. Internal representation — pleme-io style end to end

**Types via shikumi.** Every config struct and runtime value derives
`Shikumi` ([`THEORY.md`](./THEORY.md) §II.5). The HCL AST nodes are
shikumi values. Plans, diffs, state, lock file are typed shikumi
values. The JSON wire format (state-file format compat) is *one
serialization* of shikumi values — typed in process, typed-on-disk,
typed-on-the-wire-to-providers.

**Lisp surface via tatara.** `(defmagma-module ...)` form maps 1:1 to
HCL2's module shape. HCL2 is already s-expression-shaped (blocks +
attributes); the Lisp surface is mostly trivial. Authors can write
tatara-lisp where convenient, HCL where required, JSON where Pangea
renders it — three syntaxes, one typed AST. Per
[`RUST-LISP-EMBEDDING.md`](./RUST-LISP-EMBEDDING.md): Rust hosts Lisp;
all three syntaxes parse into the same `magma-types::Config` value.

**WASI execution.** The magma binary ships as a WASI module via
`cargo-component` + the tatara WASM runtime ([`WASM-STACK.md`](./WASM-STACK.md)).
Providers stay native subprocesses — the gRPC protocol works across the
WASI boundary via the host's `wasi-sockets`. Plans execute deterministically
in the sandbox; deterministic output is the basis for attestation.

**Work-graph via shigoto.** The apply engine is a `shigoto::Scheduler`
consumer ([`SHIGOTO.md`](./SHIGOTO.md)). Each resource's CRUD becomes
a typed `shigoto::Job<Resource, ResourceState, ProviderError>`. The
resource dependency graph is a `shigoto::Dag`. Retries / budgets /
audit / FSM-resumption-after-crash are inherited from shigoto, not
re-implemented.

**Constructive emission via nix-ast peer.** Magma emits HCL (when it
emits — for `magma fmt`, `magma import` config generation, lock-file
serialization) via a typed `HclValue` AST, never `format!()` of HCL
syntax. Peer of [`NIX-AST.md`](./NIX-AST.md) — same rule, different
target language.

**Attestation via tameshi.** Every plan is BLAKE3-hashed (config +
state + computed plan); the hash is the plan's identity. `magma apply
--plan <hash>` requires a verified prior plan. Plans land in tabeliao
as receipts ([`docs/compliant-artifacts.md`](../docs/compliant-artifacts.md)).

This is the prime-directive payoff: magma is *not* a 1.3 MLOC bespoke
executor. It's the composition of typed pleme-io primitives (shikumi
types + tatara-lisp surface + shigoto work-graph + nix-ast-peer
emission + tameshi attestation + cofre secret materialization) at the
IaC-execution layer. The crate count is high (16); the per-crate code
volume is moderate; the shared primitives carry most of the weight.

### II.4. Where magma fits in the typescape

Per [`THEORY.md`](./THEORY.md) §III.1, the typescape is owned by
arch-synthesizer. Magma extends the typescape with one new dimension:
**executor primitives** (`Plan`, `Resource`, `Action`, `Diff`, `State`,
`Lock`, `ProviderSchema`).

The chain that today is:

```
Pangea Ruby DSL → Terraform JSON → opentofu binary → cloud
```

becomes:

```
Pangea Ruby DSL → arch-synthesizer typescape → magma-types Rust values → magma-apply → cloud
```

Three Rust-crate boundaries, all typed. The arch-synthesizer
`TerraformSynthesizer` is rewritten to emit `magma-types::Config`
directly (no JSON intermediary) for in-process flows; the JSON
intermediary is preserved for compat with non-pleme-io consumers and
for state-file persistence.

### II.5. The Rust + Lisp pattern at the IaC layer

Per [`THEORY.md`](./THEORY.md) §II.1, every typed domain follows the
Rust + Lisp pattern. Magma is no exception:

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, Shikumi)]
#[tatara(keyword = "defmagma-module")]
pub struct ModuleSpec {
    pub source:      ModuleSource,
    pub version:     Option<VersionConstraint>,
    pub inputs:      HashMap<String, Expr>,
    pub providers:   HashMap<String, ProviderReference>,
    pub count:       Option<Expr>,
    pub for_each:    Option<Expr>,
    pub depends_on:  Vec<ResourceAddress>,
}

pub fn register() {
    tatara_lisp::domain::register::<ModuleSpec>();
}
```

Authors write either of:

```clojure
(defmagma-module my-vpc
  :source "registry.terraform.io/terraform-aws-modules/vpc/aws"
  :version "~> 5.0"
  :inputs {:name "prod-vpc" :cidr "10.0.0.0/16"})
```

```hcl
module "my_vpc" {
  source  = "registry.terraform.io/terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "prod-vpc"
  cidr    = "10.0.0.0/16"
}
```

Both parse into the same `ModuleSpec`. Pangea Ruby renders JSON that
also parses into the same `ModuleSpec`. Three surfaces, one typed AST.

### II.6. Test corpus — the proof of compatibility

Compatibility is a **gate, not an aspiration**. "Fully compatible" means
every Tier 1 provider passes all five test levels including real-cloud
apply — not just schema-load smoke tests. The corpus is enumerable,
versioned, and stored in `magma-test`.

#### Three tiers of providers

**Tier 1 — pleme-io's own usage. M0 gate, every level.** These
providers are already in every pleme-io workspace; magma cannot ship
M0 without all of them passing all five test levels.

| Provider | Where used | Why it stresses the protocol |
|---|---|---|
| `hashicorp/aws` | pangea-aws, every cluster | the deepest surface — ~1100 resource types, nested-block edge cases, sensitive marks in IAM/secrets |
| `cloudflare/cloudflare` | cloudflare-pleme, every tunnel/zone | typed-list resources, computed-then-known IDs |
| `akeyless-community/akeyless` | akeyless-dev, akeyless-platform | pleme-io owns both sides — first to find regressions |
| `hashicorp/kubernetes` | every k3s cluster | manifest-mode + native-mode CRDs |
| `hashicorp/helm` | every k3s cluster | HelmRelease lifecycle, value templating |
| `hashicorp/google` | gcp-gke-cluster | GCP IAM + workload-identity edge cases |
| `hashicorp/azurerm` | azure-aks-cluster | Azure managed-identity surface |
| `datadog/datadog` | full-observability-stack, convergence-dashboard | provider-time pricing surface |
| `splunk/splunk` | splunk-log-pipeline | nested config blocks |
| `hetznercloud/hcloud` | hcloud-web-stack | small-cloud sanity baseline |
| `hashicorp/random` | every workspace | small surface; fast sanity |
| `hashicorp/null` | every workspace | provisioner-only path |
| `hashicorp/local` | every workspace | local fs read/write |
| `hashicorp/tls` | DNS / cert workspaces | crypto material generation |
| `hashicorp/external` | several workspaces | external data source |
| `hashicorp/archive` | a few workspaces | file packaging |
| `carlpett/sops` | sops-secrets workspace | OpenTofu-time secret resolution; small surface, load-bearing |

**Tier 2 — most-downloaded public providers. M2 gate, levels 1–4.**
Ecosystem credibility — magma must work for the wider Terraform /
OpenTofu user, not just for pleme-io's slice. Real-cloud apply
(level 5) is desired but not gating because CI cost dominates.

| Provider | Surface |
|---|---|
| `digitalocean/digitalocean` | popular starter cloud |
| `mongodb/mongodbatlas` | DB-as-a-service |
| `snowflake-labs/snowflake` | data warehouse |
| `integrations/github` | git platform mgmt |
| `gitlab/gitlab` | git platform mgmt; non-HashiCorp upstream |
| `oracle/oci` | OCI |
| `linode/linode` | small cloud |
| `vultr/vultr` | small cloud |
| `equinix/equinix` | bare-metal cloud |
| `auth0/auth0` | identity |
| `okta/okta` | identity |
| `pagerduty/pagerduty` | oncall surface |
| `newrelic/newrelic` | observability |
| `postgresql/postgresql` | DB schema |
| `mysql/mysql` | DB schema |
| `stripe/stripe` | payments — sensitive-mark edge case |

**Tier 3 — protocol-edge-case providers. Ongoing.** Providers chosen
specifically to exercise unusual protocol surfaces: deeply-nested
recursive types, unusual import semantics, sensitive marks flowing
through outputs, write-only attributes, ephemeral values. Curated, not
exhaustive. Lives in `magma-test/tests/protocol-edge-cases/`; the list
grows as we discover corners. Failing on a Tier 3 provider isn't a
release blocker — it's an issue filed against `magma-protocol` or
`magma-state` plus a regression-test pin.

#### Five test levels

Every Tier 1 provider passes all five at M0. Every Tier 2 provider
passes levels 1–4 at M2 (level 5 follows as CI capacity allows).
Tier 3 is targeted; levels per case.

| # | Level | What it proves | Cloud cost |
|---|---|---|---|
| 1 | **Smoke** | magma spawns the provider binary, completes the go-plugin handshake, reads its schema, terminates cleanly | none |
| 2 | **Schema diff vs tofu** | magma's loaded `ProviderSchema` is byte-equal to tofu's — every resource type, every attribute, every nested block | none |
| 3 | **Plan diff vs tofu** | given identical config + prior state + variables, `magma plan` produces byte-equal JSON to `tofu show -json` | none (mocked providers) |
| 4 | **State round-trip** | (a) a state file from `tofu apply` round-trips byte-equal through `magma show -json` and back; (b) a state file from `magma apply` round-trips byte-equal through `tofu show -json` and back | none (golden state corpus) |
| 5 | **Real-cloud apply** | `magma apply` against a real cloud account creates resources functionally indistinguishable from `tofu apply` (verified by existing InSpec profiles — `inspec-aws-k3s`, `inspec-akeyless`, future per-Tier-2-provider profiles) | yes |

Level 5 uses `pangea-jit-builders` for ephemeral test capacity; created
resources are tagged `magma-test=true` + `magma-test-run-id=<uuid>` and
torn down within 30 minutes by a TTL janitor. Cost capped per-run via
a budget gate in `magma-test`'s harness.

#### Test harness shape

`magma-test` is its own crate. Provider tests are written either as
hand-authored fixtures (for edge cases) or generated from provider
schemas (for the long tail):

```rust
#[magma_test(provider = "aws", level = Level::PlanDiff)]
fn aws_vpc_with_subnets() {
    let cfg   = include_str!("fixtures/aws_vpc_with_subnets.tf.json");
    let state = include_str!("fixtures/aws_vpc_with_subnets.tfstate");
    assert_plan_byte_equal_vs_tofu(cfg, state);
}

#[magma_test(provider = "aws", level = Level::Apply, tier = 1)]
async fn aws_s3_bucket_lifecycle() {
    let env = TestEnv::ephemeral_aws().await;
    let cfg = aws_s3_bucket_with_lifecycle(&env);
    let result = magma::apply(&cfg, &env).await.unwrap();
    let inspec = InSpec::profile("inspec-aws-s3");
    assert!(inspec.run(&result).passed());
    env.teardown().await;
}
```

Per-resource-type coverage is auto-generated: a code generator reads
each provider's schema and emits a minimal valid configuration for
each resource type, exercising create / read / update / delete +
import. Hand-written fixtures cover the compositions a generator
can't reach (resources that depend on outputs of other resources,
`count`/`for_each` with dependencies, sensitive values flowing
through module boundaries).

#### Continuous compatibility gate

CI runs:

- **Every PR:** levels 1–4 on full Tier 1 + Tier 2 (no cloud cost; ~15 min)
- **Nightly:** level 5 on a rotating Tier 1 subset (cloud cost; ~45 min)
- **Release tag:** level 5 on full Tier 1 (cloud cost; ~2 h); zero failures or no tag

The gate is non-negotiable. Any Tier 1 regression blocks release.
Tier 2 regressions block the affected milestone but not unrelated
work. The harness emits a tameshi receipt per run; receipts are
prerequisites for `magma`'s own release attestation chain.

### II.7. Pleme-io substrate integration

Magma is not a Rust binary that happens to live in pleme-io — it's a
substrate-native primitive. Every pleme-io standard applies with **no
escape hatches**, and the NixOS / home-manager / blackmatter module
surfaces are first-class M0 deliverables, not afterthoughts.

#### Strict standards compliance

| Pleme-io primitive | How magma adopts it |
|---|---|
| substrate `rust-workspace-release` flake | Workspace flake is generated by repo-forge; no hand-rolled flake.nix; every crate's build is hermetic Nix |
| **shikumi for config — end to end** | Every config struct, every runtime value, every state-file value derives `Shikumi`. No `serde_json::Value`, no `HashMap<String, String>`, no untyped JSON anywhere except at the wire boundary to providers + state files. The shikumi type is the source of truth; HCL/JSON/tatara-lisp are three serializations of the same value |
| tatara-lisp surface | `(defmagma-module …)`, `(defmagma-resource …)`, `(defmagma-data …)`, `(defmagma-output …)` — full HCL2 module shape mirrored in tatara-lisp; auto-registered via `#[derive(TataraDomain)]` |
| forge-gen via OpenAPI | The OpenTofu registry's API is OpenAPI-described; `magma-providers`'s registry client is generated, not hand-written |
| repo-forge | Magma's repo is one repo-forge invocation: `repo-forge new --archetype rust-workspace --name magma --variant cli-tool` |
| pleme-actions | CI workflows are 5-line shims around `pleme-io/release-action@v1` + `pleme-io/test-action@v1`. Zero inline composite actions |
| nix-ast | `magma fmt`'s `.tfvars.nix` emission goes through the typed `NixValue` AST; no `format!()` of Nix syntax |
| shigoto | `magma-apply` consumes `shigoto::Scheduler`; the resource DAG is a `shigoto::Dag`; retries via `shigoto::RetryPolicy` |
| tameshi | Every plan + apply emits a tameshi receipt; the magma binary itself is tameshi-signed at release |
| cofre | Every secret (provider credentials, attestation keys, backend creds) is materialized via cofre; magma processes never see plaintext |
| pleme-lib (Helm charts) | Cluster-side magma support (optional `magma-runner` k8s operator, M5+) consumes `pleme-lib` charts — no inline templates |
| observability | Vector universal-ingest sink; structured logs (no human-readable prose); mandatory typed alert layer per Pillar 11 |
| compliant-artifact-provability | Release artifacts ship through cartorio + lacre + provas + tabeliao — the magma binary is itself a compliant artifact |

`skip-<primitive>:` markers exist for downstream workspaces deferring
magma adoption (per §I). They do **not** exist for magma's own
implementation. Magma is the primitive that proves every other
primitive composes — if magma can't be built without escape hatches,
the substrate has a gap to close. Each gap is a separate ticket
against the gap-owning primitive, never a workaround in magma.

#### NixOS module — `services.magma`

`pleme-io/magma` exposes `nixosModules.default` for system-wide install
and service definition. Per-host declarative config:

```nix
{
  services.magma = {
    enable = true;

    # Workspace registry. Each entry produces a systemd unit
    # magma-workspace@<name>.service that watches the directory
    # and runs `magma plan` on change.
    workspaces = {
      seph-vpc = {
        path        = "/etc/magma/workspaces/seph-vpc";
        backend     = "s3://pleme-dev-terraform-state/pangea/seph-vpc";
        autoApprove = false;                          # plan only
        attestation.enable = true;
      };
      cloudflare-pleme = {
        path        = "/etc/magma/workspaces/cloudflare-pleme";
        backend     = "s3://pleme-dev-terraform-state/pangea/cloudflare-pleme";
        autoApprove = true;                           # zone-only, low blast
      };
    };

    providers = {
      cacheDir     = "/var/lib/magma/plugins";       # survives upgrades
      lockfileMode = "strict";                        # no download outside lock
      registries = [
        "https://registry.opentofu.org"
        "https://registry.terraform.io"
        "filesystem:///var/lib/magma/mirror"          # offline-mirror fallback
      ];
    };

    telemetry = {
      enable       = true;
      vectorSocket = "/run/vector/magma.sock";        # observability stack
    };

    attestationKey = config.cofre.refs."magma/attestation-signing-key";
  };
}
```

The module provisions: a `magma` system user/group; the binary in
PATH; `/var/lib/magma/{plugins,state,audit}` with correct mode bits;
systemd `magma-workspace@<name>.service` per workspace (started on
boot, restarted on file change via path-unit, watched for drift); a
Vector source for structured telemetry; the attestation key
materialized via cofre at boot. Each unit is hardened
(`ProtectSystem=strict`, `NoNewPrivileges=true`,
`MemoryDenyWriteExecute=true`, dedicated cgroup with `MemoryHigh=2G`).

#### Home-manager module — `programs.magma`

The flake's `homeManagerModules.default` is the operator-workstation
surface. Per-user declarative config:

```nix
{
  programs.magma = {
    enable = true;

    defaultBackend = "s3://pleme-dev-terraform-state";

    accounts = {
      "akeyless-development" = {
        provider    = "aws";
        credentials = config.cofre.refs."aws/akeyless-development";
      };
      "pleme-io-prod" = {
        provider    = "aws";
        credentials = config.cofre.refs."aws/pleme-io-prod";
      };
    };

    attestation = {
      enable     = true;
      signingKey = config.cofre.refs."magma/operator-signing-key";
    };

    completions = { bash = true; zsh = true; fish = true; };
  };
}
```

The HM module drops the binary in the user's profile, generates shell
completions (typed Clap-derived), materializes `~/.config/magma/config.toml`
from typed shikumi values, wires operator-side cofre credentials, and
installs the operator plugin cache at `~/.local/share/magma/plugins`.

#### Blackmatter integration — `blackmatter.components.magma`

`blackmatter-pleme` exposes `blackmatter.components.magma` that bundles
the HM module above plus opinionated pleme-io defaults (aws / cloudflare
/ akeyless / datadog / splunk / hcloud / gcp / azure credentials wired
to operator cofre, default backend pointed at `pleme-dev-terraform-state`,
attestation enabled with the operator's signing key). Operators turn it
on with `blackmatter.components.magma.enable = true` in
`~/code/github/pleme-io/nix`; everything else follows.

The NixOS counterpart `blackmatter.profiles.pleme-io-iac-host` enables
`services.magma` with the cluster's workspace registry declared. Used
on the `kazoku` / `seph` / `rio` control nodes that act as IaC apply
endpoints — one declaration per cluster, the rest follows from the
typescape.

#### What "extensive" means concretely

The NixOS + HM module surface is **itself typed and generated**, not
hand-written. A `magma-nix-options` crate produces the `.nix` option
declarations mechanically from the shikumi config schema. Module
options expand automatically as magma's typed surface grows: new
resource attributes never require hand-editing the module; a new
backend never requires a new `services.magma.backend.<x>` option
written by hand. This is Pillar 12 (generation over composition) at
the Nix-options layer — typed primitive once, every render surface
(NixOS module, HM module, JSON Schema for editor support, shell
completions, CLI help, doc site) auto-emits from one source.

### II.8. Multi-interface surface — drop-in compat + native interfaces

Drop-in compatibility is non-negotiable: operators must be able to add
`alias terraform=magma` (or `alias tofu=magma`) and have every existing
Pangea-rendered-JSON workflow continue working without modification.
Beyond drop-in, magma exposes three additional first-class interfaces.
**Every interface ships in M0** — none is a follow-up.

#### Interface 1: Drop-in CLI compat

The magma binary inspects `argv[0]` and adjusts behavior:

| `argv[0]` | Behavior |
|---|---|
| `terraform` or `terraform-*` | Terraform compat mode: matching flags, env vars, output format, exit codes |
| `tofu` or `tofu-*` | OpenTofu compat mode (same as terraform, modulo the post-fork divergences) |
| `magma` or anything else | Magma native mode |

In compat mode magma honors the Terraform / OpenTofu env-var inventory
(see §II.2 row 15). The `.terraform/` directory layout matches exactly:
`.terraform/providers/<host>/<ns>/<name>/<version>/<os_arch>/`,
`.terraform/modules/` (module-call expansion records only; not module
sources), `.terraform.lock.hcl` at workspace root.

Exit codes match Terraform's documented codes: 0 (success), 1 (error),
2 (`plan` reports changes when `-detailed-exitcode` is set).

stdout/stderr format matches Terraform's verbatim by default.
`--magma-native` (or env `MAGMA_OUTPUT=native`) opts into pleme-io's
structured-JSON output stream.

**Important scope:** drop-in is for *Pangea-rendered Terraform JSON
workflows*. Operators with raw HCL workflows render to JSON via
`terraform-config-inspect` first, or migrate to Pangea Ruby. Magma
never grows an HCL parser per §I, §II.1, §IX.

#### Interface 2: Native typed CLI

`magma <subcommand> [flags...]` with pleme-io conventions on top of the
full Terraform command surface:

- Typed JSON output via `--json` everywhere (consistent shape across subcommands)
- `--magma-version` shows magma's version + supported protocol versions
- Subcommands Terraform doesn't have:
  - `magma mcp` — starts the MCP server (stdin/stdout transport)
  - `magma daemon` — system-side workspace watcher (NixOS `services.magma`)
  - `magma watch` — operator-side watcher (HM `programs.magma`)
  - `magma attest verify <plan-id>` — verify a tameshi receipt
  - `magma config get|set|edit` — manipulate `~/.config/magma/magma.yaml`
  - `magma flow run <flow.lisp>` — execute a tatara-lisp DAG of magma operations (§II.9)

#### Interface 3: MCP server

`magma mcp` starts a Model Context Protocol server on stdin/stdout
(JSON-RPC 2.0). Typed tools:

| MCP tool | Wraps | Destructive |
|---|---|---|
| `magma_init` | `magma init` | no |
| `magma_plan` | `magma plan` | no |
| `magma_apply` | `magma apply` — requires `confirm: true` AND verified `plan_id` | **yes** |
| `magma_destroy` | `magma destroy` — requires `confirm: true` | **yes** |
| `magma_state_list` | `magma state list` | no |
| `magma_state_show` | `magma state show <address>` | no |
| `magma_state_mv` | `magma state mv` — requires `confirm: true` | **yes** |
| `magma_workspace_list` | `magma workspace list` | no |
| `magma_output` | `magma output [-json]` | no |
| `magma_show` | `magma show [-json]` | no |
| `magma_validate` | `magma validate` | no |
| `magma_fmt` | `magma fmt -check` | no |
| `magma_attest_verify` | `magma attest verify` | no |
| `magma_flow_run` | `magma flow run` — accepts tatara-lisp DAG inline (§II.9) | conditional (gated by inner Jobs' destructiveness) |

Tool schemas are generated mechanically from `magma-types` — **never
hand-authored** — per Pillar 12. Destructive operations require
explicit `confirm: true` that MCP clients must surface to the human;
magma-mcp rejects unconfirmed calls with a typed `UnconfirmedDestructive`
error.

Lives in `magma-mcp`; transport defaults to stdin/stdout, `--port <n>`
switches to TCP for editor / inspector use.

#### Interface 4: Rust library

Direct consumption via `cargo add magma` (or per-crate). The umbrella
`magma` crate re-exports the public surface; consumers compose typed
values without going through CLI or MCP. Internal pleme-io code paths
(arch-synthesizer, Pangea Ruby's renderer, future operators, **the
future pangea-api** — see §II.10) consume magma as a library — not a
subprocess. The library interface is also what makes §II.9's in-memory
pipelines possible: typed values flow through Rust calls, never
through CLI shell-outs.

#### Interface ship matrix

| Interface | M0 ship | M3 ship | Notes |
|---|---|---|---|
| Drop-in CLI compat | ✓ (Tier 1 subcommand subset) | ✓ (full parity) | argv[0] sensing + TF env vars wired in M0 |
| Native typed CLI | ✓ (all subcommands declared; many stubbed) | ✓ (all implemented) | `--json` works on completed subcommands |
| MCP server | ✓ (skeleton + 14 tools with destructive gating) | ✓ (full tool surface) | `magma mcp` launches; `--port` for TCP |
| Rust library | ✓ (typed API public) | ✓ (stable) | All workspace crates `pub`-exposed |

### II.9. In-memory pipelines + shigoto work-graph

Pangea Ruby evaluates **in-process** inside magma (via magnus +
`pangea-ruby-eval` per §II.1); the rendered architecture **never
touches disk** in the in-memory chain. This unlocks a category of
infrastructure operations that Terraform / OpenTofu cannot express:
typed values flow through a `shigoto`-scheduled DAG, with cross-workspace
chaining, suspendable / resumable flows, budget enforcement, and full
audit trail — all without serializing values to JSON, writing to
`tempfile`, or reading back. This is what makes magma not just an
OpenTofu replacement, but *the runtime that supercharges Pangea's
expressive DSL into a typed work-graph*.

#### The typed value flow (M0)

```
   ┌──────────────────────────────────────────────────────────────┐
   │ magma binary process (one CRuby interpreter; one Tokio rt)   │
   │                                                              │
   │  ┌──────────────┐    ┌────────────┐    ┌─────────┐           │
   │  │ magma-pangea │──▶ │ magma-     │──▶ │ magma-  │           │
   │  │ Evaluate-    │    │ config     │    │ plan    │──▶┐       │
   │  │ Pangea Job   │    │ Parse Job  │    │ Plan    │   │       │
   │  └──────────────┘    └────────────┘    │ Job     │   │       │
   │       ▲                                 └─────────┘   │       │
   │       │ Workspace source                              │ Plan  │
   │       │ (path or in-memory string)                    │       │
   │                                                       ▼       │
   │  ┌──────────────┐    ┌─────────────┐    ┌─────────────────┐   │
   │  │ magma-attest │◀── │ magma-state │◀── │ magma-apply     │   │
   │  │ AttestPlan   │    │ UpdateState │    │ ApplyChange × N │   │
   │  │ Job          │    │ Job         │    │ (one Job/res)   │   │
   │  └──────────────┘    └─────────────┘    └─────────────────┘   │
   │                                                               │
   └──────────────────────────────────────────────────────────────┘
```

Every typed value (`LoadedWorkspace`, `Config`, `Plan`, `ResourceChange`,
`StateInstance`, `PlanId`) lives in RAM throughout the operation;
`tempfile`-on-disk JSON roundtrips are an anti-pattern (§IX). Disk
persistence is reserved for: state file commits (durability), lock
file (concurrency), attestation receipts (provenance) — never for value
passing between in-process Jobs.

#### Operations as typed shigoto Jobs

Each magma operation surfaces as a typed `shigoto::Job`
(per [`SHIGOTO.md`](./SHIGOTO.md)):

| Job kind | Input | Output | Idempotent | Side effects |
|---|---|---|---|---|
| `EvaluatePangea` | `WorkspaceShape` (path or in-memory source string) | `LoadedWorkspace` | yes (pure Ruby) | none |
| `ParseConfig` | `LoadedWorkspace` | `magma_config::Config` | yes | none |
| `BuildGraph` | `Config` | `ResourceGraph` (`shigoto::Dag` over `ResourceAddress`) | yes | none |
| `Plan` | `Config × State` | `Plan` + `PlanId` | yes (cacheable reads) | provider RPC reads |
| `ApplyChange` | `ResourceChange` (one Job per change) | `StateInstance` | yes (provider contract) | provider RPC writes — the cloud change |
| `UpdateState` | `State × []StateInstance` | `State` | yes | state file write |
| `AttestPlan` | `Plan` | `tameshi::Receipt` | yes | receipt emission |

Job composition is a `shigoto::Dag`. The default DAG for `magma plan`
is `EvaluatePangea → ParseConfig → BuildGraph → Plan → AttestPlan`.
The default DAG for `magma apply --plan-id <id>` is
`LoadPriorPlan → VerifyPlanId → (ApplyChange × N as a graph)
→ UpdateState → AttestApply`. Custom DAGs are authored as tatara-lisp:

```clojure
(defmagma-flow seph-cluster-deploy
  :workspaces [:seph-vpc :seph-cluster :seph-fluxcd]
  :graph
    (-> :seph-vpc apply
        :seph-cluster (apply :after [:seph-vpc])
        :seph-fluxcd  (apply :after [:seph-cluster])
        :verify       (run-inspec :seph-cluster :profile inspec-k3s-cis))
  :budget {:total-duration "30m" :parallelism 4}
  :retry  {:max-attempts 3 :backoff :exponential})
```

The flow compiles to a `shigoto::Dag`. `magma flow run <flow.lisp>`
executes it. `magma mcp` exposes `magma_flow_run` so AI agents drive
multi-workspace deploys with full audit + budget.

#### Cross-workspace chaining without disk

Workspace A's outputs are typed values; workspace B's inputs reference
them by typed handle. The classical Terraform pattern
`data "terraform_remote_state"` (read another state file from disk via
S3) becomes an in-process typed-value pass:

```rust
// Old (Terraform / OpenTofu): disk-bound; two processes; serialize/deserialize.
//   data "terraform_remote_state" "vpc" { backend = "s3"; config = { ... } }
//   resource "aws_subnet" "web" { vpc_id = data.terraform_remote_state.vpc.outputs.vpc_id }

// New (magma in-memory): one process; typed values; no JSON in the path.
let vpc_outputs = magma::flow::apply(&vpc_workspace).await?.outputs;
let subnet_cfg  = magma::pangea::evaluate_with_input(
    &subnet_workspace_src,
    "cross.seph_vpc.vpc_id" => vpc_outputs["vpc_id"].clone(),
).await?;
let subnet_plan = magma::flow::plan(&subnet_cfg, &subnet_state).await?;
magma::flow::apply_plan(&subnet_plan, &subnet_state).await?;
```

Same workflow, the VPC's `vpc_id` is a typed value flowing through
Rust — no state file persisted between A and B unless durability is
explicitly requested via `magma::state::persist(...)`.

#### Work-like behavior — suspend / resume / budget / replay

Because every operation is a `shigoto::Job`:

- **Suspend / resume.** Magma's process can crash mid-apply; the next
  start reads the partial state, reconstructs the DAG, and resumes
  from the first `Pending` / `Retrying` Job. No `terraform
  force-unlock` ritual — shigoto's FSM owns the lock + resumption.
- **Budgets.** Operations carry typed budgets (cost / time /
  parallelism). `magma apply --budget time=15m,parallelism=4` is a
  typed constraint shigoto enforces.
- **Audit.** Every Job transition emits a `shigoto::TickReceipt`;
  receipts feed `magma-attest` and ultimately tameshi.
- **Replay.** Given a `PlanId`, magma replays the apply DAG from a
  clean state for verification (per §II.6 level 4 round-trip).

#### What this unlocks

- **Composable infrastructure operations.** Chains like "evaluate
  Pangea, plan, attest, gate on human approval, apply, verify with
  InSpec, attest again, sign-off" become typed DAGs — not shell pipelines.
- **No artificial JSON serialization.** Pangea outputs feed Plan
  inputs as typed values. Eliminates bugs caused by losing type
  information at the JSON boundary (sensitive marks dropping, cty
  type-tag erasure, numeric precision loss).
- **Multi-workspace deploys without state-file gymnastics.**
  Constellation-style flows (per [`PANGEA-WORKSPACE-RECONCILIATION.md`](./PANGEA-WORKSPACE-RECONCILIATION.md))
  become single in-process DAGs.
- **MCP-driven flows.** `magma_flow_run` accepts a tatara-lisp DAG as
  a JSON string, runs it, streams TickReceipts back. AI agents drive
  complex multi-workspace deploys with full audit + budget enforcement.
- **Pangea supercharged.** Pangea Ruby's expressive DSL becomes one
  layer of a typed work-graph, not a one-shot rendering. The same
  Pangea workspace evaluates under different inputs / states / budgets
  within a single `magma flow run` — impossible in the disk-bound
  Terraform model.

§II.9 is the bridge: magma stops being merely "an OpenTofu replacement"
and becomes *the runtime that makes Pangea's expressive DSL a
first-class typed infrastructure work-graph*.

### II.10. Substrate for pangea-api — the future product layer

Magma + Pangea Ruby + shigoto is **the substrate** for a future
product layer called **pangea-api**: the ultimate infrastructure
creation and management tool. magma is not the destination — it's
the typed-work-graph runtime that makes pangea-api possible.

pangea-api consumes magma via the §II.8 Rust library interface and
abstracts, for any Terraform-provider-backed infrastructure:

| pangea-api dimension | Implementation via pangea-magma |
|---|---|
| **Instantiation** | Operator declares what infrastructure they want (typed Pangea Ruby or higher-level pangea-api DSL); pangea-api drives `magma::flow::plan` + `magma::flow::apply` to provision it. |
| **State management** | `magma-state` + `magma-backend` own the on-disk state surface; pangea-api owns the *user-facing* state model (drift detection, reconciliation cadence, alerting). |
| **Exposure** | DNS / ingress / networking / routing are derivable from typed infrastructure declarations — pangea-api emits magma operations that wire `cloudflare_record`, `kubernetes_ingress`, `aws_lb_listener`, etc. without the user thinking in those primitives. |
| **Troubleshooting** | shigoto TickReceipts + tameshi attestations + Vector telemetry feed pangea-api's diagnostics surface; typed audit trail makes "what changed, why, when, by whom" queryable. |

The "pangea-magma pattern" generalizes:

```
            operator intent (typed pangea-api DSL)
                            │
                            ▼
                  Pangea Ruby (in-process)
                            │
                            ▼
                  magma typed work-graph
                  (shigoto Jobs over magma-types)
                            │
                            ▼
                 existing Terraform providers
                  (any vendor, any cloud)
```

Any infrastructure expressible via existing Terraform providers
(thousands across the OpenTofu + Terraform registries) becomes
addressable through pangea-api — without pangea-api needing to
implement a single provider client itself.

**Design implication for magma:**

- The library interface (§II.8 interface 4) must stay symmetric with the
  CLI / MCP interfaces — pangea-api consumes the library; AI agents
  consume MCP; humans consume CLI; the typed core is shared.
- Avoid baking pangea-api-specific abstractions into magma. magma stays
  a substrate primitive: typed work-graph runtime for IaC. pangea-api
  builds its product layer on top.
- The in-memory chain (§II.9) is non-negotiable: pangea-api's
  responsiveness depends on disk-free typed-value flow through the
  pangea → magma → cloud pipeline.

pangea-api itself is out of scope for this destination doc — it gets
its own spec when it lands. This section exists to anchor magma's
design decisions against the trajectory it's serving.

### II.11. Pangea backend selection + auto-discovery + capability probing

Pangea Ruby supports two execution backends — `tofu` (the historical
default; current pleme-io fleet behavior) and `magma` (the pleme-io
Rust-native executor). Both backends consume Pangea-rendered Terraform
JSON; magma also accepts the in-process Pangea-Ruby path (§II.9).
**Backend selection is fully configurable** and the Ruby side
auto-discovers each binary's capabilities at runtime so it can give
graceful errors when the chosen backend lacks a feature the workspace
needs.

#### Selecting a backend

Per-workspace `pangea.yml` gains a typed `backend:` field:

```yaml
namespace: pleme-io/seph-vpc
state:
  s3:
    bucket: pleme-dev-terraform-state
    key:    pangea/seph-vpc
backend: magma            # tofu | magma (default: tofu)
```

Operator override at invocation time:

```bash
# Force-override backend for a single invocation.
bundle exec pangea plan seph_vpc.rb --backend=magma

# Or via env, for batch / CI flows.
PANGEA_BACKEND=magma bundle exec pangea apply seph_vpc.rb
```

Substrate's [`pangea-arch-workspace.nix`](../substrate/lib/infra/pangea-arch-workspace.nix)
helper's `executor` parameter (§II.7) hardwires the choice into the
workspace's `flake.nix`; `pangea.yml`'s `backend:` is the Ruby-side
default that the helper's app respects.

#### Auto-discovery

On every `pangea <verb>` invocation, the Ruby side:

1. **Probes each backend's binary.** `which tofu` + `which magma`; cache
   the discovery result keyed on workspace dir + day.
2. **Queries capabilities** via the canonical `<binary> capabilities`
   subcommand (magma) / `<binary> version -json` (tofu) — both must
   return a typed JSON manifest. Cached for the session.
3. **Validates compatibility.** Compares the workspace's requirements
   (declared in `pangea.yml` + the Pangea Ruby template) against the
   chosen backend's reported capabilities. Examples:
   - HCL input → not supported by magma → fail-early with helpful
     migration guidance
   - Provider protocol v7 → not in `magma capabilities`'s
     `supported_protocols` → fail with the supported list
   - In-memory chain → not in `tofu capabilities` → fail when
     pangea workspaces want §II.9 mode

#### `magma capabilities` JSON manifest

The canonical schema (the live `magma capabilities` output):

```json
{
  "tool":               "magma",
  "version":            "0.1.0",
  "schema_version":     1,
  "supported_protocols": ["tfplugin5", "tfplugin6"],
  "input_formats":      ["pangea-ruby-inprocess", "terraform-json"],
  "input_formats_excluded": ["hcl2"],
  "backends":           ["local", "s3 (planned M1)"],
  "subcommands":        ["init", "plan", "apply", "destroy", "state",
                         "workspace", "mcp", "daemon", "watch",
                         "attest", "config", "flow", "capabilities", …],
  "interfaces": {
    "drop_in_cli":   { "argv0_modes": ["terraform", "tofu", "magma"] },
    "native_cli":    { "json_flag": true },
    "mcp":           { "transport": "stdin-stdout-jsonrpc2", "tools": 14 },
    "library":       { "umbrella_crate": "magma" }
  },
  "mcp_tools":          ["magma_init", "magma_plan", "magma_apply", …],
  "env_vars_honored":   ["TF_VAR_*", "TF_DATA_DIR", …],
  "exit_codes":         { "0": "success", "1": "error",
                          "2": "plan: changes pending (-detailed-exitcode)" },
  "workspace_primitive_supported": true,
  "workspace_chain_supported":     true,
  "in_memory_pipeline_supported":  true,
  "shigoto_job_wrapping": "planned (M1)"
}
```

The schema is versioned (`schema_version`); operators / CI gate on
specific schema versions when probing capabilities programmatically.

#### tofu capabilities manifest

`tofu` doesn't natively emit a capabilities manifest — Pangea Ruby
constructs one by probing:

- `tofu version -json` → version + protocols
- `tofu providers schema -json` → installed providers (where applicable)
- Compile-time table of "tofu always has X subcommand" → subcommand
  list

The Ruby side normalizes both backends into a common
`Pangea::BackendCapabilities` struct so workspace-side code interacts
with one shape regardless of executor.

#### Graceful failure modes

When a workspace requires a feature the chosen backend lacks, Pangea
Ruby raises a typed `Pangea::BackendIncompatible` exception with:

1. The requested backend name
2. The feature that's unavailable
3. Available alternatives (other backend names that DO support it)
4. A migration suggestion (e.g. "switch to backend: magma to use
   in-memory chains")

Example surfacing:

```
$ bundle exec pangea plan seph_vpc.rb --backend=tofu
Pangea::BackendIncompatible:
  backend=tofu does not support in-memory workspace chains.
  workspace seph-vpc declares `chain:` referencing seph-cluster.

  Options:
    1. Switch the workspace to backend=magma (in-memory chains supported).
    2. Materialize the chain as separate workspaces with terraform_remote_state
       data sources (legacy disk-bound flow; supported by tofu).
```

#### Where the implementation lives

| Layer | What it owns | Status |
|---|---|---|
| `magma capabilities` Rust subcommand | Emits the JSON manifest | M0 ✓ |
| `tofu` version probing | Pangea Ruby wraps `tofu version -json` | M0.x |
| `Pangea::BackendCapabilities` Ruby struct | Normalized cross-backend shape | M0.x |
| `Pangea::Backend{Tofu,Magma}` dispatch | Runtime dispatch on `pangea.yml`'s `backend:` | M0.x |
| `Pangea::BackendIncompatible` typed error | Graceful failure surfacing | M0.x |
| `pangea-arch-workspace.nix` `executor` knob | Nix-layer hardwire | M0 ✓ |
| `--backend=` CLI flag | Override at invocation | M0.x |
| `PANGEA_BACKEND` env var | Override across CI flows | M0.x |

The `magma capabilities` end of the dialogue is implemented now (the
substrate side magma owns); the Ruby side ships in pangea-core M0.x
alongside the first workspace that opts into `backend: magma` in
production.

---

## III. The typed surface

Compact code blocks for the load-bearing types. The prose carries the
invariants.

### III.1. `ResourceAddress` — typed identity

```rust
pub struct ResourceAddress {
    pub module:   ModulePath,                 // root or a.b.c
    pub kind:     ResourceKind,               // Managed | Data | Output | Local | Variable
    pub type_id:  ResourceTypeId,             // "aws_vpc", "cloudflare_record", ...
    pub name:     String,
    pub key:      Option<InstanceKey>,        // for count / for_each
}

pub enum InstanceKey {
    Index(u64),                               // count.index
    Key(String),                              // for_each key
}

pub struct ResourceTypeId(pub &'static str); // interned; provider-scoped
```

A `ResourceAddress` is the canonical identifier used in state, plan,
DAG, error messages, and audit. It must round-trip exactly through the
OpenTofu state-file format (`"aws_vpc.foo[\"bar\"]"`).

### III.2. `Resource` — declared shape

```rust
pub struct Resource {
    pub address:   ResourceAddress,
    pub provider:  ProviderReference,
    pub schema:    ProviderSchemaRef,
    pub config:    ShikumiValue,              // typed; conforms to schema
    pub depends_on: Vec<ResourceAddress>,
    pub lifecycle: LifecycleRules,
    pub provisioners: Vec<Provisioner>,       // deprecated but supported
}

pub struct LifecycleRules {
    pub create_before_destroy: bool,
    pub prevent_destroy:       bool,
    pub ignore_changes:        Vec<AttributePath>,
    pub replace_triggered_by:  Vec<ResourceAddress>,
}
```

The schema reference is required — `config` is validated against the
provider's reported schema at parse time, not at plan time. Schema
mismatches are caught before any provider RPC.

### III.3. `Plan` — the planning output

```rust
pub struct Plan {
    pub id:           PlanId,                 // BLAKE3 hash, see magma-attest
    pub created_at:   DateTime<Utc>,
    pub config_root:  PathBuf,
    pub variables:    HashMap<String, ShikumiValue>,
    pub prior_state:  StateSnapshot,
    pub resource_changes: Vec<ResourceChange>,
    pub output_changes:   Vec<OutputChange>,
    pub provider_schemas: HashMap<ProviderReference, ProviderSchema>,
    pub backend:      BackendConfig,
}

pub struct ResourceChange {
    pub address: ResourceAddress,
    pub action:  Action,
    pub before:  Option<ShikumiValue>,
    pub after:   Option<ShikumiValue>,
    pub before_sensitive: SensitivePaths,
    pub after_sensitive:  SensitivePaths,
    pub importing: Option<ImportTarget>,
    pub reasons: Vec<ChangeReason>,
}

pub enum Action {
    NoOp,
    Create,
    Read,                                     // data source
    Update,
    Replace { reason: ReplaceReason },
    Delete,
    Forget,                                   // remove from state without destroy
    CreateThenDelete,                         // create_before_destroy
    DeleteThenCreate,                         // default replace
}

pub enum ReplaceReason {
    RequiresReplace,                          // provider said so
    ReplaceTriggeredBy(ResourceAddress),      // lifecycle hint
    Tainted,
    ReplacedByCli,                            // -replace=...
}
```

The `Plan` is BLAKE3-hashed by `magma-attest` to produce `PlanId`. The
hash covers the prior state + variables + resource changes + provider
schemas. Two plans with the same `PlanId` are bit-equal; `magma apply
--plan <id>` re-derives the plan from the persisted snapshot and
asserts hash-equality before applying.

### III.4. `State` — what we record

```rust
pub struct State {
    pub version:    u64,                      // matches OpenTofu's
    pub terraform_version: VersionString,     // emitted for compat
    pub serial:     u64,
    pub lineage:    Uuid,
    pub outputs:    HashMap<String, OutputValue>,
    pub resources:  Vec<StateResource>,
    pub check_results: Vec<CheckResult>,
}

pub struct StateResource {
    pub address:    ResourceAddress,
    pub provider:   ProviderReference,
    pub instances:  Vec<StateInstance>,
}

pub struct StateInstance {
    pub schema_version: u64,
    pub attributes:     ShikumiValue,
    pub sensitive_attrs: SensitivePaths,
    pub private:        Vec<u8>,              // opaque provider state
    pub dependencies:   Vec<ResourceAddress>,
    pub status:         InstanceStatus,
}
```

Serialization to `terraform.tfstate` is byte-equal to OpenTofu's
output for every snapshot the proptest harness throws at it. This is
the most fragile interop surface — `magma-state`'s integration tests
read every state file produced by a year's worth of pleme-io workspaces
and round-trip them.

### III.5. `Provider` — the typed gRPC wrapper

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    fn reference(&self) -> &ProviderReference;
    fn schema(&self)    -> &ProviderSchema;

    async fn configure(&self, config: ShikumiValue) -> Result<(), ProviderError>;

    async fn read_resource(
        &self,
        type_id: ResourceTypeId,
        prior:   StateInstance,
    ) -> Result<StateInstance, ProviderError>;

    async fn plan_resource_change(
        &self,
        type_id: ResourceTypeId,
        prior:   Option<StateInstance>,
        proposed: ShikumiValue,
    ) -> Result<PlannedChange, ProviderError>;

    async fn apply_resource_change(
        &self,
        type_id: ResourceTypeId,
        prior:   Option<StateInstance>,
        planned: PlannedChange,
    ) -> Result<StateInstance, ProviderError>;

    async fn import_resource(
        &self,
        type_id: ResourceTypeId,
        id:      String,
    ) -> Result<Vec<StateInstance>, ProviderError>;

    async fn validate(
        &self,
        type_id: ResourceTypeId,
        config:  ShikumiValue,
    ) -> Result<Vec<Diagnostic>, ProviderError>;
}
```

Two impls of `Provider` ship:
- `GrpcProvider` — wraps a tonic gRPC client to a subprocess provider
  binary (the 99% case)
- `MockProvider` — in-process test double, used by `magma-test`'s
  proptest harness and by consumers' synthesis tests

The trait is the only surface `magma-plan` and `magma-apply` see —
they don't know whether they're talking to a real provider or a mock.

---

## IV. The protocol layer — the load-bearing technical unknown

### IV.1. go-plugin handshake

HashiCorp's go-plugin protocol (the framework underneath tfplugin5/6):

1. Parent sets handshake env: `PLUGIN_MIN_PORT=1024`,
   `PLUGIN_MAX_PORT=65535`, `PLUGIN_MAGIC_COOKIE=<the well-known TF value>`,
   `PLUGIN_PROTOCOL_VERSIONS=5,6`, `PLUGIN_CLIENT_CERT=<base64 PEM>`.
2. Parent spawns the provider binary as a subprocess.
3. Provider validates the magic cookie. If wrong, exits 1 with a
   friendly error.
4. Provider generates a self-signed leaf cert (signed by the parent's
   cert chain), binds to a port in `[PLUGIN_MIN_PORT, PLUGIN_MAX_PORT]`,
   prints **one** handshake line on stdout:

   ```
   1|6|tcp|127.0.0.1:42839|grpc|<base64 provider cert PEM>
   ```

   Fields: `CORE_PROTOCOL|APP_PROTOCOL|NETWORK|ADDRESS|PROTO_TYPE|CERT`.
5. Parent parses the handshake line, builds a tonic gRPC channel with
   mTLS using the parent cert + provider cert, ready to dial.
6. All subsequent calls go over the gRPC channel.
7. Parent kills the subprocess via SIGTERM (then SIGKILL after grace
   period) when the `GrpcProvider` is dropped.

Implemented in `magma-plugin` as:

```rust
pub struct Plugin {
    process:  Child,
    channel:  tonic::transport::Channel,
    protocol: PluginProtocol,                 // V5 or V6
}

impl Plugin {
    pub async fn spawn(
        binary_path: &Path,
        magic_cookie: &str,
        accepted_protocols: &[PluginProtocol],
    ) -> Result<Self, PluginError> { /* ... */ }
}

impl Drop for Plugin {
    fn drop(&mut self) { /* SIGTERM, SIGKILL after 5s */ }
}
```

The reverse-engineering work fits in one focused crate. The tonic gRPC
stack handles mTLS + protobuf + connection lifecycle for free. The
fragile parts are: the stdout-handshake-line parser, the cert chain
bootstrap, the subprocess lifecycle (clean kill on parent panic, clean
kill on parent OOM, no zombies).

### IV.2. tfplugin5/6 protobuf → Rust types

`magma-protocol`'s `build.rs`:

```rust
fn main() {
    tonic_build::configure()
        .build_server(false)
        .compile(
            &["proto/tfplugin5.proto", "proto/tfplugin6.proto"],
            &["proto/"],
        )
        .unwrap();
}
```

The .proto files are vendored from OpenTofu's `internal/tfplugin{5,6}/`
directories (vendored, not git-submodule'd — they're stable and we
want auditable inputs). prost-build emits idiomatic Rust types; tonic
emits async gRPC client stubs.

`magma-types` wraps the raw protobuf types in typed-domain values
where helpful (e.g., a tfplugin6 `Schema` becomes `magma-types::ProviderSchema`
with typed `Block` / `Attribute` / `NestedBlock` instead of nested
protobuf prost types).

### IV.3. Provider-binary discovery

`magma-providers` mirrors OpenTofu's discovery:

| Source | Path | Notes |
|---|---|---|
| Lock-file local | `.terraform/providers/<host>/<namespace>/<name>/<version>/<os_arch>/` | After `magma init` |
| User cache | `~/.terraform.d/plugin-cache/...` | If `TF_PLUGIN_CACHE_DIR` set |
| System | `/usr/share/terraform/plugins/...` | rare |
| Registry download | OpenTofu / Terraform registry | First-time install |
| Filesystem mirror | `filesystem_mirror` block | Air-gapped envs |
| Network mirror | `network_mirror` block | Corporate mirrors |

The lock-file format (`.terraform.lock.hcl`) round-trips byte-exact
with OpenTofu's via the typed `HclValue` AST. Per-platform hashes
(`h1:`) compute identically.

---

## V. Naming + theme

`magma` per the [`THEORY.md`](./THEORY.md) §II Brazilian-Portuguese
naming convention. The geological frame is load-bearing:

> *Pangea* (the supercontinent) declares the shape of the world;
> *magma* is the molten executive force beneath the crust that
> realizes that shape on the physical substrate.

The metaphor maps 1:1 onto the architecture:

| Geological | Architectural |
|---|---|
| Pangea (supercontinent) | Pangea Ruby DSL — the declarative top |
| Magma (molten force below crust) | magma binary — the execution engine |
| Tectonic plates | Cloud providers (AWS, Cloudflare, Akeyless, ...) — the substrate that takes shape |
| Heat + motion | Plan + apply — the dynamic force that realizes shape |
| Crust solidifying | Resources reaching `Created` state |
| Continental drift | State drift; refresh / re-plan |
| Subduction | Resource deletion + recreation |

Other geological / continental candidates considered: `manto` (mantle —
too passive), `relevo` (terrain — too outcome-flavored, not engine-flavored),
`lava` (surface flow — too visible / output-flavored), `placa` (tectonic
plate — name of the substrate, not the executor), `crisol` (crucible —
craft, but disconnected from continental metaphor).

Industrial / craft candidates considered: `usina` (factory / power plant —
strong but no Pangea tie), `engenho` (mill — strong but craft-flavored
not force-flavored), `forja` (forge — already taken by CI/CD; semantic
collision).

Session-manager / craft candidates considered: `tear` (loom) — already
taken by the pleme-io session manager. Memory:
[`bp-name-tear-taken`](../../.claude/projects/-Users-luis-d-code-github-pleme-io-pangea-architectures/memory/bp-name-tear-taken.md).

`magma` wins on: (a) explicit geological tie to Pangea, (b) force /
engine flavor (active, hot, motive), (c) brevity (5 chars), (d)
English-friendly loanword (operators reading code don't trip on
pronunciation), (e) no collision in the pleme-io BP corpus, (f) clean
pairing with `forja` (forge = metal-shaping craft; magma = earth-shaping
force — same depth-of-craft, different element).

---

## VI. Phases — the path-down from destination

Per pleme-io/CLAUDE.md Operating Principle #0: the destination is
§II–V. The phases below are the **interim path**, not the goal. Each
milestone delivers a usable executor that strictly grows toward
§II–V. Time-pressure deviation never collapses §II–V — we may ship
M0 alone for six months without M1, but we do not redefine the
destination to make M0 "enough."

### M0 — Pangea-Ruby in-process + JSON executor + drop-in CLI + MCP (≈4 months)

**Goal:** Replace `tofu plan/apply` for the Pangea-rendered-JSON
pipeline AND wire in-process Pangea Ruby evaluation, drop-in CLI
compat, and the MCP server skeleton — the four §II.8 interfaces all
land in M0.

Crates landed: `magma-types`, `magma-pangea` (in-process CRuby via
magnus + `pangea-ruby-eval` behind the `magnus` feature; Terraform
JSON reader always available), `magma-config`, `magma-protocol`,
`magma-plugin` (go-plugin handshake), `magma-providers` (lock-file
read only), `magma-state` (local backend), `magma-graph`, `magma-plan`
(subset of Action variants — Create/Delete/NoOp; updates pending
provider RPC), `magma-apply` (subset; no provisioners), `magma-attest`,
`magma-mcp` (skeleton + 14 typed tools with destructive gating),
`magma-cli` (load-bearing subcommands + drop-in argv[0] sensing +
TF_* env vars).

Inputs: Pangea Ruby DSL (in-process via magnus, the canonical M0
path) **AND** Terraform JSON. **No HCL parser** (§I, §IX).

Backends: local only.

Providers smoke-tested (real provider binaries, real gRPC): the full
Tier 1 set from §II.6.

Drop-in compat (§II.8 interface 1): argv[0] sensing distinguishes
`terraform` / `tofu` / `magma`; TF_VAR_* / TF_DATA_DIR / TF_LOG /
TF_CLI_ARGS_* / TF_INPUT / TF_IN_AUTOMATION / TF_TOKEN_* /
TF_PLUGIN_CACHE_DIR / TF_WORKSPACE all honored.

MCP (§II.8 interface 3): `magma mcp` launches the server with 14
initial tools; destructive operations require `confirm: true`.

**Gate:** All Tier 1 providers (§II.6) pass test levels 1–5; the
NixOS + home-manager module surfaces ship per §II.7 (not deferred).
`magma plan` produces byte-equal JSON to `tofu show -json` on the
entire `pangea-architectures/workspaces/` corpus when invoked in
Terraform-JSON mode. `magma apply` produces a byte-equal state file
to `tofu apply` on the same corpus, and the resulting resources pass
the existing `inspec-aws-k3s` / `inspec-akeyless` profiles. Pangea
Ruby workspaces (the M0 Pangea-first input) render in-process via
magnus and produce a typed `LoadedWorkspace` identical to the
disk-rendered JSON path (verified by round-trip).

### M1 — Multi-backend + multi-workspace + shigoto Job wrappers (≈3 months)

S3 backend (the only one pleme-io uses today;
`pleme-dev-terraform-state`), HTTP backend, DynamoDB locking.
`workspace select/list/new/delete` parity.

Magma operations wrapped as typed `shigoto::Job`s (per §II.9); the
apply engine is a `shigoto::Scheduler` consumer; cross-workspace
chaining via typed-value passing (no disk roundtrip per §II.9).

`tatara-lisp` `(defmagma-flow …)` authoring surface lands for in-memory
DAGs; `magma flow run <file.lisp>` executes them.

**Gate:** Pangea-architectures' M0 corpus runs against S3 backend
with DynamoDB locking, parity preserved. A constellation deploy of
three chained workspaces (e.g. `seph-vpc → seph-cluster →
seph-fluxcd`) completes entirely in-process (no intermediate disk
JSON) and the typed-value flow is observable as a single
`shigoto::Dag` TickReceipt stream.

### M2 — Provider registry + lock file round-trip (≈2 months)

Local, OpenTofu, Terraform registry protocols.
`.terraform.lock.hcl` round-trip byte-exact with OpenTofu (typed
`HclValue` AST emitter for the lock file — narrow surface; **no HCL
parser**).

**Gate:** Provider downloads from OpenTofu + Terraform registries
succeed; lock files round-trip byte-equal.

### M3 — Full CLI parity + complete MCP surface (≈2 months)

`state list/show/mv/rm/replace-provider`, `import` with config
generation, `taint`/`untaint`/`force-unlock`, `output -json`,
`refresh`, `get`, `console`, `attest`, `config get/set/edit`,
`flow run`. MCP server exposes the full tool surface (every native
subcommand has a typed MCP wrapper).

**Gate:** Every documented `tofu` CLI command has a `magma` equivalent
with matching flags + exit codes + stdout/stderr format. Every
native magma subcommand has a typed MCP tool.

### M4 — Attestation + tameshi integration (≈1 month)

`magma-attest`: BLAKE3 over (config + state + plan);
`magma plan -attest=<path>` writes a tameshi-compatible receipt;
`magma apply --plan-id <hash>` requires a verified prior plan;
tabeliao registry for cluster-applied plans.

**Gate:** `tameshi typed-primitive-audit` proves the plan-attestation
surface; a plan applied by `magma` and re-verified by tameshi /
sekiban / kensa carries the full compliance chain.

### M5 — Fleet rollout (≈ongoing)

`substrate/lib/infra/pangea-arch-workspace.nix` swaps `tofu`
invocations for `magma` invocations. pleme-io workspaces migrate one
at a time behind `skip-magma:` markers. `akeylesslabs/*` Terragrunt
workspaces remain on tofu (per pleme-io/CLAUDE.md ★ Pangea-always
exception — Terragrunt wraps tofu; magma doesn't yet implement the
Terragrunt override surface).

**Gate:** `skip-magma:` audit across pleme-io shows zero outstanding
deviations except the documented `akeylesslabs/*` exception.

### Total

~12 focused months for M0–M4; M5 is rollout, not implementation.

The HCL parser milestone from the v1 draft (~4 months) is
**eliminated**: magma reads Pangea Ruby + Terraform JSON only, never
raw HCL. The Pangea-Ruby-in-process path arrives at M0, not as an
afterthought. Trade-off: operators with raw-HCL workspaces must
render to Terraform JSON first or migrate to Pangea Ruby — that's
the deliberate scope choice, not a gap.

---

## VII. Compounding Directive alignment

| Principle | Magma satisfies how |
|---|---|
| 0. Path of least resistance is forbidden | Easy path: "shell out to tofu forever." Hard path: "own the executor." This doc is the hard path locked in writing **before** any code, exactly as the principle requires. M0–M6 are stations on the way to §II–V; they are not the destination. |
| 1. Solve once, in one place | Today's eight failure modes (§I) live in eight ad-hoc handlers across pleme-io workspaces. Magma collapses them to one typed primitive that every workspace consumes. |
| 2. Load-bearing fix | Instead of patching state-file edge cases workspace-by-workspace, or chasing tofu's plan-diff quirks, replace the executor. The fix lands once, at the layer that owns the concern. |
| 3. Idiom-first | shikumi types + tatara-lisp surface + shigoto work-graph + nix-ast-peer emission + tameshi attestation + cofre secret materialization. Magma is the composition of existing pleme-io primitives applied to IaC, not a new foreign-idiom import. |
| 4. Models stay current | This doc, [`theory/README.md`](./README.md) index, [`pleme-io/CLAUDE.md`](../CLAUDE.md) ★★ primitive index, [`pangea-architectures/CLAUDE.md`](../pangea-architectures/CLAUDE.md) all updated synchronously with each milestone. |
| 5. Direction beats velocity | Magma *is* direction — long-term IaC sovereignty. Velocity says "use tofu, it works." Direction says "the substrate must own its execution layer or it doesn't compound." |
| 6. Single goals are anti-goals | Not "make workspace X deploy faster." Make IaC executors a typed primitive that compounds across all workspaces, forever — and across markets that won't accept the opaque Go binary today (regulated, life-safety, air-gapped). |
| 7. Acquire and contextualize | OpenTofu / Terraform / HCL / gRPC-plugin-protocol knowledge is acquired *via* magma's typed surface, not consumed raw. The HCL AST becomes a shikumi value; the provider protocol becomes magma-protocol; the state file becomes magma-types::State. Foreign-idiom isolation per principle #7. |

---

## VIII. Twelve-pillar alignment

| Pillar | Magma satisfies how |
|---|---|
| 1. Rust + tatara-lisp + WASM/WASI | Magma is Rust end-to-end; tatara-lisp `(defmagma-module …)` is one of three input syntaxes; ships as a WASI module via cargo-component. |
| 2. shikumi configuration | Every type derives `Shikumi`; HCL parses *into* shikumi values; runtime flow is shikumi end-to-end. |
| 3. forge-gen API generation | `magma-protocol` is generated from tfplugin5/6.proto via prost-build (the protobuf analog of forge-gen's OpenAPI codegen). |
| 4. SeaORM + shinka migrations | State-file schema migrations follow shinka's shape; Postgres backend (when implemented) is SeaORM. |
| 5. Pangea Ruby DSL → Terraform JSON → magma | Magma is the new Pillar-5 **execution** layer; declaration stays in Pangea. Direct in-process flow (Ruby → magma-types) skips the JSON intermediary for pleme-io consumers; JSON remains for compat. |
| 6. typescape owned by arch-synthesizer | arch-synthesizer renders typed Pangea forms into magma-types Rust values; magma extends the typescape with the executor-primitives dimension (§II.4). |
| 7. Helm + Kustomize + FluxCD rendered from typescape | The Kubernetes provider in magma reads the same typescape; no separate K8s flow. Helm releases for cluster-side stay on FluxCD (out of scope for magma); cluster-bootstrap and IaC stay on magma. |
| 8. Nix-only image building | Magma's `flake.nix` uses substrate's `rust-workspace-release` builder; no Dockerfile, no hand-rolled CI. |
| 9. SDLC: substrate flakes + `nix run .#app` + fleet DAG + repo-forge archetypes | Magma is scaffolded via repo-forge (`rust-workspace` archetype), built via substrate, run via fleet DAG, distributed via tatara-packaging. |
| 10. Proof discipline: `cargo test` = verification | proptest against opentofu's output for every plan-diff algorithm; rspec for golden state files; tameshi BLAKE3 chain for plans; kensa compliance dimensions for regulated rollouts. |
| 11. JIT Infrastructure | Magma's apply engine is shigoto-scheduled; long-running applies dispatch via [`pangea-jit-builders`](../docs/pangea-jit-builders.md) (Mac dispatches to Linux JIT capacity for the heavy lifting). |
| 12. Generation over composition | `magma-protocol` is generated from .proto. `magma-types::ResourceKind` enums for typed-resource access are generated per provider schema (analog of crossplane-forge's typed-AST emission for Go providers). The executor framework itself is composition; the per-provider typed-resource surface is generation. |

---

## IX. Anti-patterns this primitive forbids

- **Shelling out to `tofu`/`terraform` from any pleme-io workspace.**
  Magma is the executor. Documented exceptions in §VI.M5 only.
- **Reading raw HCL.** Magma never reads `.tf` files. Pangea Ruby
  (in-process) + Terraform JSON are the only input contracts.
  Operators with raw-HCL workflows render to JSON via
  `terraform-config-inspect` or migrate to Pangea Ruby. Adding an
  HCL parser to magma is a foreign-idiom leak forbidden by §I, §II.1,
  §II.2.
- **Disk roundtrips in the in-memory chain.** Within a single magma
  process (`magma flow run`, multi-workspace constellation,
  MCP-driven multi-step flow), typed values pass between Jobs as
  Rust values — never `tempfile + serde_json::to_writer + read back`.
  Disk persistence is reserved for state file (durability), lock file
  (concurrency), and attestation receipts (provenance). Per §II.9.
- **Bypassing the shigoto Job wrapper.** Every magma operation
  surfaces as a typed `shigoto::Job` (per §II.9). Side channels
  (direct provider RPC outside `magma-plugin`, raw state mutation
  outside `magma-state`'s API, ad-hoc retry loops outside
  `shigoto::RetryPolicy`) are forbidden.
- **Custom plan executors per cluster, per workspace, per service.**
  Every cluster, workspace, fleet uses one magma binary. Per the
  [`CASCADE-RULE.md`](./CASCADE-RULE.md): one executor across N
  consumers is the only acceptable shape.
- **Hand-written gRPC clients for tfplugin5/6.** `magma-protocol`
  owns the protobuf surface. No consumer rewrites it.
- **Hand-rolled MCP tool definitions.** `magma-mcp::tool_specs` is
  generated mechanically from `magma-types` schemas (Pillar 12).
  Hand-authored tool definitions diverge silently from the typed
  surface; per §II.8 interface 3, this is forbidden.
- **Parallel terraform-shim implementation outside `magma-cli`.**
  Drop-in compat lives in `magma-cli/src/main.rs`'s `detect_mode` +
  `capture_tf_env` machinery (§II.8 interface 1). Re-implementing
  argv[0] sensing or TF_* env handling in a sibling crate or external
  wrapper is duplication forbidden by Compounding Directive #1.
- **Lock-file HCL parsing.** `.terraform.lock.hcl` is **emitted** via
  a typed `HclValue` AST (M2; peer of [`NIX-AST.md`](./NIX-AST.md));
  it is **never parsed** by magma — the lock file is content magma
  owns end-to-end, so the emitted form is the canonical form.
- **Inline state-file mutation.** `magma-state` is the only writer.
  State-migration scripts go through `magma-state`'s migration API,
  never raw JSON. State-file editing tools (the rare manual
  intervention) use `magma state mv/rm/import`, never `vim
  terraform.tfstate`.
- **Provider-specific magic.** Every provider talks tfplugin5/6 over
  gRPC. No provider gets a hand-rolled non-gRPC backdoor; if a
  provider needs something the protocol doesn't expose, the protocol
  is extended (and the upstream proto file changes).
- **Plan caching without attestation.** Every cached plan has a
  `PlanId` (BLAKE3). Plans without an attestation receipt are
  inadmissible to `magma apply --plan-id <id>`.

---

## X. Open questions

A short list to resolve before M0 ships. Each is tracked as an open
issue in the magma repo once it exists; this section is the
authoritative anchor:

1. **Plan-diff non-determinism in OpenTofu itself.** OpenTofu has known
   plan-diff quirks (ordering of `aws_security_group_rule` ingress,
   `for_each` over computed keys). Do we match bug-for-bug, or improve
   and risk silent drift?
   - **Tentative answer:** Match bug-for-bug for M0–M2 (compatibility
     is the gate). Track each bug as a `magma_known_quirk!` proptest
     case. After M5, audit each quirk; upstream the fix to OpenTofu
     when both projects agree; only then drop the matching quirk in
     magma.

2. **WASI provider plugins.** The WASM component-model providers (when
   they ship — currently a Crossplane-led effort) won't speak the
   tfplugin5/6 protocol. How much surface area do we reserve now?
   - **Tentative answer:** Reserve a `Provider` trait shape that
     accommodates both. `GrpcProvider` is one impl; `WasmProvider`
     becomes another later. No premature abstraction beyond the trait
     boundary.

3. **State-file v5.** OpenTofu hasn't shipped one, but it's likely in
   1.x. Versioning strategy?
   - **Tentative answer:** `magma-state` versions its serialization
     independently; the on-disk format follows OpenTofu's. Internal
     shikumi typing absorbs the format change with a `From<StateV4>
     for StateInternal` impl.

4. **Terraform Cloud workspace integration.** Out of scope, or
   `magma-cloud` crate later?
   - **Tentative answer:** Out of scope through M5. Terraform Cloud is
     a HashiCorp commercial surface; pleme-io doesn't use it; if a
     consumer needs it, `magma-cloud` is the natural crate.

5. **Provisioners (`local-exec`, `remote-exec`, `file`).** Deprecated
   upstream but still in production use.
   - **Tentative answer:** Supported in M0, behind a `provisioners`
     feature flag. Documented as legacy in the magma CLI help.
     `local-exec` runs in the WASI sandbox with `wasi-cli` —
     deterministic where commands are deterministic.

6. **Cross-provider implicit dependencies.** OpenTofu's "data source
   reads on apply if dependencies change" semantics are subtle.
   - **Tentative answer:** Match exactly; proptest case per documented
     behavior; flag any divergence as a M0 release blocker.

7. **Sensitive-mark propagation through functions.** A particularly
   tricky byte-exact surface.
   - **Tentative answer:** `magma-functions` propagates `SensitivePaths`
     through every function call per OpenTofu's `marks` package
     semantics; proptest per function.

---

## XI. How this doc evolves

- Status field updates per milestone gate. After M0 ships: status →
  "M0 shipped; M1 in flight."
- Open questions resolve into prose in the relevant section, never
  delete-without-trace — closed questions move to a "resolved
  questions" appendix.
- This doc is the canonical destination. If implementation diverges,
  this doc updates first, then code. Per [`README.md`](./README.md):
  *if there is ever a contradiction between this doc and the
  implementation, this doc wins and the implementation is a bug to
  fix* — *unless* this doc is itself wrong, in which case the
  contradiction is the bug-report and the doc updates as the fix.
- Cross-references: every commit that changes a load-bearing piece
  of magma updates this doc, [`README.md`](./README.md) index,
  [`pleme-io/CLAUDE.md`](../CLAUDE.md) ★★ primitive index, and
  [`pangea-architectures/CLAUDE.md`](../pangea-architectures/CLAUDE.md)
  synchronously. Per Compounding Directive principle #4 (models stay
  current).
