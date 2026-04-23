# theory

The canonical statement of the pleme-io unified theory of computing.

This repo exists so that every other repo in pleme-io can stop restating the
theory. If you want to know what pleme-io believes about how software is
built, distributed, deployed, and maintained as a living system, you read it
here — once. Every repo-level `CLAUDE.md` is trimmed to its specific mission
and points at this repo for the shared frame.

## What's in this repo

| File | Purpose |
|---|---|
| [`THEORY.md`](./THEORY.md) | The unified theory in one document. Nine parts: frame, language, structure, motion, verification, generation, operation, connecting threads, quick reference. |
| [`VOCABULARY.md`](./VOCABULARY.md) | Canonical term glossary. Every named concept, verb, primitive — defined once. Splits terms that have historically been used with multiple meanings (convergence-as-lattice vs convergence-as-process, proof-as-verification vs proof-as-attestation, etc.). |
| [`CLAUDE.md`](./CLAUDE.md) | Repo-level instructions for AI agents editing this material. |

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

**Returning:** jump to the part you need. Each of the nine parts stands on
its own well enough that you don't have to re-read the whole thing.

**As an AI agent:** if a user asks you a question whose answer depends on
the pleme-io theoretical frame, load the relevant section(s) of
`THEORY.md` and cite them. Don't paraphrase from memory. Don't re-derive
from other repos' CLAUDE.mds — those are downstream of this document.

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
