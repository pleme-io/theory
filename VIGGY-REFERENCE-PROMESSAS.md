# VIGGY-REFERENCE-PROMESSAS — one canonical example per kind

> **Thesis.** This doc is the **starter template catalog** for the
> five canonical PromessaTarget kinds. An operator authoring a new
> promessa lifts the closest match, edits the typed slots, and ships.
> Per VIGGY-AUTHORING.md §16 roadmap item #6.
>
> Each kind below carries: source `.tatara` block, rendered YAML CR,
> materialization output (the `/viggy` skill's Mode A output template),
> and a representative tick trace. **No prose; lift and adapt.**

Companion docs:

- [`VIGGY-AUTHORING.md`](./VIGGY-AUTHORING.md) — the typed authoring discipline
- [`VIGGY-OPERATING.md`](./VIGGY-OPERATING.md) — daily operator playbook
- [`CONTINUOUS-SOLUTION-MACHINE.md`](./CONTINUOUS-SOLUTION-MACHINE.md) §V.4 — the five canonical kinds

---

## Part I — SLA promessa (`lilitu-prod-sla`)

### I.1 `.tatara` source

```lisp
;; lilitu/promessas/prod-sla.tatara

(defpromessa lilitu-prod-sla
  :namespace        lilitu-prod
  :scope            (services [api-v1 api-v2 api-v3])
  :target           (sla
                      :availability  "99.99%"
                      :latency-p99   "200ms"
                      :latency-p50   "50ms"
                      :error-rate    "0.01%")
  :window           "30d"
  :observation      (shinryu
                      :query "SELECT
                                avg(availability) AS availability,
                                quantile(latency_ms, 0.99) AS p99,
                                quantile(latency_ms, 0.50) AS p50,
                                sum(errors) / sum(requests) AS error_rate
                              FROM events
                              WHERE service IN (?services)
                                AND ts > now() - INTERVAL '30 days'"
                      :bindings { :services [api-v1 api-v2 api-v3] }
                      :materialize SlaSnapshot)
  :reconcile-every  "60s"
  :remediation      { :on-cosmetic   (alert :sink slack-forge:#lilitu-ops)
                      :on-functional (auto-correct
                                       :action (flux-commit
                                                 :path "k8s/clusters/lilitu/apps/api/values.yaml"
                                                 :patch (json-patch
                                                          { :op "replace"
                                                            :path "/replicaCount"
                                                            :value-fn (+ current 1) })))
                      :on-critical   (compose
                                       [(auto-correct
                                          :action (flux-commit
                                                    :path "k8s/clusters/lilitu/apps/api/values.yaml"
                                                    :patch (json-patch
                                                             { :op "replace"
                                                               :path "/replicaCount"
                                                               :value-fn (+ current 2) })))
                                        escalate]) }
  :escalation       (ladder
                      :name "sla-default"
                      :step1 (controller :timeout "5m"  :retry-with same-action)
                      :step2 (on-call    :timeout "15m" :who pagerduty:lilitu-prod)
                      :step3 (manager    :timeout "1h"  :who pagerduty:eng-manager)
                      :step4 (exec       :timeout "4h"  :who pagerduty:exec))
  :dependencies     [{ :to lilitu-prod-cost-budget :op meet }
                     { :to lilitu-prod-compliance  :op observe }])
```

### I.2 Rendered YAML CR (committed under `k8s/clusters/lilitu/promessas/`)

```yaml
apiVersion: viggy.pleme.io/v1alpha1
kind: Promessa
metadata:
  name: lilitu-prod-sla
  namespace: lilitu-prod
  labels:
    app.kubernetes.io/part-of: viggy
    app.kubernetes.io/name: lilitu-prod-sla
    viggy.pleme.io/kind: sla
spec:
  scope:
    services: [api-v1, api-v2, api-v3]
  target:
    kind: sla
    availability: 99.99%
    latencyP99: 200ms
    latencyP50: 50ms
    errorRate: 0.01%
  window: 30d
  observation:
    source: shinryu
    query: |
      SELECT avg(availability) AS availability,
             quantile(latency_ms, 0.99) AS p99,
             quantile(latency_ms, 0.50) AS p50,
             sum(errors) / sum(requests) AS error_rate
      FROM events
      WHERE service IN (?services)
        AND ts > now() - INTERVAL '30 days'
    bindings:
      services: [api-v1, api-v2, api-v3]
    materialize: SlaSnapshot
  reconcileEvery: 60s
  remediation:
    onCosmetic:
      kind: alert
      sink: slack-forge:#lilitu-ops
    onFunctional:
      kind: autoCorrect
      action:
        kind: fluxCommit
        path: k8s/clusters/lilitu/apps/api/values.yaml
        patch:
          - op: replace
            path: /replicaCount
            valueFn: (+ current 1)
    onCritical:
      kind: compose
      ops:
        - { kind: autoCorrect, action: { kind: fluxCommit, …, valueFn: (+ current 2) } }
        - { kind: escalate }
  escalation:
    name: sla-default
    steps:
      - { kind: controller, timeout: 5m,  retryWith: sameAction }
      - { kind: onCall,    timeout: 15m, who: pagerduty:lilitu-prod }
      - { kind: manager,   timeout: 1h,  who: pagerduty:eng-manager }
      - { kind: exec,      timeout: 4h,  who: pagerduty:exec }
  dependencies:
    - { to: lilitu-prod-cost-budget, op: meet }
    - { to: lilitu-prod-compliance,  op: observe }
```

### I.3 Materialization output (`/viggy` Mode A output)

```
═══════════════════════════════════════════════════════════════════
VIGGY MATERIALIZATION — lilitu-prod-sla
═══════════════════════════════════════════════════════════════════

§1 — Diagnostic
   Q1: ✓  Q2: ✓ (SLA — customer commitment)
   Q3: ✓ (shinryu metrics)  Q4: ✓ (customer contract SLA)
   Q5: SLA
   → AuthoringDecision::Promessa(SLA)

§2 — Granularity
   Commitment subject: lilitu prod API services
   Scope:              services [api-v1 api-v2 api-v3]
   Window:             30d rolling

§3 — Existing primitives
   Citing: caixa Servicos `api-v1/v2/v3` :limits.latency declarations
   Wrapping: observes shinryu metrics emitted by those Servicos
   No new TargetController needed (uses shipping SlaController)

§4 — Severity (defaults)
   Cosmetic:   p99 > 1.2× target (240ms+)
   Functional: p99 > 1.5× target (300ms+) OR availability < target − 0.001%
   Critical:   p99 > 2× target (400ms+) OR availability < target − 0.01%

§5 — Actions per severity
   on-cosmetic:   Alert (slack-forge:#lilitu-ops)
   on-functional: AutoCorrect(FluxCommit replicaCount + 1)
   on-critical:   Compose([AutoCorrect(FluxCommit replicaCount + 2), Escalate])

§6 — Composition
   Dependencies:
     - lilitu-prod-cost-budget :meet  (don't scale up if budget breached)
     - lilitu-prod-compliance  :observe (record posture in receipts)

§7 — Cadence
   reconcile_every: 60s (SLA default)
   sampling:        none

§8 — Lifecycle
   Shadow mode: 7 days (one week of traffic patterns)
   Spec-revision strategy: tighten-in-place

§9 — Failure policy
   EscalationLadder: controller(5m) → pagerduty(15m) → manager(1h) → exec(4h)

§11 — Gaps
   None — every ambiguity auto-resolved via prime directives.

───────────────────────────────────────────────────────────────────
RENDERED PRIMITIVES
───────────────────────────────────────────────────────────────────

# File 1: lilitu/promessas/prod-sla.tatara
<see §I.1>

# File 2: k8s/clusters/lilitu/promessas/lilitu-prod-sla.yaml
<see §I.2>

───────────────────────────────────────────────────────────────────
SUBSTRATE TOUCHES
───────────────────────────────────────────────────────────────────
  TargetController: SlaController (shipping in engenho-promessa-controllers)
  ObservationSource: Shinryu (existing query infrastructure)
  TypedAction: FluxCommit (existing via GithubRepoReconciler)
  Reconciler dispatch: GithubRepoReconciler @ pleme-io/k8s (shipping)
  PromessaDependency Controller: dependency-controller (engenho)
  
  Existing primitive citations (no new code):
  - caixa Servicos api-v1/v2/v3 (already deployed via caixa-publish)

───────────────────────────────────────────────────────────────────
EPHEMERAL PREVIEW
───────────────────────────────────────────────────────────────────
  kenshi env up viggy-preview-lilitu-prod-sla
  kubectl apply -f promessas/lilitu-prod-sla.yaml
  (mock 10 observations → verify AnomalyChain integrity → tear down)

───────────────────────────────────────────────────────────────────
PR DESCRIPTION DRAFT
───────────────────────────────────────────────────────────────────
Title:  feat(viggy): lilitu-prod-sla — 99.99% availability / p99 < 200ms / 30d rolling

Body:
  ## Summary
  Authors the lilitu prod API SLA promessa attesting 99.99%
  availability and p99 < 200ms over a 30-day rolling window.
  Auto-corrects via FluxCommit replica-count bumps; escalates per
  the SLA EscalationLadder on Critical.

  ## Commitment cite
  - Customer contract v3 §4.2 — production API SLO
  - lilitu internal SLO doc §"production tier"

  ## Substrate touches
  - SlaController (shipping)
  - shinryu query against existing metrics pipeline
  - FluxCommit → GithubRepoReconciler → k8s/clusters/lilitu/apps/api/values.yaml
  - PromessaDependency Meet edge to lilitu-prod-cost-budget

  ## Test plan
  - [x] trait_laws_obeyed!(SlaController) passes
  - [x] Golden trajectory test matches
  - [x] kind-cluster integration: 100 ticks clean
  - [ ] Shadow mode for 7 days after merge
  - [ ] Promote to AutoCorrect once calibrated
```

### I.4 Representative tick trace (after a load spike)

```
T+00:00:00  observe → SlaSnapshot { availability=0.9991, p99=242ms, p50=58ms, errors=0.00012 }
            diff    → SlaDrift { availability_delta=-0.0008, p99_ratio=1.21, … }
            classify→ Functional (p99 1.21× target → just past Cosmetic threshold; classify Functional per latency edge)
            decide  → AutoCorrect(FluxCommit replicaCount: 5 → 6)
            dep_chk → cost-budget Cosmetic ✓
            act     → commit pushed @ blake3:af3c…; FluxCD reconciling
            attest  → OutcomeReceipt { tick=1247, sev=Functional, dec=AutoCorrect, out=Committed }

T+00:01:00  observe → SlaSnapshot { availability=0.9994, p99=215ms, … }    ; capacity arrived
            diff    → SlaDrift { p99_ratio=1.075 }
            classify→ Cosmetic
            decide  → Alert
            act     → Anomalia emission on `anomaly.lilitu.lilitu-prod-sla.cosmetic`
            attest  → OutcomeReceipt { tick=1248, sev=Cosmetic, dec=Alert, out=Anomalia }

T+00:05:00  observe → SlaSnapshot { availability=0.9999, p99=187ms, … }    ; converged
            diff    → SlaDrift::Empty
            classify→ Cosmetic (no drift)
            decide  → Noop
            attest  → OutcomeReceipt { tick=1252, sev=Cosmetic, dec=Noop, out=Empty }
```

---

## Part II — CostBudget promessa (`lilitu-prod-cost-budget`)

### II.1 `.tatara` source

```lisp
;; lilitu/promessas/prod-cost-budget.tatara

(defpromessa lilitu-prod-cost-budget
  :namespace        lilitu-prod
  :scope            (namespace lilitu-prod)
  :target           (cost-budget
                      :max-spend       "$5000"
                      :period          monthly
                      :on-overspend    scale-down-non-critical)
  :window           "calendar-month"
  :observation      (shinryu
                      :query "SELECT sum(cost_usd) AS spend
                              FROM events
                              WHERE namespace = ?ns
                                AND ts > date_trunc('month', now())"
                      :bindings { :ns "lilitu-prod" }
                      :materialize CostSnapshot)
  :reconcile-every  "5m"
  :remediation      { :on-cosmetic   (alert :sink slack-forge:#finops)
                      :on-functional (auto-correct
                                       :action (flux-commit
                                                 :path "k8s/clusters/lilitu/apps/non-critical/values.yaml"
                                                 :patch (json-patch
                                                          { :op "replace"
                                                            :path "/replicaCount"
                                                            :value-fn (max 1 (- current 1)) })))
                      :on-critical   escalate }
  :escalation       (ladder
                      :name "cost-default"
                      :step1 (controller :timeout "10m" :retry-with magma-right-size)
                      :step2 (on-call    :timeout "30m" :who pagerduty:finops)
                      :step3 (manager    :timeout "2h"  :who pagerduty:cfo))
  :dependencies     [])
```

### II.2 Notes on typed slots

- `:scope (namespace lilitu-prod)` — encompasses all workloads in the namespace; matches FinOps's "per-namespace" billing view.
- `:window "calendar-month"` — billing cycle alignment; reconcile cadence is `5m` but the budget is monthly.
- `:reconcile-every 5m` — cost data lags; minute-level reconcile wasted (per VIGGY-AUTHORING §7.1).
- `:dependencies []` — CostBudget is typically a *dependee*, not a *depender*. SLA promessas have it as a Meet edge.

### II.3 Materialization output (compressed)

```
§1 ✓ Promessa(CostBudget)  §2 namespace=lilitu-prod, window=calendar-month, kind=CostBudget
§3 cite shinryu CUR pipeline; no wrap (cost data already typed)
§4 default thresholds (10% / 25% / 50%+ over budget pace)
§5 actions: alert / scale-down-non-critical / escalate-FinOps-ladder
§6 no dependencies (CostBudget is a Meet target for others)
§7 5m cadence (CostBudget default)
§8 shadow mode 30 days (one billing period)
§9 EscalationLadder: controller(10m) → finops-pager(30m) → cfo-pager(2h)
§11 no gaps
§12 materialized
```

### II.4 Tick trace (excerpt — month-end overrun)

```
2026-02-28 12:00  observe → CostSnapshot { spend=$4200, days_elapsed=28, days_in_month=28 }
                  diff    → CostDrift { burn_rate=$4200/$4929=85% projection of monthly budget }
                  classify→ Cosmetic (within 10% of pace)
                  decide  → Alert (FinOps)
                  attest  → tick

2026-02-28 18:00  observe → CostSnapshot { spend=$4750 }
                  diff    → 95% of monthly budget
                  classify→ Functional (5% over budget pace projection)
                  decide  → AutoCorrect(scale-down non-critical-workload replicas by 1)
                  dep_chk → no dependencies
                  act     → commit pushed; flux reconciles
                  attest  → tick
```

---

## Part III — Compliance promessa (`lilitu-regulated-compliance`)

### III.1 `.tatara` source

```lisp
;; lilitu/promessas/regulated-compliance.tatara

(defpromessa lilitu-regulated-compliance
  :namespace        lilitu-regulated
  :scope            (namespace lilitu-regulated)
  :target           (compliance
                      :baseline      pci-dss-4-0
                      :on-violation  quarantine-workload)
  :window           "rolling"
  :observation      (kensa-project
                      :baseline pci-dss-4-0
                      :scope    (namespace lilitu-regulated))
  :reconcile-every  "15m"
  :remediation      { :on-cosmetic   (alert :sink slack-forge:#compliance)
                      :on-functional (auto-correct
                                       :action (reconciler-apply
                                                 :kind "GithubRepoReconciler"
                                                 :config { :repo "pleme-io/lilitu"
                                                           :branch-protection-required true
                                                           :branch-protection-strict true }))
                      :on-critical   (compose
                                       [(auto-correct
                                          :action (cracha-patch
                                                    :policy lilitu-regulated-access
                                                    :patch (json-patch
                                                             { :op "add"
                                                               :path "/spec/quarantine"
                                                               :value true })))
                                        escalate]) }
  :escalation       (ladder
                      :name "compliance-default"
                      :step1 (controller :timeout "5m"  :retry-with same-action)
                      :step2 (on-call    :timeout "15m" :who pagerduty:compliance))
  :dependencies     [])
```

### III.2 Materialization output (compressed)

```
§1 ✓ Promessa(Compliance)
§2 namespace lilitu-regulated, rolling window, baseline pci-dss-4-0
§3 cite kensa control map (no wrap of existing primitive); observe via kensa-project
§4 defaults: any non-critical failing → Functional; any critical control failing → Critical
§5 actions: alert / reconciler-apply (branch protection) / cracha-patch (quarantine) + escalate
§6 no dependencies — Compliance is a Meet target for stricter promessas
§7 15m cadence (Compliance default; posture changes propagate slowly)
§8 shadow mode 30 days (one audit cycle)
§9 EscalationLadder: controller(5m) → compliance-pager(15m); no manager/exec (compliance is auditor-routed)
§11 no gaps
```

### III.3 Audit output

```bash
$ kensa verify outcome-chain \
    --promessa lilitu-regulated-compliance \
    --window 2026-01-01..2026-03-31 \
    --baseline pci-dss-4.0

{
  "promessa": "lilitu-regulated-compliance",
  "window":   { "start": "2026-01-01T00:00:00Z", "end": "2026-03-31T23:59:59Z" },
  "baseline": "pci-dss-4.0",
  "ticks":    8640,
  "ticks_critical":   0,
  "ticks_functional": 3,
  "ticks_cosmetic":   8637,
  "outcome": "HELD",
  "remediations": [
    { "tick": 2341, "control": "PCI-DSS-4.0-3.5.1", "auto_corrected_at": "2026-01-29T14:22:00Z", "action": "GithubRepoReconciler force branch-protection" },
    { "tick": 5012, "control": "PCI-DSS-4.0-7.2.3", "auto_corrected_at": "2026-02-19T09:15:00Z", "action": "GithubRepoReconciler" },
    { "tick": 7234, "control": "PCI-DSS-4.0-1.4.5", "auto_corrected_at": "2026-03-12T03:48:00Z", "action": "GithubRepoReconciler" }
  ],
  "chain_root":  "blake3:af3c…",
  "signature":   "ed25519:…",
  "signer":      "lilitu-prod-cluster-attestor"
}
```

**Auditor's deliverable.** Hand to PCI assessor + the public verification
key. The chain replays; the proof emerges.

---

## Part IV — CustomerKpi promessa (`lilitu-product-nps`)

### IV.1 `.tatara` source

```lisp
;; lilitu/promessas/product-nps.tatara

(defpromessa lilitu-product-nps
  :namespace        lilitu-product
  :scope            (product lilitu)
  :target           (customer-kpi
                      :metric   nps
                      :minimum  50)
  :window           "rolling-90d"
  :observation      (http-endpoint
                      :url      "https://api.mixpanel.com/api/2.0/insights"
                      :auth     (cofre-secret-ref "mixpanel-api-key")
                      :params   { :project   "lilitu-prod"
                                  :insight   "nps-rolling-90d" }
                      :parse-as json
                      :materialize NpsSnapshot)
  :reconcile-every  "1h"
  :remediation      { :on-cosmetic   (alert :sink slack-forge:#product)
                      :on-functional (require-approval
                                       :approver-group product-management
                                       :approval-via github-issue)
                      :on-critical   (compose
                                       [(auto-correct
                                          :action (flux-commit
                                                    :path "k8s/clusters/lilitu/apps/api/values.yaml"
                                                    :patch (json-patch
                                                             { :op "replace"
                                                               :path "/featureFlags/recent-rollouts"
                                                               :value false })))
                                        escalate]) }
  :escalation       (ladder
                      :name "customer-kpi-default"
                      :step1 (controller :timeout "1h" :retry-with same-action)
                      :step2 (manager    :timeout "1d" :who pagerduty:product-mgr)
                      :step3 (exec       :timeout "3d" :who pagerduty:exec))
  :dependencies     [])
```

### IV.2 Notes on typed slots

- `:on-functional require-approval` — KPI drops are sensitive; auto-rollback is gated.
- `:on-critical auto-correct` — explicit rollback of feature flags + escalation.
- `cofre-secret-ref` — Mixpanel API key never touches the operator's hands.
- `:reconcile-every "1h"` — NPS data aggregates over hours; minute-level wasted.

### IV.3 Materialization output (compressed)

```
§1 ✓ Promessa(CustomerKpi)
§2 product lilitu, rolling 90d, metric nps minimum 50
§3 cite Mixpanel; wrap via HttpEndpoint ObservationSource + cofre secret
§4 defaults: within 5% / 5-15% below / >15% below
§5 actions: alert / require-approval (PM) / auto-rollback + escalate
§6 no dependencies
§7 1h cadence (CustomerKpi default)
§8 shadow mode 30 days
§9 EscalationLadder: controller(1h) → product-mgr(1d) → exec(3d)
§11 no gaps
```

### IV.4 Sample tick trace (NPS dropping)

```
2026-03-15 09:00  observe → NpsSnapshot { nps=52, sample=5421 }
                  classify→ Cosmetic (within 5% of minimum)
                  decide  → Alert (#product)
                  attest  → tick

2026-03-20 14:00  observe → NpsSnapshot { nps=45 }
                  classify→ Functional (5-15% below)
                  decide  → RequireApproval (open GitHub issue for product-management)
                  act     → issue opened: #4521; controller blocks pending approval
                  attest  → tick (decision=PendingApproval)

2026-03-22 10:00  approval received from product-management
                  controller proceeds with previously declared action
                  → reconciles
```

---

## Part V — Security promessa (`lilitu-prod-security`)

### V.1 `.tatara` source

```lisp
;; lilitu/promessas/prod-security.tatara

(defpromessa lilitu-prod-security
  :namespace        lilitu-prod
  :scope            (namespace lilitu-prod)
  :target           (security
                      :max-cve-age { :critical "24h" :high "7d" :medium "30d" }
                      :banned-packages [openssl-1.0.* log4j-1.* spring-core-4.*])
  :window           "rolling"
  :observation      (composite
                      [(shinryu
                         :query "SELECT cve_id, severity, image, max(detected_at) AS detected_at
                                 FROM events
                                 WHERE kind = 'cve-detected'
                                   AND namespace = ?ns
                                 GROUP BY cve_id, severity, image"
                         :bindings { :ns "lilitu-prod" }
                         :materialize CveSnapshot)
                       (kanshi-runtime
                         :scope (namespace lilitu-prod)
                         :events [banned-package-loaded])])
  :reconcile-every  "15m"
  :remediation      { :on-cosmetic   (alert :sink slack-forge:#security)
                      :on-functional (auto-correct
                                       :action (flux-commit
                                                 :path "k8s/clusters/lilitu/apps/api/values.yaml"
                                                 :patch (json-patch
                                                          { :op "replace"
                                                            :path "/image/tag"
                                                            :value-fn latest-patched-tag })))
                      :on-critical   (compose
                                       [(auto-correct
                                          :action (cracha-patch
                                                    :policy lilitu-prod-access
                                                    :patch (json-patch
                                                             { :op "add"
                                                               :path "/spec/quarantine"
                                                               :value true })))
                                        escalate]) }
  :escalation       (ladder
                      :name "security-default"
                      :step1 (controller :timeout "10m" :retry-with renovate-bump)
                      :step2 (on-call    :timeout "30m" :who pagerduty:security)
                      :step3 (manager    :timeout "2h"  :who pagerduty:ciso))
  :dependencies     [])
```

### V.2 Notes on typed slots

- `:observation composite` — joins shinryu CVE feed table with kanshi
  eBPF runtime events. Both flow into the same `Snapshot` shape.
- `:on-functional auto-correct` — pins to a patched image via FluxCommit.
- `:on-critical compose [crachá-patch, escalate]` — quarantines the workload
  AND escalates simultaneously.
- `:retry-with renovate-bump` — first controller retry bumps the
  dependency via the Renovate Reconciler.

### V.3 Materialization output (compressed)

```
§1 ✓ Promessa(Security)
§2 namespace lilitu-prod, rolling window, max-cve-age + banned-packages
§3 cite shinryu CVE feed + kanshi runtime; composite ObservationSource
§4 default: high-sev > 7d = Functional; critical > 24h = Critical
§5 actions: alert / FluxCommit image-pin / cracha-patch quarantine + escalate
§6 no dependencies
§7 15m cadence (Security default)
§8 shadow mode 30 days
§9 EscalationLadder: controller(10m, retry-with-renovate) → security(30m) → ciso(2h)
§11 no gaps
```

### V.4 Tick trace (Critical CVE detected)

```
2026-04-12 08:30  observe → CveSnapshot { 1 critical CVE detected 25h ago in image api:v1.4.2 }
                  diff    → SecurityDrift { critical_cve_aged_beyond_24h: 1 }
                  classify→ Critical
                  decide  → Compose([
                              AutoCorrect(CrachaPatch(lilitu-prod-access + quarantine: true)),
                              Escalate(security-default-ladder)
                            ])
                  act     → crachá patched; quarantine in effect
                            EscalationLadder Step 1 fires: controller retry with renovate-bump
                  attest  → tick

2026-04-12 08:40  EscalationLadder Step 1 timeout (10m) — renovate-bump didn't resolve
                  Step 2 fires: pagerduty:security paged

2026-04-12 09:15  on-call manually bumps to api:v1.4.3 (patched)
                  observe → CveSnapshot { 0 critical CVEs }
                  diff    → Empty
                  classify→ Cosmetic
                  decide  → Noop (and unquarantine via separate spec change)
                  attest  → tick
```

---

## Part VI — Composition example (SLA `Meet` CostBudget)

The canonical worked composition. SLA promessa from §I depends on
CostBudget promessa from §II via `Meet`.

### VI.1 Behavior at runtime

```
Scenario: SLA p99 spike at 14:00; cost-budget is currently at 95% of monthly budget (Functional severity).

T+00:00:00  [SlaController tick]
            observe → SlaSnapshot { p99=240ms }
            diff    → SlaDrift { p99_ratio=1.20 }
            classify→ Cosmetic (borderline)
            decide  → Alert (per :on-cosmetic policy)
            dep_chk → cost-budget Functional (95% of budget)
            act     → emit Anomalia (DependencyDegraded { dependency: cost-budget })
                      DO NOT scale up — SLA Meet CostBudget invariant blocks
            attest  → tick

T+00:01:00  [SlaController tick]
            observe → SlaSnapshot { p99=265ms }
            classify→ Functional
            decide  → AutoCorrect(FluxCommit scale-up by 1)
            dep_chk → cost-budget STILL Functional → emit DependencyBlocked anomaly
            act     → DO NOT commit scale-up
                      emit Anomalia { kind: DependencyBlocked }
                      escalates to on-call (per cost-budget's escalation, since SLA can't autocorrect)
            attest  → tick (decision=DependencyBlocked)

T+00:05:00  [on-call evaluates] either:
            (a) commits a typed Override raising the CostBudget temporarily (with waiver), OR
            (b) accepts the SLA breach for now, files substrate-improvement ticket
```

The PromessaLattice + PromessaDependency turn what would have been
**two controllers acting independently and amplifying a budget
breach** into **a typed cross-promessa interaction** with a clear
escalation path.

---

## Part VII — Adoption guide

To use these references when authoring a new promessa:

1. **Pick the closest match** by PromessaTarget kind.
2. **Copy the `.tatara` source** into `<repo>/promessas/<new-name>.tatara`.
3. **Edit the typed slots** for your scope / target / observation /
   action / escalation.
4. **Confirm via §1 diagnostic** (`/viggy diagnose <concern>`).
5. **Materialize** via `/viggy <natural-language declaration>` —
   the skill auto-validates against the reference shape.
6. **Ship in shadow mode** for the recommended window (§8.3 + this
   doc's per-kind notes).
7. **Calibrate; promote**.

Per VIGGY-AUTHORING.md §16: reference promessas are typed templates,
not running infrastructure. Adapt them; don't copy them verbatim
without editing the scope to match your commitment.

---

## Part VIII — Closing

Five canonical kinds. Each carries `(defpromessa …)` source +
materialization output + tick trace. Lift; edit; ship. The substrate
provides; the operator declares.

The compounding payoff: every new promessa joins a chain that's
typed, attested, replayable, and audit-ready by construction. **The
template catalog widens with every new typed kind we add.**

---

> *Five kinds; one shape; the substrate produces everything else.
> **The chain IS the proof.***
