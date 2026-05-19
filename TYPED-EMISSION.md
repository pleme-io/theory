# Typed Emission — `format!()` is a code smell

> **Status:** canonical (2026-05-18). Prime-directive-tier. Complements
> [`THEORY.md`](./THEORY.md) §II (Language) + §VI (Generation) and the
> existing `format!()`-of-syntax bans in
> [`NIX-AST.md`](./NIX-AST.md) and
> [`CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md`](./CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md).
>
> This is the language-level generalization of those per-domain bans:
> the `format!()` macro is forbidden across every pleme-io Rust crate.
> Every string the platform emits comes from a **typed surface** — either
> the `Display`/`Debug` impl of a value, the `Serialize` impl through a
> typed AST, or a typed logging/error macro.

---

## The rule

In pleme-io Rust code, the `std::format!()` macro is **forbidden**. So is
`format_args!()` outside the typed-macro contexts that consume it
internally.

The two — and only two — places strings get composed at runtime are:

1. **Inside a `Display` / `Debug` / `Serialize` impl block.** Use
   `write!(f, ...)` / `writeln!(f, ...)`. The impl block *is* the typed
   render surface for the type. The format string is part of the
   serialization contract, not free-form glue.
2. **Through a typed logging or error macro.** `tracing::info!()` /
   `tracing::error!()` / `tracing::debug!()` / `anyhow::anyhow!()` /
   `anyhow::bail!()`. These accept format-args inputs but the macro
   itself is the typed surface — it knows whether it produces a log
   record, a typed `anyhow::Error`, an `eyre::Report`, etc.

There is no third case. Every other apparent "build a string" need is
actually one of:

- **A typed AST emission concern.** Building Nix → `nix-synthesizer`'s
  `NixValue` + pretty-printer. Building Helm template fragments →
  `helmworks` named templates. Building YAML → `serde_yaml`. Building
  Go → `goast`. Building HCL → `iac-forge::Backend`. Building K8s
  manifests → typed `k8s-openapi` Resources. Building GitHub-Actions YAML
  → arch-synthesizer's `Action` domain. *Every* target syntax has a
  typed AST + deterministic serializer.
- **A typed value-construction concern.** Building a `PathBuf` →
  `Path::join`. Building a URL → `url::Url::parse_with_params`.
  Building a shell command argv → `tokio::process::Command::arg`.
  Building an HTTP request body → typed struct + `Serialize`. Building a
  SQL query → `sea-orm`'s typed query builder.
- **A typed error or log surface.** See bullet 2 above.

If you reach for `format!()` to compose a string, the substrate is
missing a typed surface for whatever you're emitting. The compounding
move is to **add that typed surface** — extend the AST, define a new
`Display` impl, or introduce a typed builder — then consume it.

---

## Why this is prime-directive-tier

This rule is the language-level corollary of [Pillar 12 — *Generation
over composition*](./BLACKMATTER.md). A free `format!()` call is the
atomic unit of composition-without-generation: a human or agent
concatenating characters by hand for some downstream consumer, instead
of routing through a typed primitive that knows what the consumer
expects.

Banning it forces the question *"what's the typed surface for this?"*
on every emission site. The answer is either:

- An **existing typed surface** the operator hadn't found yet (the
  substrate carries the load — find it, use it), or
- A **gap in the substrate** (the substrate needs widening — propose
  the type, extend the AST, define the `Display` impl, then consume it).

Either path **grows the substrate.** The free-`format!()` path leaves
the substrate untouched and silently regresses on the compounding
thesis.

This is the same shape as:

- The **`no shell beyond 3 lines`** rule (forces tatara-script) — bans
  ad-hoc bash composition, demands a typed orchestration primitive.
- The **`no Dockerfiles`** rule (Pillar 8) — bans ad-hoc image building,
  demands the Nix substrate.
- The **`samba for every rate-limited API`** rule — bans ad-hoc HTTP
  retry/quota logic, demands the typed broker primitive.
- The **`shigoto for every work graph`** rule — bans ad-hoc work
  orchestration, demands the typed scheduler.
- The **`magma for IaC execution`** rule (drafted) — bans `tofu apply`
  shell-outs, demands the typed Rust executor.

`format!()` is the language-level peer of these. Each rule replaces a
local-glue path with a typed substrate path. The platform compounds
when **every** local-glue path has been replaced. This is the last
of the major ones in Rust.

---

## What's allowed, by example

```rust
// ✓ ALLOWED — Display impl is the typed render surface for the type.
impl fmt::Display for HostnameTriple {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}.{}.{}", self.app, self.cluster, self.location)
    }
}

// ✓ ALLOWED — typed logging macro.
tracing::info!(repo = %repo.name, "reconciling repo");

// ✓ ALLOWED — typed error macro through anyhow's contract.
anyhow::bail!("unexpected workspace state: {state:?}");

// ✓ ALLOWED — typed AST emission via serde.
let yaml = serde_yaml::to_string(&deployment_spec)?;

// ✓ ALLOWED — typed URL builder.
let url = base.join(&format!("v1/repos/{repo}"))?;
//                  ^^^^^^^^ STILL BANNED — this format!() is the
//                  smell. The right form is a typed UrlBuilder or
//                  a Path-segment join, not raw format!().

// ✗ BANNED — composing structure by hand.
let s = format!("{} {}", a, b);

// ✗ BANNED — building a path with format!().
let p = format!("/tmp/{}/cache", workspace);
//      Replace with: PathBuf::from("/tmp").join(workspace).join("cache")

// ✗ BANNED — building Nix syntax.
let nix = format!("{{ inputs.{name}.url = \"{url}\"; }}");
//        Replace with: nix-synthesizer's NixValue + render.
```

---

## Enforcement

Two layers, both shipped through the substrate so any pleme-io Rust
crate adopts the rule by depending on the shared lint config.

### Layer 1 — clippy `disallowed_macros`

Each Rust workspace's `clippy.toml` (or the workspace's `Cargo.toml`
under `[workspace.lints]`) carries the disallowed-macros entry. The
substrate's `with-format-ban` Nix function injects this snippet at
build time so it lands in every workspace by construction:

```toml
# clippy.toml — rendered by substrate's with-format-ban.
disallowed-macros = [
  { path = "std::format",
    reason = "use write!() inside Display/Debug impls, tracing::* for logs, anyhow::anyhow!()/bail!() for errors, or a typed AST renderer. See pleme-io/theory/TYPED-EMISSION.md." },
]
```

CI runs `cargo clippy --all-targets -- -D warnings`. Any `format!()`
call fails the build with the rationale string.

The clippy lint is intentionally **coarse** — it flags every
`format!()` regardless of context. The handful of legitimate cases
(`Display` impls that genuinely need `format!()` rather than
`write!()`) get `#[allow(clippy::disallowed_macros)]` at the
function level, with a justification comment.

### Layer 2 — typed audit pass

A small Rust tool (`pleme-io/format-ban`) walks the workspace via
`syn`, identifies every remaining allow-attribute, and produces a
fleet-wide report of:

- count of `format!()` calls per repo
- count of `#[allow(clippy::disallowed_macros)]` justifications
- diff vs baseline (regression detection: PR adds a new allow → CI
  surfaces it for review)

The tool runs as a substrate-provided nix-run app + a typed
pleme-action (`pleme-io/format-ban-check@v1`).

### Per-repo opt-out

A repo that genuinely can't adopt the rule (vendor SDK port, generated
code, legacy migration in progress) declares
`skip-format-ban: <reason>` at the top of its `CLAUDE.md`. The Nix
substrate function honors the marker and omits the clippy injection
for that repo. The format-ban-check tool reports skipped repos
separately so the operator can track migration progress without
blocking on every repo at once.

---

## Migration plan

The rule is enforced **on net-new code immediately** (any new pleme-io
Rust workspace adopts the lint by default). Existing repos migrate on
the substrate-driven rollout described below.

### Phase 0 — substrate scaffold (immediate)

1. Land this theory doc in `pleme-io/theory/`.
2. Land the clippy snippet + Nix function in `pleme-io/substrate/`.
3. Land the `pleme-io/format-ban` tool (initially a ripgrep-based
   first-cut; AST-aware follow-up).
4. Produce the fleet-wide migration map: count of `format!()` per repo,
   sorted by hotspot.

### Phase 1 — adoption rollout (mechanical)

For each repo in hotspot-descending order:

1. Identify whether the `format!()` calls are (a) syntax-emission (use
   the typed AST), (b) value-construction (use the typed builder),
   (c) log/error (use the typed macro), or (d) a genuine `Display`
   need (move into a `Display` impl).
2. Migrate the call sites — usually the typed surface already exists;
   when it doesn't, **extend the substrate** first, then migrate.
3. Enable the clippy lint on the repo. CI now blocks regressions.
4. Update `format-ban` baseline.

### Phase 2 — substrate completeness audit

For every `#[allow(clippy::disallowed_macros)]` justification that
survives Phase 1, ask: *what typed surface is missing?* Each survivor
is a gap report against arch-synthesizer / typescape — the substrate
needs that surface added before the repo can become clean.

---

## What this connects to

- [`THEORY.md`](./THEORY.md) §II (Language), §VI (Generation), §IV
  (Motion — the eight-phase loop applies to this rule's rollout)
- [`NIX-AST.md`](./NIX-AST.md) — the original per-domain ban that this
  rule generalizes
- [`CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md`](./CONSTRUCTIVE-CROSSPLANE-PROVIDERS.md)
  — the second per-domain ban (Go + YAML syntax)
- [`BLACKMATTER.md`](./BLACKMATTER.md) Pillar 12 — generation over
  composition; this rule is the Rust-language operationalization
- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
  — the operator–agent–substrate frame; banning `format!()` is one of
  the moves that keeps "what the substrate knows" growing per
  Operating Principle #0

---

## One-line summary

`format!()` is banned. Use `write!()` inside a `Display` impl, a
typed logging/error macro, or a typed AST renderer. Free-form string
composition is a substrate gap; close it.
