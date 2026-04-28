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
| 3 | Sustained dispatch rate ≤ `requests_per_minute`. | `LeakyBucket` admits one token per `60 / rpm` seconds. |
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
stand on. New consumers (the next throttle) live entirely at L4 — the
lower layers never need to be re-implemented.

```
L5 Operator integration  (defrate-limited-consumer …) Lisp form via #[derive(TataraDomain)]
                         ────────────────────────────────────────────────────────────────────
L4 Specific consumers    pleme-tend-throttle, pleme-datadog-throttle (future), …
                         5-line Helm wrapper + ~30 LOC `impl UpstreamApi for FooClient`
                         ────────────────────────────────────────────────────────────────────
L3 Typed Rust primitive  samba — UpstreamApi trait, LeakyBucket, JetStreamPullWorker
                         crates.io library; binaries depend, instantiate, run.
                         ────────────────────────────────────────────────────────────────────
L2 Generic broker        pleme-nats — broker-only chart; streams owned by consumers
                         (Cilium-style identity, not central registry)
                         ────────────────────────────────────────────────────────────────────
L1 pleme-lib templates   _jetstream_stream.tpl, _jetstream_init.tpl,
                         _rate_limit_worker.tpl, _rate_limit_alerts.tpl
                         every chart drops in {{ include … }} and gets the shape
                         ────────────────────────────────────────────────────────────────────
L0 Theory                this doc — names the pattern, lists invariants
                         (zero code, infinite leverage; never re-derived)
```

| Layer | Repo / path | Status |
|---|---|---|
| L0 | `pleme-io/theory/RATE-LIMITED-CONSUMERS.md` | **★ load-bearing** (this doc) |
| L1 | `pleme-io/helmworks/charts/pleme-lib/templates/_{jetstream_*,rate_limit_*}.tpl` | **★ load-bearing** |
| L2 | `pleme-io/helmworks/charts/pleme-nats/` | **★ load-bearing** |
| L3 | `pleme-io/samba/` (Rust crate) | **◐ scaffold** (trait + types, future workers stand on it) |
| L4 | `pleme-io/helmworks/charts/pleme-tend-throttle/` (canonical reference) | **◐ chart ready, awaiting `tend throttle` subcommand** |
| L5 | `(defrate-limited-consumer …)` via `#[derive(TataraDomain)]` | **○ aspirational** (next compounding move) |

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
