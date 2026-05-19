# magma-rubygems — Magma as the Pangea Platform

Status: ★★★ canonical (destination locked; M0 not yet started)
Implementing crate: `pleme-io/magma::magma-rubygems` (to land)
Related: [`MAGMA.md`](./MAGMA.md), [`MAGMA-OPERATOR-BACKEND.md`](./MAGMA-OPERATOR-BACKEND.md), [`PANGEA-MAGMA-ORCHESTRATION.md`](./PANGEA-MAGMA-ORCHESTRATION.md), [`TESTING-SUBSTRATE.md`](./TESTING-SUBSTRATE.md)

## I. Thesis

**Magma owns the Pangea Ruby gem dependency tree end-to-end.** Bundler, rubygems.org tarballs, `vendor/bundle` directories, and per-workspace `bundle install` startup are all subsumed by typed in-memory primitives inside magma. The Pangea Ruby DSL evaluates against a virtual filesystem materialized + attested by magma; no on-disk gem cache, no subprocess `bundle exec`, no version-pinning surprises across workspaces.

Magma is the **platform Pangea executes on** — the bridge between the typed Pangea world (Ruby DSL → Terraform JSON) and the typed provider world (gRPC plugins → cloud state). The existing `pangea-ruby-eval` crate is the interpreter half; `magma-rubygems` is the dependency-resolution + gem-materialization half.

> Per the Compounding Directive principle #0 — *destination locked first, then phase down.* This doc names the destination; milestones M0–M5 are the path to it.

## II. What bundler does + what magma replaces

| Bundler responsibility | Magma replacement | Status |
|---|---|---|
| Parse `Gemfile` DSL | `magma_rubygems::gemfile_parser::parse` | not yet started |
| Parse `*.gemspec` DSL | `magma_rubygems::gemspec_parser::parse` | not yet started |
| Resolve dependency graph (Molinillo algorithm) | `magma_rubygems::resolver::resolve` | not yet started |
| Fetch gems from rubygems.org / git / path | `magma_rubygems::fetcher` (typed source enum) | not yet started |
| Compile native extensions | `magma_rubygems::native` (delegates to system toolchain or pre-built attestation) | not yet started |
| Persist `Gemfile.lock` | `magma_rubygems::lockfile::emit` | not yet started |
| `bundle exec` runtime (`GEM_PATH`, `RUBYLIB`) | `magma_rubygems::runtime::materialize` — produces a virtual `GEM_PATH` rooted in a tmpfs / in-process tree | not yet started |
| Ruby version pinning | `magma_rubygems::manifest::Ruby` field | not yet started |
| `bundle outdated` / `bundle update` | `magma rubygems outdated` / `magma rubygems update` CLI subcommands | not yet started |

What magma adds **on top of** bundler:

* **Typed gem closure** — every resolved tree carries a BLAKE3 hash over the canonical projection of `(gem name, version, source, gemspec hash) × N`. Two operators with the same `Gemfile.lock` materialize bit-identical gem closures.
* **In-memory virtual filesystem** — gems live in a typed `VirtualGemTree` (no disk I/O after fetch; subsequent workspace evaluations read from RAM).
* **Cross-workspace sharing** — pangea-operator running 100 workspaces with the same Gemfile.lock pays one resolution cost, not 100.
* **Attestation propagates** — the `magma_bundle::Bundle` carries the gem-tree BLAKE3 hash; compliance teams verify the resolved gem closure matches the recorded one.
* **Pangea DSL is the only language** — Ruby is opaque infrastructure under magma; the operator's surface is typed Plans, not Ruby objects.

## III. Existing pieces magma builds on

* [`pangea-ruby-eval`](https://github.com/pleme-io/pangea-operator/tree/main/pangea-ruby-eval) — in-process CRuby evaluator via `magnus` + `rb-sys`. Single interpreter per process (GVL constraint). Pangea DSL evaluates here.
* [`magma-config`](https://github.com/pleme-io/magma/tree/main/magma-config) — typed parser for the rendered Terraform JSON the Pangea DSL emits.
* [`magma-pangea`](https://github.com/pleme-io/magma/tree/main/magma-pangea) — the bridge between Pangea-rendered JSON workspaces and magma's typed Config.
* [`pangea-architectures`](https://github.com/pleme-io/pangea-architectures) — the canonical set of typed architecture compositions (SecureVpc, TieredSubnets, K3sDevCluster…) the DSL produces.
* [`magma-test-laws`](https://github.com/pleme-io/magma/tree/main/magma-test-laws) — universal contract test surface every Pangea-rendered workspace passes through.

`magma-rubygems` slots in **between** the Pangea Ruby source code and `pangea-ruby-eval`'s `RubyEvaluator`. It materializes the gem tree the evaluator's CRuby thread needs, then hands the evaluator a typed `RubyEnvironment` carrying the virtual `GEM_PATH`.

## IV. Crate layout

```
magma-rubygems/
  src/
    lib.rs                  # Public surface + types
    manifest.rs             # Gemfile / *.gemspec typed AST
    gemfile_parser.rs       # `Gemfile` DSL → typed Manifest
    gemspec_parser.rs       # `*.gemspec` → typed GemSpec
    lockfile.rs             # Gemfile.lock parse + emit
    resolver.rs             # Molinillo-style dependency resolver
    source.rs               # Typed source enum: RubyGemsOrg | Git | Path
    fetcher.rs              # Async fetcher per source variant
    cache.rs                # In-memory blob cache (BLAKE3-keyed)
    native.rs               # Native-extension compilation orchestration
    tree.rs                 # `VirtualGemTree` — typed materialized closure
    runtime.rs              # Bridge to pangea-ruby-eval (GEM_PATH wiring)
    attestation.rs          # BLAKE3 hash over canonical closure projection
```

Public API surface (M0 destination):

```rust
pub struct Manifest { pub ruby: RubyVersion, pub deps: Vec<Dependency> }
pub struct Lockfile { pub gems: Vec<ResolvedGem>, pub specs: Vec<Spec> }
pub struct VirtualGemTree { /* opaque, BLAKE3-attested */ }

pub async fn resolve(manifest: &Manifest) -> Result<Lockfile, ResolverError>;
pub async fn materialize(lock: &Lockfile) -> Result<VirtualGemTree, FetchError>;
pub fn attestation(tree: &VirtualGemTree) -> String; // 64-hex BLAKE3

pub fn into_ruby_env(tree: &VirtualGemTree) -> pangea_ruby_eval::RubyEnvironment;
```

## V. Milestones

### M0 — Lockfile parser (Gemfile.lock-shaped, no resolution)

Goal: read existing `Gemfile.lock` files emitted by bundler. No resolution, no fetching. Just parse the typed shape.

* `lockfile::parse` reads bundler's lockfile format and produces typed `Lockfile`.
* Round-trip test: parse → emit produces byte-identical output for every Pangea workspace's lockfile.
* Coverage gate: every `pangea-architectures/workspaces/*/Gemfile.lock` parses.

Why M0: it's the smallest piece that unblocks downstream work. The resolver (M2) consumes lockfiles for incremental updates; the fetcher (M3) consumes lockfiles for cache keys; the runtime (M5) consumes lockfiles to materialize.

### M1 — Gemfile + gemspec parsers

Goal: produce a `Manifest` from a `Gemfile` source string + parse `*.gemspec` files. Both are Ruby DSLs but a narrow subset; parse via a hand-rolled typed parser (avoid evaluating Ruby).

* `gemfile_parser::parse` reads the `source`, `gem`, `group`, `gemspec` directives. Out of scope: arbitrary Ruby (we refuse `Gemfile`s that embed eval — there are none in Pangea workspaces).
* `gemspec_parser::parse` reads `Gem::Specification.new` blocks.
* Coverage: every Pangea workspace + every Pangea gem ships parseable Gemfile + gemspec.

### M2 — Dependency resolver (Molinillo port)

Goal: given a `Manifest`, produce a `Lockfile` solving the dependency graph.

* Pure-Rust port of [Molinillo](https://github.com/CocoaPods/Molinillo) (the algorithm bundler uses).
* Handles: version constraints (`~>`, `>=`, `<`), platform pinning, dependency groups, transitive deps, conflict resolution via backtracking.
* Property test: `resolve(parse(real_gemfile))` matches the bundler-produced lockfile byte-for-byte across all Pangea workspaces.

### M3 — Fetcher + cache

Goal: download gem tarballs from typed sources (RubyGems.org / git / path), cache them BLAKE3-keyed in memory.

* Source enum: `RubyGemsOrg { name, version }` | `Git { url, ref }` | `Path { dir }`.
* Async fetcher with rate-limit awareness (samba pattern per `theory/RATE-LIMITED-CONSUMERS.md`).
* In-process LRU cache; magma-operator's reconcile path hits cache once per gem version, never twice.

### M4 — Virtual gem tree + native extensions

Goal: materialize a `VirtualGemTree` in tmpfs (or pure-RAM via FUSE/overlayfs).

* `VirtualGemTree::materialize` extracts gem tarballs into a tmpfs root; emits `GEM_PATH`.
* Native extension compilation: delegate to system `cc` / `clang` for now; M4.x considers pre-built per-(version, platform) attestations to skip compile entirely.
* BLAKE3 attestation over the canonical projection of the materialized tree.

### M5 — Pangea-ruby-eval bridge

Goal: a `pangea-ruby-eval::RubyEvaluator` can be constructed against a `VirtualGemTree` (no bundler subprocess).

* `magma_rubygems::runtime::into_ruby_env(&tree)` returns a `RubyEnvironment { gem_path, ruby_lib, ruby_version }`.
* `pangea-ruby-eval::RubyEvaluator::with_env(env)` accepts it.
* Pangea-operator opt-in: `MagmaExecutorConfig::rubygems_mode = InMemory | Subprocess` (default Subprocess until M5 burn-in).

### M6 — Pangea-operator wiring + bundler removal

Goal: pangea-operator switches to magma-rubygems by default. `bundle install` removed from workspace SDLC.

* `pangea-operator-init` materializes the shared gem tree on startup.
* Per-CR reconciles share the tree; no per-workspace gem resolution.
* Workspace startup goes from ~10s (bundle install) to <100ms (read shared tree).
* Compliance: every reconcile's `magma_bundle::Bundle` includes the gem-tree BLAKE3 hash; compliance teams export bundles + verify gem-closure attestation against the bundler-removed runtime.

## VI. Integration with the existing magma substrate

* **`magma-test-laws`** gains `rubygems-laws` feature: `resolver_is_deterministic`, `lockfile_round_trips`, `materialize_is_idempotent`, `attestation_is_stable`.
* **`magma-bundle`** gains a `gem_tree_attestation: Option<String>` field carrying the BLAKE3 of the materialized tree. Compliance bundles thus attest to the full closure (gem tree + plan + drift + lifecycle).
* **`magma-stream`** event types: `GemTreeMaterialized`, `GemResolved`, `GemFetched` emitted during the resolution + materialization phases.
* **`magma-converge`** gets a future `RubyGemReconciler` impl: given a desired Gemfile, reconcile the materialized tree (re-fetch if drift, evict stale gems, etc.).

## VII. Why "highly leveraged"

A single resolution + materialization cycle today costs each workspace ~10s × N reconciles per day. Across pleme-io's fleet:

* ~25 production workspaces × ~10 reconciles/day × 10s = **~2500s / day = ~42 min / day** wasted on bundle install.
* Each workspace's gem closure is on-disk in `vendor/bundle/` — duplicated N times across the fleet.
* No attestation of the closure — operators can't prove "the gem tree we resolved is the gem tree we ran."

After M6:

* One resolution at operator startup; tree shared via in-memory ref + BLAKE3 attestation.
* Workspace reconcile startup: ~100ms (read tree handle).
* Every `magma_bundle::Bundle` carries the gem-tree hash; compliance audits prove closure identity end-to-end.
* `Gemfile.lock` becomes a typed magma artifact, not an opaque YAML-ish file bundler-managed.

The leverage compounds: every new Pangea workspace inherits microsecond startup. Every new reconciler kind inherits attestation. Every new compliance requirement (gem-closure auditability, supply-chain provenance) is one substrate primitive away.

## VIII. Open questions

1. **Native extensions**: do we compile at materialize-time or ship pre-built `.so` artifacts attested per-platform? Probably both — pre-built where possible, fall back to compile.
2. **Ruby version drift**: magma owns the gem tree, but who owns the Ruby binary? Likely magma-rubygems materializes Ruby itself (via rbenv-compatible distribution tarballs) for full closure ownership.
3. **Git source security**: Git-sourced gems need provenance attestation. Plumb through tameshi receipts per `theory/MAGMA.md` §II.3.
4. **Cross-platform**: pleme-io targets Linux x86_64 + aarch64 + macOS. Native-ext compilation matrix gets large fast; pre-built attestations make this tractable.
5. **Migration path**: the `pangea-architectures` Gemfile + Gemfile.lock are the M0 test corpus. Round-trip parsing them all is the M0 acceptance gate.

## IX. Status snapshot

* M0: ⬜ not started
* M1: ⬜ not started
* M2: ⬜ not started
* M3: ⬜ not started
* M4: ⬜ not started
* M5: ⬜ not started
* M6: ⬜ not started

Crate skeleton: ⬜ not yet landed in `pleme-io/magma`.

Per the Compounding Directive principle #0 — *path of least resistance is a cardinal sin*: this doc commits to the long-term destination (magma-as-Pangea-platform). Interim "still use bundler" exists today, but the destination is the in-memory closure. Every workspace migration toward that destination is one less filesystem operation, one more BLAKE3-attested artifact.
