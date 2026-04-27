# theory

The canonical statement of the pleme-io unified theory of computing.

This repo exists so that every other repo in pleme-io can stop restating the
theory. If you want to know what pleme-io believes about how software is
built, distributed, deployed, and maintained as a living system, you read it
here — once. Every repo-level `CLAUDE.md` is trimmed to its specific mission
and points at this repo for the shared frame.

## What's in this repo

The theory expanded across two generations. **Read in this order:**

### Foundation (read first)

| File | Purpose |
|---|---|
| [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md) | **★ The methodology.** Definition of Constructive Substrate Engineering (CSE), aka Knowable Construction — the named pleme-io approach to building software systems. Read this *before* THEORY.md if you want the operational frame; read THEORY.md *with* this if you want the underlying frame. The renderer-reliability section is non-optional. |
| [`THEORY.md`](./THEORY.md) | The unified theory in one document. Nine parts: frame, language, structure, motion, verification, generation, operation, connecting threads, quick reference. The 12 pillars live here. |
| [`VOCABULARY.md`](./VOCABULARY.md) | Canonical term glossary. Every named concept defined once. |

### Generation 1 (the WASM/WASI runtime + tatara-lisp packaging)

The fourteen documents added 2026-04-26 cover the runtime, packaging,
patterns, and deployment story. **Read in this order** — each builds
on the prior:

| # | File | One-line summary |
|---|---|---|
| 1 | [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) | The 4-layer compute hierarchy + decision tree for "where does this go?" |
| 2 | [`SCRIPTING.md`](./SCRIPTING.md) | Tatara-lisp as canonical scripting standard |
| 3 | [`BREATHABILITY.md`](./BREATHABILITY.md) | Fleet-wide use-causes-spin-up + helm-first invariant |
| 4 | [`TATARA-PACKAGING.md`](./TATARA-PACKAGING.md) | Git+nix native, content-addressed, no separate package manager |
| 5 | [`WASM-STACK.md`](./WASM-STACK.md) | The runtime — 5 program shapes, capability tokens, breathability per shape |
| 6 | [`WASM-PATTERNS.md`](./WASM-PATTERNS.md) | 49-pattern cookbook |
| 7 | [`WASM-PACKAGING.md`](./WASM-PACKAGING.md) | URL grammar (`github:owner/repo/path?ref=tag`) + cache layout |
| 8 | [`WASM-RUNTIME-COMPLETE.md`](./WASM-RUNTIME-COMPLETE.md) | Closes the 3 runtime loops: elasticity, container images, recursive bootstrap |
| 9 | [`LISP-YAML-CONTROLLERS.md`](./LISP-YAML-CONTROLLERS.md) | Lisp logic + YAML policy + tiny escape hatches; 4 authoring tiers |
| 10 | [`FLEET-DECLARATION.md`](./FLEET-DECLARATION.md) | Cluster's whole program inventory in one Helm release; perfect-state contract |
| 11 | [`HELLO-WORLD-LIVE.md`](./HELLO-WORLD-LIVE.md) | Canonical "we got there" doc — the proof + standardization mandates |
| 12 | [`TATARA-CODEGEN.md`](./TATARA-CODEGEN.md) | OpenAPI → tatara-lisp domain via homoiconic Sexp transformation |
| 13 | [`TATARA-CODEGEN-MATRIX.md`](./TATARA-CODEGEN-MATRIX.md) | Same pattern across gRPC, GraphQL, Terraform, SQL DDL, K8s CRDs, MCP, CLI |
| 14 | [`LIVE-DEPLOYMENT.md`](./LIVE-DEPLOYMENT.md) | Hot-replacement triggered by git hashes; 4 state-preservation levels |
| 15 | [`RUST-LISP-EMBEDDING.md`](./RUST-LISP-EMBEDDING.md) | Bidirectional hosting — Rust hosts Lisp + Lisp hosts Rust (3 paths) |
| 16 | [`EXTENSIBILITY.md`](./EXTENSIBILITY.md) | The closing principle — boundlessly extensible by mechanical extension of 5 patterns |
| 17 | [`BASE-PRIMITIVES.md`](./BASE-PRIMITIVES.md) | The 31 base primitives across 5 axes; every higher-level pattern is a tuple |

### Repo conventions

| File | Purpose |
|---|---|
| [`CLAUDE.md`](./CLAUDE.md) | Repo-level instructions for AI agents editing this material |

## What's NOT in this repo

- Implementation. Code lives in the repos that implement each pillar. This
  document only cites those repos; it does not replicate their READMEs.
- Operational runbooks. Those live in the relevant repo (`pangea-jit-builders`
  skill for JIT capacity, `attestation-deployment` skill for integrity gating,
  etc.).
- Product decisions about specific services. Those live in the service's repo.
- Historical notes or pre-decision memos. This document is the settled frame,
  not the decision log.

## How this repo relates to `BLACKMATTER.md` and `pleme-io/CLAUDE.md`

- `~/code/github/pleme-io/BLACKMATTER.md` — the twelve-pillar manifesto.
  `THEORY.md` absorbs and extends it: the twelve pillars appear in Part I,
  with every pillar's downstream implications traced in the later parts.
- `~/code/github/pleme-io/CLAUDE.md` — the org-level AI-agent instructions.
  After this repo lands, that file is trimmed to: prime directive, NO SHELL
  law, repo map, Rust+Lisp pattern, JIT rule. All theory paragraphs move
  here and the CLAUDE.md carries one-line pointers instead.

If there is ever a contradiction between `THEORY.md` and another document in
the fleet, `THEORY.md` wins and the other document is a bug to fix.

## How to read this

**First time:** read `THEORY.md` front-to-back. It's one document; there is
no substitute for reading it linearly. Keep `VOCABULARY.md` open in another
tab for term lookups.

**For the runtime + packaging story** (Generation 1): walk the 17 docs above
in numbered order. Each is ~200-500 lines; the entire generation reads in
~3 hours and lands a complete operational frame for WASM/WASI on Kubernetes
with tatara-lisp authoring. The progression compounds — early docs name
primitives, middle docs combine them, late docs prove the closure.

**Returning:** jump to the part you need. Each document stands on its own
well enough that you don't have to re-read the whole thing.

**As an AI agent:** if a user asks you a question whose answer depends on
the pleme-io theoretical frame, load the relevant section(s) of
`THEORY.md` and cite them. Don't paraphrase from memory. Don't re-derive
from other repos' CLAUDE.mds — those are downstream of this document.

**For implementation work**: cross-reference [`BASE-PRIMITIVES.md`](./BASE-PRIMITIVES.md)
to identify which of the 31 primitives the work touches, then drill into
the matching detailed doc:
- compute primitives → `WASM-STACK.md`
- composition primitives → `WASM-PATTERNS.md` + `LISP-YAML-CONTROLLERS.md`
- lifecycle primitives → `LIVE-DEPLOYMENT.md`
- infrastructure primitives → `BREATHABILITY.md`
- delivery primitives → `TATARA-PACKAGING.md` + `WASM-PACKAGING.md`

## How this document evolves

- Changes to the theory go through this repo, not through scattered
  CLAUDE.mds. A commit here can ripple into twelve repo updates; those
  updates are the _consequence_, not the _source_.
- Term additions: put the definition in `VOCABULARY.md` first, then use
  the term in `THEORY.md`. Never the other way around.
- Pillar changes: twelve pillars are the organizing spine. Adding, removing,
  or re-scoping a pillar is a major version event for the whole fleet.
- Naming: follow the [Japanese × Brazilian-Portuguese convention](./THEORY.md#naming).

## License

MIT.
