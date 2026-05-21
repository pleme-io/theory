# VIGGY-MIGRATING — legacy migration discipline

> **Thesis.** Every legacy operational artifact — runbook, cron job,
> PagerDuty schedule, Datadog dashboard, compliance spreadsheet,
> Renovate auto-PR, cert-manager renewal, manual feature-flag
> rollback procedure, on-call playbook — has a **typed translation
> procedure** into a promessa. This document is the catalog of
> per-shape procedures. **No legacy artifact gets to stay** unless it
> carries a typed `skip-continuous-convergence:` waiver with an
> acceptable reason.
>
> The migration discipline is mechanical, not creative. Extract the
> typed shape; render the promessa; shadow-window; calibrate;
> promote; retire the legacy. Per Compounding Principle #1 — solve
> once, in one place, at one time.

Companion docs:

- [`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md) — the framework spec
- [`VIGGY-AUTHORING.md`](./VIGGY-AUTHORING.md) §8.3 — the canonical migration sequence
- [`VIGGY-OPERATING.md`](./VIGGY-OPERATING.md) §VI — shadow-mode calibration
- [`VIGGY-LEGOS.md`](./VIGGY-LEGOS.md) — controller universe + the legs

---

## Part I — Frame

### I.1 Who this is for

The **operator migrating an existing operational concern** into
Viggy. You inherit a runbook / dashboard / cron job / on-call rotation
and need to:

1. Extract the typed shape
2. Render the `(defpromessa …)`
3. Shadow-window-calibrate
4. Promote to AutoCorrect
5. **Retire the legacy artifact**

This is sibling to VIGGY-AUTHORING.md (which assumes greenfield) but
explicit about the *translation step* from informal-artifact to
typed-primitive.

### I.2 The migration principle

| Phase | What you do | Substrate output |
|---|---|---|
| **Extract** | parse the legacy artifact's `(desired, observed, action, escalation)` shape from its prose / config | typed shape candidate |
| **Diagnose** | run VIGGY-AUTHORING §1 (the five-question test) on the candidate | `AuthoringDecision::Promessa(<kind>)` or `PeerController(…)` |
| **Author** | render `(defpromessa …)` for the diagnosed kind | `.tatara` source + YAML CR |
| **Shadow** | run with `RemediationPolicy::Alert` for one full window | AnomalyChain + comparison evidence |
| **Calibrate** | tune severity overrides / remediation policy to match the legacy's effective behavior | tuned `:severity-override` + reviewer attestation |
| **Promote** | flip to AutoCorrect via PR | controller actively reconciles |
| **Retire** | mark the legacy artifact superseded; archive | cleaned operational surface |

### I.3 The compounding payoff

Each migration:

- Adds a typed audit chain for the concern (was unmaintained / human-only audit)
- Removes a human-only artifact (was operational debt)
- Validates the substrate's coverage of one more shape (every successful migration confirms the algebra grows)
- Identifies substrate gaps where translation isn't clean (file substrate-improvement tickets)

After enough migrations, **the legacy operational surface goes to
zero.** Per the Compounding Directive: the substrate solves; humans
declare.

---

## Part II — The canonical migration sequence (recap)

Per VIGGY-AUTHORING.md §8.3:

| # | Action |
|---|---|
| 1 | Extract typed shape `(desired, observed, action, escalation)` from the legacy artifact |
| 2 | Run §1 diagnostic; confirm promessa or branch to `§1.5.B` (peer controller) |
| 3 | Author `(defpromessa …)` using the appropriate TargetController kind |
| 4 | **Ship in shadow mode** — `RemediationPolicy::Alert` for one full window |
| 5 | Compare AnomalyChain (would-have-done) to actual human remediations |
| 6 | Tune severity calibration + RemediationPolicy + EscalationLadder |
| 7 | Promote to AutoCorrect via a calibration PR |
| 8 | **Retire the legacy artifact** — the chain IS the new system of record |

Parts III–IV below are the per-shape specializations.

---

## Part III — Per-shape translation procedures

For each common legacy shape: extract pattern + canonical PromessaTarget
kind + ObservationSource + TypedAction + EscalationLadder. The
mechanical translation; no judgment calls except in flagged spots.

### III.1 Runbook PDF / Notion / Confluence

**Shape:** prose document with sections like "Detect → Investigate →
Mitigate → Resolve" or "if X observed then do Y."

| Extract from runbook | → | Promessa slot |
|---|---|---|
| "Detect" section (what metric / log / signal) | → | `:observation (shinryu :query …)` |
| The desired threshold ("p99 < 200ms") | → | `:target (<kind> …)` |
| "Investigate" + "Mitigate" actions | → | `:remediation { :on-functional (auto-correct :action …) }` |
| "Resolve" or "escalate to X" | → | `:escalation (ladder …)` |
| The team / owner | → | `:scope` (services / namespace / product) |

**Typical kind:** matches the runbook's metric — SLA for latency /
availability runbooks, CostBudget for cost runbooks, Compliance for
compliance runbooks, Security for security runbooks.

**Retirement check:** add header to the legacy doc:

```markdown
> **SUPERSEDED:** This runbook has been migrated to promessa
> `<name>`. See OutcomeChain at `s3://outcome-chains-<cluster>/<name>/`.
> This page is archived as of `<date>` and should not be consulted
> for live operations.
```

After 90 days of successful AutoCorrect operation, **delete the
runbook page entirely.**

### III.2 Cron job (kube CronJob / system cron / GitHub Action schedule)

**Shape:** time-triggered action, no observation source — fires on
schedule regardless of state.

**Diagnostic question:** is this *truly* unconditional time-driven
work? Or does it actually have an implicit observation?

| Case | Decision |
|---|---|
| Pure schedule (e.g., "rotate the report every Monday 9am") | **Peer controller** — keep as kube CronJob; not Viggy |
| Conditional schedule ("check daily; act if drift exceeds X") | **Promessa** with cadence = polling interval, observation = the check |
| "Run cleanup every hour" (cost / disk / log retention) | **Custom promessa** observing the metric being cleaned up |

If it's truly a Cron-shaped concern, keep it as a kube CronJob (or
add the `CronReconciler` from CONVERGENCE-SUBSTRATE §III.2 once
shipped). If it's secretly a polled observation, **migrate to a
promessa** with the appropriate kind.

### III.3 PagerDuty rotation / on-call schedule

**Shape:** "if X breaks, page Y for time Z, then escalate to W."

This is **half-promessa, half-EscalationLadder**:

| Legacy PD | → | Promessa slot |
|---|---|---|
| The alerting metric ("p99 > 300ms") | → | `:observation` |
| The threshold | → | `:target` |
| The first responder service ("primary on-call") | → | `:escalation.step2.who = pagerduty:<service-id>` |
| Escalation policy (15 min → manager) | → | `:escalation.step3 (manager :timeout "15m" :who …)` |
| Acknowledgment behavior | → | `RemediationPolicy::Escalate { ladder: <…> }` |

**The schedule itself** (rotation, calendar) stays in PagerDuty; the
promessa references `pagerduty:<service-id>` and PagerDuty's
internal rotation logic decides who's actually paged.

**Retirement check:** the PagerDuty service stays; what changes is
that pages now arrive with **typed AnomalyChain context** instead of
raw Datadog/Splunk events.

### III.4 Datadog SLO dashboard

**Shape:** dashboard tracks availability / latency / error rate
against a stated SLO target.

This is the **canonical SLA promessa** migration target.

| Datadog SLO | → | Promessa |
|---|---|---|
| SLO target ("99.9% over 30 days") | → | `:target (sla :availability "99.9%") :window "30d"` |
| Underlying metrics (latency p99, error-rate query) | → | `:observation (shinryu :query …)` (or HttpEndpoint to Datadog if not already in shinryu) |
| Burn-rate alerts | → | `:remediation { :on-functional alert :on-critical escalate }` |
| Dashboard | → | **Becomes a sink** of the OutcomeChain — Datadog dashboard renders chain receipts, no longer authoritative |

**Retirement check:** the dashboard stays as a visualization sink;
its data sources flip from raw metrics to OutcomeReceipt projections.
The **dashboard is no longer the system of record** — that's
`kensa verify outcome-chain`.

### III.5 Splunk alert / Datadog monitor

**Shape:** "if log pattern X matches OR metric crosses Y, fire alert."

Per VIGGY-LEGOS §V.3 anti-pattern catalog: **Alertmanager rules
without typed remediation are anti-patterns.** Migrate:

| Alert | → | Promessa |
|---|---|---|
| Observed signal | → | `:observation` |
| Threshold | → | `:target` field |
| Notification destination | → | `:remediation { :on-cosmetic (alert :sink slack-forge:#oncall) }` |
| Severity / pager mapping | → | `:remediation :on-critical escalate; :escalation (ladder …)` |

**Retirement check:** the Splunk / Datadog alert config is deleted
(or kept as a read-only legacy reference). All alerting flows through
the AnomalyController → declared sinks.

### III.6 Compliance spreadsheet / quarterly review

**Shape:** spreadsheet tracks compliance controls; quarterly review
checks each against current cluster state.

**The canonical Compliance promessa migration.**

| Spreadsheet | → | Promessa |
|---|---|---|
| Each compliance control row ("PCI-DSS-4.0-3.5.1: encryption at rest") | → | bound to a kensa-projected control under `:target (compliance :baseline pci-dss-4.0)` |
| Quarterly review cadence | → | `:reconcile-every "15m"` (continuous instead of quarterly) |
| Pass/fail per row | → | per-tick OutcomeReceipt |
| Auditor walks the spreadsheet | → | `kensa verify outcome-chain --baseline pci-dss-4.0` |

**Retirement check:** the spreadsheet is archived. The auditor never
sees it again — only the chain.

### III.7 Renovate auto-PR (security / dependency updates)

**Shape:** Renovate watches dependency manifests; auto-opens PRs for
updates.

**Mostly a peer controller — but with a Viggy security promessa
wrapping it for audit attestation.**

| Renovate | → | Substrate |
|---|---|---|
| Renovate itself | → | **stays** — peer controller; runs as-is |
| The audit requirement ("no critical CVE > 24h in prod images") | → | **NEW Security promessa** wrapping Renovate's auto-PR + image-pin behavior |
| Renovate's PR-open action | → | one of the `act()` paths the Security promessa can dispatch |

**Retirement check:** Renovate doesn't retire — it becomes a peer
controller cited by a Security promessa. The audit chain attests
Renovate did its job.

### III.8 cert-manager auto-renewal

**Shape:** cert-manager renews certificates before expiry.

Same pattern as Renovate — **cert-manager stays; Security promessa
wraps for fleet-wide attestation:**

| cert-manager | → | Substrate |
|---|---|---|
| Per-Certificate renewal | → | cert-manager (unchanged) |
| Fleet-wide audit ("every cert in prod renewed before expiry") | → | Security promessa observing `Certificate.status.notAfter` across the fleet |

**Retirement check:** cert-manager doesn't retire. The Viggy chain
attests fleet-wide cert health.

### III.9 Custom kube CronJob (cleanup / rotation / right-sizing)

**Shape:** "every N hours, run script X to clean / rotate / resize."

Many are actually polled observations with a one-shot action — i.e.
secret promessas in disguise.

| CronJob purpose | → | Promessa kind |
|---|---|---|
| Disk cleanup ("when /var > 80%, delete old logs") | → | **CostBudget promessa** (or Custom) with observation = disk usage |
| Secret rotation ("rotate API keys every 90 days") | → | **Security promessa** observing secret age + `CofreRotate` action |
| Right-sizing ("if RAM unused for 24h, scale down") | → | **CostBudget promessa** with `:on-functional` scale-down |
| Backup ("snapshot every 6h") | → | keep as CronJob (pure schedule, no observation); `CronReconciler` from CONVERGENCE-SUBSTRATE once shipped |

### III.10 Manual feature-flag rollback runbook

**Shape:** "if NPS / CSAT drops > X, manually flip feature flag back."

**Canonical CustomerKpi promessa migration.**

| Runbook step | → | Promessa slot |
|---|---|---|
| The KPI ("NPS > 50") | → | `:target (customer-kpi :metric nps :minimum 50)` |
| The KPI source (Mixpanel / Amplitude) | → | `:observation (http-endpoint :url … :auth (cofre-secret-ref …))` |
| The rollback action ("set feature.recent-rollouts=false") | → | `:remediation { :on-critical (auto-correct :action (flux-commit … {:featureFlags.recent-rollouts false})) }` |
| Manager involvement ("notify PMM if rollback fires") | → | `:escalation (ladder :step2 (manager :who product-mgr-pager))` |

**Retirement check:** runbook deleted after 90 days of successful
shadow + AutoCorrect.

### III.11 NPS / customer-KPI Mixpanel dashboard

Same as §III.10 (the dashboard becomes a sink of the
CustomerKpi promessa's OutcomeChain).

### III.12 Cost / FinOps spreadsheet

Same as §III.6 (the spreadsheet becomes archived; `kensa verify
outcome-chain` replaces it).

### III.13 Security advisory mailing list / CVE response

**Shape:** team subscribes to NVD / GitHub Security / vendor mailing
lists; when a CVE drops, manually triage + patch.

Identical to §III.7 (Renovate). The Security promessa wraps both
Renovate + the CVE feed observation.

### III.14 SOC 2 / PCI / FedRAMP / ISO audit binder

**Shape:** annual audit; auditor walks a binder of evidence.

**The canonical Compliance promessa migration target** (§III.6 +
scale).

| Audit binder section | → | Substrate |
|---|---|---|
| Per-control evidence (screenshots, configs, sign-offs) | → | **Compliance promessa** per baseline + automatic OutcomeReceipt evidence |
| Annual cadence | → | continuous (15m reconcile) |
| Auditor's process | → | `kensa verify outcome-chain --baseline <baseline> --window 365d` returning a typed `OutcomeVerificationReport` |
| Auditor's deliverable | → | the verified chain root + signature + the report |

**Retirement check:** the binder no longer exists. The auditor
receives the public key + the `kensa verify` invocation; the auditor's
deliverable becomes the chain replay.

---

## Part IV — Calibration discipline during migration

Per VIGGY-OPERATING.md §VI. Migration-specific notes:

| Calibration source | What to compare |
|---|---|
| AnomalyChain (would-have-done) | "the promessa would have AutoCorrected at tick 1247" |
| Actual human action (during shadow window) | "at tick 1247, on-call did X" |

Discrepancies — three categories:

1. **The promessa under-reacts**: legacy human did something the
   promessa wouldn't have. → **Tighten severity** or extend the
   RemediationPolicy action.
2. **The promessa over-reacts**: promessa would have acted but
   humans wisely waited. → **Loosen severity** (with typed
   justification) or change the action.
3. **The promessa would have acted wrong**: would have scaled up when
   the human scaled down. → **Substrate gap.** Don't promote until
   the TargetController + TypedAction is corrected.

The shadow window is **one full reconcile cycle of the legacy
artifact**. For weekly runbooks, 7 days. For quarterly compliance
spreadsheets, 90 days (yes — substantial; matches the audit
cadence).

---

## Part V — The retirement protocol

Once promotion to AutoCorrect succeeds, **retire the legacy artifact
within 90 days**.

### V.1 Mark legacy as superseded

For each retired artifact, prepend the header:

```markdown
> **SUPERSEDED by promessa `<name>` on `<date>`**.
> Live operational truth: `kensa verify outcome-chain --promessa <name>`.
> This document is archived; do not consult for live operations.
> Will be deleted after `<date+90d>`.
```

For dashboards / monitors / spreadsheets: rename / move to an
`archive/` directory or namespace.

### V.2 Cross-link in the promessa

The promessa's `.tatara` source records the migration:

```lisp
(defpromessa lilitu-prod-sla
  …
  :migrated-from { :from "runbook" :url "<legacy-url>" :superseded-at "2026-05-21" }
  …)
```

The first OutcomeReceipt's metadata cites the migration. The chain's
genesis is **the moment the promessa took over** — never back-fill
historical receipts (per R-1.6 in VIGGY-AUTHORING.md).

### V.3 Archive the legacy artifact

| Artifact type | Archive method |
|---|---|
| Runbook PDF / Notion / Confluence | Add SUPERSEDED header; move to `archive/` section after 30 days; delete after 90 |
| Datadog dashboard | Rename to `lilitu-prod-sla-superseded-2026-05`; remove from default visibility; delete after 90 days |
| Splunk alert / Datadog monitor | Disable; delete after 30 days |
| PagerDuty service config | Keep (still routes pages); update service description with link to promessa |
| Cron job | Delete the CronJob CR / system crontab entry |
| Compliance spreadsheet | Add SUPERSEDED header; archive in storage; auditor never sees again |
| Audit binder | Archive; auditor receives `kensa verify` invocation in lieu |

### V.4 Update the team's mental model

| Action | When |
|---|---|
| Document the migration in the cluster's `RUNBOOK.md` under "Migrated to Viggy" section | at retirement |
| Brief the team on the new audit flow (`kensa verify` instead of dashboards) | within 1 week of promotion |
| Update onboarding docs to point to VIGGY-OPERATING.md, not the legacy artifact | within 30 days |

---

## Part VI — Migration anti-patterns

| Anti-pattern | Why forbidden |
|---|---|
| Keeping the legacy artifact "as a backup" | per Compounding #1 — two systems of record = drift |
| Skipping shadow mode for "obvious" migrations | per VIGGY-AUTHORING §8.5 R-8.1 — calibration is the whole point |
| Migrating a runbook that has unresolved typed gaps without filing substrate tickets | per §15.7 — the gap must be acknowledged |
| Modifying the legacy artifact post-migration | confusion; the promessa is the only live truth |
| Promoting to AutoCorrect mid-shadow-window | wait the full window; partial calibration is worse than no calibration |
| Back-filling OutcomeChain receipts from historical observability data | per R-1.6 — receipts are signed assertions at tick T; back-fill compromises chain integrity |
| Migrating a non-Viggy concern by stretching it into a promessa | per VIGGY-AUTHORING §1.5 — peer controller / existing primitive / defer is sometimes the right answer |
| Authoring a `Custom` TargetController kind just for a one-off legacy | per Compounding #1 — three uses, then extract |
| Letting legacy artifacts linger beyond 90 days | retirement is a hard deadline; the chain is the SoR |

---

## Part VII — Migration order: prioritization

Not every legacy artifact migrates at once. Prioritize:

| Tier | Examples | Why first |
|---|---|---|
| **Tier 1** (highest value) | Compliance binders, SOC 2 / PCI evidence; production SLAs; on-call runbooks for prod | Audit-bearing; substantial human time saved; chain-as-SoR replaces error-prone manual evidence |
| **Tier 2** | Cost / FinOps spreadsheets; CVE / Security mailing-list workflows; customer-KPI dashboards | Continuous attestation valuable but cycle is longer |
| **Tier 3** | Dev / staging runbooks; one-off rotations; legacy CronJobs with implicit observations | Lower stakes; can migrate as bandwidth allows |
| **Tier 4** (legitimate skips) | Pure-schedule CronJobs with no observation; one-shot recovery procedures; truly stateless build hooks | Apply `skip-continuous-convergence:` waiver with typed reason |

---

## Part VIII — Closing

Migration is mechanical. Every legacy shape has a typed translation
procedure (Part III). Calibrate via shadow mode. Retire on a
deadline. **The legacy operational surface goes to zero over time;
the substrate carries the work.**

Per the Compounding Directive: solve once, retire forever. **Viggy
is the practice; legacy is the debt being paid down.**

---

> *Extract typed shape. Author. Shadow. Calibrate. Promote. Retire.
> The chain IS the new system of record. **Migrate; don't preserve.***
