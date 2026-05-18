# Vitrine — pleme-io's pre-merge evidence delivery pattern

> **Status:** draft (2026-05-18). The pattern is canonical; the substrate is stub.
>
> This document specifies the pattern by which pleme-io delivers
> infrastructure and application changes to a target environment with
> **proof-of-life evidence already captured** by the time a PR enters
> review. Reviewers cross-check running state against code rather than
> evaluating a proposal in the abstract.
>
> Companion: [`../docs/vitrine.md`](../docs/vitrine.md) (operator HOW).
> Skill: [`../blackmatter-claude/skills/vitrine/SKILL.md`](../blackmatter-claude/skills/vitrine/SKILL.md).

---

## The frame

A *vitrine* (Portuguese: "showcase / display window") is what a PR becomes
under this pattern: a display window onto a change that is already running
in a target environment. The code change is **applied to the target before
the PR opens for review**; the reviewer sees the diff alongside cited
evidence of what the diff produced — `terragrunt plan` output, `terragrunt
apply` confirmation, `kubectl` / `gcloud` / `aws` cloud-API verification,
end-to-end functional probes — captured at apply time and embedded in the
PR description.

```
Conventional review:
  PR opens → reviewer reasons about whether the code is correct →
  merge → operator applies → discover whether it works.

Vitrine review:
  Operator implements + applies to target env (via GitOps pre-merge override) →
  captures structured evidence → opens PR with diff + evidence inline →
  reviewer cross-checks code against the cited running state → merge.
```

Conventional review answers *"would this work?"*. Vitrine review answers
*"does this work, and is the code that produced it readable?"*. The second
question is strictly easier — the unknown is collapsed before the human
review starts.

---

## Principles vitrine rests on

1. **Reviewers cross-check; they don't simulate.** No human can reliably
   simulate a multi-resource cloud apply in their head. Replace simulation
   with cited observation.
2. **The target environment is the source of truth, not the diff.** A diff
   is what you intend; the running state is what is. Reviewers should see
   both.
3. **Evidence has provenance.** Each cited observation links to its
   command, timestamp, target identifier, and operator. Stale evidence is
   worse than no evidence.
4. **GitOps unlocks pre-merge runtime.** Annotation overrides on the
   cluster registration (e.g. ArgoCD's `targetRevision` from cluster
   annotations instead of labels — "Pattern A") let a feature branch
   deploy to the target while the PR is still in draft, without touching
   `master`.
5. **Drift before vitrine.** Pre-clear environment drift before applying
   the PR's change, so the evidence is bounded to the PR's effect.

---

## The recurring shape

Every change shipped through vitrine has the same six-step shape:

| # | Step | What it produces | Where in PR |
|---|------|------------------|-------------|
| 1 | Pre-flight | confirmation of: credentials valid, target reachable, working tree clean, target environment matches `master` (drift swept) | "Pre-flight" section |
| 2 | Plan | `terragrunt plan` (or `helm template --debug`, etc.) on a target consumer of the change — output trimmed to the relevant resource | "Plan" section, fenced code block |
| 3 | Apply (pre-merge) | `terragrunt apply` against the target environment, via a GitOps pre-merge override (Pattern A) for chart-driven changes, or direct apply for pure-TF changes | "Apply" section |
| 4 | Verify | three layers in this order: TF state outputs → cloud-API queries → functional probes (`curl`, `kubectl get`, health endpoints) | "Verification" section |
| 5 | Rollback noted | the inverse command for each apply step; cleanup commands for any Pattern A override; explicit "what survives if I abort here" | "Rollback" section |
| 6 | Evidence committed | the four prior sections live in the PR description (or `EVIDENCE.md` for very large captures); reviewers open the PR and see them inline | embedded in PR body |

---

## Pattern A — the GitOps pre-merge override

Most pleme-io fleet charts deploy via ArgoCD ApplicationSets that compute
`targetRevision` as:

```yaml
targetRevision: '{{ coalesce (index .metadata.annotations "<chart-name>") (index .metadata.labels "<chart-name>") }}'
```

The `coalesce` means the **annotation wins over the label**. By setting the
annotation to your feature-branch name on the relevant cluster's
`argocd_cluster` registration, ArgoCD pulls chart values from that branch
without anything on `master` having to change. Post-merge cleanup is one
operator action: remove the annotation, re-apply the cluster registration.
The chart re-syncs from `master`, but the values are identical (master
now has what the feature branch had), so no Service recreate, no traffic
blip.

Pattern A is the load-bearing trick that lets the entire vitrine flow
exist for GitOps-driven changes. For pure-TF changes that don't involve a
GitOps controller, the equivalent is just running `terragrunt apply` from
the feature branch directly.

---

## Connection to existing pleme-io primitives

- **Stacked PRs** ([`../docs/stacked-prs.md`](../docs/stacked-prs.md)) —
  Vitrine prefers stacks: substrate change → consumer flip → chart
  enablement. Each PR is its own vitrine. The stack lets each apply be
  small, atomic, reversible.
- **Tameshi attestation** — Tameshi proves *artifacts*; vitrine proves
  *deployments*. They are siblings: both replace human trust with
  mechanical evidence, at different layers of the supply chain.
- **Convergence-flow** — Vitrine is the operator-facing surface of the
  convergence loop's "deploy → verify → reconverge" trailing phases. The
  eight-phase loop describes *what converges*; vitrine describes *how the
  operator captures evidence of convergence* per change.
- **Knowable platform** — "Every capability proven by construction."
  Vitrine extends that to the deployment plane: every change proven by
  execution before review, not by review alone.
- **Caixa SDLC** ([`../docs/caixa.md`](../docs/caixa.md)) — Caixa says how
  to package; vitrine says how to ship. A caixa-defined service whose CI
  pipeline integrates vitrine evidence collection produces PRs that
  reviewers can mechanically validate against running state.

---

## Anti-patterns vitrine forbids

- **PR opens with no evidence.** A diff-only PR forces reviewers into
  simulation. Block via PR template + lint.
- **Evidence as screenshots without commands.** A screenshot of a healthy
  Service isn't evidence; the `kubectl` command that produced it is. The
  reviewer must be able to re-run.
- **Apply via merge.** The "merge to deploy" flow defers verification
  past review; vitrine inverts it.
- **Evidence collected days before merge.** Drift between apply time and
  merge time invalidates evidence; cite timestamps and re-verify if
  needed.
- **Hidden side effects on shared resources.** Pre-merge applies must be
  bounded to new resources or in-place updates with documented rollback.
  No destroying existing state to make a clean plan; sweep drift via a
  separate, named operator action.
- **Pattern A on production.** Production deployments should follow
  staging via reviewed merge, not pre-merge annotation override. Vitrine
  is a staging-side practice; production is the merged-and-promoted
  output of a staging vitrine.

---

## Implementation status

| Layer | State |
|-------|-------|
| Pattern (this document) | canonical |
| Operator reference | [`../docs/vitrine.md`](../docs/vitrine.md) |
| Skill (Claude Code operator-invocable) | [`../blackmatter-claude/skills/vitrine/SKILL.md`](../blackmatter-claude/skills/vitrine/SKILL.md) |
| Substrate (automation) | **v0.1** — `pleme-io/vitrine` Rust workspace, caixa-Binario kind, built via substrate `rust-tool-release-flake.nix`. `isolate` + `release` (Pattern A) implemented; other subcommands stubbed. Module trio auto-emitted by the flake; consumers `imports = [ vitrine.homeManagerModules.default ]; programs.vitrine.enable = true;`. |

### Substrate roadmap

| Subcommand | Status | Replaces operator toil for… |
|---|---|---|
| `vitrine isolate <chart> --branch <feature> --cluster-terragrunt <path>` | ✅ v0.1 | Setting the Pattern A annotation on argocd_cluster TF + running `terragrunt apply` |
| `vitrine release <chart> --cluster-terragrunt <path>` | ✅ v0.1 | Removing the Pattern A annotation + running `terragrunt apply` |
| `vitrine plan <module>` | 🚧 stub | Capturing `terragrunt plan` output trimmed for PR evidence |
| `vitrine apply <module> --planfile <file>` | 🚧 stub | Planfile-pinned apply with structured output + timestamp |
| `vitrine verify --config <path>` | 🚧 stub | Three-layer evidence capture (TF state, cloud API, functional probes) |
| `vitrine embed <pr> --evidence <path>` | 🚧 stub | `gh pr edit --body-file` push of evidence into PR description |
| `vitrine ship --config <path> --pr <num>` | 🚧 stub | Full workflow orchestration as a shigoto-typed Dag |

### Other deferred substrate

- A `pleme-io/actions-vitrine` reusable GitHub Actions workflow that runs
  plan-on-PR and posts a structured comment
- A PR template (`.github/PULL_REQUEST_TEMPLATE.md`) enforcing the four
  evidence sections; lintable by CI
- An `iac-forge` backend that emits per-resource evidence stubs at codegen
  time
- A `shigoto` binding modeling evidence collection as a typed work graph
  (one Job per step; failures captured per phase) — already noted as the
  shape for `vitrine ship` when implemented

The pattern stands on its own without any of the substrate — operators
were shipping vitrine PRs by hand long before the binary existed. The
substrate exists to remove typing toil from the operator, not to enable
the pattern. v0.1 collapses the two highest-toil steps (Pattern A
annotation set/remove + terragrunt apply) into one binary call each.
