# Carve — pleme-io's monolithic-branch → ticket-aligned stacked-PR pattern

> **Status:** v0.1 (2026-05-20). The pattern was codified from the ASM-18003
> DBK-staging asia-southeast1 rollout, which was performed entirely by
> hand; the substrate (`pleme-io/carve`) reifies that procedure into
> attestable software.
>
> Companion: `../docs/carve.md` (operator HOW — TODO).
> Skill: `../blackmatter-claude/skills/carve/SKILL.md`.
> Pre-merge sibling: [VITRINE](./VITRINE.md).

---

## The frame

A *carve* is the act of taking a single development branch — where the
operator did all the work, in whatever order made sense at the time — and
splitting it into a sequence of ticket-aligned stacked pull requests the
team can actually review.

```
Operator's natural development flow:
  one branch, freeform commits, until everything works.

Team's required review surface:
  ticket-sized, scope-clean, dependency-ordered PRs.

Carve is the lossless transform between them.
```

Conventional approaches force the operator to author the stack from day
one (`gt create` etc.); this is unergonomic when the design is still in
motion. Carve inverts: develop in one branch, then carve.

---

## Principles carve rests on

1. **The operator authors freely; the team reviews structured.** The
   development branch is the operator's working surface and need not be
   structured. The review surface is the team's contract and must be
   structured. Carve is the translator — never the author.

2. **No commit is silently moved, dropped, or merged.** Every commit
   reaches the stack with a recorded *assignment* (which ticket it's
   on) and *confidence* (how that assignment was reached). Cross-cutting
   commits surface explicitly; the operator chooses a split, and carve
   applies it deterministically.

3. **The source branch is sacred.** Carve never mutates the source
   branch. Before any branch creation, carve writes a BLAKE3-attested
   annotated tag (`carve-backup/<epic>-<timestamp>`) pointing at the
   original tip. Recovery is one `git checkout` away, indefinitely.

4. **Tree-hash equivalence is the integrity contract.** After carving,
   the top of the stack's tree hash MUST equal the source branch's tree
   hash. If it doesn't, content has drifted; carve refuses to push.

5. **The plan is operator-editable; the execution is mechanical.** The
   `plan.yaml` is the single source of truth, encoding every decision
   the carve makes. Carve *proposes*; the operator *decides*; carve
   *executes attestably*. There is no scenario where execution silently
   does something not represented in the plan.

6. **Empty placeholders are first-class.** If a JIRA sub-ticket has no
   repo-side scope (a Confluence design-docs ticket, a CICD ticket whose
   pipelines live in another repo), the operator marks it `placeholder:
   true` and carve emits a draft PR with a single `--allow-empty`
   commit. The 1:1 ticket↔PR mapping is preserved without manufacturing
   code.

7. **Per-org policy is configured, not hard-coded.** JIRA story-point
   scales differ across teams (1 point = 1 day; 1 point = 0.5 days;
   fibonacci). Auto-transition policy differs (some workflows permit
   automation past "Ready to Work", others don't). Carve reads these
   facts from `.carve.toml` and adapts.

---

## The recurring shape

Every carve follows the same three-phase shape:

### Phase 1 — analyse and propose

```
carve plan --epic ASM-18003
```

Carve walks `origin/<master>..HEAD` (using `--first-parent --no-merges`
to exclude automated merges from master), fetches the JIRA epic's
sub-tickets, and emits a `plan.yaml` skeleton: tickets named, commits
fingerprinted, scope path-globs empty.

### Phase 2 — operator decides

The operator opens `plan.yaml` and fills in each ticket's `paths`
glob set. They may pin a difficult assignment, override a proposed
split, or mark a ticket as `placeholder: true`. Then:

```
carve plan --refresh   # re-score with the new globs
carve verify           # check all invariants
```

`--refresh` runs path-glob intersection on every commit, classifying
each as single-scope (high confidence), cross-cutting (with a proposed
`ByPath` split), or unassigned (operator must extend a scope's globs).

### Phase 3 — execute attestably

```
carve execute
```

Carve takes the plan and:

1. Creates the BLAKE3-attested backup tag pointing at the source tip.
2. Creates each stack node's branch off its base, in stack order.
3. For each commit: cherry-picks (clean assignments) or runs `git show
   <sha> -- <paths> | git apply` (split halves), preserving original
   author + date metadata.
4. Verifies tree-hash equivalence: the top of the stack must equal the
   source branch tip. Mismatch = refuse to push.
5. Pushes branches via `--force-with-lease`.
6. Opens stacked PRs via `gh pr create --base <parent>` in dependency
   order.

After review, fixes that land on a parent PR propagate down via
`carve restack --from <branch>`, which `git rebase --onto`s every
descendant.

---

## Plan-centric design

The `plan.yaml` document is the carve substrate. Every command reads or
writes it; nothing is stored elsewhere. Its top-level shape:

```yaml
meta:        { carve_version, generated_at, jira_epic, operator }
source:      { name, master_branch, tip, merge_base }
tickets:     [ TicketScope ]      # one per JIRA sub-task
commits:     [ CommitFingerprint ]  # immutable input, captured at plan time
assignments: [ CommitAssignment ]   # commit → ticket, with confidence
cross_cutting: [ CrossCuttingCommit ]   # multi-scope commits + split decisions
stack:       { root, nodes: [ StackNode ] }
```

The plan is round-trip-serialisable YAML; operators can `git diff`
between plan revisions; CI can lint plans without running carve at all.

---

## Connection to existing pleme-io primitives

```
single branch  ──→  carve  ──→  stacked PRs  ──→  vitrine  ──→  merge
(operator)         (CARVE)     (team review)    (VITRINE)
```

- **[VITRINE](./VITRINE.md)** — sibling pattern, *post*-carve. Once the
  stack is up, each PR may use vitrine to embed pre-merge evidence (TF
  plans, applied state, probe output) into its description. Carve
  produces the PRs; vitrine fills them with evidence.

- **cordel** — BLAKE3-attestation pattern. Carve borrows cordel's
  attestation discipline for backup tags: every tag's annotated-tag
  message embeds a hash over the plan content, making post-hoc tampering
  detectable.

- **shikumi** — typed-config pattern. `CarveConfig` follows shikumi's
  "configuration is data, not behaviour" stance; per-team JIRA policy
  becomes a TOML document, not a code path.

- **substrate** — `rust-workspace-release-flake.nix` provides the Nix
  build, the home-manager module trio, and the release pipeline.

---

## Anti-patterns carve forbids

| Anti-pattern | Carve's countermeasure |
| --- | --- |
| Silently dropping a commit | Every commit needs an explicit assignment or `Drop` decision; `Drop` requires a non-empty `rationale`. |
| Silently merging cross-cutting scope | Multi-scope commits become `CrossCuttingCommit` entries with an explicit `SplitDecision` the operator must approve. |
| Mutating the source branch | Source branch is read-only; backup tag is mandatory; `git push --force-with-lease` is the only push form used. |
| Out-of-order merge | `carve gate` is the CI hook refusing merge if any stack parent is still open. |
| Manufactured placeholder content | Empty-scope tickets get `--allow-empty` commits and ship as draft PRs. |
| Branch base drift after review feedback | `carve restack` walks descendants and replays them via `git rebase --onto`, then `carve diagram` regenerates the embedded PR-body stack diagram so reviewers see the new state. |

---

## Implementation status

| Primitive | Status |
| --- | --- |
| `carve plan` (with --refresh) | v0.1 |
| `carve verify` | v0.1 |
| `carve execute` | v0.1 |
| `carve jira-sync` (policy-capped) | v0.1 |
| `carve restack` | v0.1 |
| `carve diagram` (idempotent fence) | v0.1 |
| `carve gate` (CI hook) | v0.1 |
| `carve status` (read-only snapshot) | v0.1 |
| `pleme-io/docs/carve.md` (operator how-to) | TODO |
| `pleme-io/blackmatter-claude/skills/carve/SKILL.md` | v0.1 |
| JIRA remote-issue-link via `remote_issue_links` API | deferred (v0.2) |
| Plan visualisation outside PR bodies (web dashboard) | deferred (v0.3) |
