# Pangea Compiler — C-Extension Audit

**M8.1 of [`PANGEA-WORKSPACE-RECONCILIATION.md`](./PANGEA-WORKSPACE-RECONCILIATION.md).**

Goal: tabulate every dependency in the pangea-compiler bundle, classify
its native-extension status, decide its fate under the M8 hollow-out
(operator absorbs HTTP + parsing in Rust; embedded CRuby via magnus
evaluates a pure-Ruby DSL).

**Source of truth:** `pangea-operator/pangea-compiler/Gemfile.lock`
@ `pangea-operator@c73d5a9`.

## Classification scheme

| Class | Meaning | Fate under M8 |
|---|---|---|
| **R-replace** | C-ext that exists only because the compiler is a separate Ruby HTTP process. | **DELETE** from bundle. Replaced by Rust crate already in `pangea-operator`. |
| **R-bypass** | C-ext that pure-Ruby DSL doesn't actually need; only used in I/O glue. | **DELETE** from bundle. Rust does the parsing; Ruby sees a pre-built Hash. |
| **K-cruby** | Ships as part of CRuby's default-gem set; provided by the embedded interpreter. | **KEEP** — but provided by `ruby_3_4` Nix derivation, not via Gemfile. |
| **K-pure** | Pure Ruby. Loaded into the embedded interpreter as-is. | **KEEP** in bundle. |
| **K-native-perf** | Has a C accelerator but a working pure-Ruby fallback. | **KEEP** in bundle. CRuby uses the C accelerator transparently when present; magnus-embed inherits that. |

## Direct dependencies (`pangea-compiler.gemspec`)

| Gem | Version | Native? | Class | Notes |
|---|---|---|---|---|
| `sinatra` | 4.2.1 | pure Ruby | **R-replace** | Replaced by axum routes in pangea-operator. Drops ~5 transitive pure-Ruby gems with it. |
| `puma` | 6.6.1 | C-ext (HTTP parser) + libpoll via `nio4r` | **R-replace** | Replaced by tokio + axum. Drops `nio4r` (the only libev-shaped native dep in the bundle). |
| `json` | 2.19.4 | C-ext (`json/ext`) + pure-Ruby fallback | **R-bypass** | Pangea Ruby stops calling `JSON.parse`/`.to_json`. CRuby still ships `json` as a default gem; the Gemfile entry simply goes away. |
| `terraform-synthesizer` | 0.0.28 | pure Ruby | **K-pure** | The DSL evaluator. Untouched by M8 — runs in embedded CRuby exactly as today. |
| `pangea-core` | 0.3.0 | pure Ruby | **K-pure** | Composition base. Uses `dry-struct` + `dry-types`. |
| `pangea-akeyless` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-aws` | 0.2.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-azure` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-cloudflare` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. The one saguão Phase 1 needs. |
| `pangea-datadog` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-gcp` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-hcloud` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-kubernetes` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-porkbun` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-splunk` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. |
| `pangea-spot` | 0.1.0 | pure Ruby | **K-pure** | Provider gem. Depends on `pangea-aws`. |

`pangea-architectures` is **intentionally absent** from the lockfile
today (the bundling was deferred at the close of the M1 session — see
the comment at the top of `pangea-operator/pangea-compiler/Gemfile`).
Under M8.4, this gem (and any future architecture gem) lives outside
the operator image entirely: the operator clones it per `ArchitectureGem`
CR and `$LOAD_PATH.unshift`-es into the embedded CRuby. So the M8
end-state has *zero* architecture gems baked into the binary; the
gemspec keeps only foundational primitives (`pangea-core`,
`terraform-synthesizer`, `dry-*`, the provider-primitive gems).

## Transitive dependencies

| Gem | Version | Native? | Class | Notes |
|---|---|---|---|---|
| `abstract-synthesizer` | 0.0.15 | pure Ruby | **K-pure** | The `bury` + `method_missing` pattern. Foundation of the DSL. |
| `base64` | 0.3.0 | pure Ruby | **K-cruby** | Default gem on CRuby 3.3+. Stays via the interpreter. |
| `bigdecimal` | 4.1.2 | C-ext (perf) + pure-Ruby fallback | **K-cruby** | Default gem. Used transitively by `dry-types`. CRuby ships its own. |
| `concurrent-ruby` | 1.3.6 | pure Ruby (optional JRuby C accel.) | **K-pure** | Used by `dry-core`. |
| `dry-core` | 1.2.0 | pure Ruby | **K-pure** | dry-rb foundation. |
| `dry-inflector` | 1.3.1 | pure Ruby | **K-pure** | |
| `dry-logic` | 1.6.0 | pure Ruby | **K-pure** | |
| `dry-struct` | 1.8.1 | pure Ruby | **K-pure** | Heavy `method_missing` user — the principal CRuby-vs-alt-Ruby risk that magnus eliminates by being CRuby. |
| `dry-types` | 1.9.1 | pure Ruby | **K-pure** | |
| `ice_nine` | 0.11.2 | pure Ruby | **K-pure** | Deep-freeze helper used by `dry-struct`. |
| `logger` | 1.7.0 | pure Ruby | **K-cruby** | Default gem. |
| `mustermann` | 3.1.1 | pure Ruby | **R-replace** | sinatra dep — falls out with sinatra. |
| `nio4r` | 2.7.5 | **C-ext (libev)** | **R-replace** | puma dep — falls out with puma. **The single hard native dep in the current bundle.** |
| `rack` | 3.2.6 | pure Ruby | **R-replace** | sinatra dep — falls out with sinatra. |
| `rack-protection` | 4.2.1 | pure Ruby | **R-replace** | sinatra dep. |
| `rack-session` | 2.1.2 | pure Ruby | **R-replace** | sinatra dep. |
| `tilt` | 2.7.0 | pure Ruby | **R-replace** | sinatra dep. |
| `zeitwerk` | 2.7.5 | pure Ruby | **K-pure** | Autoloader used by every dry-rb gem. |

## C-extension surface — final tally

The compiler bundle today has exactly **two** non-CRuby C extensions
that aren't already shipping with the interpreter itself:

1. **nio4r** — libev-style native pollset, dragged in by puma.
2. **puma** — its own native HTTP-parser C accelerator + uses nio4r.

Plus one optional C accelerator that has a pure-Ruby fallback:

3. **json (json/ext)** — replaceable without removing the gem; CRuby
   ships `json` as a default gem regardless.

**Both 1 and 2 disappear with sinatra/puma → axum.** The third becomes
a non-issue once Pangea Ruby stops calling JSON.parse. **Net: M8
removes every non-default-gem native extension from the bundle.**

## What the post-M8 bundle looks like

Embedded into pangea-operator's binary via magnus + `ruby_3_4` Nix
derivation:

```
# Equivalent of the post-M8 Gemfile (no sinatra, puma, json, rack-*)
gem 'terraform-synthesizer',  '~> 0.0.28'
gem 'pangea-core'
gem 'pangea-akeyless'
gem 'pangea-aws'
gem 'pangea-azure'
gem 'pangea-cloudflare'
gem 'pangea-datadog'
gem 'pangea-gcp'
gem 'pangea-hcloud'
gem 'pangea-kubernetes'
gem 'pangea-porkbun'
gem 'pangea-splunk'
gem 'pangea-spot'

# Pulled in transitively by every pangea-* gem:
# abstract-synthesizer, dry-core, dry-inflector, dry-logic, dry-struct,
# dry-types, ice_nine, zeitwerk, concurrent-ruby
```

13 explicit gems + ~9 transitive pure-Ruby deps. Architecture gems
(currently `pangea-architectures`, future composer gems) are NOT in
this list — they ship as `ArchitectureGem` CRs and clone in at runtime.

## What this means for M8.2

The next milestone (axum absorbs the 7 sinatra routes) is bounded by
this audit. Concretely:

- **Routes to port** (`pangea-compiler/app.rb`):
  - `GET /healthz` (line 237) → trivial axum handler.
  - `GET /v1/architectures?gem=…` (line 265) → axum + magnus call into
    `Pangea::Architectures.constants`.
  - `POST /v1/architectures/smoke-test` (line 311) → axum reads YAML
    fixture via serde_yaml, hands Ruby a Hash, calls `klass.build(synth, args)`.
  - `POST /compile` (line 391) → axum + magnus; handles `template_path`
    + `rubylib_paths` + ENV-overrides validation in Rust before calling
    Ruby. **The structurally interesting one** — captured-block pattern,
    `TOPLEVEL_BINDING` shenanigans, `instance_eval` — all get re-implemented
    via `magnus::eval` + `rb_funcall`.
  - `POST /compile-packer` (line 564) → thin wrapper over /compile-any.
  - `POST /compile-any` (line 592) → axum + magnus + the dynamic
    `TypedArraySynthesizer` Ruby class.
  - `GET /formats` (line 638) → returns the registered synthesizer
    formats list — Rust-side once the dynamic-format CRD store moves.

- **Helpers to port** (Ruby-only logic that disappears with the routes):
  - `extend_synthesizer(synth)` — currently iterates `Pangea::Resources::*`
    modules. Stays Ruby; called from inside the embed.
  - `_stringify_keys(obj)` — recursive Hash key normalize. Move to Rust
    (`serde_json::Value` walk) so the magnus call sees only-string-keys.
  - `create_binding(variables)` — only used by the legacy `source` mode
    of /compile. Stays Ruby; magnus has equivalent binding APIs.

- **Bundle deletions to commit on M8.2 completion:**
  - `gemspec`: drop `sinatra`, `puma`, `json` deps. Drop `bin/pangea-compiler`
    (sidecar entrypoint).
  - `Gemfile`: drop `gem 'sinatra'`, `gem 'puma'`, `gem 'json'`.
  - Delete `app.rb` HTTP routing, `config.ru`, `Dockerfile.minimal`. Keep the
    Ruby-side helper module (`PangeaCompiler::Synthesizer` or similar) that
    the embed loads and calls into.

## Risk register (post-audit)

| Risk | Likelihood | Mitigation |
|---|---|---|
| `dry-struct` metaprogramming relies on a CRuby-private API | low — it's pure Ruby per `dry-rb/dry-struct` source | embedded CRuby IS CRuby; no compatibility surface |
| `terraform-synthesizer` `bury` pattern interacts oddly with magnus block-passing | low — block invocation in magnus is a published-API operation | M8.4 spike confirms with a 30-line fixture before rolling forward |
| `zeitwerk` autoload misbehaves under per-request `$LOAD_PATH` mutation | low — exact same shape as today's CRuby sidecar | Rust controls $LOAD_PATH bracketing more cleanly than the current `unshift` + `shift` count-based approach |
| Per-request Ruby state leaks across magnus calls (constants, $globals) | medium — single interpreter, multiple `/compile` requests | Use `rb_protect` + new `Binding` per request; ensure-block clears injected globals (mirrors today's app.rb pattern) |
| Multi-replica scale-out blocked by GVL | high — but not a regression | Already single-replica today; M8 doesn't change the constraint |

## Cross-references

- [`PANGEA-WORKSPACE-RECONCILIATION.md` § M8](./PANGEA-WORKSPACE-RECONCILIATION.md) — the umbrella plan this audit grounds.
- [`pangea-operator/pangea-compiler/Gemfile.lock`](https://github.com/pleme-io/pangea-operator/blob/main/pangea-compiler/Gemfile.lock) — the source-of-truth this audit was derived from.
- [magnus](https://github.com/matsadler/magnus) — the chosen CRuby-embed crate.
- [rb-sys](https://github.com/oxidize-rb/rb-sys) — the lower-level FFI under magnus, fallback if magnus's value model proves too restrictive.

## Maintained by

This file. Update on every change to the compiler's Gemfile / gemspec
during M8.1–M8.5. Once M8.5 lands, this audit is the historical record
of what the bundle contained pre-magnus.
