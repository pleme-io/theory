# VIGGY-AUTHORING — the canonical authoring discipline

> The typed authoring discipline for the **Viggy Method**
> ([`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md)).
> A new operator (human or agent) reads this once and can produce any
> promessa / anomalia / extension with zero ambiguity *by construction*.
> Each section is a decision rubric — a generator of correct authoring
> decisions, not a list of answers.
>
> Refined under tight-cycle disambiguation: each section lands one
> round at a time, with surfaced open questions for operator
> refinement. When an ambiguity recurs three times in practice, it
> earns a row.

Companion docs:

- [`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md) — the canonical Viggy spec
- [`THEORY.md`](./THEORY.md) §IV.7, §VIII.9, §I.1 — the frame
- [`VOCABULARY.md`](./VOCABULARY.md) — typed term definitions
- `pleme-io/CLAUDE.md` ★★ The Viggy Method — the directives

---

## Table of contents

| §  | Topic | Status |
|----|-------|--------|
| §1 | **Diagnostic** — is this a promessa? | ✓ landed |
| §2 | **Granularity** — one promessa per *what*? | ✓ landed |
| §3 | **Cite-vs-wrap** — when to use existing primitives | ✓ landed |
| §4 | **Severity calibration** — Cosmetic / Functional / Critical thresholds | ✓ landed |
| §5 | **Action selection** — FluxCommit / ReconcilerApply / MagmaApply / CrachaPatch / Custom | ✓ landed |
| §6 | **Composition** — Meet / Join / Observe dependency semantics | ✓ landed |
| §7 | **Cadence rules** — reconcile_every per kind | ✓ landed |
| §8 | **Lifecycle playbook** — spec evolution, decommissioning, runbook→promessa migration | ✓ landed |
| §9 | **Failure-mode catalog** — wedged controller, observation down, repeated action failure | ✓ landed |
| §10 | **Testing minimum** — trait_laws, golden trajectories, synthetic observation | ✓ landed |
| §11 | **Operator workflows** — Declare / Audit / Override as concrete sequences | ✓ landed |
| §12 | **Federation rule** — single global promessa vs. per-cluster + coordinator | ✓ landed |
| §13 | **Cross-domain composition** — Viggy ∩ caixa / saguao / cofre | ✓ landed |
| §14 | **Anti-pattern catalog** — typed look-alikes that aren't Viggy + spec location rule | ✓ landed |
| §15 | **Extracted abstractions** — the typed surfaces that emerge from the discipline | ✓ landed |

---

## §1 — Diagnostic: is this a promessa?

A five-question typed test. **All five must answer YES** for the
concern to become a promessa. If you stop at any NO, follow the
branch in §1.5 — the concern is still useful work, just lives
elsewhere in the substrate.

### §1.1 The five questions

**Q1 — Convergence shape.** Can the concern be expressed as
`(desired-state, observed-state, action-to-close-gap)`?

- YES → it's at least controller-shaped. Continue.
- NO → it's not Viggy-shaped at all. (Examples: one-shot data
  migration; pure compute; exploratory analysis; build-time
  generation.) Use shinka (migrations), kenshi (ephemeral tests),
  forge-gen (build-time), or a one-off CLI tool.

**Q2 — Business-outcome shape.** Is the desired-state a *business-outcome
commitment* — something a contract / SLA / audit / regulator / FinOps
/ PM / security team would name and care about?

- YES → it's promessa-shaped at the type level. Continue.
- NO → it's a **peer controller**, not a promessa. Branch to §1.5.B.

Concrete test: would *anyone outside engineering* ever ask "is this
held?" If yes, Q2 = YES. If the only stakeholder is the engineer who
wrote it, Q2 = NO.

**Q3 — External observation.** Does observed-state come from a
*measurement source outside the resource's own `.status` field*?

- YES → continue.
- NO → the controller is kube-native, observing its own resource. It
  is a peer controller. Branch to §1.5.B.

Concrete test: is the observation in shinryu / a metrics pipeline / a
CVE feed / a billing export / an external API / cross-system signal?
If yes, Q3 = YES. If the observation is "`kubectl get <kind>
<name>`," Q3 = NO.

**Q4 — Attestation required.** Will an auditor / regulator / customer
/ FinOps / PM / security team ever ask *"prove this was continuously
held across <window>"*?

- YES → THIS IS A PROMESSA. Continue to Q5.
- NO → defer. A regular peer controller + alert layer suffices.
  Re-evaluate annually or when an audit appears. Branch to §1.5.D.

Concrete test: does any of {SOC 2, PCI-DSS, HIPAA, FedRAMP, ISO
27001, NIST 800-53, customer contract, board-reported metric,
regulatory commitment} name this concern? If yes, Q4 = YES.

**Q5 — Target kind.** If Q1-Q4 all YES, classify into one of five
canonical TargetControllers:

| Observed-state shape | Target kind |
|---|---|
| availability / latency / error-rate over a window | **SLA** |
| spend / billing over a period | **CostBudget** |
| regulatory posture against a baseline | **Compliance** |
| customer-facing metric (NPS / CSAT / retention / activation) | **CustomerKpi** |
| CVE age / banned packages / runtime posture | **Security** |
| genuinely none of the above | **Custom** (extend cautiously; prefer adopting one of the 5 by analogy) |

### §1.2 The decision tree

```
START: I have an operational concern.
  │
  ▼
Q1: (desired, observed, action) shape?
  ├── NO  → §1.5.A — Not Viggy-shaped
  └── YES ▼
Q2: business-outcome commitment?
  ├── NO  → §1.5.B — Peer controller (no promessa wrapping)
  └── YES ▼
Q3: observation outside .status (measurement source)?
  ├── NO  → §1.5.C — Kube-native controller (consider future promotion)
  └── YES ▼
Q4: continuous cryptographic attestation required?
  ├── NO  → §1.5.D — Peer controller + alerts (defer promessa; re-evaluate)
  └── YES ▼
PROMESSA. Q5: which kind?
   → SLA / CostBudget / Compliance / CustomerKpi / Security / Custom
```

### §1.3 Worked examples

| Concern | Q1 | Q2 | Q3 | Q4 | Result |
|---|---|---|---|---|---|
| "lilitu API p99 latency < 200ms over 30d" | ✓ | ✓ SLA | ✓ shinryu metrics | ✓ contract | **SLA promessa** |
| "monthly cloud spend < $5,000" | ✓ | ✓ FinOps | ✓ CUR pipeline | ✓ FinOps board | **CostBudget promessa** |
| "all `regulated` namespace pods PCI-DSS-4.0 compliant" | ✓ | ✓ regulatory | ✓ kensa projection | ✓ auditor | **Compliance promessa** |
| "NPS > 50" | ✓ | ✓ board metric | ✓ Mixpanel | ✓ board-reported | **CustomerKpi promessa** |
| "no critical CVE older than 24h in prod images" | ✓ | ✓ security baseline | ✓ CVE feed | ✓ audit | **Security promessa** |
| "all images SLSA L3 attested" | ✓ | ✓ supply-chain | ✓ tameshi chain | ✓ supply-chain audit | **Security promessa** |
| Single Deployment HPA scaling | ✓ | ✗ per-resource | n/a | n/a | **Peer controller** — HPA suffices |
| HelmRelease lifecycle | ✓ | ✗ per-resource | n/a | n/a | **Peer controller** — FluxCD native |
| cert-manager cert renewal | ✓ | ✗ per-resource | ✗ `.status.notAfter` | n/a | **Peer controller** — cert-manager native (unless wrapped by a fleet-wide Security promessa) |
| Renovate auto-PR for deps | ✓ | ✓ security | ✓ CVE feed | ✓ audit | **Security promessa** wrapping Renovate (Renovate becomes one Reconciler under the promessa) |
| Prod database availability | ✓ | ✓ uptime | ✓ probe | maybe | **SLA promessa** if customer SLA names it; else peer controller + StatefulSet |
| One-shot DB schema migration | ✗ one-shot | n/a | n/a | n/a | **Not Viggy** — shinka migration controller |
| Build-time codegen | ✗ build-time | n/a | n/a | n/a | **Not Viggy** — forge-gen pipeline |
| Internal dashboard for ad-hoc query | ✗ exploration | n/a | n/a | n/a | **Not Viggy** — shinryu directly |

### §1.4 The "borderline" tier

When Q4 is *plausibly* YES but not yet — author the promessa anyway.
Three reasons:

1. **Promessas are wrappers, not replacements.** A Security promessa
   over a cert-manager-driven fleet doesn't replace cert-manager; it
   observes cert-manager's outputs and chains them. Zero discontinuity
   to add later — but also zero discontinuity to add *now*.
2. **The audit chain accrues value over time.** A 90-day chain is
   trivially derivable from a chain that has been ticking for 90
   days; not so for one started yesterday when the auditor calls.
3. **The substrate cost is constant.** Once the TargetController
   exists, additional promessas are ~5 lines of YAML each.

Rule of thumb: **if Q1, Q2, Q3 are all YES and Q4 is "maybe within 12
months,"** author the promessa.

### §1.5 The branches — what to do if NOT a promessa

#### §1.5.A — Not Viggy-shaped (Q1 = NO)

The concern is one-shot, build-time, exploratory, or otherwise lacks
a continuous (desired, observed, action) shape. Route to:

| Shape | Substrate primitive |
|---|---|
| DB schema migration | **shinka** Migration CRD |
| Ephemeral test gate | **kenshi** test operator |
| Build-time codegen | **forge-gen** + caixa-publish |
| One-shot recovery | manual; document in cluster `RUNBOOK.md`; if it recurs, build a controller |
| Pure compute / batch job | engenho `Job` / `CronJob` (kube-native); add Viggy wrapping only if cross-resource attestation needed |
| Exploratory query | shinryu MCP directly; no persistent state |

#### §1.5.B — Peer controller, not a promessa (Q2 = NO)

The concern is reconciliation-shaped but not a business-outcome
commitment. The fractal-controller pattern (THEORY.md §VIII.9) says
this is a *peer* controller — same seven-beat shape, different state
space. Decision tree for which kind:

| The state being reconciled is… | Substrate primitive |
|---|---|
| A K8s workload (Deployment, Service, Ingress, StatefulSet, Job, …) | **Existing kube controllers** (FluxCD HelmRelease + kube-controller-manager). Cite, don't wrap. |
| A cloud primitive (VPC, S3, RDS, IAM, …) | **Pangea declaration + magma reconciler**. Cite the typed primitive. |
| A declarative external API (GitHub repo, DNS zone, Vault policy, Slack channel, …) | **`Reconciler` impl via pangea-operator** (one of the 23 catalogued in CONVERGENCE-SUBSTRATE.md). |
| A custom CRD shape with its own semantics | **engenho native controller** with kube-rs informer + `Reconciler` trait. New code is just the reconcile loop. |
| An admission-time constraint (e.g., "all pods must have a CPU limit") | **sekiban policy**. Admission rejects bad declarations; no continuous reconciliation needed. |
| A secret-material concern (rotation, materialization, backend reference) | **cofre** SecretRef + cofre controller. |
| An identity / authz concern | **saguao** crachá AccessPolicy + crachá-controller. |

#### §1.5.C — Kube-native controller, observation from `.status` (Q3 = NO)

The controller is kube-native and observes its own resource. Author
as a regular engenho controller — kube-rs informer + Reconciler. No
promessa wrapping needed at this stage.

**Promotion path:** if later the concern needs cross-resource
attestation (e.g., "prove every Certificate in prod was renewed
before expiry"), promote to a Security promessa observing certificate
expiry across the fleet (Q3 → YES, Q4 → YES via audit). The
underlying kube-native controller keeps running unchanged; the
promessa wraps it.

#### §1.5.D — Defer the promessa (Q4 = NO)

The concern is observable, business-shaped, cross-resource — but no
auditor / regulator / customer / FinOps / PM has asked. Author as a
peer controller with the standard alert layer (per Pillar 11), and:

- **Document in the cluster's `RUNBOOK.md`** under a "candidate
  promessa" section
- **Re-evaluate annually**, on audit, or when an attestation question
  arises
- **Be ready to promote** — promotion is a wrapper, not a rewrite

### §1.6 Resolved via prime directives

Per ★★★ Compounding Directive + ★★ The Viggy Method directives. The
five §1 questions are auto-answered. **No operator round needed.**

| # | Question | Auto-answer | Cited directive |
|---|---|---|---|
| **R-1.1** | Internal cross-team SLOs — promessa-shaped? | **YES.** Internal stakeholders are stakeholders; the canonical test is "named in a commitment," not "outside engineering." | Compounding #1 (solve once); ★★ PROVABLE OUTCOMES |
| **R-1.2** | CI-time build-quality (clippy / fmt / coverage) — promessa-shaped? | **NO.** CI is Pillar 9 (SDLC), not Pillar 7 (runtime). | Pillar 9 vs. Pillar 7 boundary |
| **R-1.3** | Operator-workstation ergonomics — promessa-shaped? | **YES** if engenho-local is in play (ENGENHO-LOCAL.md); else build-time metric. Shape transfers; substrate scales down. | ★★ The Viggy Method §VIII.9 (fractal pattern); Pillar 7 widened |
| **R-1.4** | Attestation-required threshold | **Required** when any of {SOC 2, PCI-DSS, HIPAA, FedRAMP, ISO 27001, NIST 800-53, customer contract, board-reported metric, regulatory commitment, **internal team-to-team SLO**} names the concern. Adversarial proofs ("did not exceed") → promessa; the chain IS the proof of holding. | ★★ PROVABLE OUTCOMES; Pillar 10 |
| **R-1.5** | Existing-tool wrapping (Renovate / cert-manager / Kyverno) | **Cite when audit doesn't name them; wrap when it does.** Default = cite. Promotion to wrap = typed Promotion (§8.3), not rewrite. | Compounding #1 — solve once, in one place |
| **R-1.6** | Back-filling OutcomeChain receipts | **Never.** Receipts are signed assertions of the controller's observation at tick T. Historical replay = one-shot kensa check, not a promessa receipt. | Pillar 10; tameshi BLAKE3 integrity |
| **R-1.7** | Multi-kind composite `ProductPromessa` | **No.** One promessa per kind; compose via `PromessaDependency` Meet + `PromessaLattice`. Composite types couple distinct TargetController logic + break monotonic compounding. | ★★ macros-everywhere (one shape per concern); Compounding #1 |

**The meta-rule.** If a §1 ambiguity cannot be auto-answered via the
directives + this doc, that itself is the signal of a missing
canonical directive → file a substrate-improvement ticket. Do not
paper over with local judgment.

---

## §2 — Granularity: one promessa per *what*?

**The rule:** one promessa per *typed business-outcome commitment*. A
commitment is the unit named in exactly one of: a contract clause, an
SLA line, a compliance baseline, a KPI definition, a security
baseline, or an internal cross-team SLO. **Not** one per resource;
**not** one per cluster; **not** one per service. Per commitment.

### §2.1 The granularity test

| Rule | Apply |
|---|---|
| **1. Match scope to commitment subject** | Find the single sentence naming the commitment; `:scope` exactly its subject. |
| **2. Match `:window` to commitment time scale** | Don't invent; use the commitment's window verbatim. |
| **3. Match `:target` kind to obligation kind** | One commitment = one kind = one promessa. Multi-kind commitments split. |

Examples:

- "api-v1 holds 99.99%" → `:scope (service api-v1)`
- "lilitu product holds 99.99% across customer-facing endpoints" → `:scope (product lilitu)`
- "regulated namespace is PCI-DSS-4.0" → `:scope (namespace regulated)`

### §2.2 Split-vs-compose decision

| Situation | Decision |
|---|---|
| Same kind, same scope, same window, multiple fields | **ONE** promessa (e.g., availability + latency in one SLA target) |
| Same kind, different scopes (prod + staging) | **TWO** promessas |
| Same kind, different windows (30d + annual) | **TWO** promessas |
| Different kinds (SLA + Cost + Compliance) | **N** promessas + `PromessaDependency` edges (§6) |
| Fleet aggregate view | Per-cluster promessas + `PromessaLattice::meet(…)` view (no separate "fleet" CR) |

### §2.3 Anti-patterns

| Anti-pattern | Why wrong | Directive |
|---|---|---|
| "One promessa per Deployment" | Per-resource, not per-commitment | Compounding #1; §1 Q2 NO branch |
| "One promessa per cluster" | Cluster isn't a commitment owner | §2.1 rule 1 |
| "Mega-promessa with all SLAs" | Spec hash unstable; chain genesis drifts | Pillar 10 |
| "One promessa per `:limits` field" | Hijacks caixa scope | §3 cite-vs-wrap |
| "Composite ProductPromessa" | Couples distinct TargetController logic | R-1.7 |

### §2.4 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-2.1** | Commitment evolves (new field added) — revise in place or new promessa? | **Revise in place** (SpecRevised receipt; chain continues) **iff** scope unchanged. **New promessa** **iff** scope changes. Invariant: spec hash MAY change; commitment subject MUST NOT. | Pillar 10; §8.1 |

---

## §3 — Cite-vs-wrap: when to use existing primitives

**The rule:** existing typed primitives declare *the constraint at
admission/build-time*; promessas declare *the continuous outcome at
runtime*. They compose, never compete. **Cite by default; wrap only
when audit demands continuous attestation.**

### §3.1 The cite-vs-wrap test

| Q | If YES |
|---|---|
| Is the primitive's audit surface sufficient (admit/reject + HeartbeatChain stamp)? | **Cite.** |
| Does audit also need "no bad state ever existed in steady state"? | **Wrap.** Drift / race / late-arriving data can produce bad state even with a perfect admission gate. |
| Does the primitive expose a typed observation surface? | If wrapping, observe its outputs. If no surface exists, **extend the primitive** (load-bearing fix) — never instrument via shinryu against the primitive's underlying API. |

### §3.2 Canonical pairings

| Existing primitive | Cite use | Wrap use |
|---|---|---|
| caixa `:limits` | Per-Servico declaration | SLA promessa attesting limits hold |
| sekiban admission policy | Reject bad creates at admission | Compliance promessa attesting no drift over window |
| kensa pre-deploy check | Compliance gate at deploy | Compliance promessa proving baseline continuously |
| cofre rotation policy | Per-secret schedule | Security promessa attesting rotation occurred |
| saguao crachá | Per-policy access decisions | Security promessa over saguao audit log |
| FluxCD HelmRelease | GitOps deployment | SLA promessa observing deployed workload |
| magma resource | Cloud declaration | CostBudget promessa observing spend |
| 23 Reconcilers | Per-API reconciliation | Promessa attesting the reconciler converged |

### §3.3 The wrap-not-compete invariant

A promessa MAY observe an existing primitive's outputs. A promessa
MUST NOT duplicate the primitive's role. Two structural checks:

1. `ObservationSource` cites the primitive's typed output.
2. `TypedAction` either acts via a different surface OR patches the
   primitive's declaration (`FluxCommit`, `CrachaPatch`, sekiban
   policy revision via Reconciler). **Never bypasses.**

### §3.4 Anti-patterns

| Anti-pattern | Directive violated |
|---|---|
| Promessa re-implements sekiban admission | Compounding #1 |
| Promessa reads raw kube state instead of citing primitive output | ★★ TYPED EMISSION |
| Promessa act() bypasses the primitive | ★★ GITOPS-NATIVE |
| Promessa for everything sekiban covers without audit need | Authoring inflation (R-1.5) |

### §3.5 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-3.1** | Primitive output isn't expressive enough — extend, observe raw, or new surface? | **Extend the primitive** (load-bearing fix). Falling back to raw is a smell. New typed surface = substrate-improvement ticket. | Compounding #2; ★★ macros-everywhere |
| **R-3.2** | A promessa's `CrachaPatch` mutates saguao crachá — competing? | **No, patching is a typed action on the typed surface.** The crachá IS being acted upon, not bypassed. | ★★ GITOPS-NATIVE; ★★ TYPED EMISSION |

---

## §4 — Severity calibration

**The rule:** thresholds owned by **TargetController** (per-kind,
well-justified); per-promessa overrides allowed *only to tighten*,
never to loosen, and require typed justification.

### §4.1 Default thresholds per kind

| Kind | Cosmetic | Functional | Critical |
|---|---|---|---|
| **SLA** (latency) | > 1.2× target | > 1.5× target | > 2× target |
| **SLA** (availability) | within 0.001% of target | within 0.01% | below target |
| **SLA** (error-rate) | 1.5× target | 2× | > 3× |
| **CostBudget** | 10% over pace | 25% over | 50%+ over or over total |
| **Compliance** | any non-critical control failing | any critical control failing | critical + audit deadline imminent |
| **CustomerKpi** | within 5% of minimum | 5–15% below | > 15% below |
| **Security** | high-sev CVE within remediation window | high-sev > 7d | critical CVE OR critical > 24h |

These live in code as `pub const COSMETIC_THRESHOLD: …` in each
TargetController. Typed values; never magic numbers.

### §4.2 Trait law: severity monotonic

For any Drift, Severity is monotonic — strictly larger drift produces
at-least-as-severe Severity. **Property-tested** as `severity_monotonic`.

### §4.3 Override mechanics

```lisp
:severity-override { :cosmetic (...) :functional (...) :critical (...) }
```

| Allowed | Required |
|---|---|
| **Tighter** thresholds | typed PR reviewer attestation |
| **Looser** thresholds | `:loosening-waiver: <typed-reason>` (recorded in chain genesis) |

### §4.4 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-4.1** | Where do thresholds come from? | TargetController owning team + canonical contract/SLA/baseline; documented in controller's `trait_laws_obeyed` rationale. | Pillar 10 |
| **R-4.2** | May a promessa loosen defaults? | **Only with typed `loosening-waiver`.** Acceptable reasons: explicit business decision; legacy migration window; vendor-imposed ceiling. **Time pressure is not acceptable.** | Compounding #0 |
| **R-4.3** | May a promessa add a 4th severity tier? | **No.** Three tiers are canonical fleet-wide; adding requires a substrate-wide directive change. | ★★ macros-everywhere |

---

## §5 — Action selection

**The rule:** **`FluxCommit` first** (★★ GITOPS-NATIVE). Reach for
other TypedAction variants only when the mutation surface lies outside
the cluster manifest tree.

### §5.1 Decision tree

```
What is being mutated?
  ├── A file under k8s/clusters/<cluster>/    → FluxCommit
  ├── A Pangea-declared cloud resource         → MagmaApply
  ├── An external declarative API              → ReconcilerApply (one of 23)
  ├── A secret material                        → CofreRotate
  ├── A saguao AccessPolicy                    → CrachaPatch
  ├── A sekiban admission policy               → ReconcilerApply (sekiban Reconciler)
  └── Genuinely none                           → Custom (justify; consider substrate extension)
```

### §5.2 Action invariants (trait laws)

| Invariant | Test name |
|---|---|
| Idempotent on Noop | `act_idempotent_on_noop` |
| Deterministic over typed inputs | `act_deterministic_over_inputs` |
| Reflective in OutcomeReceipt | `act_audit_complete` |
| No raw exec / shell | `act_no_shell` (★ NO SHELL) |
| No raw HTTP | `act_typed_dispatch_only` |

### §5.3 Anti-patterns

| Anti-pattern | Directive violated |
|---|---|
| AutoCorrect runs shell | ★ NO SHELL |
| AutoCorrect mutates outside manifest tree without typed dispatch | ★★ GITOPS-NATIVE / ★★ TYPED EMISSION |
| AutoCorrect issues raw HTTP | ★★ macros-everywhere (every API surface is a Reconciler first) |
| AutoCorrect bypasses sekiban admission | ★★ The Viggy Method §VII.2 |

### §5.4 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-5.1** | May AutoCorrect chain multiple actions? | **Yes**, via `TypedAction::Compose(Vec<TypedAction>)`. The chain is a shigoto Dag; receipt records each step. | ★★ Shigoto |
| **R-5.2** | When ReconcilerApply but Reconciler doesn't exist? | **Author the Reconciler first**, then the promessa. Never inline. | Pillar 12; R-3.1 |
| **R-5.3** | Custom — when acceptable? | **Only with substrate-improvement ticket.** Three Custom uses = extract a new TypedAction variant. | Pillar 12; ★★ macros-everywhere (three-times rule) |

---

## §6 — Composition: Meet / Join / Observe

### §6.1 Cross-kind composition (PromessaDependency)

| Op | Semantics | When to use |
|---|---|---|
| **Meet** | A's act() blocks until B is Cosmetic+; degraded B → A emits `DependencyBlocked` | A's remediation could *worsen* B |
| **Join** | A acts if either A or B resolves | Rare; alternative resolutions exist |
| **Observe** | A acts normally; B's state recorded as context in A's receipts | Correlation matters; coupling undesirable |

### §6.2 Same-kind composition (PromessaLattice)

| Op | Result |
|---|---|
| `meet(A, B)` | Strictest of each field (min availability, max p99, …) |
| `join(A, B)` | Weakest of each field |
| `leq(A, B)` | A strictly weaker than B (observable from spec hash) |

Lattice ops are defined only for *same-kind* promessas. Cross-kind
composition is dependency edges, not lattice.

### §6.3 Anti-patterns

| Anti-pattern | Directive |
|---|---|
| Cycle in dependency graph | Pillar 10 (typed error at admission) |
| Cross-kind lattice meet | ★★ macros-everywhere |
| Mega-Meet of >5 promessas | Authoring smell; introduce derived view |
| Bypassing PromessaDependencyController inline | Compounding #1 |

### §6.4 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-6.1** | Default dependency op when unsure | **Observe.** Coupling is the default sin; require typed justification to upgrade to Meet/Join. | Compounding #0, #5 |
| **R-6.2** | Materialized derived promessa CR or view? | **View** at query time from current participating promessas. Materialize as a CR only if the derived is itself an audit target. | Pillar 10; R-1.7 |

---

## §7 — Cadence

### §7.1 Defaults

| Kind | `reconcile_every` | Rationale |
|---|---|---|
| SLA | 60s | sub-minute latency-spike reaction |
| CostBudget | 5m | cost data lags |
| Compliance | 15m | posture changes propagate slowly |
| CustomerKpi | 1h | KPI data aggregates over hours |
| Security | 15m | CVE feeds + runtime events |

### §7.2 Per-promessa override

`:reconcile-every <duration>`; bounded **[10s, 24h]**; outside =
typed error. Chain growth ∝ 1/cadence; retention planned in §8.4.

### §7.3 Sampling

```lisp
:observation-sampling { :strategy reservoir :size 1000 }
```

Recorded in every OutcomeReceipt; replay accounts for it.

### §7.4 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-7.1** | May cadence vary across the chain (adaptive)? | **No.** Cadence is part of the spec; changes = SpecRevised. Adaptive cadence couples chain growth to observed state — breaks reproducibility. | Pillar 10 |
| **R-7.2** | What if tick takes longer than `reconcile_every`? | Skip ticks; emit `BackpressureDetected` anomaly; chain records gaps via `tick_index`. Skipping > 10% over window = Critical (controller misconfigured). | ★★ Shigoto; Pillar 11 |

---

## §8 — Lifecycle

### §8.1 Spec evolution

| Kind | Action | Receipt |
|---|---|---|
| **Tightening** (more conservative) | In-place; bump `:spec-revision` | `SpecRevised` with old + new hashes |
| **Loosening** (less conservative) | In-place + mandatory `:loosening-waiver: <typed-reason>` | `SpecRevised` with waiver |
| **Reshape** (scope or kind changes) | **New promessa** + retire old | New chain genesis cross-links old's `Retired` |

Invariant: **spec hash MAY change; commitment subject MUST NOT.**

### §8.2 Decommissioning

1. Mark `:retiring true`
2. AnomalyController stops AutoCorrect; only Alerts emit
3. After 2× window, emit `Retired` receipt
4. PromessaCR deleted; OutcomeChain remains immutable in MinIO

### §8.3 Runbook → promessa migration

The canonical typed sequence (per Compounding #1):

| Step | Action |
|---|---|
| 1 | Extract typed shape (desired, observed, action, escalation) from runbook |
| 2 | §1 diagnostic; confirm promessa |
| 3 | Author `(defpromessa …)` using existing TargetController |
| 4 | **Ship in shadow mode** — `RemediationPolicy::Alert` only — for one full window |
| 5 | Compare AnomalyChain to actual human remediations |
| 6 | Tune severity calibration / remediation policy |
| 7 | Promote to AutoCorrect |
| 8 | **Retire the runbook** — the chain IS the system of record |

### §8.4 Chain retention

| Chain | Default retention | Triggered by |
|---|---|---|
| OutcomeChain | Promessa lifetime + 7 years post-retirement | Compliance audit horizons |
| AnomalyChain | Cluster lifetime + 7 years post-cluster-retirement | Postmortem horizons |

Per-cluster MinIO bucket lifecycle; tameshi receipt-sink discipline.

### §8.5 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-8.1** | Can a promessa skip shadow mode? | **Only with `:shadow-mode-skip-waiver: <typed-reason>`.** Acceptable for low-stakes Cosmetic-only promessas. | Compounding #0, #4 |
| **R-8.2** | Keep runbook "as backup" after promotion? | **Forbidden.** Two systems of record = drift. The chain IS the record. | Compounding #1 |

---

## §9 — Failure modes

### §9.1 Wedged PromessaController

**Detection:** per-cluster `LivenessController` observes last receipt
ts. Stale > 3× cadence → `ControllerWedged` anomaly.

**Recovery:** restart pod → if fails, promote to Critical →
EscalationLadder fires. Chain gap is auditable via LivenessController's
own anomaly receipt.

### §9.2 ObservationSource unreachable

| Behavior | Constraint |
|---|---|
| Tick produces receipt with `observation = Failed { reason }` | Chain continuity preserved |
| **AutoCorrect MUST NOT fire on missing observation** | Trait law `no_action_without_observation`; substrate does not act on guesses |
| Consecutive failures: 1→Cosmetic, 3→Functional, 10→Critical | Via LivenessController promotion |

### §9.3 Action repeatedly fails

| Tick | Behavior |
|---|---|
| 1 | Retry per shigoto-typed `RetryPolicy` |
| 2 | `RemediationFailed` anomaly; declared policy applied |
| 3 | Critical; EscalationLadder fires |
| 5+ | `remediation-disabled` state; receipt `decision = SuppressedAfterRepeatedFailure`; human required |

Trait law: `repeated_failure_terminates`. No infinite silent retries.

### §9.4 Dependency cycle / spec collision

Both checked at admission; typed errors (`CyclicDependency`,
`NameCollision`). Rejected before chain begins.

### §9.5 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-9.1** | EscalationLadder exhausts — all steps fired, nothing resolved? | Emit `EscalationExhausted`; chain entry. **The substrate has a missing primitive** — the gap is a substrate-improvement ticket, not runtime failure to paper over. | Compounding #0, #1 |
| **R-9.2** | Multi-tenant: cluster A controller crashes cluster B? | **Cannot happen.** Single-replica AnomalyController per cluster; cross-cluster federation via denshin. Failure bounded per cluster. | ★★ Shigoto; ★★ The Viggy Method §X.6 |

---

## §10 — Testing minimum

Every new TargetController + PromessaController + AnomalyController
ships with the test discipline below. **No exceptions; CI gates.**

### §10.1 trait_laws_obeyed (proptest)

Mandatory proptest suite, 10 invariants, expanded by macro
`trait_laws_obeyed!(MyController)`:

| # | Invariant |
|---|---|
| 1 | `observe_referentially_transparent` |
| 2 | `diff_deterministic` |
| 3 | `classify_deterministic` |
| 4 | `classify_monotonic` (§4.2) |
| 5 | `decide_deterministic` |
| 6 | `act_idempotent_on_noop` |
| 7 | `tick_converges` |
| 8 | `attest_canonical` |
| 9 | `no_action_without_observation` (§9.2) |
| 10 | `repeated_failure_terminates` (§9.3) |

Per ★★ macros-everywhere — one macro across every TargetController.

### §10.2 MockObservationSource

Ships in `promessa-types-test`. Returns scripted Snapshot values for
test inputs. **Never write a custom mock; use MockObservationSource.**

### §10.3 Golden trajectory tests

Per TargetController: PromessaSpec + 10 mock observations → expected
AnomalyChain BLAKE3 root (signatures excluded). Catches silent
behavior regressions.

### §10.4 kind-cluster integration

CI ships a `kind`-based integration per TargetController. **No merge
if it fails.**

### §10.5 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-10.1** | TargetController skipping trait_laws? | **No.** Failure = not a TargetController. | Pillar 10 |
| **R-10.2** | Mocks beyond MockObservationSource? | **Forbidden.** One mock per typed surface fleet-wide. | ★★ macros-everywhere |

---

## §11 — Operator workflows

### §11.1 Declare

| # | Action |
|---|---|
| 1 | Diagnose via §1 |
| 2 | Granularity via §2 |
| 3 | Cite-vs-wrap via §3 |
| 4 | Author `(defpromessa …)` in `<repo>/promessas/<name>.tatara` |
| 5 | Render via `caixa-publish` → YAML CR under `k8s/clusters/<cluster>/promessas/<name>.yaml` |
| 6 | PR with cross-links to contract / audit baseline / SLA doc |
| 7 | CI: `trait_laws_obeyed`, golden trajectories, kind integration all pass |
| 8 | Merge + FluxCD apply; controller registers |
| 9 | Shadow window if migrating from runbook (§8.3) |
| 10 | Promote to AutoCorrect when calibrated |

### §11.2 Audit

```bash
kensa verify outcome-chain --promessa <name> --window <range> [--baseline <baseline>]
  → typed OutcomeVerificationReport

kensa replay anomaly-chain --cluster <name> --window <incident-range>
  → typed PostIncidentReport

promessa get <name>            # current spec + last receipt
promessa history <name> <range>
anomaly history <range>
```

### §11.3 Override (typed break-glass)

| # | Action |
|---|---|
| 1 | Confirm controller stuck > escalation max for >1h |
| 2 | Open typed Override issue (mcp__github__) — what controller would have done; why it can't; action taken |
| 3 | Act manually via direct API |
| 4 | Emit `AnomalyEmission { kind: ManualBreakGlass, github_issue: <url> }` |
| 5 | **File substrate-improvement ticket.** Every Override is data toward a missing primitive |

### §11.4 Resolved via directives

| # | Question | Auto-answer | Directive |
|---|---|---|---|
| **R-11.1** | Skip filing substrate ticket post-Override? | **No.** Every Override that doesn't yield a ticket is data lost. | Compounding #1, #7 |

---

## §12 — Federation

### §12.1 Decision

```
Does the business outcome name a SINGLE target ACROSS clusters?
  ├── YES → Federated promessa (coordinator at fleet edge)
  └── NO  → Per-cluster promessas + PromessaLattice::meet(…) view
```

### §12.2 Architecture

- **Coordinator** at fleet edge (seph, PID 1) — holds federated spec
- **Executor** per member cluster — holds typed `ClusterScopedView`
- Coordinator tick: observe via cross-cluster shinryu; act via denshin
  directives to executors; attest federated OutcomeReceipt
- Executors keep autonomy; coordinator influences via directives

### §12.3 Anti-patterns

| Anti-pattern | Directive |
|---|---|
| Coordinator mutates executor state directly | ★★ The Viggy Method §X.6 |
| Federation by default | Compounding #0; §12.1 |
| Per-cluster promessas summed in dashboard instead of `PromessaLattice::meet(…)` | ★★ TYPED EMISSION |

---

## §13 — Cross-domain composition

Promessas weave above existing typed primitives. **Never duplicate.**

| Existing surface | Citation use | Wrap use |
|---|---|---|
| caixa `:limits` (per-Servico) | per-Servico declaration | SLA promessa attesting limits hold |
| caixa `:contratos` (typed inter-Servico) | declared in mesh | Compliance promessa attesting contract conformance |
| caixa `:upgrade-from` | typed migration | Compliance promessa blocks migrations violating baseline |
| saguao crachá AccessPolicy | per-policy decisions | Security promessa over saguao audit log; CrachaPatch quarantines |
| saguao vigia forward-auth | observable via saguao audit chain | Security promessa correlates anomalous auth |
| cofre rotation policy | per-secret schedule | Security promessa attests rotation occurred; CofreRotate triggers |
| cofre backend availability | observable | SLA promessa observes backend response time |
| tameshi HeartbeatChain | static attestation | OutcomeReceipt cites leaves for cross-chain audit |
| sekiban admission | rejection at admission | Compliance promessa patches policies mid-tick |
| kensa pre-deploy | gate at deploy | Promessa act() can request kensa check before committing |
| inshou Nix integrity | static check | Security promessa observes inshou chain |
| kanshi eBPF runtime | runtime events | Security promessa observes kanshi events |
| **shinryu** | universal SQL surface | **Canonical ObservationSource for every promessa** |

**Composition invariant:** every cross-domain promessa cites the
underlying primitive's typed output in `ObservationSource` and patches
via the primitive's typed surface in `TypedAction`. Never bypasses.

---

## §14 — Anti-patterns + spec location

### §14.1 Look-alikes that aren't Viggy

| Look-alike | Why not Viggy | What it actually is |
|---|---|---|
| Controller watching a CRD | Per-resource | Peer controller (engenho native) |
| CronJob cleanup | One-shot per fire | kube CronJob; promote only if cross-resource attestation needed |
| Webhook responding to events | Event-driven not continuous | engenho event-driven controller |
| `Reconciler` trait impl alone | Reconciliation without business outcome | Peer controller; can be a promessa's Action target |
| Compliance dashboard | Visualization not attestation | Sink of OutcomeChain receipts |
| Alertmanager rule | Alert without typed remediation | Promote to anomalia emission inside a promessa |
| GitOps repo with prune: true | Per-resource | FluxCD native; cite |
| Custom operator with finite loop | Finite not continuous | shigoto Job or make continuous |
| Health check endpoint | Per-resource liveness | kube `livenessProbe`; promote to SLA promessa for fleet attestation |

### §14.2 Spec location (canonical paths)

| Artifact | Location |
|---|---|
| `(defpromessa …)` source | `<repo>/promessas/<name>.tatara` |
| Rendered YAML CR | `k8s/clusters/<cluster>/promessas/<name>.yaml` (FluxCD-tracked; never hand-edited) |
| `(defanomalia …)` source | `<repo>/anomalias/<name>.tatara` |
| Rendered Anomaly schema CR | `k8s/clusters/<cluster>/anomalias/<name>.yaml` |
| TargetController Rust impl | `engenho-promessa-controllers/<kind>/src/` |
| AnomalyController Rust impl | `engenho-anomaly-controller/src/` |
| OutcomeChain receipts | `s3://outcome-chains-<cluster>/<promessa>/<receipt-hash>.json` |
| AnomalyChain receipts | `s3://anomaly-chain-<cluster>/<receipt-hash>.json` |
| trait_laws proptest | `<crate>/tests/trait_laws.rs` (macro-expanded) |
| Golden trajectories | `<crate>/tests/golden/<kind>.json` |
| Replay UI | `varanda/promessa-replay/` |
| MCP tools | `convergence-controller/mcp/{promessa,anomaly}.rs` |

The `.tatara` file IS the source of truth. YAML is rendered. Per
★★ TYPED EMISSION: every step through typed AST renderers.

---

## §15 — Extracted abstractions

The typed surfaces that emerged from §1–§14. These ARE the generative
substrate — the next 100 ambiguities resolve against these, not
against this doc.

### §15.1 The five typed enums

```rust
// All in `promessa-types` / `anomalia-types`.

pub enum AuthoringDecision {              // §1 output
    Promessa(PromessaTargetKind),
    PeerController(PeerControllerKind),
    CitePrimitive(PrimitiveRef),
    Defer(DeferReason),
    NotViggy(SubstrateRoute),
}

pub enum PromessaTargetKind {             // §1 Q5
    Sla, CostBudget, Compliance, CustomerKpi, Security, Custom,
}

pub enum PeerControllerKind {             // §1.5.B branches
    KubeNative,
    PangeaMagma,
    PangeaOperatorReconciler { kind: String },
    EngenhoCustomCrd,
    SekibanAdmissionPolicy,
    CofreSecretRef,
    SaguaoCracha,
}

pub enum SubstrateRoute {                 // §1.5.A branches
    ShinkaMigration,
    KenshiTestGate,
    ForgeGenCodegen,
    ManualWithRunbook,
    PureCompute,
    ShinryuExplore,
}

pub enum DeferReason {                    // §1.5.D
    NoAuditRequirement { reevaluate_at: Date },
    ObservationSourceMissing { needed: String },
    KindNotYetSupported { proposed: String },
}
```

### §15.2 The ten trait laws

Per §10.1; macro-expanded; property-tested. **One macro across every
TargetController; the trait IS the surface.**

### §15.3 The granularity invariant

```rust
debug_assert_eq!(Promessa::scope, Commitment::subject);
debug_assert_eq!(Promessa::window, Commitment::time_scale);
```

Violation = `ScopeMismatch` admission error.

### §15.4 The wrap-not-compete invariant

`ObservationSource` cites existing primitive output; `TypedAction`
acts elsewhere OR patches via typed surface. **Property-tested** as
`wrap_not_compete` in every PromessaController.

### §15.5 The migration sequence (typed states)

```rust
pub enum MigrationStage {
    Extracted, Diagnosed, Authored, Shadow,
    Calibrated, Promoted, RunbookRetired,
}
```

Each transition is a typed step; the migration itself is a shigoto
Dag of 7 nodes.

### §15.6 The Override → substrate-improvement loop

Every typed Override emits `AnomalyEmission { kind: ManualBreakGlass }`
+ a substrate-improvement ticket. Override is data; the ticket is the
compounding move. **The substrate grows monotonically with every
Override.**

### §15.7 The directive-derived auto-answer principle

**When in doubt about a Viggy authoring decision, derive the answer
from the prime directives, not from local judgment.** The directives
encode the substrate's discipline; deviating from them = drifting
from the substrate. If a question cannot be auto-answered via the
directives + this doc, **that itself is the signal of a missing
directive** — file the gap; do not paper over.

This is the meta-rule that makes the discipline generative: future
ambiguities resolve against the directive set; the directive set is
canonical; ambiguities that survive force the directive set to grow.

### §15.8 The discipline's compounding property

| Inputs | Outputs |
|---|---|
| 1 `(defpromessa …)` source | + 1 YAML CR (rendered) |
| + the TargetController Rust impl (already exists per kind) | + 1 OutcomeChain (signed receipts, indefinitely) |
| + the substrate (already standardized on) | + 1 AnomalyChain (response record) |
| | + MCP tool surface (auto-derived) |
| | + audit-ready verification via `kensa verify outcome-chain` |
| | + replay-ready forensics via `kensa replay anomaly-chain` |
| | + admission gating via sekiban (auto-derived from CRD schema) |
| | + monitoring dashboards via tameshi receipt sinks (auto-derived) |

Per-promessa **author effort: ~20 lines of `.tatara`**. Substrate
amortized cost: zero (everything already exists). This is the
generative property — the discipline compounds because the author
writes a typed declaration; the substrate produces every downstream
artifact.

---

## §16 — Roadmap: extending the discipline

§1–§15 covers **authoring**. Seven sibling artifacts round out the
substrate. Priority is the substrate-side discipline + the agent
skill; concrete adoption follows.

| # | Artifact | Scope | Priority |
|---|---|---|---|
| 1 | **VIGGY-IMPLEMENTING.md** | Substrate-side discipline: how to author a new TargetController; add a TypedAction variant; extend ObservationSource; write the trait-laws proptest harness. The Rust patterns + the macro shapes. | **first** |
| 2 | **`skills/viggy/SKILL.md`** | Invocable agent skill. `/viggy` loads canon (THEORY §IV.7 + CONTINUOUS-SOLUTION-MACHINE.md + VIGGY-AUTHORING.md) into agent context. Per ★★ macros-everywhere. | **first** |
| 3 | **VIGGY-OPERATING.md** | Daily-operator discipline: PR templates; on-call workflow under Viggy; audit prep; postmortem via AnomalyChain replay; the operator's three verbs as concrete day-to-day playbooks. | second |
| 4 | **VIGGY-MIGRATING.md** | Legacy migration discipline: mechanical translation for common shapes (PagerDuty rotations, cron jobs, Datadog/Splunk dashboards, compliance spreadsheets, on-call runbooks). Sibling to §8.3; expands the canonical sequence per legacy type. | second |
| 5 | **Implementation M0** | `promessa-types` crate scaffolded via `repo-forge new`. Per CONTINUOUS-SOLUTION-MACHINE.md §XIV roadmap. The Rust substrate begins. | third |
| 6 | **Reference promessas (one per kind)** | Canonical worked examples committed in lilitu / nexus: `prod-sla.tatara`, `prod-cost-budget.tatara`, `regulated-compliance.tatara`, `product-nps.tatara`, `prod-security.tatara`. | alongside M1 |
| 7 | **Cross-references in existing skills** | One-line additions in `compliant-systems`, `observability`, `compliant-artifact-provability`, `attestation`, `convergence-flow` skills citing Viggy where concerns intersect. Keeps the substrate's skill graph monotonically informed. | rolling |

The directive set under which all seven artifacts are authored: **★★★
Compounding Directive + ★★ The Viggy Method + ★★ macros-everywhere +
Pillar 12 (generation over composition)**. Per §15.7: ambiguities
that survive the directive set force the directive set to grow.

---

## Closing

VIGGY-AUTHORING is the typed authoring discipline. Each section is a
generator. Open questions are typed → auto-answered via prime
directives → unresolved ambiguities earn substrate-improvement tickets.

The author writes one `(defpromessa …)`. The substrate produces every
downstream artifact. **The substrate's discipline is the proof.**

---

> *The cluster is the algebra. The author writes one (defpromessa …).
> The substrate produces every downstream artifact. **Viggy is the
> practice.***
