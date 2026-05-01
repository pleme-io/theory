# Pangea Workspace Reconciliation — pangea-operator design

> **Frame.** Read [`THEORY.md` §IV (Motion)](./THEORY.md) and
> [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> first. This doc operationalizes the operator-side contract that
> makes Pangea-driven infrastructure converge predictably.

## North star (one sentence)

`pangea-operator` is **a workspace reconciler** whose
Rust + tatara-lisp + WASM/WASI + Nix + YAML harness loads
architecture gems (Ruby Pangea stays canonical), smoke-tests every
typed primitive against shipped fixtures, and gates `kubectl apply`
of an `InfrastructureTemplate` at admission so by the moment a
workspace enters reconciliation **everything it needs is provably
present and contract-consistent**.

This doc is the substrate-level plan that emerged from the saguão
Phase 1 cloudflare-tunnel `LoadError` — a class of failure that
should be *impossible by construction* under the design below.

---

## ★ The four primitives

| Primitive | What it is | Authored in |
|---|---|---|
| **ArchitectureGem** | A typed catalog of cloud-resource primitives (e.g. `pangea-architectures`, `pangea-aws`, `pangea-akeyless`, `pangea-cloudflare`). Owned by gem maintainer. Published as a Ruby gem (today) or WASM module (later — same Ruby gem stays canonical). | Ruby Pangea DSL |
| **Workspace** | A git tree containing infrastructure templates + per-environment YAML config (e.g. `pangea-architectures/workspaces/cloudflare-pleme/`). Cloned per-template by the operator. Owned by workspace owner. | Ruby Pangea + YAML |
| **Template** | A unit of infrastructure declaration: existing `pangea.pleme.io/v1alpha1 InfrastructureTemplate` CR. Calls into ArchitectureGems to declare resources. Owned by template author. | YAML CR (today) + optional tatara-lisp `(defpangeatemplate ...)` form (M6+) |
| **Resource** | An atomic Terraform resource (e.g. `cloudflare_record`, `aws_ec2_instance`). Match-selector-targetable from Template policies. | Implicit — emitted by gem code |

Pangea-operator's job is to **reconcile every Template against every
Workspace, using the loaded ArchitectureGem catalog, applying the
hierarchical policy cascade**.

## ★ Hierarchical policy cascade

Reconciliation policies + reactions are typed and live at four
levels. Inheritance flows outer → inner; inner overrides outer with
**safety precedence** (`refuse > requireApproval > autoApply`):

```
ArchitectureGem  →  Workspace  →  Template  →  Resource
```

| Level | Sets defaults for | Authored by | Example use |
|---|---|---|---|
| ArchitectureGem | every workspace whose templates load this gem | gem maintainer | "Cloudflare Tunnel resources NEVER auto-destroy without explicit override" |
| Workspace | every template in the workspace | workspace owner | "cloudflare-pleme workspace requires approval for any change to a `cloudflare_zone` resource" |
| Template | every resource the template declares (existing CR field) | template author | "rio-drive-cloudflare-tunnel: refuse delete/replace on the tunnel resource itself" |
| Resource | this specific cloud resource (terraform-typed match selectors) | template author | "this one DNS record: autoApply, no approval" |

Policy fields are typed and consistent across levels:

- `reconcileInterval` — how often to plan
- `driftReaction` — `autoApply | requireApproval | refuse | alert`
- `settlingPolicy.maxConsecutiveDriftCycles` + `onExhaustion`
- `destroyProtection`
- `approvalRouting` — oncall ntfy topic, Slack channel, GitHub issue template
- `breakGlass` — explicit annotation required to override a gem's `refuse` at a lower level

Effective policy at a Resource is computed by walking the cascade,
merging in order, conflicts resolved by safety precedence. Operator
exposes resolved policy via `kubectl pangea policy explain <resource>`
so a human can see exactly which level each field came from.

## ★ Reconciliation state machine

Each Workspace × Template pair traverses this state machine. Every
transition has typed inputs/outputs in Rust; transitions are
auditable; humans can predict what happens next.

```
                    ╭──── failure ────► Failed
                    │                    (terminal)
   Pending ──► Cloning ──► Loading ──► Verified ──► Planning ──► Applying ──► Settled
                                          │                                      ▲
                                          │                                      │
                                          │              ╭──── drift detected ───╯
                                          │              │
                                          ▼              ▼
                                       (admission     Drifted ──► (per-policy reaction:
                                        gate)                     autoApply, alert,
                                                                  requireApproval, refuse)
```

Phases:

1. **Pending** — InfrastructureTemplate just admitted; not yet picked up by reconciler.
2. **Cloning** — operator clones the workspace's git ref into a scratch dir.
3. **Loading** — operator queries the typed ArchitectureGem registry: are all gems this workspace's templates need actually loaded + smoke-tested?
4. **Verified** ★ — the typed gate. Past this point, gem-loading-failure modes are eliminated. Failures here surface as `Loading=False/ArchitectureGemMissing` with a specific message naming the missing primitive.
5. **Planning** — Ruby compiler runs the workspace template; OpenTofu produces a plan against the resolved policy cascade.
6. **Applying** — OpenTofu applies. Per-resource policies determine whether the apply happens automatically or stalls awaiting approval.
7. **Settled** — terminal happy state. Drift watcher monitors at `reconcileInterval`.
8. **Drifted** — drift detected; reaction depends on resolved policy.
9. **Failed** — terminal sad state. Specific reason field names the substrate-level cause (e.g. `ArchitectureGemMissing`, `TerraformPlanError`, `CompilerCrash`).

## ★ The seven additions

What gets built INTO `pangea-operator` to deliver the design above.
None replace Ruby Pangea — every addition wraps or hardens it.

### M1 — Typed ArchitectureGem loader (Rust) + smoke-test harness

**Goal:** the `cloudflare_tunnel` `LoadError` becomes impossible.

- New Rust module `pangea_arch_loader` in pangea-operator.
- New CRD `ArchitectureGem` (cluster-scoped):
  ```yaml
  apiVersion: pangea.pleme.io/v1alpha1
  kind: ArchitectureGem
  metadata:
    name: pangea-architectures
  spec:
    gemName: pangea-architectures
    version: "0.x"
    sourceRef:                # gitRepository or OCI image (gem published as OCI artifact)
      gitRepository:
        url: https://github.com/pleme-io/pangea-architectures
        ref: main
    expectedClasses:          # operator refuses to advance if any missing
      - Pangea::Architectures::CloudflareTunnel
      - Pangea::Architectures::CloudflareDomain
      - Pangea::Architectures::CloudflareDnsRecords
      - Pangea::Architectures::CloudflareHeadlessBlog
    fixtures:                 # smoke-test inputs that must compile to valid TF JSON
      - className: Pangea::Architectures::CloudflareTunnel
        path: spec/fixtures/cloudflare_tunnel.rb
      - className: Pangea::Architectures::CloudflareDomain
        path: spec/fixtures/cloudflare_domain.rb
    policy:                   # gem-level defaults (cascade root)
      destroyProtection: true
      driftReaction: requireApproval
  ```
- Loader RPCs the compiler sidecar at startup: "load this gem; report which classes are present; run each fixture; confirm valid TF JSON output."
- Operator's typed registry records: `{ gemName → { version, classes, schemas, smokeStatus } }`.
- Operator refuses to advance any reconcile if registry is empty or any expected class failed to load. Surfaces as a fleet-wide alert.

**Saguão tonight unblocks here.** Adding `Pangea::Architectures::CloudflareTunnel` to the registry's expected list, with a fixture, surfaces the live `LoadError` as a typed condition on the `ArchitectureGem` CR — no more silent retry-loop. Once the gem packaging is fixed (separate session), the loader passes smoke-test and the saguão `auth.quero.cloud` + `cracha.quero.cloud` records reconcile cleanly.

### M2 — Workspace state machine (typed Rust)

- Reify the state machine above into a Rust enum + transition table.
- Each transition has a typed precondition: `Verified` requires `ArchitectureGem.smokeStatus == Passed` for every gem the workspace needs.
- Status conditions on `InfrastructureTemplate`:
  - `Type=Loading, Reason=ArchitectureGemMissing/Loaded`
  - `Type=Verified, Reason=SmokeTestPassed/Failed`
  - `Type=Planning, Reason=TerraformPlan{Created,Errored}`
  - `Type=Applying, Reason=AwaitingApproval/Applied`
  - `Type=Settled, Reason=Healthy/Drifted`
- Drift detection runs on `reconcileInterval` from the resolved policy; reaction follows the cascade.

### M3 — Hierarchical policy resolver (Rust)

- Cascade computation: walk gem → workspace → template → resource policies, merge with safety precedence.
- New CRD `WorkspaceCatalog` (cluster-scoped, declares which workspaces the operator watches + workspace-level policy):
  ```yaml
  apiVersion: pangea.pleme.io/v1alpha1
  kind: WorkspaceCatalog
  metadata:
    name: cloudflare-pleme
  spec:
    workspace:
      gitRepository:
        url: https://github.com/pleme-io/pangea-architectures
        ref: main
      path: workspaces/cloudflare-pleme
    requiredGems: [pangea-architectures]
    policy:
      reconcileInterval: 5m
      driftReaction: autoApply
      settlingPolicy:
        maxConsecutiveDriftCycles: 4
        onExhaustion: alert
      approvalRouting:
        ntfyTopic: rio-pangea-approvals
  ```
- Effective policy at a Resource queryable via `kubectl pangea policy explain rio-architectures/rio-drive-cloudflare-tunnel/cloudflare_record.auth_quero_cloud`. Output names which level each field came from + why.

### M4 — Admission webhook (Rust → WASM)

- ValidatingAdmissionWebhook on `InfrastructureTemplate` and `WorkspaceCatalog` CRs.
- Webhook policy compiled from Rust to a WASM module (`pangea-arch-admission.wasm`) — sandboxed, hermetic, fast.
- Webhook queries the operator's typed registry: "is the architecture this template references actually loaded? do its spec values type-check against the registered schema? does the policy cascade resolve cleanly without unauthorized lower-level overrides of `refuse` policies?"
- Reject up front. The user sees a clear `kubectl apply` error like:
  ```
  error: admission webhook "pangea-arch-admission.k8s.pleme.io" denied:
  Template references Pangea::Architectures::CloudflareDoesNotExist,
  but the operator's loaded registry has classes: [CloudflareTunnel,
  CloudflareDomain, CloudflareDnsRecords, CloudflareHeadlessBlog].
  Either fix the template or add the missing class to the
  pangea-architectures ArchitectureGem.
  ```

### M5 — `WorkspaceCatalog` + `ArchitectureGem` CRDs shipped

- Both CRDs land in `pleme-io/pangea-operator`'s chart.
- Existing `InfrastructureTemplate` CR remains; gains a `workspaceCatalogRef` field linking it to its parent (for cascade resolution).
- Backward compat: templates without a `workspaceCatalogRef` use a default catalog matching their git source.

### M6 — Tatara-lisp authoring layer (optional)

- New tatara-lisp domain: `(defpangeatemplate ...)` form via `#[derive(TataraDomain)]`.
- Renders to the existing `InfrastructureTemplate` YAML CR.
- Fails at Lisp expansion if it references an architecture not in the operator's loaded registry — fastest possible feedback.
- Authors who prefer YAML directly keep doing that. This is purely additive.
- Example:
  ```clojure
  (defpangeatemplate rio-cf-tunnel
    :workspace-catalog cloudflare-pleme
    :architecture cloudflare-tunnel
    :name 'rio'
    :zone 'quero.cloud'
    :ingress [{:hostname "auth.quero.cloud" :service "http://lareira-authentik-server.authentik.svc.cluster.local:80"}
              {:hostname "cracha.quero.cloud" :service "http://cracha-lareira-cracha.cracha.svc.cluster.local:80"}
              {:hostname "drive.bristol.quero.cloud" :service "http://ocis.drive.svc.cluster.local:9200"}
              {:hostname "rio.bristol.quero.cloud" :service "ssh://127.0.0.1:22"}])
  ```

### M7 — Cross-architecture rollout

- Apply M1–M5 to every architecture gem the fleet uses:
  - `pangea-architectures` (Cloudflare primitives)
  - `pangea-aws` (AWS primitives)
  - `pangea-akeyless` (Akeyless primitives)
  - `pangea-azure`, `pangea-gcp`, `pangea-hcloud`, etc.
- Each gem's CI gate proves its `ArchitectureGem` CR set ships with valid fixtures and clean smoke-tests.
- Fleet-wide guarantee: every operator deployment reconciles only architectures it has provably loaded.

### M8 — Hollow-out + embed Artichoke (eliminate the compiler sidecar)

The motivation arrived during M1: bundling `pangea-architectures`
into the compiler image required regenerating Gemfile.lock +
gemset.nix in lockstep with every gem change, then rebuilding +
publishing the compiler image, then bumping the rio HelmRelease.
**Adding a gem became a five-step image-rebuild loop.** That
contradicts "operator UNDERSTANDS gems as runtime artifacts" — the
M1 design intent.

The fix is structural, not workflow: hollow out the Ruby compiler so
its only job is *evaluating the Pangea DSL*. Everything else moves
to Rust. Once Ruby's surface is reduced to "pure-Ruby DSL evaluator
with no I/O", Artichoke (embedded Ruby in Rust) becomes a viable
runtime — and the compiler sidecar disappears entirely.

**The C-dep audit:**

Pangea's Ruby is **mostly pure Ruby**. C extensions live in I/O
glue, not the DSL itself:

| Component | Status | Migration |
|---|---|---|
| pangea-core, terraform-synthesizer, dry-struct, all pangea-* | pure Ruby | keep as-is |
| sinatra | pure Ruby (pulls puma) | replaced by axum already in pangea-operator |
| puma | C ext for performance | replaced by axum |
| json (Ruby) | C-accelerated parser | serde_json (Rust side) |
| psych (YAML) | libyaml C ext | serde_yaml (Rust side) |
| bundler | mostly pure with some C edges | replaced by typed `Gem` Rust struct + `git clone` per CR |

**The migration shape (M8.1 → M8.5):**

- **M8.1** ✅ (2026-04-30): Audited Gemfile.lock; classified every dep as
  R-replace / R-bypass / K-cruby / K-pure / K-native-perf. Net finding:
  the bundle has exactly two non-default-gem native extensions (puma,
  nio4r), both fall out with sinatra → axum. Full table:
  [`PANGEA-COMPILER-CDEP-AUDIT.md`](./PANGEA-COMPILER-CDEP-AUDIT.md).
  Spike result on Artichoke: archived 2025-11-03 — pivoted to magnus
  (active CRuby FFI). Hollow-out plan structure unchanged; only the
  embed crate substitutes.
- **M8.2**: Move HTTP routing from sinatra/puma to axum endpoints
  in pangea-operator. Compiler RPCs (`/compile`, `/v1/architectures`,
  `/v1/architectures/smoke-test`) become axum handlers that internally
  call into the embedded Ruby evaluator. Sinatra/puma deleted from
  the bundle.
- **M8.3**: Move JSON + YAML parsing to Rust. Operator parses request
  bodies via serde, hands Ruby a typed Hash; Ruby returns a Hash;
  operator serializes via serde back to the wire. Pangea Ruby stops
  importing `json` and `yaml`/`psych`. Bundle shrinks again.
- **M8.4**: Embed Artichoke. Operator's evaluator module is a Rust
  function `eval_pangea(input: Hash, source: &str) -> Hash` backed
  by an Artichoke interpreter. Per-CR `git clone` + `$LOAD_PATH`
  prepend handled in Rust before invoking Artichoke. Each `ArchitectureGem`
  CR's clone is cached in a workspace dir keyed on `{gemName, version}`;
  only invalidated on gemRef change.
- **M8.5**: Delete the compiler sidecar from the helm chart values.
  Operator pod becomes single-container. `pangea-compiler` container
  image is archived. End state: pangea-operator is a single Rust
  binary that evaluates Pangea DSL in-process via Artichoke against
  per-CR-cloned gems.

**Why this matters:**

Adding a new architecture gem to the fleet becomes a one-line
ArchitectureGem CR — the operator clones the gem at the requested
ref, smoke-tests every fixture, registers the typed namespace, no
image rebuilds anywhere in the loop. Workspace SDLC iteration on
gem code happens at git-push speed, not image-publish speed. This
is the substrate guarantee the M1 design promised but couldn't fully
deliver while a Ruby image was in the path.

**Risks + mitigations:**

| Risk | Mitigation |
|---|---|
| ~~Artichoke metaprogramming gaps~~ — Artichoke archived 2025-11-03 | RESOLVED: pivoted to magnus + embedded CRuby. dry-struct + terraform-synthesizer run unmodified against real CRuby; no compatibility surface to validate. |
| C extension we missed | M8.1 audit ([`PANGEA-COMPILER-CDEP-AUDIT.md`](./PANGEA-COMPILER-CDEP-AUDIT.md)) tabulated every dep; bundle has zero non-default-gem natives after sinatra/puma/json removal. CI gate refuses to publish operator builds where the magnus embed fails any smoke fixture. |
| Performance regression vs sidecar MRI | Pangea evaluation is per-CR + cached; throughput isn't the bottleneck. Magnus calls real CRuby — same eval cost as today. Compile latency for cold-start template is bounded by gem-clone time, not eval time. |
| Loss of CRuby debugging tools | Compiler sidecar can stay alongside Artichoke during the cutover as a fallback; deprecate only after M8.4 has hit a green-build week on rio. |

**This is the recommended next major after M2–M7 land.** Aggregates
to: pangea-operator becomes the single Rust binary that reconciles
workspaces, loads architecture gems at runtime via Artichoke, and
makes the rebuild-per-gem cycle structurally impossible.

## ★ Nix packaging — the bottom-line CI gate

Operator + compiler sidecar + every architecture gem + WASM admission
policy ship as ONE Nix package set. CI gate
`nix run pleme-io/pangea-operator#verify-coverage` is the bottom-line
guarantee: every architecture the operator claims to support is
actually loadable + smoke-tested + has an admission policy entry.
Builds that fail this gate refuse to publish.

This makes the load-test-verify chain immutable across releases.

## ★ Saguão Phase 1 as the M1 use case

The cloudflare-tunnel `LoadError` blocking auth + cracha DNS is the
**first concrete validation** of the design. Tonight's path:

1. **Manual bridge** — directly create the auth + cracha DNS records
   in Cloudflare via the existing token (one-shot, documented as a
   bridge in `clusters/rio/architectures/drive-cloudflare-tunnel.yaml`).
2. **M1 implementation** — package pangea-architectures properly
   (publish as gem or vendor into compiler image's Gemfile.lock), add
   the smoke-test fixture for `Pangea::Architectures::CloudflareTunnel`,
   add the loader Rust module, add the `ArchitectureGem` CRD.
3. **Re-verify** — once the loader passes smoke-test, remove the
   manual bridge; pangea-operator reconciles cleanly. The `LoadError`
   class of failure is now structurally impossible.

Every architecture migrated through M1+ adds compounding fleet
guarantees. The user-facing promise: *if you `kubectl apply` an
InfrastructureTemplate and it's admitted, the operator will reconcile
it (modulo environmental failures). No more "stuck in Compiling" for
substrate-internal reasons.*

## ★ Canonical contracts (ruthless standardization)

Per the user directive (2026-04-30): every architecture gem and every
workspace conforms to ONE typed contract. CI-gated, not optional.

### ArchitectureGemContract — what every `pangea-*` gem MUST ship

```
pangea-<name>/
├── pangea-<name>.gemspec        # standard rubygems metadata
├── lib/
│   └── pangea/
│       ├── architectures/       # IFF this gem defines architectures
│       │   └── <name>.rb        # one file per architecture
│       └── resources/           # IFF this gem defines primitive resources
│           └── <provider>.rb
├── spec/
│   ├── architectures/<name>_spec.rb         # RSpec coverage REQUIRED
│   └── fixtures/<name>.yaml                 # smoke fixture REQUIRED
└── pangea-gem.yaml              # the contract declaration (NEW; M7)
```

`pangea-gem.yaml` declares:

- `gem_name` + `version`
- For each architecture / resource: `class_name`, `fixture_path`,
  `emit_resource_types` (list of TF resource types it generates),
  `default_policy` (destroyProtection, driftReaction).
- Required Ruby gem dependencies (already in gemspec, mirrored here
  so the operator can verify without parsing Ruby).

CI gates (operator-side):

- `nix run pleme-io/pangea-operator#verify-gem -- <gem>` runs every
  fixture against the operator's compiler image and validates
  `pangea-gem.yaml` matches reality. Operator builds refuse to
  publish if any gem in the bundle fails this gate.
- `cargo test pangea_arch_loader::contract_tests` round-trips every
  gem's `pangea-gem.yaml` into the typed `ArchitectureGemContract`
  Rust struct.

### WorkspaceContract — what every workspace tree MUST ship

```
<workspace>/
├── account.yaml              # cloud account + credentials slot
├── workspace.yaml            # NEW: typed contract declaration
├── domains/                  # OR: clusters/, accounts/, fleet/ — per-shape
│   └── <name>.yaml           # per-domain config (loaded by templates)
├── <name>.rb                 # one Ruby template wrapper per "thing"
└── Gemfile                   # required architecture gems
```

`workspace.yaml` declares:

- `name`
- `required_gems` (with version constraints)
- `templates` — one entry per `.rb` file with: `name`, `path`,
  `architecture_classes` (which Pangea::Architectures::* it calls).
- `policy` (workspace-level cascade).

The operator's `WorkspaceCatalog` CR points at a workspace, the
operator parses `workspace.yaml`, and refuses to admit any
InfrastructureTemplate whose architecture isn't in the workspace's
declared list.

### Reconciler protocol — the ONE state machine

Implementation lives in pangea-operator (Rust). All five stages
have typed inputs/outputs in Rust; transitions are auditable.

| Stage | Input | Action | Typed output |
|---|---|---|---|
| Discover | ArchitectureGem CR set | RPC compiler `/v1/architectures` per gem | typed registry: `Map<gem, Vec<ClassSchema>>` |
| Verify | typed registry + WorkspaceContract | run every fixture; type-check spec values; refuse if any unknown class | `Verified` typed witness, attached to InfrastructureTemplate status |
| Plan | Verified template | clone workspace; compile via existing /compile RPC; tofu plan | terraform plan JSON, typed |
| Apply | plan + cascaded policy | tofu apply (or stall on requireApproval) | applied state, typed |
| Drift-watch | reconcileInterval timer | tofu plan; diff against last apply | `Drifted` or `Settled`, typed; reaction per cascade |

### Adopt-once, compose-many

Once these contracts land, ANY operator that consumes a pangea
architecture gem (kenshi for ephemeral testing, shinka for
migrations, future operators built on these primitives) inherits the
same contract via the typed `ArchitectureGemContract` Rust struct.
Same loader. Same smoke gate. Same reconciliation guarantees. The
substrate compounds: adding a new architecture to ANY gem becomes a
five-file change (Ruby class + RSpec + YAML fixture + gem.yaml entry
+ ArchitectureGem CR), and every consumer benefits without knowing
the new architecture exists.

## ★ Open questions

To resolve before M1 lands:

1. **Gem distribution** — publish `pangea-architectures` to RubyGems.org
   (clean, but requires public release process), or vendor as a
   Bundler git source in the compiler image's Gemfile (less clean,
   pleme-io-internal). Lean toward vendoring for now;
   RubyGems publish is a follow-up.
2. **Compiler RPC shape** — Rust→Ruby IPC. Options: HTTP API on the
   sidecar (simple), Thrift-shaped RPC (typed), MessagePack over
   Unix socket (fast). Recommend HTTP+JSON for M1; revisit if
   throughput becomes an issue.
3. **Fixture format** — should fixtures be Ruby files
   (`spec/fixtures/<class>.rb`) or YAML (`spec/fixtures/<class>.yaml`)
   that get fed to the architecture's `.build` method? Recommend YAML
   for cross-language readability; Ruby fixtures only when needed.
4. **WASM build target** — admission webhook can compile to
   `wasm32-wasip2`. Need to verify wasmtime-rs integration with
   kube-rs admission flow. Spike in M4.
5. **State persistence** — operator's typed registry is in-memory
   today. For multi-replica HA, register goes into the operator's
   Postgres or etcd-via-kube-API. Current single-replica pangea-operator
   is fine for M1; revisit when HA becomes a requirement.

## Cross-references

- [`THEORY.md`](./THEORY.md) — frame for typed substrate engineering.
- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md) — the canonical essay this design implements.
- [`SAGUAO.md`](./SAGUAO.md) — the use case driving M1.
- [`SAGUAO-MIGRATION.md`](./SAGUAO-MIGRATION.md) — saguão Phase 1+ rollout, of which this operator hardening is the load-bearing substrate fix.
- [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) — the four-layer compute hierarchy this operator instantiates.
- [`RUST-LISP-EMBEDDING.md`](./RUST-LISP-EMBEDDING.md) — the tatara-lisp embedding pattern M6 uses.

## Maintained by

`pleme-io/pangea-operator` repo + theory contributions. Update this
doc whenever the operator's reconciliation contract changes; it is
the single source of truth.
