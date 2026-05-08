# FLAKE-DEDUP — typescape verification pass

> **Status:** companion to [`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md). This
> doc walks every typed primitive in that spec and answers: **is the
> claimed invariant actually proven by construction, or just
> documented?**
>
> A typescape's value is exactly the gap between "claimed proof" and
> "actual proof." Claimed-but-unproven invariants are silent debt; the
> compiler doesn't catch their violation, so they degrade to comments.
> This doc surfaces the gaps so they get fixed before the substrate
> ships.
>
> Each section: **Claim** → **Verification method** → **Verdict**
> (✅ proven / ⚠️ partially proven / ❌ documented-only) → **Required
> change** if not ✅.

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
   serde-skip-validation, public field exposure, etc.
6. **Verdict + required change**.

---

## 1. `SharedDep` — name + priority + constraint

**Claim:** Names are non-empty + path-separator-free; priority is
`Copy` (no silent mutation); constraint is exhaustive.

**Trace:**
- ✅ `SharedDep::new` rejects empty name + `/` containing names — the
  only public constructor, fields are private.
- ✅ `Priority` is a newtype around `u32` with `Copy`. Constants
  prevent magic-number drift.
- ⚠️ `ConstraintKind::SourceOnly` has no associated data → fine. But
  `BranchCompatible(BranchName)` — what's `BranchName`? Spec doesn't
  define it. **Gap: `BranchName` needs its own typed primitive
  (smart constructor, validation: matches Git ref name rules, no
  embedded refs/heads/ prefix).**
- ⚠️ `Serialize`/`Deserialize` derives — does deserialization bypass
  the `new` validation? With default `serde::Deserialize` derive on
  a struct with private fields, **the derive emits a constructor
  that bypasses `new`**. This is a real escape hatch.

**Verdict:** ⚠️ partially proven.

**Required change:**
1. Define `BranchName(String)` with its own smart constructor.
2. Add `#[serde(try_from = "SharedDepRaw")]` + an explicit
   `TryFrom<SharedDepRaw>` impl that calls `new` to enforce
   validation on deserialize. Same pattern for `Rev`, `IntentionalPin`,
   any type with private fields.

---

## 2. `Rev` — locked Git revision

**Claim:** Always 40 lowercase hex chars.

**Trace:**
- ✅ `Rev::parse` validates length + hex.
- ⚠️ Same deserialize escape hatch as above.
- ⚠️ `Rev(String)` — single-field tuple struct. `Rev::parse` lowercases
  the input but `Eq`/`Hash` on the inner string assumes already-lowercased.
  If the deserialize escape hatch fires, two `Rev`s with same content but
  different case would compare unequal. **Gap: serde validation OR
  enforce lowercase via custom `Hash` impl.**

**Verdict:** ⚠️ partially proven.

**Required change:** Same `try_from` deserialize wrapping. With that,
the case-mismatch issue can't arise.

---

## 3. `LockNode` and `LockGraph` — parsed flake.lock

**Claim:** Cycle-free; all input refs resolve; root present.

**Trace:**
- ✅ `LockGraph::from_file` runs `validate_no_cycles` (Tarjan SCC) +
  `validate_all_refs`. Both errors propagate as `LockParseError`.
- ✅ Fields are private; no setter exposes mutation post-construction.
- ⚠️ `LockNode` has `pub` fields. Anyone holding a `&mut LockGraph`
  could (today) mutate a `LockNode` without re-validating. The struct
  isn't *constructed* outside `from_file`, but `pub` fields invite
  drift.
- ⚠️ `pub fn nodes()` returns `impl Iterator<Item = &LockNode>` — read-
  only. ✅ on this one. But there's no `nodes_mut()` either, so the
  graph is effectively immutable after construction. **OK.**
- ⚠️ `InputResolution::Follows(Vec<String>)` — the path elements are
  freeform strings. A consumer could construct `Follows(vec!["nonexistent"])`
  and `LockGraph` couldn't tell because the validation was at
  parse-time only. **Gap: if consumers construct `LockNode`/`InputResolution`
  outside the parser, validation is bypassed.**

**Verdict:** ⚠️ partially proven.

**Required change:**
1. Make `LockNode` fields private; expose accessors only.
2. Make `InputResolution::Follows` carry a `FollowsPath` newtype,
   constructible only via `LockGraph::resolve_follows_path` which
   round-trips through validation. Or simpler: don't expose
   `InputResolution` as `pub` at all; expose only the resolved
   `&LockNode` via accessor methods.

---

## 4. `RevHistogram` and `DedupAudit` — measurement

**Claim:** Histograms keyed by validated `Rev`s; `is_optimal()` is a
pure boolean function over total fragmentation.

**Trace:**
- ✅ `BTreeMap<Rev, _>` — `Rev`'s validity comes from its own
  constructor, transitively guaranteed.
- ✅ `total_fragmentation` uses `saturating_sub` — no underflow panics.
- ✅ `is_optimal()` is `total_fragmentation() == 0` — no hidden state.
- ⚠️ `RevHistogram::build(graph, dep)` walks the graph but doesn't
  check that `dep` is actually in `graph`'s shared-deps spec. If the
  spec evolves (a dep is removed) but old code calls
  `RevHistogram::build` with the old dep, the result is "the histogram
  for an unspecified dep" — semantically meaningful but
  **typescape-wise a category error.** The type doesn't prove
  "this histogram is for a dep the spec sanctions."

**Verdict:** ✅ for what it claims; ⚠️ for the implicit "matches the
active spec" claim.

**Required change:** Optional. Either:
- Tighten: `DedupAudit::for_spec(graph: &LockGraph, spec: &FlakeDedupSpec)
  -> DedupAudit` so the build is anchored.
- Or accept the looseness as out-of-scope: `RevHistogram` is just
  measurement, the spec-anchor lives in `DedupRun`.

I lean toward tightening because it makes the spec the single source
of truth for "what we audit."

---

## 5. `IntentionalPin` — relief valve

**Claim:** Empty-reason pins unrepresentable; expired pins
unrepresentable at construction; expiration forces periodic
re-justification.

**Trace:**
- ✅ `IntentionalPin::new` checks both invariants.
- ⚠️ `is_active(now: Date)` — the operator passes `now`. If they pass
  a far-future date, expired pins falsely report active. **Gap: this
  is delegating clock-trust to the caller.**
- ⚠️ Serialize/deserialize escape hatch (same as #1, #2).

**Verdict:** ⚠️ partially proven.

**Required change:**
1. `try_from` deserialize wrapping with `expires_at_construction` re-
   validation against an injected clock.
2. Inject clock via `trait Clock { fn now(&self) -> Date; }` + a
   `tameshi::clock::SystemClock` default impl. `is_active(&self, &dyn
   Clock)` — caller can't lie about `now`.
3. Better still: drop the `is_active` method; force callers to use a
   typed `ActivePin<'a>` wrapper produced by passing the clock to
   `try_into_active`. Then `&[IntentionalPin]` filtered to active is a
   typed transition, not a boolean check.

---

## 6. `Remediation` — typed action

**Claim:** Cannot construct against a no-op state; applying strictly
decreases fragmentation OR fails (never worsens).

**Trace:**
- ⚠️ `Remediation::propose(graph, histogram)` returns `Vec<Self>` —
  but the variants `AddFollowsAtRoot`, `UpstreamPR`, etc. have public
  fields. A consumer can construct a `Remediation::AddFollowsAtRoot
  { direct_input: "doesnt-exist", dep_to_follow: "fenix" }` directly,
  bypassing `propose`.
- ⚠️ `UpstreamDiff::additions: Vec<String>` — freeform-string drift
  at the typed boundary. The doc says "typed list of `inputs.X.follows
  = "X";` lines" but a `Vec<String>` is not actually typed.
- ⚠️ The "monotonicity" claim ("applying strictly decreases
  fragmentation") is enforced only by `DedupRun::build` (post-hoc),
  NOT by the `Remediation` type. So a buggy `propose` impl could emit
  a `Remediation` that doesn't actually improve things; the constraint
  is detected at run-build time, not construction time.

**Verdict:** ❌ documented-only. The typescape claims "monotonic" but
the type system doesn't enforce it.

**Required change:**
1. Make `Remediation` variants' fields private; only `Remediation::propose`
   constructs them.
2. Replace `UpstreamDiff::additions: Vec<String>` with `Vec<FollowsLine>`
   where `FollowsLine { input: InputName, dep: DepName }` is typed.
3. Stronger: introduce a `MonotonicRemediation { remediation: Remediation,
   estimated_delta: usize }` newtype that `propose` returns. The estimated
   delta is the *type-level promise*; `DedupRun::build` validates it's
   met. Now the type carries the "I will improve things by ≥1" claim
   that `DedupRun` checks.

---

## 7. `DedupRun` — single-step audit witness

**Claim:** A run with `Outcome::Improved { delta: 0 }` is
unrepresentable; runs with post > pre fragmentation cannot be
constructed.

**Trace:**
- ✅ `DedupRun::build` does the math; rejects `delta = 0` (returns
  `NoOp` instead of `Improved`); errors on increasing fragmentation.
- ✅ Smart constructor; no public field exposure of `outcome`.
- ⚠️ Same serde escape hatch — a deserialized `DedupRun` could carry
  any state.
- ⚠️ The error case `DedupRunError::FragmentationIncreased` panics-
  in-spirit ("the deduper's invariant is broken; bail loudly") but
  the type just returns `Result`. Caller can `.ok()` and continue.

**Verdict:** ⚠️ partially proven.

**Required change:**
1. Same `try_from` deserialize wrapping.
2. Make `FragmentationIncreased` a separate panic variant or use a
   `Verified<DedupRun>` wrapper that proves the invariant held at
   construction time (callers can't ignore the failure).

---

## 8. `FlakeDedupSpec` + `ConvergenceState` — top-level contract

**Claim:** Three terminal states are exhaustive; `Converged { final_audit }`
requires `final_audit.is_optimal()`; `PendingUpstreamPRs` requires at
least one emitted PR.

**Trace:**
- ⚠️ The `Converged.final_audit.is_optimal()` constraint is asserted
  in the spec but I haven't shown the smart constructor that enforces
  it. The default-derived enum constructor lets anyone build
  `ConvergenceState::Converged { final_audit: <non-optimal>, runs:
  vec![] }` — the type allows the impossible.
- ⚠️ Same for `PendingUpstreamPRs { emitted_prs: vec![] }` — empty
  list isn't rejected by the type.

**Verdict:** ❌ documented-only.

**Required change:**
1. Make `ConvergenceState` variants' fields private.
2. Add smart constructors: `converged(audit: DedupAudit, runs: Vec<DedupRun>)
   -> Result<Self, ConvergenceError>` that returns `Err` if `audit`
   is non-optimal.
3. `pending_prs(audit, prs, runs) -> Result<Self, _>` that returns
   `Err` on empty `prs`.
4. Or richer: have a `OptimalAudit` newtype that wraps `DedupAudit`
   and is only constructible via `try_from(DedupAudit)` (which checks
   `is_optimal`). Then `Converged { final_audit: OptimalAudit, ... }`
   is type-level proof.

---

## Cross-cutting gaps

### A. Serde validation bypass

Affects: `SharedDep`, `Rev`, `IntentionalPin`, `DedupRun`,
`ConvergenceState`. 5 of 8 primitives.

**Fix:** Standard pattern — for every type with private fields and a
smart constructor, derive `Serialize` directly but use `try_from` for
deserialize. The `try_from` target is a "raw" struct with public
fields whose `TryFrom` impl calls the smart constructor.

```rust
#[derive(Serialize)]
#[serde(into = "SharedDepRaw")]
pub struct SharedDep { /* private */ }

#[derive(Deserialize)]
#[serde(try_from = "SharedDepRaw")]
pub struct SharedDep { /* … */ }

#[derive(Serialize, Deserialize)]
struct SharedDepRaw { name: String, priority: Priority, constraint: ConstraintKind }

impl TryFrom<SharedDepRaw> for SharedDep {
    type Error = SharedDepError;
    fn try_from(r: SharedDepRaw) -> Result<Self, Self::Error> {
        Self::new(r.name, r.priority, r.constraint)
    }
}
```

### B. Public-field escape hatches

Affects: `LockNode`, `Remediation` variants, `ConvergenceState`
variants. 3 primitives.

**Fix:** Make fields private + add accessor methods.

### C. `String`-everywhere boundaries

Affects: `UpstreamDiff::additions`, `EscalationReason::AmbiguousFix::candidate_paths`.
Less severe but still an opportunity to tighten.

**Fix:** Replace `Vec<String>` with `Vec<TypedLine>` where TypedLine
is the structural unit. Even if the underlying serialization is
strings, the type carries the schema.

### D. Post-hoc invariant enforcement

Affects: `Remediation` claims "monotonic" but enforces only via
`DedupRun::build`. The MonotonicRemediation wrapper newtype fixes this.

### E. Clock as ambient global

Affects: `IntentionalPin::is_active`. Caller passes `now`; nothing
prevents a lying clock.

**Fix:** Inject via `dyn Clock` trait + use a `tameshi`-style
authenticated time source (or at minimum a `SystemClock` with a
`MockClock` for tests). Better: drop ambient time entirely; the
caller has to actively claim "I am now using a checked clock".

---

## Verification scorecard

| Primitive | Constructor proof | Field privacy | Exhaustiveness | Serde safety | Verdict |
|---|:-:|:-:|:-:|:-:|:-:|
| `SharedDep` | ✅ | ✅ | ⚠️ (BranchName undef) | ❌ | ⚠️ |
| `Rev` | ✅ | ✅ | n/a | ❌ | ⚠️ |
| `LockGraph` | ✅ | ⚠️ (LockNode pub fields) | n/a | n/a (file-only) | ⚠️ |
| `LockNode` | n/a (built by parser) | ❌ | n/a | ❌ | ❌ |
| `RevHistogram` | ✅ | ✅ | n/a | ✅ | ✅ |
| `DedupAudit` | ✅ | ✅ | n/a | ✅ | ✅ |
| `IntentionalPin` | ✅ | ✅ | n/a | ❌ | ⚠️ |
| `Remediation` | ❌ (variants pub) | ❌ | ✅ | ❌ | ❌ |
| `UpstreamDiff` | ⚠️ (Vec<String>) | ❌ | n/a | ⚠️ | ⚠️ |
| `DedupRun` | ✅ | ✅ | n/a | ❌ | ⚠️ |
| `Outcome` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `FlakeDedupSpec` | ⚠️ | ❌ | n/a | ⚠️ | ⚠️ |
| `ConvergenceState` | ❌ | ❌ | ✅ | ❌ | ❌ |

3 ✅ / 7 ⚠️ / 3 ❌ out of 13 typed surfaces.

---

## What this verification reveals about the typescape pattern itself

The audit surfaces a recurring shape across pleme-io's typescapes:

1. **Smart constructors are necessary but insufficient** — without
   serde-validation wrapping, every typescape with private fields
   has a roundtrip escape hatch. The fix is mechanical (try_from
   pattern) but easy to forget.
2. **`pub` enum variant fields are anti-typescape** — they let any
   consumer construct any variant without going through the smart
   constructor. The audit caught this in `Remediation` and
   `ConvergenceState`; the same pattern likely affects other
   typescapes in pleme-io and merits a fleet-wide audit.
3. **Documented invariants ≠ proven invariants** — the gap is the
   point. Every typescape benefits from this verification pass; without
   it the doc claims drift from the actual compile-time guarantees.

**Suggested substrate-level addition:** a `tameshi::typed-primitive-audit`
crate that, given a Rust file, runs the same verification methodology
mechanically:

- Parses the file (via `syn`)
- For every `pub struct` / `pub enum` with `#[derive(TataraDomain)]`,
  checks: are fields private? Is there a smart constructor? Are
  Serialize/Deserialize derives wrapped via try_from? Are enum
  variants' associated data private?
- Emits a verification scorecard like the one above
- Fails CI on regression

That's the typed compounding move — the verification methodology
itself becomes a typed primitive that audits future typescapes
mechanically. The cardinal directive applied to typescape authoring.

---

## Action items extracted from this verification

In commit-able order, smallest-blast-radius first:

1. **Add try_from serde wrapping** to all 5 affected primitives
   (`SharedDep`, `Rev`, `IntentionalPin`, `DedupRun`,
   `ConvergenceState`). Mechanical, ~50 LOC.

2. **Privatize fields** in `LockNode`, `Remediation` variants,
   `ConvergenceState` variants. Add accessor methods. Mechanical,
   ~30 LOC.

3. **Define `BranchName`** typed primitive with its own smart
   constructor. ~15 LOC.

4. **Replace `UpstreamDiff::additions: Vec<String>`** with
   `Vec<FollowsLine>` typed. ~20 LOC + a renderer to/from string.

5. **Define `MonotonicRemediation` wrapper** + change
   `Remediation::propose` return type to `Vec<MonotonicRemediation>`.
   Forces the monotonicity proof to live in the type, not the run-build
   post-condition. ~30 LOC.

6. **Define `OptimalAudit` newtype**, change
   `ConvergenceState::Converged.final_audit` to use it. The type
   carries "is_optimal == true" by construction. ~20 LOC.

7. **Inject `Clock` dependency** in `IntentionalPin::is_active`.
   ~30 LOC + a `MockClock` for tests.

8. **Tighten `RevHistogram::build` to require a `&FlakeDedupSpec`**
   so histograms are spec-anchored. ~10 LOC.

After all 8 changes the scorecard goes to 13 ✅ / 0 ⚠️ / 0 ❌.

Total: ~200 LOC of mechanical hardening, no public-API breakage if
done before the deduper ships. The verification pass paid for itself
by surfacing changes that would have shipped as silent debt.
