# Rate-limited consumers

> **Frame.** [`THEORY.md`](./THEORY.md) §VI.2 (typescape rendering as
> morphism), §V (verification by construction), §VII.3 (JIT capacity).
> [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) (compute hierarchy).
> [`RUNTIME-PATTERNS.md`](./RUNTIME-PATTERNS.md) (caixa-enabled
> patterns). This doc names *one specific compounding pattern* —
> rate-limited external API consumption — and codifies it across
> the typescape's six layers (theory → templates → broker → typed
> primitive → consumer → operator integration). The typescape gains
> the ability to *prove* that a given consumer cannot exceed its
> share of an upstream API budget.

---

## I. The problem statement

Some upstream APIs (GitHub, Datadog, Slack, Stripe, OpenAI) have
**hard rate limits** that, if exceeded, can:

- return errors that disturb downstream correctness (apply paths fail,
  audit logs gap, payments stall);
- trigger **behavioral abuse-detection** that locks the credential for
  hours or days, with no in-band recovery;
- consume **shared budget** with other internal callers using the
  same credential, so one greedy caller starves the rest.

Naive defaults — "set the polling interval lower" — don't compose:

- Each caller has its own per-pod budget; **N pods × per-pod budget**
  silently sums past the upstream cap.
- Per-caller backoff doesn't share knowledge across callers.
- Workstation daemons + cluster operators on the same token race.
- Human-tuned interval drifts as the workspace grows.

The fix is **structural, not procedural**: replace inline dispatch
with a queue, and put exactly *one* rate-limited drain on the
upstream side. The queue absorbs bursts; the drain enforces the
invariant.

This doc is the canonical statement of that pattern in pleme-io.

---

## II. The pattern, named

**Rate-limited consumer** (`samba` in the typescape — Brazilian-Portuguese
for "rhythm / cadence"; the worker dispatches at a fixed rhythm).

```
producers  ──►  JetStream stream  ──►  rate-limited drain  ──►  upstream API
(operators,    (pleme-nats broker,     (samba-based worker:    (GitHub,
 daemons,       per-consumer stream     leaky bucket +          Datadog,
 controllers)   declared by consumer)   adaptive backoff)       Slack, …)
```

Three load-bearing properties:

1. **Decoupled publish.** Producers cannot exceed budget by definition —
   they don't dispatch HTTP, they publish to NATS. The queue absorbs
   any burst, including pathological ones (a misbehaving controller
   that publishes 10 000 jobs in a second).

2. **Single drain.** Exactly one consumer with `MaxAckPending=1`
   (broker-side) plus a `LeakyBucket` (worker-side) gives a fleet-wide
   rate ceiling, regardless of how many producers exist or where they
   run (k8s cluster, workstation daemon, GitHub Actions job).

3. **Adaptive shrinkage.** When the upstream's own counter
   (`X-RateLimit-Remaining` for GitHub) crosses pressure thresholds,
   the worker halves / quarters / eighths its own pace. The fleet
   automatically yields when other callers (humans, third-party
   integrations sharing the token) consume budget.

The result: **the rate-limit invariant becomes a theorem of the
typescape**, not a runtime guess. The `samba` trait + `LeakyBucket`
type + `JetStreamPullWorker` types refuse, by construction, to
dispatch faster than the configured pace.

---

## III. Invariants (the theorems)

| # | Invariant | Where enforced |
|---|---|---|
| 1 | A producer cannot dispatch HTTP to the upstream API directly. | Type signature: producers see `Publisher<U>` not `Client<U>`. |
| 2 | At most one in-flight request per upstream credential at a time. | NATS consumer `MaxAckPending: 1`. |
| 3 | Sustained dispatch rate ≤ `quota_pct × X-RateLimit-Limit / 60` (dynamically tracking the upstream's reported ceiling). | `LeakyBucket` admits one token per `60 / rpm` seconds; rpm updated on every response via `record_observed_limit`. |
| 4 | Upstream budget pressure shrinks the rate. | `pressure_warn_pct` / `pressure_critical_pct` thresholds in worker; halves / quarters / eighths the bucket refill rate. |
| 5 | Wallclock-aligned cron-shaped traffic is forbidden (anti-fingerprint). | `jitter_pct ≥ 0.10` in worker config (rejected at chart-render time if lower). |
| 6 | ETag conditional reads don't consume budget. | Worker forwards `If-None-Match` from the operator's ETag cache; 304 responses don't take a bucket token. |
| 7 | Every dispatch is observable. | `samba_requests_total{outcome,upstream}` counter is mandatory; chart refuses to install without a ServiceMonitor. |
| 8 | Every consumer publishes its own stream + consumer config. | Stream declaration owned by the consumer chart, not the broker (Cilium-style identity). |

Invariants 1–6 are the operational primitives. 7–8 are the
typescape-correctness primitives. All eight follow from the type
choices in `samba` + `pleme-lib.rate-limit-worker.*` templates.

---

## IV. The six compounding layers

Each lower layer is a *load-bearing primitive* the higher layers
stand on. New consumers (the next upstream) live entirely at L4 —
the lower layers never need to be re-implemented.

```
L5 Lisp form        (defrate-limited-consumer …) via #[derive(TataraDomain)]   — aspirational
                    ────────────────────────────────────────────────────────────────────────
L4 Consumer         pleme-tend-throttle (the canonical and currently-only consumer)
                    ~30 LOC `impl UpstreamApi for FooClient` + ~5-line Helm values
                    ────────────────────────────────────────────────────────────────────────
L4' Operator        pleme-tend-operator: fleet_watch + fleet_advance_consumer + nats_throttle
                    Same binary as the consumer (operator + throttle subcommands)
                    ────────────────────────────────────────────────────────────────────────
L3 Rust primitive   samba — UpstreamApi trait, LeakyBucket (dynamic-rate via observed_limit),
                    JetStreamPullWorker, standardized samba_* metrics, HTTP server
                    ────────────────────────────────────────────────────────────────────────
L2 Generic broker   pleme-nats — broker-only chart; streams owned by consumers
                    (Cilium-style identity, not central registry)
                    ────────────────────────────────────────────────────────────────────────
L1 pleme-lib templates   _jetstream_stream.tpl + _rate_limit_worker.tpl
                         + _jetstream_init.tpl + standardized alerts
                         every chart drops in {{ include … }} and gets the whole shape
                         ────────────────────────────────────────────────────────────────
L0 Theory                this doc — names the pattern, lists invariants
                         (zero code, infinite leverage; never re-derived)
```

| Layer | Path | Status |
|---|---|---|
| L0 | `pleme-io/theory/RATE-LIMITED-CONSUMERS.md` | **★ load-bearing** (this doc) |
| L1 | `pleme-io/helmworks/charts/pleme-lib` v0.12.0+ | **★ load-bearing** |
| L2 | `pleme-io/helmworks/charts/pleme-nats` v0.2.x | **★ load-bearing** |
| L3 | `pleme-io/samba` v0.1.0 (dynamic-rate) | **★ load-bearing** |
| L4' | `pleme-io/helmworks/charts/pleme-tend-operator` v0.5.x — fleet_watch + bridge | **★ load-bearing** |
| L4 | `pleme-io/helmworks/charts/pleme-tend-throttle` v0.3.x | **★ load-bearing** |
| L5 | `(defrate-limited-consumer …)` via `#[derive(TataraDomain)]` | **○ aspirational** |

### IV.1 The single load-bearing knob: `quota_pct` × observed limit

samba's `LeakyBucket` derives its admission rate dynamically from
**what the upstream reports**, not from a hardcoded constant:

```text
target_rpm = quota_pct × X-RateLimit-Limit / 60
```

`X-RateLimit-Limit` is observed on every response and stashed via
`LeakyBucket::record_observed_limit`. So `quotaPct: 0.01` ("use 1%
of the upstream's reported quota") tracks the credential's actual
ceiling. For GitHub:

| Token type | GitHub limit | quota_pct=0.01 → rpm |
|---|---|---|
| Classic / fine-grained PAT | 5000/hr | 0.83 (period ~72s) |
| GitHub App installation | 15000/hr | 2.5 (period ~24s) |
| GitHub Enterprise custom | varies | derived from observed |

Cold start uses `INITIAL_BUDGET_PER_HOUR` from the trait const as a
fallback; first response with the header re-rates the bucket.

### IV.2 Continuous fleet-wide watching: fleet_watch + bridge

The operator (L4') runs a `fleet_watch` task alongside the
FlakeUpdatePolicy reconciler. It:

1. Periodically (default 1h) enumerates the configured GitHub org
   via `list_repos` API. **Auto-discovers new repos** as they're
   created.
2. Filters to repos that have a `flake.nix` at HEAD.
3. Runs a round-robin scheduler ticking at the throttle's drain rate.
   Each tick publishes one refresh-job through the NATS throttle.
4. Subscribes to `tend.github.jobs.results.>` and tracks per-repo
   state. Detects advances by comparing `result.head.info.upstream_rev`
   to its last-seen rev.
5. On advance, publishes a typed `FleetAdvanceEvent` to
   `tend.fleet.advances.<sanitized-key>`.

A separate `fleet_advance_consumer` task subscribes to
`tend.fleet.advances.>` and **annotates every enrolled
FlakeUpdatePolicy CR** with `tend.pleme.io/last-fleet-advance`. The
annotation change triggers an immediate kube-rs Controller reconcile
→ discovery hits the hot ETag cache → Proposal lands within seconds.

For Auto-mode inputs (e.g. `fenix`, `kindling`): fleet-wide upstream
advance → proposal → gates → flake.lock updated + git pushed by the
operator within seconds.

---

## V. How to add a new consumer (operator playbook)

Adding `pleme-datadog-throttle` once `samba` is published:

1. **Rust binary** (~30 LOC). In whatever crate owns Datadog integration:
   ```rust
   use samba::{UpstreamApi, LeakyBucket, JetStreamPullWorker};

   struct DatadogApi { client: reqwest::Client, base: String }
   impl UpstreamApi for DatadogApi {
       const NAME: &'static str = "datadog";
       const BUDGET_PER_HOUR: u32 = 30_000;  // Datadog Pro tier
       type Request = DatadogRequest; type Response = DatadogResponse; type Error = ...;
       async fn dispatch(&self, req: Self::Request) -> Result<Self::Response, Self::Error> { ... }
       fn rate_limit_remaining(&self, r: &Self::Response) -> Option<u32> { ... }
   }

   #[tokio::main]
   async fn main() -> Result<()> {
       let cfg = samba::Config::from_env()?;  // reads /etc/datadog-throttle/config.yaml
       JetStreamPullWorker::new(DatadogApi::new(&cfg.api_key), cfg).run().await
   }
   ```

2. **Helm chart** (5 lines). Copy `pleme-tend-throttle/values.yaml`,
   substitute three values:
   ```yaml
   image: { repository: ghcr.io/pleme-io/datadog-throttle, ... }
   rateLimit: { requestsPerMinute: 50 }     # 10% of 30000/hr Datadog cap
   nats:      { stream: DATADOG_JOBS, consumer: datadog-throttle }
   ```
   Templates are one-line `{{ include "pleme-lib.rate-limit-worker.deployment" . }}` etc.

3. **HelmRelease** in `k8s/clusters/<cluster>/infrastructure/datadog-throttle/`
   referencing the published chart with cluster-specific overrides.

4. **Producers** in operators / controllers replace `client.post(...)`
   with `nats.publish("datadog.jobs.<key>", ...)` + result subscription.

That's it. Layers L0–L3 are unchanged; L1's templates absorb every
boilerplate concern (Deployment shape, NetworkPolicy, ServiceMonitor,
PrometheusRule). L4's specific chart is values-only; the lib carries
the structure.

---

## VI. Anti-patterns this pattern forbids

- **Per-caller rate budget**, summed silently. *Fix:* one shared
  drain.
- **Shell-based rate limiting** (`sleep N` between API calls in a bash
  loop). *Fix:* typed `LeakyBucket` in `samba`.
- **Hand-rolled NATS code in every consumer.** *Fix:* `JetStreamPullWorker<U>`
  is the one shape; consumers parameterize, not re-implement.
- **Streams declared centrally in pleme-nats.** *Fix:* consumer owns
  its stream declaration via `pleme-lib.jetstream-stream` template
  (Cilium-style identity).
- **Per-consumer alert rules drifting in spelling / threshold.** *Fix:*
  `pleme-lib.rate-limit-worker.alerts` template is the single source.
- **"Just bump the per-pod budget"** in response to growth. *Fix:* if
  the queue is growing, lower the pace and investigate which
  producer is publishing more — the rate is the budget; no internal
  knob can lift it past the typescape's invariant.

---

## VII. Relationship to other typescape primitives

| Composes with | How |
|---|---|
| **caixa** ([CAIXA-SDLC.md](./CAIXA-SDLC.md)) | a rate-limited consumer can be authored as `(defcaixa :kind Servico …)` once L5 lands; `samba` is its runtime contract. |
| **breathability** ([BREATHABILITY.md](./BREATHABILITY.md)) | the broker (`pleme-nats`) can scale-to-zero via KEDA when no producers are publishing; the worker stays at 1 (the invariant requires it). |
| **typescape** ([THEORY.md §III](./THEORY.md)) | `samba::UpstreamApi` trait IS the typescape Upstream node; `LeakyBucket` IS the Pacing node; both BLAKE3-attestable via tameshi. |
| **observability** ([THEORY.md §VII](./THEORY.md)) | `samba_requests_total`, `samba_pace_factor`, `samba_rate_limit_remaining` standardize the metric surface — Pangea-rendered Grafana panels work for every consumer with no per-instance tweaking. |
| **tameshi attestation** ([THEORY.md §V](./THEORY.md)) | the worker's config (rate, pressure thresholds, stream names) is content-addressed; rotation = swap the config and the BLAKE3 root, no code changes. |
| **fleet hostname pattern** (CLAUDE.md) | each broker / worker exposes its metrics endpoint at `<app>.<cluster>.<location>.quero.cloud` — consumers find each other by typed name, not IP. |

---

## VIII. Open questions / future moves

- **Cross-cluster sharing.** Today each cluster has its own pleme-nats;
  fleet-wide budget needs leafnode federation or a single SaaS broker.
  M3 deliverable; not load-bearing yet.
- **JetStream KV for ETag cache.** Migrating tend-operator's per-pod
  ETag file cache to a fleet-shared KV bucket (`tend.etag.v1`) lets
  workstation daemons + cluster operators share 304 wins. Schema-shaped
  in `samba`'s scaffold; impl pending.
- **Pangea Ruby cross-cloud rendering.** A `pangea-rate-limited-consumer`
  resource that synthesizes K8s+NATS+chart on rio AND AWS
  Lambda+EventBridge+SQS on saas clusters from one Ruby declaration.
  Pangea-side render trait absent today.
- **Lisp authoring surface (L5).** `(defrate-limited-consumer
  :upstream :github :budget {:per-hour 5000 :pct 10} …)` via a new
  `TataraDomain` derive. Compresses L4's 5-line Helm + 30 LOC Rust
  into one declaration; future renderers emit both.

These are the next compounding moves on top of this pattern. Each
landed move is a new theorem the substrate gains, never re-derived.
