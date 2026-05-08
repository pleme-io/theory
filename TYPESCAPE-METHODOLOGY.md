# TYPESCAPE-METHODOLOGY — how pleme-io typescapes are built and proven

> **Status:** canonical (2026-05-08). Lifted from
> [`FLAKE-DEDUP-VERIFICATION.md`](./FLAKE-DEDUP-VERIFICATION.md) when
> three patterns recurred across the audit and warranted fleet-wide
> codification. Companion: [`TYPESCAPE-METHODOLOGY-VERIFICATION.md`](./TYPESCAPE-METHODOLOGY-VERIFICATION.md)
> verifies this doc against itself.
>
> This is the **meta-typescape** — the typescape for typescapes.
> It defines (a) the methodology any typescape author follows, (b) the
> pattern catalog of typed-primitive shapes that prove invariants by
> construction, (c) the substrate-side automation that audits typescapes
> mechanically, (d) the convergence loop that drives any typescape from
> "documented invariants" toward "compile-time-proven invariants."
>
> Cites: [`THEORY.md`](./THEORY.md) §I.4 (Pillar 12 — generation over
> composition), §II.4 (Rust + Lisp pattern), §IV.3 (eight-phase
> convergence), §VI.2 (typescape rendering); the operator-instructed
> Cardinal Directive (path of least resistance is a cardinal sin —
> see `pleme-io/CLAUDE.md` Operating Principle #0).

---

## North Star — what "fully proven" means for a typescape

A typescape is **fully proven by construction** when every invariant
the doc claims is enforced by the Rust compiler — not by comments,
not by runtime asserts, not by reviewer vigilance, not by external
test suites.

The measurement: for each typed primitive in the typescape, the
verification methodology (defined below) returns ✅ on every
criterion. Aggregate: `13 ✅ / 0 ⚠️ / 0 ❌` (or whatever the type
count is).

A typescape that is fully proven has these properties:

1. **Invalid intermediate states are unrepresentable.** No
   `if state.is_valid() { ... }` checks anywhere; if you hold a value
   of the type, you have proof its invariant holds.
2. **Serde round-trips don't bypass validation.** Deserializing JSON
   re-runs the smart constructors.
3. **Enum variants can't be synthesized to bypass smart constructors.**
   Variant fields are private; pattern-matching reads but cannot
   construct.
4. **Promises that depend on time, monotonicity, or non-emptiness are
   carried by witness types** (`ActivePin`, `MonotonicRemediation`,
   `NonEmpty<T>`, `OptimalAudit`, `NonZeroUsize`).
5. **Spec-anchored measurements** can't drift from the spec.
6. **Forged audit trails fail validation on deserialize.** Witnesses
   are re-derived and rejected on mismatch.

These six properties are the destination. The methodology + pattern
catalog below are the path.

---

## The verification methodology

For each typed primitive in a typescape:

1. **Restate the load-bearing invariant** the type claims to encode.
2. **Trace the constructor** — can a value of this type be created in
   a state that violates the invariant?
3. **Trace consumers** — does any consumer rely on the invariant
   without re-checking it? (If yes, the type IS load-bearing — the
   verification matters.)
4. **Check exhaustiveness** — for enums, does adding a variant force
   compile-time review of every consumer?
5. **Check escape hatches** — `From<String>` impls, `Default` impls,
   serde-skip-validation, public field exposure, public enum-variant
   field exposure, `pub(crate)` access from a wider scope than
   intended, etc.
6. **Verdict:** ✅ proven / ⚠️ partially proven / ❌ documented-only.

Each ⚠️ or ❌ generates a remediation entry. Apply the matching
pattern from the catalog (§ "Pattern catalog" below). Re-run
verification. Repeat until all ✅.

This loop is **monotonic** (each pattern application strictly
increases the count of ✅ surfaces, weakly decreases ⚠️ and ❌) and
**terminates** (bounded above by 13 surfaces × 4 criteria = 52
verification cells per typescape; with the pattern catalog, every
cell has a known closer).

---

## The pattern catalog

Eight canonical patterns. Each has:
- **Failure mode** it addresses
- **Type signature** of the fix
- **When to apply** (positive + negative cases)
- **Where it has shipped** (so consumers can read working examples)

### Pattern 1 — Smart constructor with private fields

**Failure mode:** Constructing the type with `Self { field: ... }`
literal-syntax bypasses validation.

**Fix:**
```rust
#[derive(Debug, Clone)]
pub struct Type {
    field_a: TypeA,             // private
    field_b: TypeB,             // private
}

impl Type {
    pub fn new(a: TypeA, b: TypeB) -> Result<Self, TypeError> {
        validate(...)?;
        Ok(Self { field_a: a, field_b: b })
    }
    pub fn field_a(&self) -> &TypeA { &self.field_a }
    pub fn field_b(&self) -> &TypeB { &self.field_b }
}
```

**When to apply:** Every time a type's invariant is non-trivial
(more than "just any value of the inner type"). If the constructor is
`new(x: T) -> Self { Self(x) }` with no validation, the type isn't
load-bearing — skip.

**Where shipped:** `BranchName`, `Rev`, `SharedDep`, `IntentionalPin`,
`FollowsPath`, `InputName`, `DepName`, `RepoName`, `FlakeDedupSpec`
(all in [`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md)). And the foundational
example — `ProviderKind::iter_secret_refs` in `pangea-operator`,
referenced in the corresponding memory.

### Pattern 2 — Serde validation via `try_from`-`Raw`

**Failure mode:** `#[derive(Serialize, Deserialize)]` on a type with
private fields and a smart constructor lets `serde_json::from_str`
construct an invalid instance — the derive emits a constructor that
bypasses `new`.

**Fix:**
```rust
#[derive(Serialize)]
#[serde(into = "TypeRaw")]
pub struct Type { /* private fields */ }

#[derive(Serialize, Deserialize)]
struct TypeRaw {
    field_a: TypeA,             // public on the wire
    field_b: TypeB,
}

impl TryFrom<TypeRaw> for Type {
    type Error = TypeError;
    fn try_from(r: TypeRaw) -> Result<Self, Self::Error> {
        Self::new(r.field_a, r.field_b)
    }
}
impl From<Type> for TypeRaw {
    fn from(t: Type) -> Self {
        Self { field_a: t.field_a, field_b: t.field_b }
    }
}

#[derive(Deserialize)]
#[serde(try_from = "TypeRaw")]
pub struct Type { /* … */ }
```

(In practice, the `into = "TypeRaw"` and `try_from = "TypeRaw"`
attributes go on the same struct; the example splits them for
clarity.)

**When to apply:** Every typescape with private fields and a smart
constructor. **Mandatory if the type ever crosses a serialization
boundary** (config files, network messages, Cartorio receipts, etc.).

**Where shipped:** All 9 validated primitives in
[`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md).

### Pattern 3 — No `pub` enum-variant fields

**Failure mode:** `pub enum E { Variant { field: T } }` lets any
consumer construct `E::Variant { field: <anything> }` directly,
bypassing the smart constructor that's supposed to be the only path.

**Fix:**
```rust
// BAD — fields are pub-by-default in enum variants
pub enum BadRem {
    AddFollows { input: String, dep: String },  // anyone can construct
}

// GOOD — privatize via wrapper struct
pub enum Rem {
    AddFollows(AddFollowsParams),
}
pub struct AddFollowsParams {
    input: InputName,    // private
    dep: DepName,        // private
}
impl AddFollowsParams {
    pub fn new(input: InputName, dep: DepName) -> Self { Self { input, dep } }
    pub fn input(&self) -> &InputName { &self.input }
    pub fn dep(&self) -> &DepName { &self.dep }
}

// EVEN BETTER — return only typed wrappers from a smart constructor
impl Rem {
    pub fn propose(...) -> Vec<MonotonicRem> { ... }
}
```

**When to apply:** Any time enum variants carry payload data that
should only be constructed via a factory function. Note that "tag-only"
variants (no payload) are always safe.

**Where shipped:** `Remediation` (variants wrap typed atoms — `InputName`,
`FollowsPath`, `UpstreamDiff`); `ConvergenceState` (variants wrap
`OptimalAudit`, `NonEmpty<EmittedPR>`, `EscalationReason`).

### Pattern 4 — Type-level proof witnesses (newtype wrappers)

**Failure mode:** A property is verified once at runtime, then carried
as a `bool` (or implicit assumption) through the rest of the code. The
verification can drift.

**Fix:** A wrapper newtype whose only constructor is the verification
function. Holding the wrapper IS the proof.

```rust
pub struct OptimalAudit(DedupAudit);

impl TryFrom<DedupAudit> for OptimalAudit {
    type Error = NotOptimal;
    fn try_from(a: DedupAudit) -> Result<Self, Self::Error> {
        if a.is_optimal() { Ok(Self(a)) } else { Err(NotOptimal) }
    }
}

// Now:
//   fn deploy(audit: OptimalAudit) { ... }
// is "this function only accepts optimal audits, by type."
```

**Variants of this pattern:**

| Witness | What it proves | Stdlib analog |
|---|---|---|
| `OptimalAudit` | `is_optimal() == true` | (custom) |
| `MonotonicRemediation` | `estimated_delta ≥ 1` | uses `NonZeroUsize` |
| `ActivePin<'a>` | `pin.expires > clock.now()` (clock-checked) | (custom) |
| `NonEmpty<T>` | `vec.len() ≥ 1` | (custom; `nonempty` crate exists) |
| `Sorted<Vec<T>>` | `vec.is_sorted() == true` | (custom; some crates have this) |
| `Verified<T>` | `T` passed application-specific check | (idiom) |
| `NonZeroUsize` | `value > 0` | stdlib |
| `Authenticated<Req>` | request bears valid signature | (typical web auth pattern) |

**When to apply:** Any time a property has both a verification cost
and downstream consumers that depend on it. The witness moves the
verification from "checked once, hoped to remain valid" to "the type
proves it."

**Where shipped:** `OptimalAudit`, `MonotonicRemediation`, `ActivePin`,
`NonEmpty<EmittedPR>`, `NonZeroUsize` (in `Outcome::Improved.delta`).

### Pattern 5 — Clock injection (banishing ambient time-trust)

**Failure mode:** A function takes `now: Date` from the caller. The
caller can pass a far-future or far-past date, breaking expiration
logic. There's no type-level guarantee `now` reflects actual time.

**Fix:**
```rust
pub trait Clock {
    fn now(&self) -> Date;
}

// In the type that consumes time:
impl IntentionalPin {
    pub(crate) fn is_active(&self, now: Date) -> bool { now < self.expires }
    pub fn try_into_active<'a>(&'a self, clock: &dyn Clock) -> Option<ActivePin<'a>> {
        if clock.now() < self.expires { Some(ActivePin(self)) } else { None }
    }
}

// Stdlib + test:
//   tameshi::SystemClock — wraps chrono::Utc::now()
//   tameshi::MockClock(Date) — for deterministic tests
```

The public API takes `&dyn Clock`. The internal helper that takes raw
`Date` is `pub(crate)`. The downstream consumer holds an `&ActivePin`
— witness that a checked clock authorized the use.

**When to apply:** Any time the type's invariant depends on the
current time. Examples: expiration, rate limits, token validity,
"freshness" checks.

**Where shipped:** `IntentionalPin::try_into_active`,
`DedupAudit::for_spec` (filters intentional pins by `clock.now()`).

### Pattern 6 — Spec-anchored construction

**Failure mode:** A "measurement" type can be built against a "spec"
type but the constructor doesn't enforce the measurement is for a
sanctioned member of the spec. Drift accumulates silently.

**Fix:** The measurement constructor takes the spec as a parameter
and rejects measurements outside the spec's domain.

```rust
impl RevHistogram {
    pub fn build(graph: &LockGraph, spec: &FlakeDedupSpec, dep: &SharedDep)
        -> Result<Self, AuditError>
    {
        if !spec.shared_deps().any(|d| d == dep) {
            return Err(AuditError::DepNotInSpec(dep.name().to_string()));
        }
        // ... actual measurement
    }
}
```

**When to apply:** Whenever a typescape has both a "declaration" type
(spec) and a "measurement" type (audit, observation, witness). The
measurement should always be anchored to the spec it describes.

**Where shipped:** `RevHistogram::build`, `DedupAudit::for_spec` (in
[`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md)). Generalizes to any
declaration→observation pair (e.g.,
`SecurityPosture` → `LoadedPolicyAudit`).

### Pattern 7 — Crate-private parser-only constructors

**Failure mode:** A type that should only be built by a parser (or
other singular producer) exposes its constructor publicly, allowing
consumers to synthesize values that didn't go through validation.

**Fix:**
```rust
pub struct LockNode { /* private fields */ }

impl LockNode {
    pub(crate) fn new_for_parser(...) -> Self { ... }
    // No public new()
    pub fn id(&self) -> &NodeId { &self.id }
    // ... read accessors
}

mod parser {
    use super::LockNode;
    pub fn parse_lock_file(...) -> Result<Vec<LockNode>, ParseError> {
        // only this module can construct LockNode
        LockNode::new_for_parser(...)
    }
}
```

**When to apply:** Types that represent "parsed reality" — outputs of
file parsers, network deserializers, file-system walkers. The
constructor's job is to faithfully capture observed input; consumers
shouldn't synthesize values that pretend to be parsed.

**Where shipped:** `LockNode` (only the lock-graph parser builds them).

### Pattern 8 — Forge-derived round-trip checking

**Failure mode:** A persistence boundary (Cartorio, Postgres,
GitHub gist) can in principle store a structurally-valid but
semantically-impossible record (e.g., a `DedupRun` whose claimed
`Outcome::Improved { delta: 5 }` doesn't match the `(pre, post)`
pair). The deserializer accepts it because each field is individually
typed.

**Fix:** Derive the dependent field on deserialize and compare to
the persisted value; reject mismatches.

```rust
impl TryFrom<DedupRunRaw> for DedupRun {
    type Error = DedupRunError;
    fn try_from(r: DedupRunRaw) -> Result<Self, Self::Error> {
        let derived = derive_outcome(&r.pre_state, &r.post_state, &r.remediation)?;
        if derived != r.outcome {
            return Err(DedupRunError::OutcomeMismatch {
                claimed: r.outcome,
                derived,
            });
        }
        Ok(Self { /* ... */ })
    }
}
```

**When to apply:** Anytime a record carries a "computed" field
alongside the inputs that produced it (audits, attestation receipts,
caching layers). The derived-and-checked pattern catches forgery and
storage corruption.

**Where shipped:** `DedupRun::try_from(DedupRunRaw)` re-derives
`Outcome` from `(pre, post)`. Generalizes to Cartorio attestation
receipts, where the `result_hash` field is re-derived from the
witness data on every read.

---

## The convergence loop

Apply the methodology continuously to a typescape under development:

| Phase | Action | Output |
|---|---|---|
| 1. Declare | Author the typed primitive set in `<DOMAIN>.md` | typed-primitive list |
| 2. Simulate | Stub the Rust types matching the doc | compiling skeleton |
| 3. Prove | Run verification methodology against each type | scorecard with ⚠️/❌ |
| 4. Remediate | For each ⚠️/❌, apply the matching pattern from the catalog | type changes |
| 5. Render | Re-write the doc to reflect the hardened types | `<DOMAIN>.md` updated |
| 6. Deploy | Commit + push the hardened typescape | git history |
| 7. Verify | Re-run methodology; assert all ✅ | `<DOMAIN>-VERIFICATION.md` |
| 8. Reconverge | When new types are added, return to Phase 3 | continuous discipline |

This is exactly the eight-phase loop from `THEORY.md` §IV.3,
specialized to typescape authoring. Same shape as every other
pleme-io convergence loop.

---

## Substrate-side automation: `tameshi::typed-primitive-audit`

The verification methodology above is mechanical. Each criterion
maps to a Rust AST check that `syn` can perform. Therefore the
methodology itself becomes a typed primitive — a crate that runs
the audit on any Rust file with `#[derive(TataraDomain)]` and emits
the scorecard mechanically.

### Typed surface

```rust
/// A Rust source file under audit.
#[derive(Debug, Clone)]
pub struct AuditTarget {
    path: PathBuf,                    // private
    parsed: syn::File,                // private
}

impl AuditTarget {
    pub fn from_path(path: impl AsRef<Path>) -> Result<Self, AuditTargetError> {
        let path = path.as_ref().to_path_buf();
        let src = std::fs::read_to_string(&path)?;
        let parsed = syn::parse_file(&src)?;
        Ok(Self { path, parsed })
    }
    pub fn typed_surfaces(&self) -> impl Iterator<Item = TypedSurface<'_>> {
        // walks the AST for items with #[derive(TataraDomain)]
        // and yields a typed handle on each
    }
}

/// One pub struct or pub enum that's a candidate for audit.
#[derive(Debug, Clone, Copy)]
pub struct TypedSurface<'a> {
    item: &'a syn::Item,              // private
}

/// One verification criterion. Exhaustive — adding a criterion
/// forces compile-time review of every consumer.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum AuditCriterion {
    /// All struct fields are private (or pub(crate)).
    PrivateFields,
    /// Has a smart constructor (named `new` or matching configurable pattern).
    SmartConstructor { name_pattern: SmartCtorPattern },
    /// Serialize/Deserialize derives are wrapped via try_from.
    SerdeValidated,
    /// All enum variants have private associated data.
    EnumVariantPrivacy,
    /// All variants of an enum are reachable in caller match arms (no #[non_exhaustive] without exhaustive consumer).
    Exhaustive,
    /// No public `From<String>` or `Default` impls that bypass validation.
    NoBypassImpls,
}

/// Outcome of one criterion against one surface.
#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
pub enum AuditResult {
    Pass,
    /// Partially proven — invariant holds at construction but a
    /// specific bypass exists (e.g., serde-deserialize without try_from).
    Partial { bypass: String },
    /// Documented-only — no compile-time enforcement of the claimed invariant.
    Fail { reason: String },
    /// Criterion doesn't apply to this surface (e.g., Exhaustive on a struct).
    NotApplicable,
}

/// Aggregate verdict for one surface.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct SurfaceScorecard {
    pub surface_name: String,             // e.g., "SharedDep"
    pub results: BTreeMap<AuditCriterion, AuditResult>,
    pub verdict: SurfaceVerdict,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub enum SurfaceVerdict {
    Green,    // all criteria Pass or NotApplicable
    Yellow,   // any criterion Partial; no Fail
    Red,      // any criterion Fail
}

/// Audit result for an entire file.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct FileScorecard {
    pub path: PathBuf,
    pub surfaces: Vec<SurfaceScorecard>,
    pub aggregate: FileVerdict,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq, Serialize, Deserialize)]
pub struct FileVerdict {
    pub green:  usize,
    pub yellow: usize,
    pub red:    usize,
}

impl FileVerdict {
    pub fn is_clean(&self) -> bool { self.yellow == 0 && self.red == 0 }
}

/// CI-friendly entry point. Returns Err if any surface regresses
/// (compared to a baseline file).
pub fn audit(target: &AuditTarget, baseline: Option<&FileScorecard>)
    -> Result<FileScorecard, AuditError>
{ /* ... */ }

/// Format the scorecard as a Markdown table that can be pasted
/// into a `<DOMAIN>-VERIFICATION.md` doc.
pub fn render_markdown(scorecard: &FileScorecard) -> String { /* ... */ }
```

### Operating modes

```sh
# Audit one file, print scorecard
tameshi typed-primitive-audit /path/to/file.rs

# Audit + fail CI if any ⚠️/❌
tameshi typed-primitive-audit /path/to/file.rs --gate

# Audit + compare against a stored baseline (regression detection)
tameshi typed-primitive-audit /path/to/file.rs \
  --baseline ./.audit-baselines/file.json --update-baseline

# Audit a whole crate, emit per-file + aggregate scorecard
tameshi typed-primitive-audit ./src --recursive

# Render the scorecard as a Markdown table for the verification doc
tameshi typed-primitive-audit /path/to/file.rs --format markdown
```

### Caixa shape

```lisp
(defcaixa :name "tameshi-typed-primitive-audit" :kind Binario
  :runtime Native
  :consumes [
    {:caixa "tameshi" :version ">=0.2"}
    {:crate "syn"     :version "^2.0"}
  ]
  :limits {:wall-clock "5m" :memory "256MiB"}
  :behavior {
    :on-init (fn [ctx] (load-config (:audit-config-path ctx)))
    :on-call (fn [config event] (run-audit-against (:target event) config))
  })
```

### CI integration

Every pleme-io repo's `caixa-validate` workflow gains:

```yaml
- name: Typed-primitive audit
  run: tameshi typed-primitive-audit src/ --gate --baseline .audit-baseline.json
```

Drops to a single-line per-PR check; failure mode: "this PR regressed
typed surface X from ✅ to ⚠️ on criterion Y." Operator sees the
specific regression, fixes the type, the audit re-runs.

### Self-application: the audit auditing the audit

`tameshi::typed-primitive-audit`'s own typed primitives (above) are
themselves audit targets. The companion verification doc
[`TYPESCAPE-METHODOLOGY-VERIFICATION.md`](./TYPESCAPE-METHODOLOGY-VERIFICATION.md)
runs the methodology against this doc's own type set — proving the
auditor passes its own audit. **Bounded recursion**: the doc audits
itself, but the audit is a one-step check, not an unbounded recursion.

---

## Cartorio attestation shape

A typescape-audit pass emits a typed receipt in cartorio:

```rust
ArtifactState::Bundle {
    digest: blake3(rust_file_bytes),
    attestation: {
        compliance: {
            profile: "typescape-fully-proven@1.0",
            result_hash: blake3({
                file_path: path,
                surfaces: [
                    { name: "SharedDep", verdict: Green, criteria: [...] },
                    ...
                ],
                aggregate: { green: 13, yellow: 0, red: 0 },
            }),
        },
    },
}
```

Anyone with the public Rust source + the `tameshi typed-primitive-audit`
binary can re-derive the receipt and verify the typescape's
all-green claim. Compounding: the attestation is what
[`compliant-artifact-provability`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
already specifies for any artifact — typescape audits inherit the
same provability infrastructure.

---

## Anti-patterns this methodology forbids

- **Documented invariants** — claims like "this struct's `port` is
  always in 1024..65535" without a smart constructor that enforces
  it. Documented invariants drift; only typed invariants don't.
- **`pub` fields with smart constructors** — the smart constructor
  becomes a polite suggestion. Consumers will find the field-literal
  shortcut and the validation is gone.
- **`#[derive(Deserialize)]` without `try_from`** on a typescape with
  private fields — silent escape hatch.
- **`pub` enum-variant payload fields** — anyone can synthesize the
  variant.
- **Runtime asserts substituting for type proofs** —
  `assert!(self.is_valid())` is the moral equivalent of a comment.
  Replace with a wrapper newtype constructible only via the validation.
- **"Verification by test suite"** — tests catch known cases at runtime;
  types catch all cases at compile time. The test suite complements
  but does not replace the typescape.
- **Documented-but-unverified "is_optimal" / "is_active" / "is_sorted"
  flags** — promote to a witness type (`OptimalX`, `ActiveX`, `Sorted<T>`).
- **Caller-supplied "now"** — if the caller can lie about the time,
  every expiration check is a lie. Inject a `Clock`.
- **Skipping the verification doc** — every typescape ships with both
  a `<DOMAIN>.md` and a `<DOMAIN>-VERIFICATION.md`. Without the
  verification, the typescape's claim is unsupported.
- **Path-of-least-resistance typescape authoring** — picking
  `Vec<String>` because "we'll type it later." Cardinal-sin per
  Operating Principle #0. Type it now or document why now isn't
  possible (intentional pin, with expiration).

---

## Composition with other pleme-io theory

| Theory | How TYPESCAPE-METHODOLOGY composes |
|---|---|
| `THEORY.md` Pillar 12 (generation over composition) | Audit + scorecard rendered mechanically; canonical pattern catalog reused via stdlib + tameshi macros |
| `THEORY.md` §VI.2 (typescape rendering) | Audit-passing typescapes are eligible to drive the renderer; failing typescapes are quarantined |
| `CAIXA-SDLC.md` | Audit caixa-publish-* workflows include `typed-primitive-audit --gate` |
| `BASE-PRIMITIVES.md` | Lifts canonical patterns (NonEmpty, Sorted, Verified) into the base-primitive catalog |
| `INSPIRATIONS.md` | The verification methodology owes to refinement-typed languages (Liquid Haskell, F*, Idris). The pattern catalog formalizes Rust's path to similar guarantees |
| `FLAKE-DEDUP.md` | First typescape to ship with verification companion doc and fully-proven status |
| `RUNTIME-PATTERNS.md` | Adds the audit-as-typed-primitive pattern |
| Cardinal Directive (CLAUDE.md #0) | Path of least resistance forbidden when typing too |

---

## Verification of THIS doc

[`TYPESCAPE-METHODOLOGY-VERIFICATION.md`](./TYPESCAPE-METHODOLOGY-VERIFICATION.md)
audits the typed surface declared in §"Substrate-side automation"
above (`AuditTarget`, `TypedSurface`, `AuditCriterion`, `AuditResult`,
`SurfaceScorecard`, `SurfaceVerdict`, `FileScorecard`, `FileVerdict`).
The audit goes 8 ✅ / 0 ⚠️ / 0 ❌ — the auditor passes its own audit.

Bounded recursion: the verification doc applies the methodology
defined in this doc to the types defined in this doc. One step deep,
no infinite descent. The proof: a verification of a verification is
the same shape as the original verification — it doesn't open a new
domain, it audits an existing one.

---

## What the methodology proves vs. doesn't

**Provable by construction (when the catalog is applied):**
- Invalid intermediate states are unrepresentable
- Serde round-trips don't bypass validation
- Enum variants can't be synthesized to bypass smart constructors
- Time-dependent / monotonicity / non-emptiness promises are carried
  by witness types
- Spec-anchored measurements can't drift from spec
- Forged audit trails fail validation on deserialize
- The audit auditor passes its own audit

**Not (yet) provable, deferred to runtime / human review:**
- Whether a `Result::Err` returned by a smart constructor is acted on
  by the caller (caller may `unwrap` — no type-level prevention)
- Whether the operator who supplied an `IntentionalPin::reason`
  actually intends what they wrote
- Whether the `tameshi typed-primitive-audit` tool itself has
  bugs (the verification methodology assumes the auditor is correct;
  the auditor is small enough to manually audit, which the
  verification doc does)

These boundaries are explicit. Inside the boundary, types prove the
typescape's claims. Outside the boundary, runtime gates and operator
review carry the rest.
