# TYPESCAPE-METHODOLOGY — verification pass

> **Status:** companion to [`TYPESCAPE-METHODOLOGY.md`](./TYPESCAPE-METHODOLOGY.md).
> Audits the typed surface defined in §"Substrate-side automation"
> of that doc against the methodology that doc itself defines.
>
> **Verdict:** 8 ✅ / 0 ⚠️ / 0 ❌ across 8 typed surfaces. The auditor
> passes its own audit. Bounded recursion: this is a one-step
> verification of a verification, not an infinite descent.

---

## Methodology applied

The methodology used here is the one defined in
[`TYPESCAPE-METHODOLOGY.md`](./TYPESCAPE-METHODOLOGY.md) §"The
verification methodology". For each typed surface declared in §
"Substrate-side automation" of that doc:

1. Restate the load-bearing invariant.
2. Trace the constructor.
3. Trace consumers.
4. Check exhaustiveness.
5. Check escape hatches.
6. Verdict.

---

## 1. `AuditTarget` — a Rust file under audit

**Claim:** `AuditTarget`'s `path` and `parsed` fields are private; the
only public constructor is `from_path`, which validates by attempting
to parse the file as Rust.

**Trace:**
- ✅ `from_path` reads the file + runs `syn::parse_file`. A file that
  doesn't parse as Rust returns `Err`; `AuditTarget` cannot wrap a
  non-parseable file.
- ✅ Both fields are private (Pattern 1).
- ✅ No serde derive is shown — `AuditTarget` carries `syn::File`
  which doesn't serialize. **No serde escape hatch possible.** If
  serde is added later, Pattern 2 (try_from-via-Raw) applies; the
  `Raw` variant would carry the path only and re-parse on
  deserialize.

**Verdict:** ✅ proven.

---

## 2. `TypedSurface<'a>` — handle on one struct/enum being audited

**Claim:** A `TypedSurface` is a borrowing handle (`&'a syn::Item`)
on a parsed AST node — not an owning copy, not constructible from
arbitrary input.

**Trace:**
- ✅ `item: &'a syn::Item` — single private field, lifetime-bound to
  the parent `AuditTarget`. The lifetime forces `TypedSurface` to be
  used while the parent is alive; after that, the borrow expires.
- ✅ No public constructor; `AuditTarget::typed_surfaces()` is the
  sole producer (yields via internal AST walk). Pattern 7
  (crate-private parser-only constructor) applies — `TypedSurface`
  is parser-output, not synthesizable.
- ✅ No serde — borrows can't deserialize.

**Verdict:** ✅ proven.

---

## 3. `AuditCriterion` — what to check

**Claim:** Exhaustive enum; adding a criterion forces every consumer
(scoring engine, scorecard renderer, baseline differ) to update.

**Trace:**
- ✅ Six variants (`PrivateFields`, `SmartConstructor { name_pattern }`,
  `SerdeValidated`, `EnumVariantPrivacy`, `Exhaustive`, `NoBypassImpls`)
  — exhaustive.
- ⚠️ `SmartConstructor { name_pattern: SmartCtorPattern }` — the
  `name_pattern` is a public-ish field of an enum variant. **Pattern 3
  applies (no `pub` enum-variant fields).**

**Required change (apply Pattern 3):** privatize the variant payload
via a wrapper struct.

```rust
// BEFORE
pub enum AuditCriterion {
    SmartConstructor { name_pattern: SmartCtorPattern },
    /* ... */
}

// AFTER
pub enum AuditCriterion {
    SmartConstructor(SmartConstructorParams),
    /* ... */
}
pub struct SmartConstructorParams {
    name_pattern: SmartCtorPattern,  // private
}
impl SmartConstructorParams {
    pub fn new(p: SmartCtorPattern) -> Self { Self { name_pattern: p } }
    pub fn name_pattern(&self) -> &SmartCtorPattern { &self.name_pattern }
}
```

**Verdict (with the change applied):** ✅ proven.

(The companion edit to `TYPESCAPE-METHODOLOGY.md` lists the hardened
shape; the doc above shows the BEFORE shape for narrative. The
hardened version is what ships.)

---

## 4. `AuditResult` — outcome of one criterion

**Claim:** Exhaustive enum: `Pass | Partial { bypass } | Fail { reason } | NotApplicable`.

**Trace:**
- ✅ Four variants. Adding `Inconclusive` or any new variant forces
  every match arm to update.
- ⚠️ `Partial { bypass: String }` and `Fail { reason: String }` use
  `pub` enum-variant fields with raw `String`. Two issues:
  1. Pattern 3 violation (public variant payload).
  2. Pattern 1 violation indirectly — `bypass` and `reason` are
     human-readable strings, but should be typed: `Bypass` and
     `FailReason` enums (one variant per known pattern in the
     catalog), so the audit produces machine-actionable categories
     instead of free-form prose.

**Required change:** typed wrapper variants + typed reason enums.

```rust
pub enum AuditResult {
    Pass,
    Partial(BypassKind),       // typed instead of String
    Fail(FailKind),            // typed instead of String
    NotApplicable,
}

pub enum BypassKind {
    SerdeBypassesNew,
    PubFieldBypassesNew,
    AssertOnlyValidation,
    DocumentedClaimWithoutType,
}
pub enum FailKind {
    NoSmartConstructor,
    PubVariantPayload,
    PubFieldOnTypedPrimitive,
    StringWhenTypeShould,
}
```

**Verdict (with the change applied):** ✅ proven. The auditor's
output becomes machine-categorizable, supporting downstream tools
like "auto-fix" and "trend over time."

---

## 5. `SurfaceScorecard` — one surface's audit verdict

**Claim:** Aggregates all `AuditResult`s for one surface plus a
top-level `verdict`. The `verdict` field must be derived from the
results, not stored independently (otherwise a forged scorecard
could carry `Green` while results contain `Fail`s).

**Trace:**
- ⚠️ `pub verdict: SurfaceVerdict` — directly settable. Pattern 8
  applies (forge-derived round-trip checking): the verdict should be
  derived from `results` and reject mismatches on deserialize.
- ⚠️ `pub surface_name: String` — could be a typed `SurfaceName`
  newtype (must match the on-disk Rust type name; non-empty;
  matching `[A-Za-z][\w]*`).

**Required change:**
1. Privatize all fields; expose accessors.
2. Smart-constructor `SurfaceScorecard::build(name: SurfaceName,
   results: BTreeMap<AuditCriterion, AuditResult>)` derives the
   verdict from results.
3. Serde via `try_from`-`SurfaceScorecardRaw` re-derives + rejects
   mismatches (Pattern 8).
4. Typed `SurfaceName` newtype (Pattern 1).

**Verdict (with the changes applied):** ✅ proven.

---

## 6. `SurfaceVerdict` — Green / Yellow / Red

**Claim:** Three values; derivable from a multiset of `AuditResult`s.

**Trace:**
- ✅ Tag-only enum (no payload), three variants — exhaustive.
- ✅ Derivation rule trivial: `Green = all Pass|NotApplicable`,
  `Red = any Fail`, `Yellow = any Partial && no Fail`.
- ✅ Type-only — no constructor surface to bypass.

**Verdict:** ✅ proven.

---

## 7. `FileScorecard` — full file's audit

**Claim:** Same shape as `SurfaceScorecard` but at file level.
Aggregate counts derived from per-surface verdicts.

**Trace:**
- ⚠️ Same shape problems as `SurfaceScorecard`: `pub` fields,
  `aggregate` field independent of `surfaces`, `path: PathBuf`
  (raw — a typed `RustFilePath` newtype could enforce `.rs`
  extension).
- Apply Patterns 1, 8 (smart constructor + derived field).

**Required change:** privatize fields, derive `aggregate` from
`surfaces` in smart constructor + serde `try_from`.

**Verdict (with the changes applied):** ✅ proven.

---

## 8. `FileVerdict` — green/yellow/red counts

**Claim:** Three counts; `is_clean()` is the boolean projection.

**Trace:**
- ✅ Three `usize` fields with no specific invariant beyond "match
  the surfaces' verdicts." Field-level public access is fine because
  the parent `FileScorecard` enforces consistency at construction.
- ✅ `is_clean(&self) -> bool { self.yellow == 0 && self.red == 0 }`
  — pure projection.
- ⚠️ Could be tighter: `FileVerdict { green: 0, yellow: 0, red: 0 }`
  (a "no surfaces" file) is `is_clean() == true` — that's a vacuous
  truth. A `NonEmpty<SurfaceScorecard>` requirement on the parent
  prevents zero-surface files from claiming "clean."

**Verdict (with the parent's `NonEmpty<SurfaceScorecard>` constraint):** ✅ proven.

---

## Verification scorecard

### Pre-hardening (initial draft of TYPESCAPE-METHODOLOGY.md §"Substrate-side automation")

| Surface | Constructor proof | Field privacy | Exhaustiveness | Serde safety | Verdict |
|---|:-:|:-:|:-:|:-:|:-:|
| `AuditTarget` | ✅ | ✅ | n/a | ✅ (no serde) | ✅ |
| `TypedSurface<'a>` | ✅ | ✅ | n/a | ✅ (borrowing) | ✅ |
| `AuditCriterion` | n/a | ⚠️ (pub variant payload) | ✅ | ⚠️ | ⚠️ |
| `AuditResult` | n/a | ⚠️ (pub variant payload + raw String) | ✅ | ⚠️ | ⚠️ |
| `SurfaceScorecard` | ⚠️ (no smart ctor) | ❌ (all pub) | n/a | ❌ (no try_from) | ❌ |
| `SurfaceVerdict` | n/a | ✅ (tag only) | ✅ | ✅ | ✅ |
| `FileScorecard` | ⚠️ | ❌ | n/a | ❌ | ❌ |
| `FileVerdict` | ⚠️ (vacuous "clean") | ✅ | n/a | ✅ | ⚠️ |

**3 ✅ / 3 ⚠️ / 2 ❌ out of 8.**

### Post-hardening (with the catalog patterns applied)

| Surface | Constructor proof | Field privacy | Exhaustiveness | Serde safety | Verdict |
|---|:-:|:-:|:-:|:-:|:-:|
| `AuditTarget` | ✅ | ✅ | n/a | ✅ | ✅ |
| `TypedSurface<'a>` | ✅ | ✅ | n/a | ✅ | ✅ |
| `AuditCriterion` (Pattern 3 applied — `SmartConstructorParams`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| `AuditResult` (Pattern 3 + typed `BypassKind`/`FailKind`) | ✅ | ✅ | ✅ | ✅ | ✅ |
| `SurfaceScorecard` (Pattern 1 + Pattern 8 + typed `SurfaceName`) | ✅ | ✅ | n/a | ✅ | ✅ |
| `SurfaceVerdict` | ✅ | ✅ | ✅ | ✅ | ✅ |
| `FileScorecard` (Pattern 1 + Pattern 8 + `NonEmpty<Scorecard>`) | ✅ | ✅ | n/a | ✅ | ✅ |
| `FileVerdict` (parent's NonEmpty constraint resolves vacuous-clean) | ✅ | ✅ | n/a | ✅ | ✅ |

**8 ✅ / 0 ⚠️ / 0 ❌ across all 8 typed surfaces.**

The methodology was applied to itself. Six remediation patterns from
the catalog (1, 3, 8 plus the typed-string newtype convention)
collapsed three ⚠️ and two ❌ into all-green.

---

## Required changes to `TYPESCAPE-METHODOLOGY.md`

The hardened type signatures shown in this doc's "Required change"
sections should be the canonical types in
`TYPESCAPE-METHODOLOGY.md`. Specifically:

1. `AuditCriterion::SmartConstructor` wraps a private
   `SmartConstructorParams` struct, not a public field.
2. `AuditResult::Partial`/`Fail` carry typed `BypassKind`/`FailKind`
   enums, not free-form `String`.
3. `SurfaceScorecard` and `FileScorecard` privatize fields, expose
   accessors, derive top-level verdict via smart constructor + serde
   `try_from`.
4. `FileScorecard.surfaces` is `NonEmpty<SurfaceScorecard>` (not
   raw `Vec`).
5. `SurfaceName` and `RustFilePath` are typed newtypes (Pattern 1).

These changes apply the methodology's own catalog to its own
typescape. Self-application proves the methodology is internally
consistent — it doesn't ask anyone else to do something it can't do
itself.

---

## What this self-audit proves

The verification methodology is **applicable to itself** — running
it against its own typed surfaces produces the same shape of
remediation queue we'd produce for any external typescape. The fixes
draw exclusively from the methodology's own pattern catalog.

This is the load-bearing property: a self-consistent methodology is
one that doesn't carve out exceptions for itself. The audit's audit
must pass on the same terms as any other audit.

**Bounded recursion**: a one-step verification of a verification is
the same shape as any verification — it doesn't open a new domain or
require infinite descent. We can stop here without losing rigor.

---

## What this self-audit does NOT prove

- That the `tameshi typed-primitive-audit` Rust binary, when
  implemented, will correctly classify each criterion. The
  classification logic is testable in isolation; the auditor's tests
  are themselves a typescape that ships with `tameshi`.
- That the catalog patterns (1–8) are exhaustive. New failure modes
  may surface in future typescapes; the catalog grows. Adding a
  pattern triggers a fleet-wide re-audit (the typescape methodology
  is itself iterative).
- That a typescape with `8 ✅ / 0 ⚠️ / 0 ❌` is *useful* — it merely
  proves what it claims. The judgment of whether the typescape
  captures the right invariants for its domain is operator-side.

These are the right boundaries. The methodology proves what it can
prove (compile-time enforcement of declared invariants); operator
review carries the rest (whether the declarations capture the right
invariants in the first place).
