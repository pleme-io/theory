# VIGGY-OPERATING — daily operator discipline

> **Thesis.** The Viggy operator's job is **three verbs**: *Declare*
> (author new promessas), *Audit* (query chains), *Override* (typed
> break-glass when reconciliation cannot resolve). Everything the
> operator used to do — deploying, scaling, patching, monitoring,
> alerting, rotating, right-sizing — is *reconciliation*, and
> reconciliation is the cluster's job, not the operator's.
>
> This document is the canonical day-to-day playbook under Viggy.
> What does on-call look like? What does an audit look like? What
> does a shadow-mode promotion look like? Concrete sequences, all
> typed.

Companion docs:

- [`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md) — the framework spec
- [`VIGGY-AUTHORING.md`](./VIGGY-AUTHORING.md) — operator-side authoring discipline
- [`VIGGY-IMPLEMENTING.md`](./VIGGY-IMPLEMENTING.md) — substrate-side discipline
- [`VIGGY-LEGOS.md`](./VIGGY-LEGOS.md) — controller universe + ten legs
- `pleme-io/CLAUDE.md` ★★ The Viggy Method — the directives

---

## Part I — Frame

### I.1 Who this is for

The **operator** — anyone with `promessa-author` ClusterRole + audit
read access. Not the substrate engineer (read VIGGY-IMPLEMENTING.md
instead), not the promessa author at the moment of authoring (read
VIGGY-AUTHORING.md instead). This doc is for the **steady-state
ongoing operation** of a Viggy fleet.

### I.2 The three verbs

| Verb | What it is | When |
|---|---|---|
| **Declare** | author a new promessa / anomalia / substrate extension | new business commitment lands; new audit requirement appears; new operational concern surfaces |
| **Audit** | query chains via `kensa verify` / `kensa replay` / `promessa.history` MCP | compliance audit; SLA report; FinOps review; postmortem; routine operational visibility |
| **Override** | typed break-glass when reconciliation cannot resolve | controller wedged, EscalationLadder exhausted, business emergency that the substrate has no typed response for |

The first verb is **creative** (extends the algebra). The second is
**analytical** (reads the algebra). The third is **exceptional** —
and signals a missing typed primitive. **Every override is data
toward a substrate-improvement ticket.**

### I.3 What Viggy operating is NOT

| Not | Because |
|---|---|
| Coding | The substrate engineers code; operators declare |
| Debugging | The chains are the source of truth; `kensa replay` is debugging |
| Imperative state mutation | Per ★★ GITOPS-NATIVE + ★★ CONTINUOUS CONVERGENCE, mutations are controller actions, not operator commands |
| Dashboard-watching as the system of record | Dashboards are *sinks* of the typed chains; the substrate IS the system of record |
| Writing runbooks | Runbooks are anti-patterns; controllers carry the work |
| On-call ad-hoc triage | The EscalationLadder fires the right humans at the right time; the operator's role on-call is to execute typed actions, not to improvise |

---

## Part II — Declare: authoring a new promessa

The canonical authoring sequence (sibling to VIGGY-AUTHORING §11.1 +
§16). Day-to-day operator workflow.

### II.1 End-to-end sequence

| # | Step | Output |
|---|---|---|
| 1 | Identify the business commitment (contract / SLA / baseline / KPI / security commitment) | one sentence — the subject of the promessa |
| 2 | Run `/viggy <natural-language declaration>` | the disambiguation interview begins (VIGGY-AUTHORING §1–§9) |
| 3 | Confirm + refine each phase (granularity, observation, action, composition, cadence) | `(defpromessa …)` source draft |
| 4 | Run `kensa verify outcome-chain --dry-run` against the draft to confirm chain layout | typed validation report |
| 5 | Open PR titled `feat(viggy): <name> — <one-line commitment summary>` | PR with cross-links to contract + audit baseline + downstream stakeholders |
| 6 | CI gates: `trait_laws_obeyed`, golden trajectory, kind-cluster integration | green |
| 7 | Reviewer attests: the typed shape matches the commitment | review approval |
| 8 | Merge + FluxCD applies | controller registers; first OutcomeReceipt within `reconcile_every` |
| 9 | **Shadow mode** (`RemediationPolicy::Alert` only) for one full window | calibration data; AnomalyChain compared to expected behavior |
| 10 | Promote to AutoCorrect by RemediationPolicy change | second PR; reviewer attests calibration; merge |

### II.2 PR template

```markdown
## Summary

This PR authors **<promessa-name>**, a `<TargetKind>` promessa with
scope = `<scope>`, window = `<window>`.

**Commitment cite:** <link to contract clause / SLA doc / audit
baseline / KPI definition>

## What this attests

The OutcomeChain produced by this promessa is the typed proof that:

> <one paragraph — the typed promise being held>

## Substrate touches

- **Existing primitives consumed:**
  - <caixa Servico's :limits> at <path>
  - <saguao crachá policy> at <name>
  - <cofre SecretRef> for <signing-key>
  - (…)
- **TypedAction(s):** `<FluxCommit / ReconcilerApply / MagmaApply / …>`
- **PromessaDependency edges:** `[<…> :op meet, <…> :op observe]`
- **No new TargetController kind needed** (uses existing <kind> controller)

## Open substrate gaps

<list any gaps that emerged during the disambiguation interview that
couldn't be auto-resolved via prime directives; each is a separate
substrate-improvement issue link>

## Test plan

- [ ] `trait_laws_obeyed` passes for `<TargetController>`
- [ ] Golden trajectory matches `<root>`
- [ ] `kind`-cluster integration: 100 ticks clean
- [ ] **Shadow mode for `<window>` after merge** (`RemediationPolicy::Alert` only)
- [ ] Calibration review: AnomalyChain compared to expected human remediations
- [ ] Promote to AutoCorrect once calibrated (separate PR)

## Cross-references

- Contract / SLA / baseline: <link>
- Existing primitives consumed: <links>
- Reviewer attestation: <name> confirms the typed shape matches the
  commitment.
```

### II.3 Reviewer expectations

Reviewer's job is **not** to verify code correctness (CI does that
via trait_laws + golden trajectories). The reviewer's job is to
attest:

| Reviewer check | Confirms |
|---|---|
| Spec scope matches commitment subject | per VIGGY-AUTHORING §2.1 |
| Window matches commitment time scale | per VIGGY-AUTHORING §2.1 |
| ObservationSource is the right typed surface (cite-vs-wrap) | per VIGGY-AUTHORING §3 |
| Severity overrides (if any) carry typed justification | per VIGGY-AUTHORING §4.3 |
| TypedAction is highest-applicable on the reuse hierarchy | per VIGGY-LEGOS §VI.1 |
| Dependencies are typed-justified (default = Observe) | per VIGGY-AUTHORING §6 R-6.1 |
| Cadence is appropriate per kind | per VIGGY-AUTHORING §7.1 |
| Shadow window is declared (unless waiver) | per VIGGY-AUTHORING §8.5 R-8.1 |
| Open substrate gaps are filed as issues | per §15.7 directive-derived auto-answer |

**Code review of the controller logic is the substrate engineer's
domain (VIGGY-IMPLEMENTING.md), not the operator's.** A promessa
author is configuring an existing TargetController; the
TargetController has already been reviewed once.

### II.4 CI gates (recap)

| Gate | Stage |
|---|---|
| `cargo clippy -- -D warnings` (typed emission ban on `format!()`) | every PR |
| `trait_laws_obeyed!(<TargetController>)` proptest | every PR (≥48 cases per law) |
| Golden trajectory test for the TargetController kind | every PR |
| `kind` cluster integration: 100 ticks clean | every PR |
| sekiban PromessaCR admission schema validation | every PR |
| `kensa verify outcome-chain --dry-run` against the draft spec | every PR |

**No merge if any gate fails.** Per Pillar 10 proof discipline.

---

## Part III — Audit: querying chains

The audit verb is **how operational truth is established under
Viggy**. No dashboards as system of record; no spreadsheets; no
human-authored compliance reports. Just chain queries.

### III.1 Routine SLA reporting (weekly / monthly)

```bash
kensa verify outcome-chain \
  --promessa lilitu-prod-sla \
  --window 30d \
  --baseline customer-contract-v3
```

Returns a typed `OutcomeVerificationReport`:

```json
{
  "promessa": "lilitu-prod-sla",
  "window":   { "start": "...", "end": "..." },
  "ticks": 43200,
  "ticks_critical": 0,
  "ticks_functional": 12,
  "ticks_cosmetic": 43188,
  "outcome": "HELD",
  "remediations": [ /* … */ ],
  "chain_root": "blake3:af3c…",
  "signature": "ed25519:…"
}
```

Hand to the customer / SRE leadership / SLO dashboard renderer. The
report IS the proof. **No marketing-graphs intermediary.**

### III.2 Compliance audit (auditor walks the chain)

The auditor receives the public verification key and runs:

```bash
kensa verify outcome-chain \
  --promessa lilitu-regulated-compliance \
  --window 2026-Q1 \
  --baseline pci-dss-4.0
```

The auditor verifies:

1. Ed25519 signature on every receipt against the public key
2. BLAKE3 chain continuity (`prev_receipt_hash` matches)
3. Per-receipt: did the PCI-DSS-4.0 control projection match observed?
4. Window aggregates: max severity observed; number of remediation
   actions; mean time to remediate

The output is auditor-ready. **No PDF; no screenshots; no
human-authored prose.** Per VIGGY-AUTHORING §11.2.

### III.3 Cost audit (FinOps quarterly review)

```bash
kensa verify outcome-chain --promessa lilitu-prod-cost-budget --window 90d
```

The chain proves:

- Monthly budget held in each of three months
- Number of auto-remediations fired (scale-downs)
- Any breaches → typed escalation steps + resolutions

FinOps receives a typed report. **No CSV exports.**

### III.4 Customer-KPI audit (board reporting)

```bash
kensa verify outcome-chain --promessa lilitu-product-nps --window 365d
```

The chain proves:

- NPS trajectory across the year
- Auto-rollback events fired (feature flag reversions on drops)
- Manager-approval gates triggered
- Stakeholder responsiveness (escalation latency)

Goes into the board deck. **The chain IS the board's source of
truth.**

### III.5 Security audit (CVE response window)

```bash
kensa verify outcome-chain --promessa lilitu-prod-security --window 30d
kensa replay anomaly-chain --window 30d
```

Twin queries — the outcome chain proves the SLA on CVE-age was held;
the anomaly chain documents every CVE detected + how it was routed.

Hand to the security team. **The chain IS the security posture.**

---

## Part IV — Override: typed break-glass

The exceptional verb. **Acceptable, but every Override is data
toward a substrate gap.**

### IV.1 When break-glass is acceptable

| Acceptable | Not acceptable |
|---|---|
| Controller wedged > escalation max for >1h, and the EscalationLadder has exhausted | "I just want to fix it faster than the controller" |
| The EscalationLadder names you and the prior steps failed | "Reconciliation feels slow" |
| Business emergency the substrate has no typed RemediationPolicy for | "I don't trust the AutoCorrect" |
| Genuine substrate failure (e.g., engenho API server crashed) | "I prefer manual" |

If your reason isn't in the left column, **don't break glass.** Per
★★ CONTINUOUS CONVERGENCE, the substrate carries the work.

### IV.2 The override sequence

| # | Action |
|---|---|
| 1 | Confirm: controller stuck > escalation max for >1h **OR** EscalationLadder has named you |
| 2 | Open typed Override issue (via `mcp__github__` or `/viggy override <reason>`) — what the controller would have done; why it can't; what manual action is being taken |
| 3 | Act manually via direct API (`kubectl`, cloud CLI, etc.) |
| 4 | Emit `AnomalyEmission { kind: ManualBreakGlass, github_issue: <url> }` to the cluster's AnomalyChain |
| 5 | **File a substrate-improvement ticket** — every override is data toward a missing primitive |
| 6 | Document the override in the cluster's `RUNBOOK.md` (so the next override of the same shape promotes to a controller) |

### IV.3 The substrate-improvement loop

Per VIGGY-AUTHORING §15.6 + the Compounding Directive: every override
contains the seed of a missing typed primitive. The substrate
engineer reads the override anomaly receipt + the substrate-improvement
ticket and either:

- Adds a new TargetController kind for this class of business outcome
- Adds a new TypedAction variant for this class of action
- Adds a new ObservationSource variant for this class of measurement
- Adds a new EscalationStep type for this class of human response
- Extends an existing primitive

The override IS the evidence that the gap exists. The next operator
with the same shape needs no override because the primitive now
ships.

---

## Part V — On-call workflow under Viggy

What being on-call means when reconciliation is the default mode.

### V.1 What an alert means

Under Viggy, an alert is **the EscalationLadder firing** — never a
raw metric breach. The escalation has already done:

1. Detected the drift via the seven-beat tick
2. Classified Severity (Cosmetic / Functional / Critical)
3. Applied the declared RemediationPolicy
4. Fired `Controller` step (if any) — controller-side retry
5. Fired `OnCall` step (e.g., PagerDuty) — **you**

By the time you're paged, **the substrate has already tried the
typed remediation.** Your job is the next step in the ladder.

### V.2 The escalation ladder firing sequence

Typical SLA promessa ladder (per VIGGY-AUTHORING §9 defaults):

```
Critical drift detected
  │
  ▼
Step 1: Controller retry (5m timeout) — auto-correct one more time
  │ (timeout)
  ▼
Step 2: OnCall (PagerDuty, 15m timeout) — operator gets paged
  │ (timeout)
  ▼
Step 3: Manager (1h timeout) — manager pager
  │ (timeout)
  ▼
Step 4: Exec (4h timeout) — exec pager
```

Each step is typed and timed. The receipt in the AnomalyChain records
which step fired, when, and how it was resolved.

### V.3 Operator actions per ladder step

When you're paged (Step 2):

| # | Action |
|---|---|
| 1 | Acknowledge the page (PagerDuty / Opsgenie / ntfy) — this also acknowledges the AnomalyChain entry |
| 2 | Inspect the OutcomeReceipt at the breach: `promessa get <name>` + `promessa history <name> 30m` |
| 3 | Inspect the AnomalyChain context: `anomaly history 30m --kind <…>` |
| 4 | Check related promessas (PromessaLattice view): `promessa.lattice meet <names>` |
| 5 | Decide: is this resolvable via spec change (open a PR) or a typed Override (file ticket + manual action)? |
| 6 | Take the typed action — never improvise outside the substrate |
| 7 | If you Override: emit the `ManualBreakGlass` anomaly + file the substrate-improvement ticket |

### V.4 Post-incident review (replay anomaly-chain)

After the incident resolves:

```bash
kensa replay anomaly-chain \
  --cluster <name> \
  --window <incident-start..incident-end>
```

Returns a typed `PostIncidentReport`:

```json
{
  "window": "...",
  "anomalies": [
    {
      "id": "uuid-...",
      "emission": { "source": "lilitu-prod-sla", "kind": "SloBreached", "severity": "Critical" },
      "resolved_policy": { "Escalate": { "ladder": "sla-default" } },
      "escalation_path": [
        { "step": "Controller", "fired_at": "...", "resolution": "Timeout" },
        { "step": "OnCall",     "fired_at": "...", "resolution": "OperatorResolved" }
      ],
      "outcome": "Resolved"
    },
    /* … */
  ],
  "mttr_by_severity": { "Critical": "12m", "Functional": "0s" }
}
```

This IS the postmortem. **No Notion doc; no narrative; the chain
speaks.** Per VIGGY-AUTHORING §11.2.

---

## Part VI — Shadow-mode → AutoCorrect promotion

The calibration discipline for any new promessa.

### VI.1 Why shadow mode exists

A newly authored promessa's severity calibration + remediation policy
+ EscalationLadder are *guesses* until they run against real data.
Shadow mode runs the promessa with `RemediationPolicy::Alert` only —
no AutoCorrect — for one full window. Receipts flow; AnomalyChain
records what *would* have happened.

The operator compares:

| Source | What it says |
|---|---|
| AnomalyChain (would-have-done) | "the promessa would have AutoCorrected at tick 1247" |
| Real-world response (what humans actually did) | "at tick 1247, on-call scaled up by 2 replicas" |

Discrepancies → tune the calibration before promoting.

### VI.2 The shadow-window discipline

| Kind | Recommended shadow window |
|---|---|
| SLA | 7 days (covers weekly traffic cycle) |
| CostBudget | one full period (monthly / quarterly) |
| Compliance | 30 days (covers typical drift cycle) |
| CustomerKpi | one observation cycle (matches `:reconcile-every`) |
| Security | 30 days (covers CVE feed propagation) |

### VI.3 Calibration mechanics

During the shadow window:

| Check | Action |
|---|---|
| Severity classifications match operator expectations | proceed |
| Severity is **too aggressive** (Critical fires for non-critical drift) | tune thresholds tighter; document in `:severity-override` |
| Severity is **too lax** (drifts go unflagged) | tune thresholds tighter; document override |
| AutoCorrect would have made things worse (would scale wrong direction) | reshape the TypedAction; do NOT promote until corrected |
| EscalationLadder fires the wrong humans / wrong timing | adjust the ladder; re-test the routing |

### VI.4 The promotion PR

Once calibrated:

```markdown
## Summary

Promote `<promessa-name>` from shadow mode (RemediationPolicy::Alert)
to active mode (RemediationPolicy::AutoCorrect / Escalate / Compose).

## Calibration evidence

Shadow window: <start..end> (<duration>)

| Tick severity | Count | Notes |
|---|---|---|
| Cosmetic   | N    | matches expected baseline |
| Functional | N    | <list of cases + would-have-actions> |
| Critical   | N    | <list of cases + would-have-actions> |

Compared to actual human remediations during shadow window:
- <N> matches — the substrate would have made the same call
- <N> mismatches — calibrated and documented below

## Calibration changes

<list any `:severity-override` or `:remediation` policy adjustments>

## Promotion

After merge, the controller's RemediationPolicy switches from
`{ :on-functional alert }` to `{ :on-functional auto-correct … }`.
Next tick after FluxCD reconcile is the live activation.
```

---

## Part VII — Lifecycle operations

Per VIGGY-AUTHORING §8.

### VII.1 Tightening a spec

Operator action: edit `.tatara` source; bump `:spec-revision`; PR.
Reviewer attests the tightening is justified.

Receipt: `SpecRevised` with old + new spec hashes. Chain continues
unbroken.

### VII.2 Loosening a spec (with waiver)

```lisp
(defpromessa lilitu-prod-sla
  …
  :loosening-waiver { :reason "explicit business decision — Q1 cost-cutting forces 99.9% SLO temporarily"
                      :approver "vp-engineering"
                      :reverts-at "2026-04-01" }
  :target (sla :availability "99.9%" …))   ;; was 99.99%
  …)
```

The waiver is recorded in the chain genesis. **Time pressure is not
an acceptable reason** per Compounding Principle #0.

### VII.3 Reshape (new promessa, retire old)

If the commitment subject changes (scope expands / kind changes /
window changes substantively), this is **not** a revision — it's a
new promessa.

| # | Action |
|---|---|
| 1 | Author new promessa with new name + new spec |
| 2 | Mark old promessa `:retiring true` |
| 3 | New chain genesis cross-links old chain's final `Retired` receipt |
| 4 | After 2× window, old promessa fully retires |

### VII.4 Decommissioning

```lisp
(defpromessa lilitu-prod-sla
  …
  :retiring { :reason "service deprecated; lilitu replaced by ledere"
              :retired-at "2026-06-01" })
```

After 2× window post-`:retiring`, the controller emits a final
`Retired` receipt + the PromessaCR is deleted. **The OutcomeChain
remains in MinIO as the immutable historical record.**

---

## Part VIII — Inter-promessa management

### VIII.1 PromessaDependency edges

| Op | Use when |
|---|---|
| `Meet` | A's act() must not run if B is degraded (e.g., SLA `Meet` CostBudget — don't scale up if budget breached) |
| `Join` | A acts if either A or B resolves (rare; alternative resolutions) |
| `Observe` | A records B's state but doesn't block (default when unsure) |

### VIII.2 Lattice views

```bash
promessa.lattice meet lilitu-prod-sla drill-sla   # fleet-wide SLA view
```

Returns a derived view (not a materialized CR) showing the strictest
of each field across both promessas.

### VIII.3 Federation across clusters

For global commitments (e.g., "global lilitu p99 < 300ms"), federate
per VIGGY-AUTHORING §12. Coordinator on `seph`; executors per
cluster; same chain shape; cross-cluster shinryu queries.

---

## Part IX — The seven primitive operator moves

The complete operator surface, per CONTINUOUS-SOLUTION-MACHINE.md §XI.5.
**Everything else collapses to one of these:**

| # | Move | How |
|---|---|---|
| 1 | **Declare** a new promessa | `/viggy <natural-language declaration>` → render → PR |
| 2 | **Observe** current state | `promessa.get <name>` |
| 3 | **Verify** chain across window | `kensa verify outcome-chain` |
| 4 | **Replay** for forensics | `kensa replay outcome-chain` / `kensa replay anomaly-chain` |
| 5 | **Compose** via lattice | `promessa.lattice meet/join <names>` |
| 6 | **Acknowledge** a queued approval | `promessa.approve <anomaly-id>` |
| 7 | **Override** via break-glass | `/viggy override <reason>` + file substrate ticket |

No 8th move. No "edit the cluster manually". No "tweak the dashboard
threshold." If you reach for an 8th move, **either the substrate has
a gap (file a ticket) or you're operating outside Viggy (use the
typed waiver).**

---

## Part X — The MCP surface

The `/viggy` skill is the operator's primary entry point. Per
VIGGY-AUTHORING.md §11 and the `skills/viggy/SKILL.md` definition:

| Mode | Invocation | When |
|---|---|---|
| **A. Render** | `/viggy <natural-language declaration>` | authoring a new promessa |
| **B. Diagnose** | `/viggy diagnose <concern>` | unsure if it's Viggy-shaped |
| **C. Migrate** | `/viggy migrate <runbook>` | moving legacy work to a controller |
| **D. Audit** | `/viggy audit <name> <window>` | verifying chain |
| **D. Replay** | `/viggy replay <window>` | forensics |
| **E. Wake-up** | `/viggy` | load canon; confirm Viggy mode |

Plus persistent lens activation via "viggy style" / "viggy lens" /
"viggify" (per `viggy-lens-trigger` feedback memory).

---

## Part XI — Anti-patterns (operator side)

| Anti-pattern | Why forbidden | Where it lives instead |
|---|---|---|
| Editing the cluster manually with `kubectl edit` | ★★ GITOPS-NATIVE + ★★ CONTINUOUS CONVERGENCE | edit `.tatara` source; commit; let FluxCD + controllers reconcile |
| Authoring runbooks in Notion / Confluence | ★★ CONTINUOUS CONVERGENCE | declare as promessa; controller carries the work |
| Reading dashboards as the system of record | ★★ PROVABLE OUTCOMES | dashboards are sinks; the OutcomeChain is the SoR |
| Quarterly compliance via screenshots | ★★ PROVABLE OUTCOMES | `kensa verify outcome-chain --baseline …` |
| Pager-only response (no typed remediation declared in the policy) | substrate carries the response | declare `RemediationPolicy` + `EscalationLadder` at promessa-author time |
| Tweaking `:reconcile-every` to "feel responsive" | per VIGGY-AUTHORING §7.4 R-7.1 | defaults per kind are right; tighter ratios are smell |
| Skipping shadow mode (no `:shadow-mode-skip-waiver:`) | per §8.5 R-8.1 | always shadow before AutoCorrect |
| Keeping the old runbook after promessa promotion | per Compounding #1 | retire the runbook; the chain IS the record |
| Overriding without filing a substrate-improvement ticket | per §11.4 R-11.1 | every override is data toward a missing primitive |
| Loosening severity without `:loosening-waiver:` | per §4.4 R-4.2 | typed justification or no change |
| Federating by default | per VIGGY-AUTHORING §12.1 + §12.3 | per-cluster + lattice view first; federate only when target demands it |
| Adding a 4th severity tier | per §4.4 R-4.3 | three tiers are canonical fleet-wide |

---

## Part XII — Closing

The operator's day-to-day under Viggy is **declarative reasoning**,
**chain queries**, and **typed exceptions**. The cluster carries the
operational load that used to fall on humans. The operator's role
collapses to authoring the typed declarations + reading the typed
attestation chains.

**The substrate is the system of record; everything else is a view.**

Three verbs. Seven primitive moves. One typed surface. The chain IS
the proof.

---

> *Declare what should be true. Audit what is true. Override only when
> the substrate has a gap — and file the ticket. **Viggy operates
> itself; the operator declares.***
