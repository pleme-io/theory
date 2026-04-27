# theory — AI agent instructions

> **★★★ CSE / Knowable Construction.** This repo operates under **Constructive Substrate Engineering** — canonical specification at [`pleme-io/theory/CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](https://github.com/pleme-io/theory/blob/main/CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md). The Compounding Directive (operational rules: solve once, load-bearing fixes only, idiom-first, models stay current, direction beats velocity) is in the org-level pleme-io/CLAUDE.md ★★★ section. Read both before non-trivial changes.


This repo is the canonical statement of the pleme-io unified theory of
computing. It is pure documentation. No code, no flake, no infrastructure.

## Editing rules

- **`THEORY.md` is the spine.** Every other document in pleme-io that talks
  about theory should cite a section here, not restate it. If you find
  yourself writing more than a paragraph of theory in another repo, that
  paragraph belongs here instead.
- **`VOCABULARY.md` is the glossary.** Every named concept gets exactly one
  canonical definition. If a term already exists, edit its entry; do not
  add a second row. If a term has drifted to cover two distinct ideas, split
  it into two rows with distinct names and fix the callers.
- **Terms are defined before they are used.** If `THEORY.md` uses a term
  that doesn't appear in `VOCABULARY.md`, that's a bug — add the row before
  merging.
- **Do not add "examples" or "tutorials" files.** Every useful how-to lives
  in a skill or in the implementing repo's `CLAUDE.md`. This repo stays
  compact on purpose.
- **Every pillar lives at Part I, ¶n of `THEORY.md`.** Parts II–VII trace
  consequences. Part VIII names the connecting threads. Part IX is the
  cheat sheet. Do not reorganize lightly.

## Style

- Prose is declarative and present-tense. "Rust owns types" — not "Rust
  should own types" or "We want Rust to own types."
- No hedging ("might," "could," "perhaps"). If a claim is uncertain, it
  doesn't belong here yet.
- No filler ("It is important to note that...", "As you may have
  noticed..."). Cut it.
- Code blocks illustrate; they are not load-bearing. The implementing repo
  carries the real example.
- Citations to implementing repos use the form `repo/path:line` or
  `repo/CLAUDE.md` without URLs. Readers are assumed to have the workspace
  checked out.

## When you're asked to "update the theory"

1. Identify the specific claim that's changing.
2. Find every section of `THEORY.md` that depends on it. Grep for the term.
3. Update those sections coherently, not just the first one you find.
4. Update `VOCABULARY.md` if the change touches a defined term.
5. If the change invalidates a citation in another repo's `CLAUDE.md`, flag
   it — but do not edit that repo in the same commit.

## When you're asked "what does pleme-io believe about X?"

Read the relevant section of `THEORY.md` and quote from it. Don't paraphrase
from memory. Don't synthesize from other repos' docs — those are downstream.
If `THEORY.md` is silent on X, say so explicitly: "The theory is silent on
this; it should be added." Don't fabricate.

## Repo conventions

- No flake.nix. No Cargo.toml. No Ruby gemspec. This is documentation only.
- Commit messages describe what piece of the theory changed and why, not
  what files were touched.
- License: MIT. Copyright pleme-io contributors.
