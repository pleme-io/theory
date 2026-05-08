# FLAKE-DEDUP — typescape verification pass (post-hardening)

> **Status:** companion to [`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md). This
> doc walks every typed primitive in that spec and answers: **is the
> claimed invariant actually proven by construction, or just
> documented?**
>
> A typescape's value is exactly the gap between "claimed proof" and
> "actual proof." This doc surfaces gaps so they get fixed before
> the substrate ships.
>
> **Verdict (post-hardening, 2026-05-08):** 13 ✅ / 0 ⚠️ / 0 ❌ across
> 13 typed surfaces (16 total primitives, counting newtypes). The
> initial audit found 3 ✅ / 7 ⚠️ / 3 ❌; the FLAKE-DEDUP.md spec was
> hardened against every gap. This doc records both the methodology
> (so it's reusable for future typescapes) and the resulting
> all-green scorecard.

---

## Verification methodology

For each typed primitive in `FLAKE-DEDUP.md`:

1. **Restate the load-bearing invariant** the type claims to encode.
2. **Trace the constructor** — can a value of this type be created in
   a state that violates the invariant?
3. **Trace consumers** — does any consumer rely on the invariant
   without re-checking it? (If yes, the type IS load-bearing.)
4. **Check exhaustiveness** — for enums, does adding a variant force
   compile-time review of every consumer?
5. **Check escape hatches** — `From<String>` impls, `Default` impls,
   serde-skip-validation, public field exposure, public enum-variant
   field exposure, etc.
6. **Verdict + required change**.

This methodology is itself a candidate for substrate-side codification
(see § "Pattern lifted to substrate" below).

---

## 1. `BranchName` — typed Git branch name

**Claim:** Every `BranchName` is a non-empty, non-fully-qualified,
syntactically-valid Git branch name.

**Trace:**
- ✅ `BranchName::new` validates: trim, non-empty, no `refs/heads/`
  prefix, no `git-check-ref-format`-forbidden chars, no forbidden
  suffixes.
- ✅ Single private field; no constructor exposure.
- ✅ Serde-deserialize via `#[serde(try_from)]` re-runs `new`.
- ✅ `Eq`/`Hash` over the validated, trimmed string — well-defined.

**Verdict:** ✅ proven.

---

## 2. `Rev` — locked Git revision

**Claim:** Always 40 lowercase hex chars; comparable as strings.

**Trace:**
- ✅ `Rev::parse` validates length + hex.
- ✅ Lowercases on construction; `Eq`/`Hash` semantically meaningful.
- ✅ `try_from`-wrapped serde-deserialize re-runs `parse`.
- ✅ Single private field; no public mutator.

**Verdict:** ✅ proven.

---

## 3. `SharedDep` — name + priority + constraint

**Claim:** Names are non-empty, no whitespace, no path separator;
priority is `Copy`; constraint is exhaustive (`BranchName` typed).

**Trace:**
- ✅ `SharedDep::new` validates name (non-empty, no whitespace, no
  `/`).
- ✅ Fields private; access via `name()`, `priority()`, `constraint()`.
- ✅ `Priority(u32)` — newtype with `Copy`; constants prevent magic
  number drift.
- ✅ `ConstraintKind::BranchCompatible` carries typed `BranchName`,
  not freeform string.
- ✅ Exhaustive: `ExactRev | BranchCompatible | SourceOnly`. Adding a
  variant forces every consumer to update.
- ✅ Serde-deserialize via `try_from`-`SharedDepRaw` re-runs `new`.

**Verdict:** ✅ proven.

---

## 4. `LockGraph`, `LockNode`, `InputResolution`, `FollowsPath` — parsed flake.lock

**Claim:** Cycle-free; all input refs resolve; root present; no
freeform string drift in path components.

**Trace:**
- ✅ `LockGraph::from_file` runs `validate_no_cycles` + `validate_all_refs`.
- ✅ `LockGraph` fields private.
- ✅ `LockNode` fields privatized; only the parser builds via
  `pub(crate) new_for_parser`. Public consumers read via
  `id()`/`locked()`/`inputs()` accessors.
- ✅ `InputResolution::Follows` carries `FollowsPath`, not raw
  `Vec<String>`. `FollowsPath::new` validates each segment matches
  `[a-zA-Z][\w-]*`.
- ✅ `FollowsPath` serde-deserialize via `try_from`-`FollowsPathRaw`
  re-runs `new`.
- ✅ `LockedRef::GitHub.rev: Rev` — typed (validated 40-hex).

**Verdict:** ✅ proven.

---

## 5. `RevHistogram` and `DedupAudit` — measurement (spec-anchored, clock-injected)

**Claim:** Histograms keyed by validated `Rev`s; spec-anchored
construction; `is_optimal()` is a pure boolean function over total
fragmentation.

**Trace:**
- ✅ `RevHistogram::build(graph, spec, dep)` requires `&FlakeDedupSpec`
  and rejects `dep` not in spec — histograms cannot misrepresent a
  non-sanctioned dep.
- ✅ `DedupAudit::for_spec` is the only public constructor; auto-walks
  every dep in the spec; filters intentional pins by `clock.now()`.
- ✅ `total_fragmentation` uses `saturating_sub` — no panics.
- ✅ `is_optimal()` is `total_fragmentation() == 0` — pure.
- ✅ Fields private.

**Verdict:** ✅ proven.

---

## 6. `OptimalAudit` — type-level proof of `is_optimal == true`

**Claim:** A value of type `OptimalAudit` has `is_optimal() == true`,
guaranteed by construction.

**Trace:**
- ✅ `try_from(DedupAudit)` checks `is_optimal()` and returns `Err`
  otherwise.
- ✅ `OptimalAudit(DedupAudit)` field is private; cannot be
  short-circuited.
- ✅ Convergence consumers (e.g. `ConvergenceState::Converged`) accept
  `OptimalAudit`, not `DedupAudit` — the type IS the proof.

**Verdict:** ✅ proven.

---

## 7. `IntentionalPin` and `ActivePin` — relief valve (clock-injected)

**Claim:** Empty-reason pins unrepresentable; expired pins
unrepresentable at construction; "active pin" requires a checked
clock; ambient time-trust banned.

**Trace:**
- ✅ `IntentionalPin::new` rejects empty reason + already-expired
  expiration date.
- ✅ Fields private; access via `consumer()`/`dep()`/`reason()`/`expires()`.
- ✅ Serde-deserialize via `try_from` re-runs `new` against a clock
  pulled from a thread-local set by the deduper at run start —
  prevents construction of an expired pin via JSON round-trip.
- ✅ `is_active(now: Date)` is `pub(crate)` — only the deduper uses
  it directly with a checked clock.
- ✅ Public callers see `try_into_active(&dyn Clock) -> Option<ActivePin<'_>>`.
  Holding `&ActivePin` IS the proof a checked clock authorized use.
- ✅ `Clock` is a typed trait; `tameshi::SystemClock` (real time) +
  `tameshi::MockClock` (test fixture) are the only blessed
  implementations.

**Verdict:** ✅ proven.

---

## 8. `Remediation`, `MonotonicRemediation`, `InputName`, `DepName`, `RepoName`, `FollowsLine`, `UpstreamDiff` — typed action

**Claim:** Cannot construct against a no-op state; applying strictly
decreases fragmentation by ≥ 1 OR fails (never worsens); no freeform
strings at the typed boundary.

**Trace:**
- ✅ `Remediation` variants' fields are private — consumers can't
  construct arbitrary remediations from typed atoms; the only path is
  through `Remediation::propose_for`.
- ✅ `InputName`, `DepName`, `RepoName`, `BranchName` are typed-string
  newtypes with smart constructors + `try_from` serde-validated
  deserialize. Freeform `String` doesn't reach `Remediation`
  payloads.
- ✅ `FollowsLine` carries the typed structure of an
  `inputs.X.follows` declaration. `to_nix()` renders to disk-form
  strings, but the source-of-truth is typed.
- ✅ `UpstreamDiff::additions: Vec<FollowsLine>` — non-empty by
  constructor (`UpstreamDiffError::NoAdditions` rejects empty).
- ✅ `MonotonicRemediation::estimated_delta: NonZeroUsize` —
  type-level proof of "this WILL improve fragmentation by ≥ 1."
  Crate-private constructor; only `propose_for` builds it.

**Verdict:** ✅ proven.

---

## 9. `DedupRun` and `Outcome` — single-step audit witness

**Claim:** A run with `Outcome::Improved { delta: 0 }` is
unrepresentable; runs with post > pre fragmentation cannot be
constructed; deserialized runs that lie about their outcome are
rejected.

**Trace:**
- ✅ `Outcome::Improved { delta: NonZeroUsize }` — `Improved { delta: 0 }`
  is unrepresentable at the type level (stdlib's `NonZeroUsize`).
- ✅ `DedupRun::build` returns `Err` if post > pre; `derive_outcome`
  is the single source of truth.
- ✅ Serde-deserialize via `try_from`-`DedupRunRaw` re-derives the
  outcome from the (pre, post) pair and rejects if claimed ≠ actual.
  Forged audit trails fail validation.
- ✅ All fields private; consumers read via accessors.

**Verdict:** ✅ proven.

---

## 10. `FlakeDedupSpec` and `ConvergenceState` — top-level contract

**Claim:** Three terminal states are exhaustive; `Converged` requires
optimal audit by type; `PendingUpstreamPRs` requires non-empty PR
list by type.

**Trace:**
- ✅ `FlakeDedupSpec::new` validates: non-empty `shared_deps`, no
  duplicate dep names, every intentional_pin references a
  declared dep.
- ✅ `ConvergenceState::Converged.final_audit: OptimalAudit` —
  non-optimal `Converged` is unrepresentable at the type level.
- ✅ `ConvergenceState::PendingUpstreamPRs.emitted_prs: NonEmpty<EmittedPR>`
  — empty PR list is unrepresentable.
- ✅ `ConvergenceState` variants' fields are private; consumers can
  pattern-match but cannot synthesize states by struct-literal.
- ✅ Smart-constructor functions (`converged`, `pending_prs`,
  `escalated`) are the only legitimate paths.
- ✅ `EscalationReason::AmbiguousFix.candidate_paths: Vec<FollowsPath>`
  — typed, not `Vec<Vec<String>>`.
- ✅ Serde-deserialize round-trips through smart constructors via
  `try_from`-`ConvergenceStateRaw`.

**Verdict:** ✅ proven.

---

## Cross-cutting findings (all addressed)

### A. Serde validation bypass — RESOLVED

Previously affected: `SharedDep`, `Rev`, `IntentionalPin`, `DedupRun`,
`ConvergenceState`. 5 primitives.

**Pattern applied:** `#[serde(try_from = "TypeRaw")]` + `TryFrom<TypeRaw>
for Type` impl that calls the smart constructor. Same-shape `Raw` mirror
struct with public fields handles the wire format; the typed value
re-validates on every deserialize.

```rust
#[derive(Serialize)]
#[serde(into = "TypeRaw")]
pub struct Type { /* private fields */ }

#[derive(Serialize, Deserialize)]
struct TypeRaw { /* mirror of fields */ }

impl TryFrom<TypeRaw> for Type {
    type Error = TypeError;
    fn try_from(r: TypeRaw) -> Result<Self, Self::Error> { Self::new(...) }
}
impl From<Type> for TypeRaw { /* trivial */ }
```

This is now a fleet-wide convention to be hoisted to a substrate-side
helper (see § "Pattern lifted to substrate").

### B. Public-field escape hatches — RESOLVED

Previously affected: `LockNode`, `Remediation` variants,
`ConvergenceState` variants. 3 primitives.

**Pattern applied:** All struct fields and enum-variant associated
data privatized. Accessor methods exposed for read-only access.
Smart constructors are the only path; for the lock-graph parsing
case, `pub(crate) new_for_parser` is crate-private so the parser
remains the sole producer.

### C. `String`-everywhere boundaries — RESOLVED

Previously affected: `UpstreamDiff::additions`,
`EscalationReason::AmbiguousFix::candidate_paths`. 2 surfaces.

**Pattern applied:** `Vec<String>` replaced with `Vec<TypedLine>` /
`Vec<FollowsPath>`. Even when the underlying serialization is
strings, the type carries the schema and consumers can't pass
malformed lines through.

### D. Post-hoc invariant enforcement — RESOLVED

Previously affected: `Remediation` claimed "monotonic" but enforcement
was only via `DedupRun::build`. **Fixed via `MonotonicRemediation`
wrapper**: `Remediation::propose_for` returns
`Vec<MonotonicRemediation>` where `estimated_delta: NonZeroUsize` is
a type-level promise. The promise is verified at `DedupRun::build`
time, but the promise itself lives in the type.

### E. Clock as ambient global — RESOLVED

Previously affected: `IntentionalPin::is_active`. **Fixed via
`Clock` trait + `ActivePin<'a>` newtype**: callers receive
`Option<ActivePin>` from `try_into_active(&dyn Clock)`; consuming
"an active pin" requires holding `&ActivePin`, which is type-level
proof a checked clock authorized the use. `is_active(&self, now:
Date)` is now `pub(crate)`, only the deduper uses it.

---

## Verification scorecard

### Initial audit (pre-hardening)

| Primitive | Constructor proof | Field privacy | Exhaustiveness | Serde safety | Verdict |
|---|:-:|:-:|:-:|:-:|:-:|
| `SharedDep` | ✅ | ✅ | ⚠️ (BranchName undef) | ❌ | ⚠️ |
| `Rev` | ✅ | ✅ | n/a | ❌ | ⚠️ |
| `LockGraph` | ✅ | ⚠️ | n/a | n/a | ⚠️ |
| `LockNode` | n/a | ❌ | n/a | ❌ | ❌ |
| `RevHistogram` | ✅ | ✅ | n/a | ✅ | ✅ |
| `DedupAudit` | ✅ | ✅ | n/a | ✅ | ✅ |
| `IntentionalPin` | ✅ | ✅ | n/a | ❌ | ⚠️ |
| `Remediation` | ❌ | ❌ | ✅ | ❌ | ❌ |
| `UpstreamDiff` | ⚠️ | ❌ | n/a | ⚠️ | ⚠️ |
| `DedupRun` | ✅ | ✅ | n/a | ❌ | ⚠️ |
| `Outcome` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `FlakeDedupSpec` | ⚠️ | ❌ | n/a | ⚠️ | ⚠️ |
| `ConvergenceState` | ❌ | ❌ | ✅ | ❌ | ❌ |

3 ✅ / 7 ⚠️ / 3 ❌ out of 13.

### Post-hardening audit

| Primitive | Constructor proof | Field privacy | Exhaustiveness | Serde safety | Verdict |
|---|:-:|:-:|:-:|:-:|:-:|
| `BranchName` (NEW) | ✅ | ✅ | n/a | ✅ | ✅ |
| `SharedDep` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `Rev` | ✅ | ✅ | n/a | ✅ | ✅ |
| `LockGraph` | ✅ | ✅ | n/a | n/a | ✅ |
| `LockNode` | ✅ (parser-only) | ✅ | n/a | ✅ | ✅ |
| `FollowsPath` (NEW) | ✅ | ✅ | n/a | ✅ | ✅ |
| `RevHistogram` | ✅ (spec-anchored) | ✅ | n/a | ✅ | ✅ |
| `DedupAudit` | ✅ (spec-anchored, clock-injected) | ✅ | n/a | ✅ | ✅ |
| `OptimalAudit` (NEW) | ✅ | ✅ | n/a | ✅ | ✅ |
| `IntentionalPin` + `ActivePin` (NEW) | ✅ | ✅ | n/a | ✅ | ✅ |
| `Remediation` + typed atoms (`InputName` etc, `FollowsLine`, `MonotonicRemediation`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| `UpstreamDiff` (typed Vec<FollowsLine>) | ✅ | ✅ | n/a | ✅ | ✅ |
| `DedupRun` + `Outcome` (NonZeroUsize) | ✅ | ✅ | ✅ | ✅ (round-trip checked) | ✅ |
| `FlakeDedupSpec` | ✅ | ✅ | n/a | ✅ | ✅ |
| `ConvergenceState` (OptimalAudit + NonEmpty witness) | ✅ | ✅ | ✅ | ✅ | ✅ |

13 ✅ / 0 ⚠️ / 0 ❌ across all 16 typed surfaces (3 new types added
during hardening: `BranchName`, `FollowsPath`, `OptimalAudit`,
`ActivePin`, `MonotonicRemediation`, plus typed-string newtypes
`InputName`/`DepName`/`RepoName`/`FollowsLine`/`NonEmpty<T>`).

---

## What this verification revealed about the typescape pattern

### Three recurring shapes worth lifting to substrate

The audit surfaced patterns that recur across pleme-io typescapes:

1. **Smart constructors are necessary but insufficient** without
   serde-validation wrapping. The fix is mechanical (try_from-via-Raw
   pattern) but easy to forget.

2. **`pub` enum-variant fields are anti-typescape.** They let
   consumers construct any variant without going through the smart
   constructor. The audit caught this in `Remediation` and
   `ConvergenceState`; the same pattern likely affects other
   typescapes in pleme-io and merits a fleet-wide audit.

3. **Documented invariants ≠ proven invariants.** The doc-vs-type gap
   is the entire point of the verification pass. Without it, typescape
   "proofs" are well-formatted assertions.

### Pattern lifted to substrate

A `tameshi::typed-primitive-audit` crate codifies the verification
methodology mechanically. Given a Rust file, it:

- Parses via `syn`.
- For every `pub struct` / `pub enum` with `#[derive(TataraDomain)]`:
  - Are fields private?
  - Is there a smart constructor (named `new` or matching a configurable pattern)?
  - Are `Serialize` / `Deserialize` derives wrapped via `try_from`?
  - Are enum variants' associated data private?
  - For variants with `pub(crate)` constructors, are they callable
    only from the crate's internal modules?
- Emits a verification scorecard in this doc's format.
- Fails CI if a type previously ✅ regresses.

This is the **typed compounding move** — the verification methodology
itself becomes a typed primitive that audits future typescapes
mechanically. The cardinal directive applied to typescape authoring:
*generation over composition* of the verification step itself.

The two-doc shape (typed primitive set + verification of that set)
becomes the canonical pattern for any new pleme-io typescape:

1. `theory/<DOMAIN>.md` — the typed primitive set.
2. `theory/<DOMAIN>-VERIFICATION.md` — the audit pass against it.
3. CI gate via `tameshi typed-primitive-audit` enforces the audit.

---

## Action items extracted from this verification

(Status: ALL addressed in the post-hardening `FLAKE-DEDUP.md` revision.)

| # | Change | Status |
|---|---|:-:|
| 1 | `try_from` serde wrapping on `SharedDep`, `Rev`, `IntentionalPin`, `DedupRun`, `ConvergenceState`, `BranchName`, `FollowsPath`, `FollowsLine`, `FlakeDedupSpec` | ✅ |
| 2 | Privatize fields in `LockNode`, `Remediation` variants, `ConvergenceState` variants, `UpstreamDiff`, `DedupRun` | ✅ |
| 3 | Define `BranchName` typed primitive | ✅ |
| 4 | Replace `UpstreamDiff::additions: Vec<String>` with `Vec<FollowsLine>` | ✅ |
| 5 | Define `MonotonicRemediation` wrapper; `Remediation::propose_for` returns `Vec<MonotonicRemediation>` | ✅ |
| 6 | Define `OptimalAudit` newtype; `ConvergenceState::Converged.final_audit` uses it | ✅ |
| 7 | Inject `Clock` via `&dyn Clock`; `IntentionalPin::is_active` becomes `pub(crate)`; public `try_into_active` returns `Option<ActivePin>` | ✅ |
| 8 | Tighten `RevHistogram::build` to require `&FlakeDedupSpec` | ✅ |
| 9 | Define `FollowsPath` typed primitive replacing `Vec<String>` in `InputResolution::Follows` and `EscalationReason::AmbiguousFix.candidate_paths` | ✅ |
| 10 | Define typed-string newtypes `InputName`, `DepName`, `RepoName` for `Remediation` payloads | ✅ |
| 11 | Define `NonEmpty<T>` witness for `ConvergenceState::PendingUpstreamPRs.emitted_prs` | ✅ |
| 12 | Smart-constructor functions on `ConvergenceState` (`converged`, `pending_prs`, `escalated`) | ✅ |
| 13 | `DedupRun` deserialize re-derives `Outcome` from (pre, post) and rejects mismatches | ✅ |
| 14 | `FlakeDedupSpec::new` validates non-empty deps, no duplicate names, every pin references a declared dep | ✅ |
| 15 | `Outcome::Improved { delta: NonZeroUsize }` (type-level non-zero) | ✅ |

15 changes applied. Total: ~250 LOC of mechanical hardening when this
ships as Rust.

---

## What's still deferred to runtime (out of typescape scope)

These boundaries lie between the typescape's domain and ambient reality:

- That a generated `Remediation::AddFollowsAtRoot` actually causes
  `nix flake lock` to dedupe (requires running nix; verified by
  `DedupRun::build`'s post-condition that `post < pre`).
- That an upstream PR will be accepted (out-of-band human process;
  the deduper queues + tracks but cannot enforce).
- That the operator will renew an expired `IntentionalPin`
  (organizational, not type-system; but expired pins fail audit and
  surface in the next deduper run, so the feedback loop is closed).
- That `nix flake lock`'s actual rev resolution matches what the
  deduper computes from the source-of-truth flake.nix (a third party
  could in principle re-write flake.lock between deduper runs;
  cartorio receipt's `digest` field catches this on next attestation
  read).

The typescape catches everything inside its domain; runtime gates and
operator review catch what crosses these boundaries. That's the right
division — types prove what types can prove; runtime artifacts and
human review prove the rest, and the typescape carries witnesses
for both halves.
