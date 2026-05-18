# Shigoto — the typed job-system primitive

> **Frame.** [`THEORY.md`](./THEORY.md) §IV (Motion) names controllers as
> fixed-point operators and the eight-phase convergence loop.
> [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> says every recurring shape becomes a typed substrate primitive. **This
> doc is that primitive at the work-graph layer:** a Job is a fixed-point
> operator, a Job DAG is a partial order over operators, a Job phase is
> the operator's position in its convergence trajectory.
>
> `shigoto` (仕事, "work / job / task") is the foundational name for the
> seventh pleme-io primitive layer alongside tatara, shikumi, sekkei,
> takumi, forge, and arch-synthesizer. The doc is normative: every
> pleme-io tool whose internals form a dependency-ordered, fallible,
> retryable, parallelism-bounded work graph adopts this shape.
> `skip-shigoto:` per-repo to deviate.
>
> **Status.** Draft v2 + M0.16 reconciliation. Bootstrap consumer
> (`tend`) migration is M0.10b through M0.15c; see §IV.5 for the
> shipped Job impls. Prime-directive promotion in §V is gated on two
> production consumers.
>
> **Shipped vs aspirational.** As of M0.16 (this rev), the v0.1
> shipped surface intentionally omits several pieces the spec below
> describes: `Job::Input` + `JobContext` are deferred (jobs hold input
> in `self` and execute as `async fn execute(&self) -> Result<Output,
> Error>`); `JobKind` as a separate trait is deferred (kind-defaults
> register against `JobKindId` on the Scheduler directly); per-Job
> override methods (`Job::gates()` / `Job::retry_policy()` /
> `Job::timeout()`) are deferred (registration is scheduler-level).
> Each is a "land when a real consumer's need forces it" decision per
> §VI.
>
> **Additions since v2 spec.** What shipped that this doc didn't yet
> describe at v2: `OutputSink<O>` (§III.10b; M0.11), concurrent wave
> execution (§III.6b; M0.13), the four tend Job wrappers proving the
> substrate composes for a real consumer (§IV.5; M0.10b–f, M0.12,
> M0.14, M0.15a–c). Sections below reflect the actually-shipped
> surface; aspirational pieces are explicitly labeled.

---

## I. The repeating pattern

Six pleme-io tools, six ad-hoc job systems:

| Tool | Work model | DAG | FSM | Parallelism | Retry | Audit |
|---|---|---|---|---|---|---|
| tend (daemon) | `planner.rs` stages | implicit | none | unbounded `tokio::JoinSet` | none (silent skip) | `audit.jsonl` ad-hoc |
| tend (operator) | `operator/dag.rs` waves | typed `petgraph` | `ProposalPhase` (5 states) | wave-bounded + `failure_set` | budget-aware | CR `status.conditions` |
| forge-gen | hardcoded pipeline | none | none | per-stage `rayon` | abort-on-first | stdout |
| pangea-operator | kube-runtime per CR | none | per-CR `phase` | per-CR rate-limit | kube-runtime | CR `status.conditions` |
| tameshi | shell glue → forge-gen | none | none | none | manual rerun | `cartorio` receipts |
| convergence-flow (methodology) | eight phases | implicit | implicit | per-impl | per-impl | per-impl |

Strip the surface and every one of these manipulates the *same seven
concerns*: typed work items, partial order over items, per-item state
machine, parallelism envelope, failure policy, transition log,
reconcile cycle. The same seven concerns appear in K8s reconcilers
(work-queue + status + retry), in build systems (targets + deps +
jobserver), and in workflow engines (activities + history + worker
pool). The shape is universal. Naming it is overdue — and the
substrate carries the typed primitive only once.

Three forces hid the pattern from us: each instance felt
domain-specific; adjacent libraries (`kube-runtime`, `petgraph`,
`tokio`) own fragments but not the composition; the pattern is "the
boring part" — invisible against the interesting work it scaffolds.

---

## II. The destination

### II.1. One typed primitive

```
pleme-io/shigoto                          (Cargo workspace)
├── shigoto-types       — Job, JobId, JobPhase, JobKindId, TickReceipt, ...
├── shigoto-dag         — Dag, edges, wave partitioning
├── shigoto-scheduler   — Scheduler trait + InProcessScheduler
├── shigoto-budget      — BudgetTree, min-intersection, fair-share
├── shigoto-retry       — RetryPolicy, BackoffStrategy, RetryDecider
├── shigoto-gate        — Gate trait + standard gates (TtlElapsed, ...)
├── shigoto-emit        — TransitionEmitter, AuditFileEmitter, NatsEmitter
├── shigoto-test        — idempotence proptest harness, golden tests
└── shigoto             — umbrella crate, re-exports
```

Pure-Rust, no IO above the `Job` trait. Side effects belong to
consumers; state-machine, scheduling, budget, retry, emission belong to
shigoto. Closes the algebra; opens to consumers.

### II.2. Two consumption surfaces

The bootstrap consumer `tend` exposes one Scheduler through two
frontends:

- **CLI daemon** — local single-node loop. `tick → wait_for_change →
  tick`. Exits on SIGTERM via cooperative drain.
- **K8s operator** — multi-cluster controller. Each CR event becomes
  one `tick` of the CR's DAG. Receipts emitted to `.status`.

Same Scheduler. Different triggers (clock vs. CR event), different
receipt sinks (audit vs. CR status), different eviction (process exit
vs. controller restart).

### II.3. Where shigoto fits in the typescape

Per [`THEORY.md`](./THEORY.md) §III.1, the typescape is owned by
arch-synthesizer. Shigoto extends it with one new dimension — **work
primitives** — alongside the existing infra, identity, data, and
verification dimensions. Authors declare work in tatara-lisp
(`(defjob …)`, `(defdag …)`, `(defbudget …)`); arch-synthesizer
renders to the Rust runtime config that `Scheduler::new()` consumes.
The Rust + Lisp pattern ([`THEORY.md`](./THEORY.md) §II.1) at the
work-graph layer.

---

## III. The typed surface

The contract every consumer codes against. Compact code blocks; the
prose carries the invariants.

### III.1. `Job` — the unit of work

**Shipped (v0.1):**

```rust
#[async_trait]
pub trait Job: Send + Sync + 'static {
    type Output: Send + 'static;
    type Error:  std::error::Error + Send + Sync + 'static;

    fn id(&self)   -> JobId;
    fn kind(&self) -> JobKindId;

    /// Side-effecting work. Called only on Ready → Running. MUST be
    /// idempotent — execute may be re-invoked after a scheduler crash.
    async fn execute(&self) -> Result<Self::Output, Self::Error>;
}
```

**Aspirational (lands when a consumer's need forces it):** `type Input`
+ `JobContext` argument to `execute`, plus `gates() / retry_policy() /
timeout()` per-Job override methods. The v0.1 substitutes:
- Inputs ride in `self` — jobs are pure-data structs constructed with
  whatever they need to do their work (workspace, repo_name, repo_path,
  pre-resolved URLs, optional sinks).
- Gates/retry/timeout register against `JobKindId` on the scheduler
  (`register_gate`, `register_retry_policy`, `set_timeout`) — the
  per-kind defaults the §III.4 `JobKind` trait would have provided.
- Cancellation/clock are external (`tokio::time::timeout` wrapping
  `execute`; SIGTERM via `tsunagu::ShutdownController`).

Invariants the shipped trait pins:
- `'static + Send + Sync` — jobs move between scheduler threads.
- Typed `Output` / `Error` — no untyped JSON in the work contract.
  Consumers serialize for transport, not for typing.
- `execute` is async on tokio. Sync work uses `spawn_blocking`.
- `id()` and `kind()` are pure — the scheduler reads them many times
  per cycle; no IO allowed.
- `execute` is idempotent (§III.13). The contract is testable via
  `shigoto-test`'s idempotence proptest.
- The scheduler's `execute_erased` blanket impl on `ErasedJob` collapses
  the typed `Output` / `Error` to `()` / `Box<dyn Error>` so the
  scheduler can hold heterogeneous Jobs in one DAG. Typed Outputs reach
  consumers via `OutputSink<O>` (§III.10b), not the FSM.

### III.2. `JobId` — typed identity

```rust
pub struct JobId {
    pub scope:   JobScope,
    pub kind:    JobKindId,
    pub subject: JobSubject,
}

pub enum JobScope {
    Global,
    Workspace(String),
    Repo { workspace: String, repo: String },
}

pub enum JobSubject {
    None,
    Repo(String),
    Org(String),
    Path(PathBuf),
    Pinned(String),
}

pub struct JobKindId(pub &'static str);
```

A `JobId` is stable across cycles: the same `(scope, kind, subject)`
always names the same logical job. This is the basis for FSM resumption
after restart — the scheduler looks up the existing `JobState` by ID
and resumes the FSM where it left off.

### III.3. `JobPhase` — the FSM

```rust
pub enum JobPhase {
    Pending,               // declared; gates not yet evaluated this tick
    Gated,                 // gates evaluated; ≥1 returned Wait
    Ready,                 // all gates Pass; awaiting budget
    Running,               // budget allocated; execute() in flight
    Succeeded,             // execute returned Ok. terminal-for-cycle.
    Failed { attempts: u32 }, // execute returned Err; retry decision pending
    Retrying { until: Instant }, // backoff scheduled
    Skipped(SkipReason),   // gate returned Skip; terminal.
    Deadlettered,          // retries exhausted; operator action required.
    WaitingForOperator,    // §III.13: human-in-the-loop.
}
```

Transitions:

```
                  ┌─────────┐
                  │ Pending │ ──── evaluate all gates ──┐
                  └─────────┘                           │
                                                        │
            ┌────────────────────┬──────────────────────┴────────────────┐
            ▼                    ▼                                       ▼
       any Wait             all Pass                                  any Skip
            │                    │                                       │
            ▼                    ▼                                       ▼
       ┌─────────┐           ┌───────┐                              ┌─────────┐
       │  Gated  │ ──Pass──► │ Ready │                              │ Skipped │ (terminal)
       └─────────┘           └───┬───┘                              └─────────┘
            ▲                   │
            │                   │  budget allocated
            │                   ▼
            │              ┌─────────┐
            │              │ Running │
            │              └────┬────┘
            │                   │
            │              ┌────┴────────────┐
            │              ▼                 ▼
            │           Ok                  Err
            │              │                 │
            │              ▼                 ▼
            │       ┌───────────┐       ┌────────┐
            │       │ Succeeded │       │ Failed │
            │       └───────────┘       └───┬────┘
            │       (terminal)              │
            │                       ┌───────┴────────────┐
            │           retry left  ▼                    ▼  exhausted
            │                  ┌──────────┐         ┌──────────────┐
            │                  │ Retrying │         │ Deadlettered │
            │                  └────┬─────┘         └──────┬───────┘
            │                       │                      │
            │                       │ backoff elapsed      │ operator transitions
            └───────────────────────┘                      ▼
                                                  ┌───────────────────┐
                                                  │WaitingForOperator │
                                                  └─────────┬─────────┘
                                                            │ operator decides
                                                            ▼
                                                       Ready or Skipped
```

The FSM is implemented as a single `fn advance(from, signal) ->
Result<JobPhase, IllegalTransition>` table. Compiler exhaustiveness
over `(JobPhase, Signal)` enforces that every cell is decided; new
phases or signals fail to build until handled.

**Signals** (the only ways phase can change):

| Signal | Effect |
|---|---|
| `EvaluateGates` | Pending → {Gated, Ready, Skipped} ; Gated → {Gated, Ready, Skipped} ; Retrying-with-elapsed-backoff → Ready |
| `AllocateBudget` | Ready → Running |
| `ExecutionDone(Ok)` | Running → Succeeded |
| `ExecutionDone(Err)` | Running → Failed |
| `RetryDecide` | Failed → {Retrying, Deadlettered} |
| `Cancel` | Running → Failed { with cancellation error } (§III.13) |
| `Timeout` | Running → Failed { with timeout error } (§III.13) |
| `OperatorTransition` | WaitingForOperator → {Ready, Skipped} ; Deadlettered → Pending |
| `DagRemoval` | any → tombstoned (§III.14) |

Every transition is recorded by the `TransitionEmitter` (§III.10).

### III.4. `JobKind` — typed work classes (aspirational)

**Status:** the standalone `JobKind` trait is deferred. v0.1 registers
kind-default budgets, gates, retry policies, and timeouts directly
against `JobKindId` on the Scheduler:

```rust
scheduler.install_budget(budget_tree).await;
scheduler.register_gate(JobKindId::new("tend.pull-repo"), gate).await;
scheduler.register_retry_policy(JobKindId::new("tend.pull-repo"), policy).await;
scheduler.set_timeout(job_id, Duration::from_secs(60)).await;
```

`JobKindId` is a typed work-class identifier (`String` wrapper). Two
Jobs sharing a `JobKindId` share the registered defaults. Examples
from the bootstrap consumer (`tend`):

| Kind id              | Spec section |
|---------------------|--------------|
| `tend.pull-repo`    | §IV.5        |
| `tend.status-repo`  | §IV.5        |
| `tend.fetch-repo`   | §IV.5        |
| `tend.sync-repo`    | §IV.5        |

The `JobKind` trait lands when a consumer's need forces a richer
defaults surface (e.g., kind-specific behavioral knobs that aren't
covered by the scheduler-level registrations).

### III.5. `Dag` — explicit topology

```rust
pub struct Dag { /* opaque */ }

impl Dag {
    pub fn add_job(&mut self, job: Box<dyn ErasedJob>) -> Result<(), DagError>;
    pub fn add_edge(&mut self, from: JobId, to: JobId) -> Result<(), DagError>;
    pub fn remove_job(&mut self, id: &JobId) -> Result<(), DagError>;
    pub fn validate(&self) -> Result<(), DagError>; // no cycles, no dangling
    pub fn waves(&self) -> Result<Vec<Vec<JobId>>, DagError>;
}
```

An edge `A → B` declares: *B may not start until A reaches a terminal
phase* (Succeeded, Skipped, or Deadlettered). Terminal includes
Deadlettered intentionally — a dead ancestor releases its descendants
to make partial progress rather than blocking forever. Descendants of
deadlettered ancestors emit `BlockedByDeadletteredAncestor` so cascading
skips are explicit.

Edges declare dependency, not data flow. Data flow rides on typed
`Input`/`Output`. B's `Input` may consume A's `Output` — or B may read
on-disk state, or take an input from scheduler config. The DAG only
sequences.

Cycles are rejected at `validate()`, not at runtime — the scheduler has
nothing to check at runtime beyond budget allocation. The DAG is
mutable across ticks (jobs added when a discovery completes, removed
when a repo is archived); each mutation re-validates.

### III.6. `Scheduler` — the runtime

**Shipped:**

```rust
#[async_trait]
pub trait Scheduler: Send + Sync {
    async fn tick(&self, dag: &mut Dag) -> Result<TickReceipt, SchedulerError>;
    async fn snapshot(&self, _dag: &Dag) -> Snapshot;
}
```

`InProcessScheduler` is the v0.1 impl. Outside the trait it exposes
the typed registration API:
- `register_job(Arc<dyn ErasedJob>)`
- `register_gate(JobKindId, Arc<dyn Gate>)`
- `register_retry_policy(JobKindId, RetryPolicy)`
- `set_timeout(JobId, Duration)`
- `install_budget(BudgetTree)`
- `with_emitter(Arc<dyn TransitionEmitter>)` — builder
- `operator_transition(&JobId, JobPhase, TransitionReason)`

A `Scheduler` is *not* a daemon. It does one tick per call. Daemons
loop `tick → sleep → tick` (the deferred `wait_for_change` is replaced
by a sleep + SIGTERM polling in v0.1). Future impls (distributed,
persistent, replayable) plug behind the trait.

### III.6b. Concurrent wave execution (M0.13)

`InProcessScheduler.tick()` runs each topological wave in three passes:

1. **Pass 1 (sequential):** drive every Job through fast FSM transitions
   (gate evaluation, budget allocation) up to `Running`. Lock-contended
   but CPU-cheap.
2. **Pass 2 (concurrent):** collect every Job in `Running`, fan out
   their `execute_erased()` calls into a `tokio::task::JoinSet`, await
   all. **A wave of N Ready jobs runs in `O(slowest_execute)`, not
   `O(sum_of_executes)`.**
3. **Pass 3 (sequential):** drain terminal transitions (`Failed →
   RetryDecide → {Retrying | Deadlettered}`) for any Job whose execute
   returned.

Inter-wave dependencies still serialize: Jobs in wave 2 can't start
their pass-1 transitions until wave 1's pass-3 has landed terminal
phases for their upstreams. The concurrency is *within* a wave.

**Correctness contract:** the FSM still goes through every legal
transition `shigoto_types::advance` declares. The wave-level
parallelism only changes *when* execute runs relative to the FSM walk;
phase transitions themselves remain serialized.

### III.7. `Budget` — typed parallelism

```rust
pub struct BudgetTree {
    pub global:   BudgetSpec,
    pub by_kind:  HashMap<JobKindId, BudgetSpec>,
    pub by_scope: HashMap<JobScope,  BudgetSpec>,
}

pub struct BudgetSpec {
    pub max_concurrent:           u32,
    pub max_failures_per_minute:  u32,
    pub queue_depth:              u32,
}
```

A job of kind `K` in scope `S` may transition Ready → Running iff
`running_now(global) < global.max_concurrent` AND `running_now(K) <
by_kind[K].max_concurrent` AND `running_now(S) < by_scope[S].max_concurrent`.
Three counters, one atomic `try_allocate`. Composition is
min-intersection — the narrowest applicable budget binds. This is the
load-bearing defense against runaway parallelism (current
`tokio::JoinSet` in tend's daemon has no upper bound — a workspace
with 600 repos spawns 600 git processes simultaneously).

When budget is exhausted across waiting Ready jobs, the scheduler
picks by deficit-round-robin across scopes within a kind, kinds within
global. Fair-share by construction; no scope or kind monopolizes.

### III.8. `RetryPolicy` — typed failure recovery

```rust
pub enum RetryPolicy {
    NoRetry,
    Fixed       { attempts: u32, delay: Duration },
    Exponential { attempts: u32, base:  Duration, max: Duration, jitter: f64 },
    Custom(Arc<dyn RetryDecider>),
}

pub trait RetryDecider: Send + Sync {
    fn decide(
        &self,
        attempt: u32,
        error:   &dyn JobError,
        history: &[FailureRecord],
    ) -> RetryDecision;
}
```

`RetryDecider` is the IO-aware escape hatch — a job that honors HTTP
`Retry-After` headers, e.g., implements its own decider. Default
policies cover 90% of cases.

### III.9. `Gate` — typed preconditions

```rust
pub trait Gate: Send + Sync + 'static {
    fn name(&self) -> &'static str;

    /// Pure. No IO. IO-dependent gating is a Job emitting a typed
    /// fact a downstream gate then checks.
    fn evaluate(&self, job: &JobId, snapshot: &Snapshot) -> GateOutcome;
}

pub enum GateOutcome { Pass, Wait, Skip(SkipReason) }
```

Standard gates: `AllUpstreamsTerminal`, `TtlElapsed { last, ttl }`,
`BudgetAvailable` (scheduler-injected), `OperatorApproved`. Consumers
add domain-specific gates (tend's `IsClean`, forge-gen's
`SpecHashMatches`, pangea-operator's `LeaderHeld`).

Gates are *pure* — two ticks against the same snapshot produce the
same schedule. IO masquerading as a gate would break determinism; it
must be a Job instead.

### III.10. `TransitionEmitter` — typed audit

```rust
pub trait TransitionEmitter: Send + Sync {
    fn emit(&self, event: TransitionEvent);
}

pub struct TransitionEvent {
    pub at:      DateTime<Utc>,
    pub job_id:  JobId,
    pub from:    JobPhase,
    pub to:      JobPhase,
    pub reason:  TransitionReason,
    pub tool:    &'static str,  // consumer identifier, set at scheduler build
}
```

Sinks compose:
- `AuditFileEmitter` — append to `<data_dir>/audit.jsonl`.
- `NatsEmitter` — publish to `shigoto.<tool>.<workspace>.<kind>.<phase>`.
- `MultiEmitter` — fan-out to ≥1 sink.

Emitters are non-blocking — events are queued and flushed in a
background task. On queue overflow events are dropped with a
`TransitionDropped` log line; observability never back-pressures the
scheduler.

The transition log is the **canonical history**. `tend report`, K8s
controller status, MCP operator surface, dashboards — all read the
same log. No second source of truth.

**Audit ≠ metrics.** The transition log is FSM history (when did this
job's phase change). Throughput / latency / queue-depth / per-tick
duration are sibling concerns surfaced separately (Vector →
VictoriaMetrics or Datadog APM) so the audit log doesn't become a
sample-rate-sensitive metrics store.

### III.10b. `OutputSink<O>` — typed output capture (M0.11)

The scheduler's `execute_erased` blanket impl collapses every Job's
typed `Output` to `()` so heterogeneous Jobs can live in one
`Vec<Box<dyn ErasedJob>>`. That erasure means the FSM never sees the
typed Output value — but consumers (reconcile receipts, dashboards,
audit trails) need it. `OutputSink<O>` is the typed surface that
bridges:

```rust
#[async_trait]
pub trait OutputSink<O>: Send + Sync + 'static
where
    O: Send + Sync + 'static,
{
    async fn record(&self, job_id: &JobId, output: &O);
}
```

A Job calls `sink.record(&self.id(), &output)` from its `execute`
after computing the outcome. Per-Job sink wiring (not scheduler-level)
keeps the typed `O` parameter from leaking into the scheduler's
heterogeneous storage. Concrete sinks in `shigoto-emit`:

- `NullSink<O>` — discards every record. Default for Jobs that don't
  surface their outputs.
- `InMemorySink<O>` (requires `O: Clone`) — stores into
  `Arc<Mutex<HashMap<JobId, O>>>`. Reader API: `drain()` /
  `snapshot()` / `len()` / `is_empty()`. Overwrites on duplicate
  JobId so retries don't accumulate.

Why per-Job (not scheduler-attached): the scheduler holds Jobs as
trait objects with the type parameter erased. A scheduler-attached
sink would either need its own type erasure (sacrificing the typed
contract) or restrict the scheduler to a single output type
(sacrificing heterogeneity). Per-Job sinks preserve both.

### III.11. `TickReceipt` — derived rollup

```rust
pub struct TickReceipt {
    pub tick_at:               DateTime<Utc>,
    pub dag_summary:           DagSummary,
    pub phase_counts:          HashMap<JobPhase, u32>,
    pub transitions_this_tick: Vec<TransitionEvent>,
    pub unhealed:              Vec<UnhealedDrift>,
}
```

Produced by the scheduler on every `tick`. Derived — not a separate
state store. Realizes the M10 "ReconcileReceipt" of the original tend
plan for free.

### III.12. `JobContext` — what execute() gets

```rust
pub struct JobContext<'a> {
    pub emitter:    &'a dyn TransitionEmitter,
    pub cancel:     &'a CancellationToken,
    pub clock:      &'a dyn Clock,
    pub log:        &'a tracing::Span,
}
```

The execute hook gets exactly four things:

- `emitter` — for *domain* events the consumer wants to publish
  alongside FSM transitions (e.g., tend's "repo updated to SHA X").
- `cancel` — cooperative cancellation token. `execute` must poll it
  and return early when triggered (§III.13).
- `clock` — injected time source; tests use a mock clock.
- `log` — `tracing` span scoped to this job; all spans/events inside
  execute inherit it.

No filesystem, no network, no scheduler reference. Consumers bring
their own clients.

### III.13. Cancellation, timeout, idempotence

**Cancellation.** Shutdown (SIGTERM) triggers the scheduler's root
`CancellationToken`. Every Running job receives a cancel signal via
`JobContext::cancel`. Jobs must poll it cooperatively — long-running
work checks `cancel.is_cancelled()` between IO operations and returns
`Err(JobError::Cancelled)` on hit. The FSM treats cancellation as a
`Cancel` signal (Running → Failed); the retry policy decides whether
to retry (typically NoRetry for cancellation errors).

**Timeout.** Per-job `timeout()` returns an optional duration; absent,
the kind default; absent, none (no timeout). The scheduler wraps
`execute()` in `tokio::time::timeout`. On elapse: `Timeout` signal,
Running → Failed with `JobError::Timeout`. Same retry-policy treatment.

**Idempotence.** Per §III.1, `execute` must be idempotent. The
scheduler does its part — Running is reached only via Ready, never
re-entered while a transition is in flight. But if a scheduler crashes
between completing a side-effect and emitting `Succeeded`, the next
cycle re-invokes execute. Jobs must tolerate it: tend's `PullRepoKind`
gets idempotence free (pulling an up-to-date repo is a no-op); tend's
`AdoptRepoKind` must check whether the repo is already in config
before adding.

`shigoto-test` ships an `idempotence_quickcheck` proptest that
invokes `execute` twice against the same input and asserts the second
is a no-op (no domain state changed, same Output value).

### III.14. Persistence — explicitly deferred

**v1 is in-memory only.** FSM state lives in the scheduler's address
space; a scheduler restart drops every job back to `Pending`. The
contract holds because §III.13 requires every consumer to be
idempotent: a side-effect interrupted mid-execute is safely re-run on
the next cycle.

Persistence is a future Scheduler impl behind the existing trait.
Specifically: `PersistentScheduler` stores phase transitions to a sled
or sqlite store, and on startup reconstructs `JobState` by replaying
the transition log. Consumers see no API change. Deferred until a
real consumer cannot guarantee idempotence (§VI.2).

A job removed from the DAG mid-flight is tombstoned: the in-memory
`JobState` is dropped, future signals to that JobId emit
`SignalToTombstone`, the cycle continues. No leaks.

### III.15. The tatara-lisp surface

```lisp
(defjob :id (:scope (:workspace "pleme-io") :kind pull-repo :subject (:repo "tend"))
        :gates (is-clean ttl-elapsed)
        :retry-policy (exponential :attempts 3 :base 30s :max 5m :jitter 0.2)
        :timeout 2m)

(defdag :name pleme-io-cycle
        :jobs (discover-org observe-org sync-missing pull-clean react-drift watch-versions)
        :edges ((discover-org → observe-org)
                (observe-org → sync-missing)
                (sync-missing → pull-clean)
                (pull-clean → react-drift)
                (pull-clean → watch-versions)))

(defbudget :scope (:workspace "pleme-io")
           :max-concurrent 16
           :max-failures-per-minute 10)
```

Authored by humans, rendered by arch-synthesizer to the Rust runtime
config that `Scheduler::new()` consumes. Compositional invariants are
proven at Rust compile time; configuration invariants are proven at
render time. (Reference: SHIGOTO.md is followed by an arch-synthesizer
domain registration; the registration is part of M0.6.)

### III.16. Worked example — `PullRepoJob`

```rust
pub struct PullRepoJob {
    pub workspace: String,
    pub repo:      String,
    pub repo_path: PathBuf,
}

#[async_trait]
impl Job for PullRepoJob {
    type Input  = (); // reads on-disk state
    type Output = PullOutcome;
    type Error  = PullError;

    fn id(&self) -> JobId {
        JobId {
            scope:   JobScope::Repo { workspace: self.workspace.clone(),
                                       repo:      self.repo.clone() },
            kind:    JobKindId("pull-repo"),
            subject: JobSubject::Repo(self.repo.clone()),
        }
    }

    fn kind(&self) -> JobKindId { JobKindId("pull-repo") }

    fn gates(&self) -> Vec<Box<dyn Gate>> {
        vec![Box::new(IsClean::for_path(self.repo_path.clone()))]
    }

    async fn execute(&self, _input: (), ctx: &JobContext<'_>)
        -> Result<PullOutcome, PullError>
    {
        if ctx.cancel.is_cancelled() { return Err(PullError::Cancelled); }
        let output = tokio::process::Command::new("git")
            .args(["pull", "--ff-only", "--quiet"])
            .current_dir(&self.repo_path)
            .output().await
            .map_err(PullError::Spawn)?;
        if !output.status.success() {
            return Err(PullError::Git(String::from_utf8_lossy(&output.stderr).into()));
        }
        Ok(PullOutcome::from_git_output(&output))
    }
}
```

Six concerns visible: identity (`id`), classification (`kind`),
domain-specific gating (`IsClean`), kind-default retry policy (omitted
— picked up from registered `PullRepoKind`), idempotence (git pull
on up-to-date is a no-op), cooperative cancellation (the explicit
poll). No FSM concerns, no DAG concerns, no parallelism concerns —
those belong to the Scheduler.

---

## IV. Bootstrap consumer — tend

### IV.1. Workspace = DAG root

Each tend workspace becomes one `Dag`. A multi-workspace tend daemon
runs one Scheduler with N Dags; budgets are shared via the global
dimension, isolated via the per-workspace dimension.

### IV.2. The eleven job kinds

| Kind | Subject | Default gates | Default retry | Notes |
|---|---|---|---|---|
| `DiscoverOrgKind`    | org | `TtlElapsed { 1h }` | Exp(3, 30s, 5m, 0.2) | hits GitHub API |
| `ObserveOrgKind`     | org | upstream of `DiscoverOrg` | Exp(3, 30s, 5m, 0.2) | per-repo metadata fetch |
| `SyncRepoKind`       | repo | upstream of `Observe`; `NotPlaceholder` | Exp(3, 30s, 5m, 0.2) | `git clone` missing |
| `PullRepoKind`       | repo | upstream of `Sync`; `IsClean` | Exp(3, 30s, 5m, 0.2) | `git pull --ff-only` |
| `FetchRepoKind`      | repo | upstream of `Sync`; `PullDisabled` | Exp(3, 30s, 5m, 0.2) | fetch-only edge case |
| `WatchRepoKind`      | repo | upstream of `Pull`; `WatchEnabled` | NoRetry | detect new versions |
| `ReactDriftKind`     | drift event | reaction config + drift type | Custom | typed reactions |
| `FlakeUpdateKind`    | flake input | upstream of `Pull`; `FlakeUpdateEnabled` | Exp(2, 5m, 30m, 0.3) | propagates flake.lock |
| `NixAuditKind`       | workspace | `NixAuditEnabled` | NoRetry | runs nix-audit check |
| `AdoptRepoKind`      | unknown repo | `OperatorApproved` | NoRetry | operator-driven |
| `MarkPlaceholderKind`| empty dir | `OperatorApproved` | NoRetry | operator-driven |

The existing `WorkKind` enum in `tend/src/planner.rs` is the primordial
version of this table.

### IV.3. Migration plan — what actually shipped

Per operator instruction, commits land directly on main, each leaving
tend buildable. Reality diverged from the v2 plan in two ways:

1. The substrate matured before the daemon migrated — M0.9a–s built
   out every shigoto crate in 10-line atomic steps rather than landing
   the full scheduler in one M0.9.
2. The daemon migration happened *after* a standalone `tend reconcile`
   CLI proved the substrate composes for tend's real workload, rather
   than going directly from planner-based to scheduler-based daemon.

| Milestone | What shipped                                                                  |
|----------:|-------------------------------------------------------------------------------|
| M0.6      | Scaffold `pleme-io/shigoto` workspace; CI green; substrate flake template.   |
| M0.7      | Lift `tend/src/operator/dag.rs` → `shigoto-dag::Dag`; tend operator migrated. |
| M0.8      | Tend operator consumes `shigoto-dag` (typed `JobId` everywhere).              |
| M0.9a     | Typed FSM `advance()` in shigoto-types (Phase × Signal exhaustive match).     |
| M0.9b     | `Job` + `ErasedJob` traits; blanket impl gives every typed `Job` an erased view. |
| M0.9c     | `InProcessScheduler` tick loop (sequential v0; concurrent in M0.13).           |
| M0.9d     | `TransitionEmitter` sinks: `NullEmitter` / `AuditFileEmitter` / `MultiEmitter`. |
| M0.9e     | `Dag::predecessors()` accessor — prerequisite for `AllUpstreamsTerminal`.      |
| M0.9f     | Standard gates + `reduce()` (worst-wins) in `shigoto-gate`.                    |
| M0.9g–h   | Wire gate + retry registries into scheduler.                                   |
| M0.9i–j   | `BudgetTree::try_allocate / release`; wire into Ready→Running.                |
| M0.9k     | Make `Scheduler::snapshot` async (drop `block_in_place`).                     |
| M0.9l     | `operator_transition` emits via `TransitionEmitter` immediately.              |
| M0.9m     | Populate `TickReceipt.unhealed` at end of tick.                               |
| M0.9n     | `Snapshot::failure_set()` derived view.                                        |
| M0.9p     | `idempotence_quickcheck` harness in `shigoto-test`.                            |
| M0.9r     | End-to-end integration test exercising the full pipeline (gates + retry + budget + emitter). |
| M0.9s     | `shigoto` umbrella crate re-exports the full surface; smoke test.             |
| M0.10a    | Add full shigoto suite as tend Cargo deps.                                    |
| M0.10b    | `PullRepoJob` — first typed Job wrapper in tend; 5 tests.                     |
| M0.10c    | Scheduler composition test — N PullRepoJobs through real scheduler.           |
| M0.10d    | `StatusRepoJob` (read-only; perfect fit for idempotence quickcheck).          |
| M0.10e    | `FetchRepoJob` + wrapper-pattern doc.                                         |
| M0.10f    | `SyncRepoJob` — completes the 4-Job reconcile primitive set.                  |
| M0.11a    | `OutputSink<O>` typed primitive + `NullSink` + `InMemorySink` (§III.10b).     |
| M0.11b    | Wire `OutputSink<O>` into all 4 tend Job wrappers.                             |
| M0.12     | `tend reconcile [--workspace W]` CLI — first user-facing surface.             |
| M0.13     | **Concurrent wave execution** in scheduler (§III.6b); 4×200ms in 200ms wall. |
| M0.14     | Budget-bounded reconcile (default `max_inflight=16`). Real-world: 118 repos in 21s. |
| M0.15a    | Migrate `tend daemon` pull-branch to Scheduler.                               |
| M0.15b    | Register Exponential retry policy for `tend.pull-repo`.                       |
| M0.15c    | Wire `AuditFileEmitter` so every transition lands in `scheduler-transitions.jsonl`. |
| M0.16     | This reconciliation pass on `SHIGOTO.md`.                                     |

Still deferred (post-M0.16): `tend daemon` sync-branch via `SyncRepoJob`
+ Dag edge sync→pull; `tend daemon` fetch-branch via `FetchRepoJob`;
the original M2–M10 reshape from §IV.4.

### IV.5. The four shipped Job wrappers

Substrate primitives that landed during M0.10b–M0.10f, proving the
shigoto surface composes for the bootstrap consumer. Each follows the
same six-piece pattern documented in `tend/src/jobs/mod.rs`:

| Job kind             | Wraps                            | Output             | Mutates? |
|----------------------|----------------------------------|--------------------|----------|
| `tend.pull-repo`     | `sync::pull_one_repo`            | `PullOutcome`      | yes (git pull --ff-only) |
| `tend.status-repo`   | `sync::check_one_repo_status`    | `RepoStatus`       | no       |
| `tend.fetch-repo`    | `sync::fetch_one_repo`           | `FetchOutcome`     | yes (git fetch --all --prune) |
| `tend.sync-repo`     | `sync::sync_one_repo`            | `SyncOutcome`      | yes (git clone) |

The pattern:

1. `pub(crate) const X_REPO_KIND: &str = "tend.x-repo";` — canonical id.
2. Pure-data struct (`workspace`, `repo_name`, `repo_path`, optional
   `quiet`, `clone_url`, etc.) — no I/O at construction.
3. `with_output_sink(sink)` builder attaches `Arc<dyn OutputSink<O>>`.
4. `XRepoError::Invocation(String)` — only invocation failures flow
   through `Err`; typed-success-with-failure outcomes (e.g.
   `PullOutcome::Failed { stderr }`) flow through `Output` so the
   scheduler decides retries from the right channel.
5. `Job::id()` returns `(JobScope::Workspace(...), JobSubject::Repo(...),
   JobKindId::new(X_REPO_KIND))` — the three coordinates fully name the
   Job across all tend workspaces.
6. `Job::execute()` is a `spawn_blocking` hop into the matching
   `sync.rs` helper, then `sink.record(&self.id(), &outcome).await` if
   an output sink is wired in.

The wrapper pattern's documentation is in `tend/src/jobs/mod.rs`'s
top-of-file docstring. The pattern is stable across four instances;
the macro-extraction question is parked until a 5th instance reveals
a uniform shape that a `macro_rules!` could template cleanly.

### IV.4. The original M2–M10 reshape

Once shigoto-scheduler exists, the original tend plan becomes mostly
declarative:

| Original | Reshape | Expected size |
|---|---|---|
| M2 discovery TTL | `DiscoverOrgKind` + `TtlElapsed` gate | ~50 LOC |
| M3 placeholder marker | `NotPlaceholder` gate + `MarkPlaceholderKind` | ~40 LOC |
| M4 extended org observation | `ObserveOrgKind` with typed `RepoState` output | ~150 LOC |
| M5 typed DriftEvent | DriftEvents emitted by FSM transitions | ~0 LOC (free) |
| M6 typed reactions | DAG edges from observe → react; config-driven kinds | ~100 LOC |
| M7 `tend report` | reads `Snapshot` + transition log | ~80 LOC |
| M8 doctor / adopt / placeholder | `WaitingForOperator` transitions | ~100 LOC each |
| M9 typescape primitive | `(defjob …)` / `(defdag …)` from §III.15 | already done |
| M10 ReconcileReceipt | `TickReceipt` exists | ~0 LOC (free) |

LOC numbers are expected, not committed — implementation will reveal
the real cost.

---

## V. Promotion to prime-directive

### V.1. Criteria

Promotion from "documented primitive" (★ in pleme-io/CLAUDE.md) to
"prime-directive rule" (★★) requires all four:

1. **Two production consumers** have migrated and run ≥30 days
   operational use. Bootstrap is `tend`. Candidate seconds: forge-gen,
   pangea-operator, tameshi. Two minimum falsifies "this is just
   tend's abstraction."
2. **Typed surface stable** — no breaking changes to `Job`, `Dag`,
   `Scheduler` traits for ≥30 days. Stability is the proof the
   abstraction held.
3. **Audit log consumed by ≥1 sink nothing else reads** — confirms
   the transition-log-as-single-source-of-truth invariant.
4. **One non-bootstrap consumer authored from scratch** as
   shigoto-native (not migrated). Confirms the green-field experience.

Until all four hold, shigoto is "documented; strongly recommended for
new work" but not enforced.

### V.2. Directive text (draft, lands on promotion)

To be inserted into `pleme-io/CLAUDE.md` immediately after the
existing ★★ PRIME DIRECTIVE + NO SHELL block:

> ## ★★ Shigoto for every work graph
>
> Any pleme-io tool whose internal work is a dependency-ordered graph
> of fallible, retryable, parallelism-bounded steps expresses that
> graph as a `shigoto` Dag of typed Jobs. The Scheduler is canonical;
> the audit log is canonical; the FSM is canonical. Hand-rolled work
> orchestration (sequential drivers, ad-hoc tokio JoinSets,
> domain-specific phase enums, bespoke retry loops) is forbidden in
> new code and a migration target in existing code. Deviation requires
> an explicit `skip-shigoto:` note at the top of the deviating repo's
> `CLAUDE.md`.
>
> Canonical spec: [`theory/SHIGOTO.md`](./theory/SHIGOTO.md).
> Theory frame: [`theory/THEORY.md`](./theory/THEORY.md) §IV.

### V.3. `skip-shigoto:` escape hatch

Legitimate exceptions:
- Single-step CLI tools (e.g., `bm-complete`). One step is not a graph.
- Pure-data crates with no orchestration (e.g., `arch-synthesizer-types`).
- Thin wrappers around external schedulers the operator doesn't own
  (e.g., a Kubernetes Job submitter where k8s does the orchestration).

Anything with ≥3 typed steps, ≥1 retry concern, or ≥2 parallel
branches: shigoto.

---

## VI. Open questions

Decisions deferred until a real consumer's need forces them.

### VI.1. Distributed coordination

Two daemons consuming the same workspace from two machines: race or
coordinate? Today tend assumes single-node. A future `SchedulerLock`
trait could back this on NATS jetstream or k8s lease. **Deferred until**
a real fleet runs ≥2 instances of one tool.

### VI.2. Persistence

§III.14 commits to in-memory v1. A `PersistentScheduler` impl behind
the existing trait is the upgrade path. **Deferred until** a consumer
cannot guarantee idempotence.

### VI.3. Sub-tick DAG mutation

§III.5 says mutations land between ticks. Streaming consumers (e.g., a
high-frequency CR event source) might need within-tick replanning.
**Deferred until** sub-second responsiveness is required.

### VI.4. Typed wire format for derived dashboards

§III.10 says emitters produce events; sinks subscribe. But the typed
schema of derived dashboards (Grafana queries, MCP responses) wants
its own spec. **Deferred until** the observability skill consumes
shigoto for its first dashboard.

---

## VII. References

### VII.1. Lift table from `tend/src/operator/*`

| File | Lines | Lifts to | Note |
|---|---|---|---|
| `dag.rs` | 212 | `shigoto-dag` | whole-file lift |
| `planner.rs` | 411 | `shigoto-scheduler::Planner` | trait surface; flake-domain stays in tend |
| `reconcile.rs` | 855 | NOT lifted | tend-side `Job` impls for flake update |
| `apply.rs` | 233 | `shigoto-scheduler::Apply` | trait surface only |
| `gates.rs` | 524 | `shigoto-gate` | trait surface; flake-specific gates remain in tend |
| `failure_set.rs` | 311 | `shigoto-scheduler::FailureSet` | derived view |
| `budget.rs` | 222 | `shigoto-budget` | whole-file lift, generalized |

Total: ~2700 LOC of operator code informs ~1200–1500 LOC of shigoto.
The reduction comes from generalizing flake-domain assumptions out of
the lifted code.

### VII.2. Related pleme-io primitives

- [`THEORY.md`](./THEORY.md) §IV — controllers as fixed-point operators
  + eight-phase loop. Shigoto is the typed realization.
- [`CAIXA-SDLC.md`](./CAIXA-SDLC.md) — SDLC primitive. A
  shigoto-consuming repo is a caixa.
- [`PANGEA-OPERATOR.md`](./PANGEA-OPERATOR.md) — future shigoto
  consumer (§V.1 candidate).
- `pleme-io/CLAUDE.md` ★★★ Compounding Directive — shigoto is one
  typed primitive widening the substrate's expressive power.

---

## VIII. Decision log

Load-bearing choices a future reader should be able to audit:

- **Japanese name `shigoto`** — direct semantic match; foundational
  tier alongside tatara, sekkei, takumi, forge.
- **Pure-async runtime (tokio)** — no sync worker pool. Consumers wrap
  sync work with `spawn_blocking`. One concurrency model.
- **Pure gates** — no IO. IO-needing gating is a Job emitting a typed
  fact. Keeps schedule deterministic per snapshot.
- **In-memory FSM only (v1)** — every consumer must be idempotent per
  §III.13. Persistence is a future Scheduler impl, not a v1 concern.
- **Cancellation is cooperative** — `JobContext::cancel` token; jobs
  poll. No forced abort.
- **Per-job timeout via `tokio::time::timeout`** — separate from retry.
- **Cascading skip on deadletter** — partial progress is the typical
  case; halt-on-error would burn operator time.
- **Transition log = single source of truth** — receipts, dashboards,
  CR status are derived views. One log, many readers.
- **Audit ≠ metrics** — FSM history vs throughput/latency. Different
  sinks. Different consumers.
- **Bootstrap = tend** — both consumption surfaces (CLI daemon + K8s
  operator) from day one.
- **Promotion gated on two consumers + stability + green-field +
  exclusive sink** — falsifies the "tend-only" objection.
- **`skip-shigoto:` per-repo escape hatch** — consistent with
  `skip-blackmatter:`, `skip-saguao:` precedent.
- **Repo lives at `pleme-io/shigoto`** — standalone foundational repo,
  not a module inside tend. Per operator instruction.

---

*v2. Replaces v1 (1243 lines) after correctness fixes + scope tightening.*
