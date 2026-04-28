# Absorption roadmap — M2 → M5 execution plan

> **Frame.** [`INSPIRATIONS.md`](./INSPIRATIONS.md) lays out the eight
> traditions whose patterns the substrate should absorb.
> [`RUNTIME-PATTERNS.md`](./RUNTIME-PATTERNS.md) catalogs what becomes
> possible. **This doc is the build plan** — concrete files, LoC
> estimates, test strategy, and dependency order for landing every
> pattern. Audience: any agent or operator picking up an M2 item, who
> wants to know *exactly* what to write, where, and how to verify it.

Convention:

- **★** = ship in M2 (next quarter; the tight typed-substrate cycle)
- **☆** = ship in M3 (mid-year; distribution + introspection)
- **◇** = ship in M4 (late year; operator surface)
- **⬡** = ship in M5 (self-hosting; the recursive loop)

Every item carries: scope (files + LoC), test strategy, dependency
order, status. Order in each milestone is the recommended landing
sequence — earlier items unblock later ones.

---

## M2 ★ — typed-substrate foundations (next quarter)

The typed slots that everything else composes on top of. These are
*additive* schema extensions to `caixa-core::Caixa`; existing caixas
keep parsing because every new slot has a default.

### M2.1 ★ caixa.lisp `:limits` — Lunatic-style per-process sandboxing

**Source pattern:** [`INSPIRATIONS.md` §III.1 — Lunatic per-process limits](./INSPIRATIONS.md#iii1-per-process-sandboxing--more-powerful-than-beam)

**Author surface:**

```lisp
(defcaixa
  …
  :limits ((:memory     "64MiB")     ; max linear memory
           (:fuel       1000000)     ; max instruction count per request
           (:wall-clock "30s")       ; max wall-clock time per request
           (:cpu        "500m")))    ; soft CPU share (cgroup hint)
```

**Files (caixa-core):**

- `caixa-core/src/limits.rs` — new (~120 LoC): `LimitsSpec` struct
  with optional fields, byte-size + duration parsers (e.g. `"64MiB"
  → u64`, `"30s" → Duration`), errors.
- `caixa-core/src/manifest.rs` — `Caixa::limits: Option<LimitsSpec>`
- `caixa-core/src/lib.rs` — re-export

**Tests:**
- parse `(:memory "64MiB")` → `LimitsSpec { memory: Some(64 * 1024 * 1024), … }`
- parse `(:fuel 1000000)` → fuel set
- round-trip via `to_lisp` / `from_lisp`
- reject invalid units (`"64YiB"`, negative numbers)
- absent slot → `None` (existing caixas unaffected)

**Estimate:** 120 LoC + 8 tests. ~1 hour focused.

**Dependency:** none. Land first.

**Downstream wiring (M2.5):** wasm-engine reads LimitsSpec at
component instantiation; sets `wasmtime::StoreLimits` + fuel +
wall-clock timer. Out of scope for caixa-core — when wasm-engine M2
lands, the slot is already there.

---

### M2.2 ★ caixa.lisp `:behavior` — OTP behaviors

**Source pattern:** [`INSPIRATIONS.md` §II.3 — OTP behaviors](./INSPIRATIONS.md#ii3-otp-behaviors-gen_server-gen_statem-gen_event)

**Author surface:**

```lisp
(defcaixa
  …
  :behavior ((:on-init         "lib/init.lisp")        ; called once before traffic
             (:on-call         "lib/handlers.lisp")    ; sync request/response
             (:on-cast         "lib/handlers.lisp")    ; async fire-and-forget
             (:on-info         "lib/handlers.lisp")    ; system messages (timeout etc)
             (:on-state-change "lib/migrations.lisp")  ; gen_server:code_change/3 analog
             (:on-terminate    "lib/cleanup.lisp")))   ; cleanup before shutdown
```

**Files (caixa-core):**

- `caixa-core/src/behavior.rs` — new (~80 LoC): `BehaviorSpec` with
  optional path-to-`.lisp` for each callback. Layout invariant: every
  declared path resolves under repo root.
- `caixa-core/src/manifest.rs` — `Caixa::behavior: Option<BehaviorSpec>`
- `caixa-core/src/layout.rs` — extend `StandardLayout::verify` to
  check declared callback paths exist.

**Tests:**
- parse all six callback slots
- partial callbacks (only `:on-init` set) → other fields None
- layout check fails when declared path doesn't exist
- round-trip
- reject paths outside repo root

**Estimate:** 80 LoC + 7 tests. ~1 hour.

**Dependency:** none. Parallel with M2.1.

---

### M2.3 ★ caixa.lisp `:upgrade-from` — OTP appup

**Source pattern:** [`INSPIRATIONS.md` §II.4 — appup/relup](./INSPIRATIONS.md#ii4-hot-code-reload--appup-relup-release_handler)

**Author surface:**

```lisp
(defcaixa
  …
  :upgrade-from
    ((:from "0.1.0"
      :instructions ((:load-module "hello-rio")
                     (:state-change "lib/migrations/v01-to-v02.lisp")
                     (:soft-purge "hello-rio-old")))))
```

**Files (caixa-core):**

- `caixa-core/src/upgrade.rs` — new (~150 LoC): `UpgradeFromSpec`
  enum of instructions:
  - `LoadModule(String)` — load v_new alongside v_old
  - `StateChange(PathBuf)` — run `.lisp` migration
  - `SoftPurge(String)` — wait for v_old to drain, then GC
  - `Purge(String)` — discard v_old immediately
  - `Restart` — full restart fallback
- `caixa-core/src/manifest.rs` — `Caixa::upgrade_from: Vec<UpgradeFromEntry>`
- `caixa-core/src/layout.rs` — verify referenced migration paths exist

**Tests:**
- parse a single upgrade entry
- parse multiple `:from` entries (chain v0.1.0 → 0.2.0 → 0.3.0)
- each instruction variant round-trips
- reject malformed `:from` (non-semver)
- absent slot → empty Vec (existing caixas unaffected)

**Estimate:** 150 LoC + 10 tests. ~1.5 hours.

**Dependency:** none. Parallel with M2.1, M2.2.

**Downstream wiring (M3):** caixa-flux renders an `Upgrade` CR
alongside the ComputeUnit; wasm-operator drives the upgrade-handler
state machine.

---

### M2.4 ★ `CaixaKind::Supervisor` — typed supervisor trees

**Source pattern:** [`INSPIRATIONS.md` §II.2, §III.2 — supervisor trees + Lunatic strategies](./INSPIRATIONS.md#ii2-supervisor-trees--restart-strategies)

**Author surface:**

```lisp
(defcaixa
  :nome      "my-app-root"
  :versao    "0.1.0"
  :kind      Supervisor
  :estrategia OneForOne
  :max-restarts 5
  :restart-window "60s"
  :children  ((:caixa "worker"      :versao "^0.1" :restart Permanent)
              (:caixa "cache-server":versao "^0.1" :restart Transient)
              (:caixa "scratch-job" :versao "^0.1" :restart Temporary)))
```

**Files (caixa-core):**

- `caixa-core/src/kind.rs` — add `CaixaKind::Supervisor` variant
- `caixa-core/src/supervisor.rs` — new (~180 LoC):
  - `RestartStrategy` enum (`OneForOne | OneForAll | RestForOne | SimpleOneForOne`)
  - `RestartPolicy` enum (`Permanent | Temporary | Transient`)
  - `SupervisorSpec` — strategy + max_restarts + window + children
  - `ChildSpec` — child caixa ref + version range + restart policy
- `caixa-core/src/manifest.rs` — `Caixa::estrategia: Option<RestartStrategy>`,
  `Caixa::children: Vec<ChildSpec>`, etc.
- `caixa-core/src/layout.rs` — Supervisor kind invariants:
  - must have `:children` (at least one)
  - must NOT have `:bibliotecas` / `:exe` / `:servicos`
  - `:max-restarts` and `:restart-window` required when strategy != SimpleOneForOne

**Tests:**
- parse `:kind Supervisor` + 3 children
- each restart strategy variant
- each restart policy variant
- layout: Supervisor without :children → error
- layout: Supervisor with :servicos → error
- round-trip all four strategies
- reject invalid `:restart-window` (non-duration)

**Estimate:** 180 LoC + 12 tests. ~2 hours.

**Dependency:** caixa-core::CaixaKind (extend), caixa-core::layout (extend).

**Downstream wiring (M3):** wasm-operator reconciles a Supervisor by
walking its `children`, ensuring each child ComputeUnit exists,
applying the restart-strategy on child failure (informer pattern).

---

### M2.5 ★ wasm-engine: enforce LimitsSpec at instantiate

**Source pattern:** [`INSPIRATIONS.md` §III.1, §III.3 — Lunatic per-process sandboxing + fuel preemption](./INSPIRATIONS.md#iii3-preemption-at-fuel-boundaries)

**Files (wasm-engine):**

- `wasm-engine/src/limits.rs` — new (~70 LoC): convert `LimitsSpec`
  → `wasmtime::StoreLimits` + fuel + wall-clock timer
- `wasm-engine/src/instance.rs` — instantiate w/ limits applied

**Tests:**
- a wasm component over fuel → trap caught
- wall-clock exceeded → trap caught
- memory cap → grow_memory returns error
- absent LimitsSpec → unbounded (today's behavior; no regression)

**Estimate:** 70 LoC + 4 tests. ~1.5 hours (when wasm-engine has the rest of its M2).

**Dependency:** M2.1 (caixa-core::LimitsSpec lands first), and
wasm-engine reaching the point where it actually instantiates wasm.

---

### M2.6 ★ tatara-lisp: `(restart-case …)` + `(handler-bind …)`

**Source pattern:** [`INSPIRATIONS.md` §V.1 — Common Lisp conditions/restarts](./INSPIRATIONS.md#v1-conditions-and-restarts--error-handling-as-a-recoverable-choice)

**Author surface:**

```lisp
(handler-bind ((file-error (lambda (c)
                              (invoke-restart 'use-default))))
  (restart-case (error 'file-error :path "/etc/whatever")
    (use-default () (default-config))
    (retry      () (load-config))
    (abort      () nil)))
```

**Files (tatara-lisp):**

- `tatara-lisp/src/condition.rs` — new (~250 LoC): `Condition` typed
  ADT, restart frame stack, `signal_condition`, `invoke_restart`,
  TataraDomain integration
- `tatara-lisp/src/macros.rs` — `(handler-bind …)` and
  `(restart-case …)` reader macros
- `tatara-lisp/tests/conditions.rs` — round-trip + behavior

**Tests:**
- restart picked by handler → execution branches accordingly
- no handler → propagate up (default behavior)
- multiple restarts available → handler picks by name
- restart-case with computation in body → executes if no condition signaled
- `invoke-restart 'foo` from inside handler → unwinds to matching restart

**Estimate:** 250 LoC + 10 tests. ~3 hours.

**Dependency:** tatara-lisp's reader-macro infrastructure (already
exists per caixa-ast).

**Downstream wiring (M3):** wasm-engine WASI errors signal typed
conditions with restarts (`(retry)`, `(use-cached)`, `(abort)`); the
supervisor (or REPL operator) picks.

---

### M2.7 ★ wasm-operator: in-place pod resize via SSA patch

**Source pattern:** [`RUNTIME-PATTERNS.md` D1 — In-place pod resize](./RUNTIME-PATTERNS.md#d1-in-place-pod-resize-k8s-127-)

**Files (wasm-operator):**

- `wasm-operator/src/reconcile.rs` — extend `ensure_deployment` to
  detect `spec.resources.{requests,limits}` drift and emit a
  server-side-apply patch against the Pod's `resources` subresource
  (K8s 1.27+ InPlacePodVerticalScaling).
- `wasm-operator/src/resize.rs` — new (~120 LoC): the typed patch
  logic + feature-gate detection.

**Tests:**
- mock kube-rs server: ComputeUnit with new memory limit → patch sent
- feature gate disabled → fallback to recreate strategy
- invalid resize (e.g. lower memory than current usage) → reject + emit event
- resize-during-rolling-restart → idempotent

**Estimate:** 120 LoC + 6 tests. ~2 hours.

**Dependency:** wasm-operator's ensure_deployment must be real first
(currently stubbed). Block until that lands.

---

### M2 summary

| item | scope | tests | hours | dep |
|---|---|---|---|---|
| 2.1 LimitsSpec | 120 LoC caixa-core | 8 | 1 | — |
| 2.2 BehaviorSpec | 80 LoC caixa-core | 7 | 1 | — |
| 2.3 UpgradeFromSpec | 150 LoC caixa-core | 10 | 1.5 | — |
| 2.4 SupervisorSpec + CaixaKind | 180 LoC caixa-core | 12 | 2 | M2.1 |
| 2.5 wasm-engine LimitsSpec enforce | 70 LoC engine | 4 | 1.5 | M2.1 + engine real |
| 2.6 conditions/restarts | 250 LoC tatara-lisp | 10 | 3 | — |
| 2.7 in-place resize | 120 LoC operator | 6 | 2 | operator real |

**Critical path:** items 2.1 + 2.2 + 2.3 + 2.4 are *additive caixa-core
type extensions*. Independent, can land in parallel. ~5 hours of focused
typed Rust + ~37 tests. Lands the foundation; everything in M3-M5 builds
on it.

---

## M3 ☆ — distribution + introspection (mid-year)

### M3.1 ☆ tatara-lisp `(remoto :node n :do …)` form + caixa-resolver peer-cluster

**Source pattern:** [`INSPIRATIONS.md` §II.5 + §IV.3 — distributed Erlang + Unison Remote](./INSPIRATIONS.md#ii5-distributed-erlang--epmd-cookies-transparent-message-passing)

- ~300 LoC in tatara-lisp (form + reader macro + WIT-channel typing)
- ~200 LoC in caixa-resolver (peer-cluster fetcher + content-addressed cache)
- 12 tests

### M3.2 ☆ Unison-style Adaptive Service Graph Compression

**Source pattern:** [`INSPIRATIONS.md` §IV.2 — Adaptive compression](./INSPIRATIONS.md#iv2-adaptive-service-graph-compression)

- ~400 LoC in wasm-engine (call-pattern profiler emitting typed `:hot-pair` events)
- ~250 LoC in wasm-operator (consume `:hot-pair`, apply pod-affinity overlays)
- ~150 LoC in caixa-flux (render co-location hint into HelmRelease)
- 15 tests
- E2E: two ComputeUnits + simulated traffic → wasm-operator co-schedules within 60s

### M3.3 ☆ BEAM mailbox + link/monitor as WIT-typed channels

**Source pattern:** [`INSPIRATIONS.md` §X — wasm-engine as the BEAM](./INSPIRATIONS.md#the-wasm-engine-as-the-beam)

- ~500 LoC in wasm-engine (typed mailbox per component instance,
  selective receive via WIT pattern, link/monitor edges in lacre graph)
- 20 tests

### M3.4 ☆ Pony-style typed capabilities (static layer over `:capabilities`)

**Source pattern:** [`INSPIRATIONS.md` §VII — Pony reference capabilities](./INSPIRATIONS.md#vii-pony--capability-secure-typed-actors)

- ~350 LoC in caixa-core (static cap-type checker, 6-cap vocabulary)
- ~100 LoC in cse-lint (new check: typed-capability-violation)
- 15 tests

### M3.5 ☆ CRIU live migration of wasm-engine pods

**Source pattern:** [`INSPIRATIONS.md` §IX.1 — CRIU](./INSPIRATIONS.md#ix1-criu--checkpointrestore-in-userspace)

- ~250 LoC in wasm-engine (checkpoint endpoint + restore boot path)
- ~150 LoC in wasm-operator (Migration CR controller)
- 8 tests + 1 E2E with two-node K3s

### M3.6 ☆ eBPF tracer crate (`caixa-bpf`)

**Source pattern:** [`INSPIRATIONS.md` §IX.2 — eBPF](./INSPIRATIONS.md#ix2-ebpf--programmable-kernel-observability)

- ~600 LoC new crate caixa-bpf (Aya-based)
- ~200 LoC integration in wasm-engine
- 12 tests

### M3.7 ☆ `feira repl --restore <hash>` (CL save-lisp-and-die analog)

**Source pattern:** [`INSPIRATIONS.md` §V.3 — save-lisp-and-die](./INSPIRATIONS.md#v3-save-lisp-and-die--image-dump)

- ~200 LoC in caixa-feira (new `repl` subcommand)
- ~150 LoC in tatara-script (image-dump + restore primitives)
- 8 tests

---

## M4 ◇ — operator surface (late year)

### M4.1 ◇ `feira upgrade --to <versao>` (release_handler analog)

- ~200 LoC in caixa-feira

### M4.2 ◇ Live REPL into running pod (`feira repl --pod <name>`)

- ~250 LoC in caixa-feira (kube-rs exec + tatara-script REPL bridge)
- ~150 LoC in wasm-engine (control socket for eval)

### M4.3 ◇ Hash-sharded ComputeUnit (Akka cluster sharding)

- ~300 LoC in wasm-operator (sharding controller)
- ~150 LoC in caixa-flux (shard config)

### M4.4 ◇ Virtual actors (Orleans grain placement)

- already partially via KEDA `minReplicas: 0`; formalize with
  `:activation Always | OnDemand | Singleton` slot
- ~120 LoC in caixa-core, ~100 LoC in wasm-operator

### M4.5 ◇ Event sourcing via tameshi log replay

- ~400 LoC in tameshi (log replay engine)
- ~250 LoC in wasm-operator (replay flag)
- ~150 LoC in caixa-feira (`feira replay --to <hash>`)

---

## M5 ⬡ — self-hosting (next year)

The substrate runs itself under itself:

- caixa-operator becomes a caixa Servico
- caixa-resolver becomes a caixa Servico
- wasm-engine becomes a caixa Servico
- wasm-operator becomes a caixa Servico
- lareira-tatara-stack hosts them all

**Effort:** mostly migration, not new code. The tests are the
existing ones — they pass when the substrate runs itself.

**The recursive loop closes:** the program that runs the program that
runs the program is the program. `caixa-operator-as-caixa.lisp` is
authored, lacre-pinned, attested via tameshi, deployed via feira deploy
to the cluster that's reconciling itself by running it.

---

## Test strategy across all milestones

Three layers of tests for every absorbed pattern:

### Layer 1 — Unit tests (caixa-core / tatara-lisp / wasm-operator / wasm-engine)

- type round-trips: `from_lisp` ∘ `to_lisp` = identity
- parse rejects: malformed inputs return typed errors
- behavior: each enum variant exercises its branch
- regression: existing manifests parse unchanged (additive-only schema)

### Layer 2 — Integration tests (per-crate)

- caixa-helm renders a chart with all M2 slots populated → valid YAML
- caixa-flux renders a programs.yaml entry with all M2 slots
- wasm-operator reconciles a Supervisor CR → emits child ComputeUnits
- tatara-script evaluates `(restart-case …)` correctly

### Layer 3 — End-to-end (against rio, after M2 is solid)

- author a caixa with M2 slots populated
- `feira chart` succeeds, output passes `helm template`
- `feira deploy --cluster rio --apply` updates the fleet manifest
- ComputeUnit appears in cluster, all M2 slots propagate to operator

### CI integration

Every test layer runs in CI:

- L1 — runs on every commit via `cargo test` in each repo's
  `caixa-validate.yml` workflow
- L2 — runs in `caixa-publish.yml` after build, before push
- L3 — runs nightly via a scheduled workflow against a kindling K3s
  cluster spun up just for the test (CRD load → caixa apply → assert
  + teardown)

---

## What we deliver, in priority order

1. **M2.1-M2.4** caixa-core type extensions (the foundation)
2. **M2.6** tatara-lisp conditions/restarts (unblocks supervisor recovery semantics)
3. **M2.7** wasm-operator real ensure_deployment + in-place resize
4. **M2.5** wasm-engine LimitsSpec enforcement (after operator real)
5. **M3.x** in priority order (3.1 distribution → 3.4 typed capabilities → 3.2 Adaptive → 3.5 CRIU → 3.3 mailbox → 3.6 eBPF → 3.7 image dump)
6. **M4.x** operator surface (mostly compositional, fast once primitives exist)
7. **M5** self-hosting (migrations, no new code)

Each item moves at least one pattern from `○ aspirational` to `★
load-bearing` in [`RUNTIME-PATTERNS.md`](./RUNTIME-PATTERNS.md). The
roadmap's job is to *retire* the ○ list.

---

## Right now, this turn

Landing M2.1, M2.2, M2.3, M2.4 — the four typed-substrate
foundations as additive optional slots in caixa-core. ~530 LoC + 37
tests. All-or-nothing semantically (the four slots together describe
"a caixa that runs as part of an OTP supervised tree with
sandboxed limits and declarative upgrade instructions"). Together they
are the typed shape of every future caixa.
