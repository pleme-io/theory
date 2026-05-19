# Magma as Platform — Pangea-Operator's End-to-End In-Memory Lifecycle

Status: ★★★ canonical synthesis (destination locked; phases coordinate across multiple crates)
Implementing crates: `magma`, `magma-rubygems`, `magma-nix`, `magma-tfmod`, plus `pangea-ruby-eval`, `pangea-operator`
Related: [`MAGMA.md`](./MAGMA.md), [`MAGMA-RUBYGEMS.md`](./MAGMA-RUBYGEMS.md), [`MAGMA-OPERATOR-BACKEND.md`](./MAGMA-OPERATOR-BACKEND.md), [`PANGEA-MAGMA-ORCHESTRATION.md`](./PANGEA-MAGMA-ORCHESTRATION.md), [`CONVERGENCE-SUBSTRATE.md`](./CONVERGENCE-SUBSTRATE.md), [`TESTING-SUBSTRATE.md`](./TESTING-SUBSTRATE.md)

## I. The thesis

**Pangea-operator runs the entire reconcile lifecycle — Ruby DSL evaluation, gem resolution + materialization, Pangea synthesis to Terraform JSON, state management, planning, drift classification, applying via Terraform provider gRPC, bundle attestation — in one process, in memory, attested end-to-end.** Nothing leaves the operator process unless it has to (provider gRPC calls to cloud APIs, durable state writes to S3/Postgres).

Magma is the platform. Every layer that today shells out (bundler, bundix, nix-build, terraform, opentofu) is replaced by a typed Rust primitive composed into the operator. The existing Terraform provider ecosystem stays — magma speaks the same provider gRPC protocol as OpenTofu — but everything around it becomes typed magma-managed substrate.

This is the **lean aggressive fully-featured operator** the directive calls for: one binary, one process, one BLAKE3-attested closure per reconcile, the complete Pangea → infrastructure pipeline.

## II. The pipeline

```
                                  ┌──────────────────────────────────┐
                                  │ pangea-operator-controller (K8s) │
                                  │   • CR watch                      │
                                  │   • reconcile loop                │
                                  └────────────────┬─────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                            magma — the platform                      │
│                                                                       │
│  ┌────────────────────────┐  preflight  ┌────────────────────────┐  │
│  │ magma-test-laws        │◄────────────┤ MagmaExecutor::plan    │  │
│  │ (Reconciler / Backend  │             │   • load Pangea ws      │  │
│  │  / Architecture /      │             └───┬────────────────────┘  │
│  │  Workspace / Chain)    │                 │                        │
│  └────────────────────────┘                 │                        │
│                                              ▼                        │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  Pangea Ruby evaluation (pangea-ruby-eval, embedded CRuby)       │ │
│  │   • magma-rubygems materializes the gem tree IN MEMORY           │ │
│  │     (M0 lockfile parser landed; M1-M6 fill in)                   │ │
│  │   • magma-nix (M8) evaluates Nix expressions IN MEMORY when       │ │
│  │     workspaces declare gemset.nix-style closures                  │ │
│  │   • bundix-equivalent (M7) emits gemset.nix from Lockfile        │ │
│  │   • Pangea DSL renders Terraform JSON                            │ │
│  └────────────────┬────────────────────────────────────────────────┘ │
│                   │ Terraform JSON                                    │
│                   ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  magma-config / magma-plan / magma-apply / magma-converge        │ │
│  │   • typed Plan generation                                         │ │
│  │   • magma-drift classifies + routes anomalies                    │ │
│  │   • magma-fsm walks lifecycle phases                             │ │
│  │   • magma-stream emits typed audit events                        │ │
│  │   • magma-bundle attests the reconcile closure                   │ │
│  └────────────────┬────────────────────────────────────────────────┘ │
│                   │ Plan + State                                      │
│                   ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  magma-plugin / magma-protocol — Terraform provider gRPC v5/v6  │ │
│  │   • talks the exact protocol OpenTofu speaks                     │ │
│  │   • leverages the full existing provider ecosystem               │ │
│  │   • magma-tfmod (M9) downloads + transpiles modules to typed    │ │
│  │     Pangea primitives                                            │ │
│  └────────────────┬────────────────────────────────────────────────┘ │
│                   │                                                   │
│                   ▼ (cloud API calls — only thing leaving memory)    │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │  magma-backend — durable state (S3 / PG / GCS / Consul / …)      │ │
│  │   • InMemoryBackend for fully-local reconciles                   │ │
│  │   • LocalBackend / S3Backend for production                      │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

Everything inside the dashed border lives in one operator process. Cloud API calls + durable state writes are the only egress points; the entire compute layer is in-memory.

## III. What's landed today

| Layer | Crate | Status |
|---|---|---|
| Pangea Ruby DSL evaluation | `pangea-ruby-eval` (magnus + CRuby) | Working — pangea-operator embedded_ruby feature |
| Gemfile.lock parser + BLAKE3 attestation | `magma-rubygems::lockfile` + `::attestation` | **Landed today (M0)** |
| Pangea Terraform JSON → typed Config | `magma-config` | Working |
| Typed plan diff + state-equivalence | `magma-plan` | Working |
| Typed apply (in-process Reconciler trait) | `magma-apply` | Working |
| Drift classification + DriftPolicy | `magma-drift` | Working |
| FSM lifecycle (Idle→Planning→…→Stable) | `magma-fsm` | Working |
| BLAKE3-chained audit stream | `magma-stream` | Working |
| Compliance Bundle (now carries gem_tree_attestation) | `magma-bundle` | **Extended today** |
| Universal trait-law battery | `magma-test-laws` | Working (5 layers) |
| MagmaExecutor preflight + drift + bundle + gem-tree | `pangea-operator` | **Landed today** |

## IV. What's next (the synthesis directive)

Per the user's expanding directives in this session:

### M7 — bundix-equivalent (in-memory gemset.nix emission)

Magma reads a typed `Lockfile` + emits the `gemset.nix` Nix derivation set today's pleme-io flakes consume. No `bundix` subprocess; no on-disk intermediate.

Crate: `magma-rubygems::nix` module. Produces a `serde_json::Value`-shaped Nix AST (per `theory/NIX-AST.md`), rendered through one canonical pretty-printer. Round-trips with bundix's output byte-identical for every Pangea Gemfile.lock.

### M8 — magma-nix-eval (in-memory Nix expression evaluation)

Magma evaluates Nix expressions in-process for the narrow subset Pangea workspaces use: `import`, `let-in`, function calls, attribute sets, fetchurl + `nixpkgs.callPackage` for gem derivations.

Crate: `magma-nix` (new). Owns:
* Nix expression lexer + parser (typed AST per `theory/NIX-AST.md`)
* Evaluator with lazy attribute resolution
* Fetcher for `<nixpkgs>` + remote tarballs
* Store-path materialization (deterministic content-addressed, no on-disk Nix store)

This is the second "highly leveraged" half of the bundler-replacement: magma owns BOTH the lockfile resolution AND the Nix materialization layer.

### M9 — magma-tfmod (Terraform module ingestion → Pangea primitive)

Magma downloads Terraform modules from `registry.terraform.io` / Terraform Cloud / Git URLs, parses the module's variables + outputs (HCL2), and emits a typed Pangea function/architecture wrapping it.

Crate: `magma-tfmod` (new). Owns:
* Module registry client (TF registry protocol)
* HCL2 variable/output parser (typed AST)
* Pangea-Ruby code generator that emits a `Pangea::Architectures::*` module wrapping the TF module call
* gRPC plugin compatibility check (verifies the module's provider deps are satisfiable)

Output: every existing Terraform module becomes a programmable Pangea primitive that can be composed in Ruby DSL alongside hand-written architectures.

### M10 — TF module-derived primitives as first-class Pangea citizens

Once M9 ships, Pangea workspaces can write:

```ruby
require 'pangea-tfmod'

# Auto-imported from registry.terraform.io/hashicorp/vpc/aws
Pangea::Architectures::HashiCorpVPC.build(synth, name: "my-vpc", cidr: "10.0.0.0/16")
```

Magma generates the wrapper, validates the args against the module's typed surface, calls the underlying module through the provider gRPC protocol, returns the typed result that composes with hand-written Pangea architectures.

This closes the loop: **every Terraform module ever published is a typed Pangea primitive**. The compounding effect is massive — the entire TF ecosystem becomes substrate, not just the provider gRPC surface.

## V. The compounding equation

The platform's leverage compounds in three orthogonal directions:

1. **Per-reconcile latency** drops from O(seconds) to O(milliseconds). No `bundle install` subprocess (~10s saved). No `tofu init` (~2s). No `nix-build` (~5s). Operator startup once + N reconciles share the resolved closure.

2. **Compliance attestation breadth** expands. Every reconcile bundle carries: plan hash + drift report + lifecycle history + audit chain + **gem-tree BLAKE3** (+ future: nix-store hash, tf-module hash). A compliance audit gets a single BLAKE3 to verify against; bit-identical inputs produce bit-identical bundles.

3. **Ecosystem absorption**. Every new Terraform module published anywhere is one M9 step away from being a typed Pangea primitive. Every new ruby gem with a Gemfile.lock works via M0. Every new Nix expression works via M8.

The Compounding Directive's "promises become theorems": today's operator-makes-cloud-calls becomes tomorrow's typed-substrate-derives-cloud-calls. The path of least resistance (keep shelling out to bundler/nix/tofu) is the cardinal sin; the path that builds compounding substrate is the destination.

## VI. The proof points that already exist

Read the test count as a leverage indicator:

* `magma` workspace: 431 tests pass, 0 failures (with all feature flags)
* `pangea-operator` magma executor: 20 tests pass, 0 failures
* 19 real Pangea-architectures fixtures pass the full universal law battery
* Every reconcile bundle carries a BLAKE3 identity verifiable end-to-end

The pipeline through M0 (Gemfile.lock attestation) is wired all the way through pangea-operator. The remaining milestones M1–M10 each add one more in-memory primitive to the platform.

## VII. Operating principle

Per the Compounding Directive principle #0 — destination locked first:

> The lean aggressive fully-featured operator runs the entire Pangea → infrastructure lifecycle in-memory, attested end-to-end, leveraging the existing Terraform provider ecosystem without depending on bundler / bundix / nix / tofu subprocesses.

Today's interim — operator-side bundler + nix + tofu still present — is the path-down. Every milestone landed (M0 today, M1–M10 future) removes one subprocess from operator startup + adds one BLAKE3-attested artifact to the bundle.

## VIII. Status snapshot

* M0 — Gemfile.lock parser + BLAKE3 attestation ✅
* M0 — wired into pangea-operator MagmaExecutor ✅
* M0 — Bundle.gem_tree_attestation field ✅
* M0.5 — Gemfile.lock emitter (canonical, structural round-trip) ✅
* M1 — Gemfile parser (narrow Pangea subset) ✅
* M1.5 — gemspec parser ⬜
* M2 — Molinillo-style resolver ⬜
* M3 — async fetcher + nix-hash sha256 ⬜
* M4 — VirtualGemTree + native ext compilation ⬜
* M5 — pangea-ruby-eval bridge (operator removes `bundle install`) ⬜
* M6 — bundler removed from operator startup ⬜
* M7 — bundix-equivalent (in-memory gemset.nix emission, structural) ✅
* M7.x — sha256 fill-in (needs M3) ⬜
* M8 — magma-nix (in-memory Nix expression evaluator + store materialization) ⬜
* M9 — magma-tfmod (TF module → Pangea typed primitive generator) ⬜
* M10 — TF module-derived primitives as first-class Pangea citizens ⬜

**CLI surfaces today:**
* `magma rubygems gemset-from-lock` — bundix-equivalent ✅
* `magma rubygems attest-lock` — BLAKE3 closure identity ✅
* `magma rubygems parse-gemfile` — typed Manifest JSON ✅
* `magma rubygems parse-lock` — typed Lockfile JSON ✅

**Test count today:** magma workspace 456 passing, 0 failing across
~36 magma-rubygems tests + all the existing substrate batteries.

The platform is being assembled one typed primitive at a time. Every primitive that lands removes one external dependency + adds one attested compliance artifact. Compound interest on the substrate.

**Today's deltas (single session, mantra-driven):** M0 parse + M0.5
emit + M1 Gemfile parse + M7 gemset.nix emission + CLI surface +
operator integration (Bundle.gem_tree_attestation populated when
workspace ships Gemfile.lock). The path from "user types
`bundle install`" to "magma owns the closure end-to-end" has had
its first half landed.
