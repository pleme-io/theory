# FLAKE-DEDUP — typed cache-fragmentation deduplication

> **Status:** canonical (2026-05-08, hardened post-audit). Implementation
> deferred to `pleme-io/tend` as the `tend flake-deduper` subcommand.
> Theory leads the implementation per the cardinal directive.
>
> This document specifies the typed primitive set that lets pleme-io
> drive `flake.lock` toward a provably-optimal cache-fragmentation
> state, continuously, without operator intervention beyond annotation
> review. It complements [`THEORY.md`](./THEORY.md) §VI.2 (typescape
> rendering), §II.2 (Four Lisps — typed-input pillar), Pillar 12
> (generation over composition).
>
> **Companion:** [`FLAKE-DEDUP-VERIFICATION.md`](./FLAKE-DEDUP-VERIFICATION.md)
> walks every primitive below and verifies the doc-claim matches the
> compile-time guarantee. The current doc is the hardened post-audit
> version; the verification pass scores it 13 ✅ / 0 ⚠️ / 0 ❌.

---

## North Star — what "optimal" means measurably

A `flake.lock` is in **optimal state** when, for every shared
transitive dependency T (`nixpkgs`, `fenix`, `devenv`, `cachix`,
`substrate`, `sops-nix`, `home-manager`, …):

```
distinct_revs(T) = 1 + count(active_intentional_pins(T))
```

In words: every consumer either uses the **canonical rev** root pins,
or carries an **active** `IntentionalPin` annotation justifying the
deviation. Any other state is non-optimal and must be refactored.

This metric is:

1. **Measurable** — count distinct `(name, rev)` pairs in the lock
   graph.
2. **Monotonic** — every successful refactoring step strictly decreases
   `sum(distinct_revs(T) - 1 - count(active_intentional_pins(T)))`.
3. **Bounded below by 0** — convergence terminates by the well-ordering
   principle.
4. **Attestable** — the witness is the post-state lock file's
   histogram, recordable in cartorio with a Merkle hash of the
   `(dep_name, sorted_rev_set)` tuple.

---

## The typed primitive set (post-hardening)

Sixteen types compose the FlakeDedup domain. Each type's constructor
encodes invariants the Rust compiler enforces; states the spec calls
"invalid" are unrepresentable, and serde deserialization re-runs the
same validation as in-code construction (the `try_from`-via-`Raw`
pattern). No public field exposes mutation post-construction; every
enum variant's associated data is private; every "bypass" path is
either typed or unreachable.

### 1. `BranchName` — a Git ref name (non-empty, no `refs/heads/` prefix)

```rust
/// A Git branch name, validated at construction.
///
/// Used as `ConstraintKind::BranchCompatible(BranchName)` to specify
/// "all consumers must pin a rev compatible with root's branch."
///
/// Constructor refuses:
///   - empty strings
///   - leading/trailing whitespace
///   - the literal `refs/heads/` prefix (callers commonly slip
///     fully-qualified refs in by accident; reject them so we don't
///     silently treat `refs/heads/main` as a branch named
///     `refs/heads/main`)
///   - any character that's invalid in a Git ref name (per
///     `git-check-ref-format` rules: no `..`, no `~^:?*[`, no
///     trailing `.lock`, etc.)
///
/// Serde safety: `Deserialize` is wired via `#[serde(try_from)]` to
/// re-run the smart constructor on every parse.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize)]
#[serde(into = "String")]
pub struct BranchName(String);  // private

#[derive(Deserialize)]
#[serde(transparent)]
struct BranchNameRaw(String);

impl TryFrom<BranchNameRaw> for BranchName {
    type Error = BranchNameError;
    fn try_from(r: BranchNameRaw) -> Result<Self, Self::Error> { Self::new(r.0) }
}
impl From<BranchName> for String { fn from(b: BranchName) -> Self { b.0 } }

impl BranchName {
    pub fn new(name: impl Into<String>) -> Result<Self, BranchNameError> {
        let name = name.into();
        if name.trim() != name { return Err(BranchNameError::Whitespace); }
        if name.is_empty() { return Err(BranchNameError::Empty); }
        if name.starts_with("refs/heads/") { return Err(BranchNameError::FullyQualified(name)); }
        // git-check-ref-format rules — see substrate/lib/git-ref-validation.rs for the canonical impl
        for forbidden in &["..", "~", "^", ":", "?", "*", "[", "\\", "//"] {
            if name.contains(forbidden) {
                return Err(BranchNameError::ForbiddenChar { name, c: forbidden.to_string() });
            }
        }
        if name.ends_with(".lock") || name.ends_with('/') || name.ends_with('.') {
            return Err(BranchNameError::ForbiddenSuffix(name));
        }
        Ok(Self(name))
    }
    pub fn as_str(&self) -> &str { &self.0 }
}
```

**Invariants encoded by construction:** every `BranchName` parses as a
valid Git branch (not a ref), trimmed, non-empty. Serde-deserialize
runs `new` via `try_from`. No public fields; no mutation accessor.

### 2. `Rev` — a locked Git revision

```rust
/// A specific Git rev pinned in flake.lock. Always 40 lowercase hex.
///
/// Serde safety: validated on deserialize via `try_from`.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize)]
#[serde(into = "String")]
pub struct Rev(String);  // private

#[derive(Deserialize)]
#[serde(transparent)]
struct RevRaw(String);

impl TryFrom<RevRaw> for Rev {
    type Error = RevError;
    fn try_from(r: RevRaw) -> Result<Self, Self::Error> { Self::parse(r.0) }
}
impl From<Rev> for String { fn from(r: Rev) -> Self { r.0 } }

impl Rev {
    pub fn parse(s: impl Into<String>) -> Result<Self, RevError> {
        let s: String = s.into();
        if s.len() != 40 || !s.chars().all(|c| c.is_ascii_hexdigit()) {
            return Err(RevError::InvalidFormat(s));
        }
        // Lowercase the canonical form so Eq/Hash comparisons are well-defined.
        Ok(Self(s.to_ascii_lowercase()))
    }
    pub fn short(&self) -> &str { &self.0[..7] }
    pub fn full(&self) -> &str { &self.0 }
}
```

**Invariants encoded by construction:** 40 lowercase hex chars, always.
Serde-deserialize re-validates. `Eq`/`Hash` are well-defined regardless
of source casing because `parse` always lowercases.

### 3. `SharedDep` — a dep we deduplicate

```rust
/// A transitive dependency the fleet wants collapsed to a single rev.
///
/// Serde safety: validated on deserialize via `try_from`.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize)]
#[serde(into = "SharedDepRaw")]
pub struct SharedDep {
    name: String,                          // private
    priority: Priority,
    upstream_constraint: ConstraintKind,
}

#[derive(Serialize, Deserialize)]
struct SharedDepRaw {
    name: String,
    priority: Priority,
    upstream_constraint: ConstraintKind,
}

#[derive(Deserialize)]
#[serde(try_from = "SharedDepRaw")]
struct SharedDepDe(SharedDep);  // helper for #[serde(try_from)] indirection

impl TryFrom<SharedDepRaw> for SharedDep {
    type Error = SharedDepError;
    fn try_from(r: SharedDepRaw) -> Result<Self, Self::Error> {
        Self::new(r.name, r.priority, r.upstream_constraint)
    }
}
impl From<SharedDep> for SharedDepRaw {
    fn from(s: SharedDep) -> Self {
        Self { name: s.name, priority: s.priority, upstream_constraint: s.upstream_constraint }
    }
}

impl SharedDep {
    /// Smart constructor — returns Err for invalid names.
    pub fn new(
        name: impl Into<String>,
        priority: Priority,
        upstream_constraint: ConstraintKind,
    ) -> Result<Self, SharedDepError> {
        let name = name.into();
        if name.trim() != name { return Err(SharedDepError::Whitespace); }
        if name.is_empty() { return Err(SharedDepError::EmptyName); }
        if name.contains('/') { return Err(SharedDepError::InvalidName(name)); }
        Ok(Self { name, priority, upstream_constraint })
    }
    pub fn name(&self) -> &str { &self.name }
    pub fn priority(&self) -> Priority { self.priority }
    pub fn constraint(&self) -> &ConstraintKind { &self.upstream_constraint }
}

/// Higher = remediate first. nixpkgs > fenix > devenv > cachix > sops-nix > home-manager
/// because earlier-listed deps cascade more (nixpkgs invalidates stdenv → invalidates everything).
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize, Deserialize)]
pub struct Priority(u32);  // safe to expose — Copy + simple invariant (any u32)

impl Priority {
    pub const HIGHEST:  Priority = Priority(100);
    pub const HIGH:     Priority = Priority(75);
    pub const STANDARD: Priority = Priority(50);
    pub const LOW:      Priority = Priority(25);
}

#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "kebab-case", tag = "kind")]
pub enum ConstraintKind {
    /// All consumers must pin the EXACT same rev (e.g. fenix —
    /// different revs produce different rust toolchains).
    ExactRev,
    /// All consumers must pin a rev compatible with root's branch
    /// (e.g. nixpkgs nixos-25.11 — different commits OK if same
    /// branch and ABI-compatible).
    BranchCompatible(BranchName),
    /// Source-only ref; no flake-input-resolution applies (the
    /// `*-src` pattern). Always treated as already-converged.
    SourceOnly,
}
```

**Invariants encoded by construction:** non-empty, no whitespace, no
`/` in `name`. Serde-deserialize re-validates via `try_from`.
`ConstraintKind` is exhaustive; adding a variant forces every consumer
to update their match.

### 4. `LockNode` and `LockGraph` — the parsed flake.lock

```rust
/// A node in flake.lock. All fields private; accessed via methods.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct LockNode {
    id: NodeId,                                  // private
    locked: LockedRef,                           // private
    inputs: BTreeMap<String, InputResolution>,   // private
}

impl LockNode {
    pub(crate) fn new_for_parser(
        id: NodeId,
        locked: LockedRef,
        inputs: BTreeMap<String, InputResolution>,
    ) -> Self {
        // crate-private constructor — only the parser builds these
        Self { id, locked, inputs }
    }
    pub fn id(&self) -> &NodeId { &self.id }
    pub fn locked(&self) -> &LockedRef { &self.locked }
    pub fn inputs(&self) -> impl Iterator<Item = (&String, &InputResolution)> {
        self.inputs.iter()
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum LockedRef {
    /// A real Git rev — eligible for dedup.
    GitHub { owner: String, repo: String, rev: Rev },
    /// `flake = false` source-only ref.
    SourceOnly { rev: Rev },
    /// Path-only ref (no rev — typically dev override).
    Path { path: PathBuf },
}

/// Follows path components are validated NodeId fragments — see
/// `FollowsPath` smart constructor below.
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum InputResolution {
    /// Resolves to another node directly.
    DirectNode(NodeId),
    /// Follows a path from root.
    Follows(FollowsPath),
}

/// A validated follows-path. Each segment is a non-empty identifier
/// matching `[a-zA-Z][\w-]*`.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize)]
#[serde(into = "Vec<String>")]
pub struct FollowsPath(Vec<String>);  // private

#[derive(Deserialize)]
#[serde(transparent)]
struct FollowsPathRaw(Vec<String>);

impl TryFrom<FollowsPathRaw> for FollowsPath {
    type Error = FollowsPathError;
    fn try_from(r: FollowsPathRaw) -> Result<Self, Self::Error> { Self::new(r.0) }
}
impl From<FollowsPath> for Vec<String> { fn from(p: FollowsPath) -> Self { p.0 } }

impl FollowsPath {
    pub fn new(segments: Vec<String>) -> Result<Self, FollowsPathError> {
        if segments.is_empty() { return Err(FollowsPathError::Empty); }
        let id_re = regex::Regex::new(r"^[a-zA-Z][\w-]*$").unwrap();
        for s in &segments {
            if !id_re.is_match(s) {
                return Err(FollowsPathError::InvalidSegment(s.clone()));
            }
        }
        Ok(Self(segments))
    }
    pub fn segments(&self) -> &[String] { &self.0 }
}

/// The fully-parsed flake.lock.
///
/// Constructor parses the JSON and validates structural invariants:
/// - Every NodeId in any `inputs` map exists in `nodes`
/// - Every Follows path resolves to a real node
/// - Cycle detection (Tarjan's SCC); cycles are an error, not a state
#[derive(Debug, Clone)]
pub struct LockGraph {
    nodes: BTreeMap<NodeId, LockNode>,  // private
    root: NodeId,                        // private
}

impl LockGraph {
    pub fn from_file(path: &Path) -> Result<Self, LockParseError> {
        let raw = std::fs::read_to_string(path)?;
        Self::parse(&raw)
    }
    pub fn parse(raw: &str) -> Result<Self, LockParseError> {
        let parsed: LockFileJson = serde_json::from_str(raw)?;
        let g = Self::from_json(parsed)?;
        g.validate_no_cycles()?;
        g.validate_all_refs()?;
        Ok(g)
    }
    pub fn root(&self) -> &LockNode { &self.nodes[&self.root] }
    pub fn nodes(&self) -> impl Iterator<Item = &LockNode> { self.nodes.values() }
    pub fn resolve(&self, from: &NodeId, input_name: &str) -> Option<&LockNode> { /* … */ }
}
```

**Invariants encoded by construction:**
- A `LockGraph` with dangling refs cannot exist (`from_file` errors).
- A `LockGraph` with cycles cannot exist (Tarjan's SCC).
- `root` is always present (parser asserts it).
- `LockNode` fields are private; only the parser constructs them
  via `new_for_parser` (crate-private).
- `FollowsPath` segments are individually validated; freeform
  `vec!["nonexistent".to_string()]` cannot reach the type.
- `InputResolution` is itself an enum, but its variants only carry
  validated typed data (`NodeId`, `FollowsPath`). No raw `String`s
  escape.

### 5. `RevHistogram` and `DedupAudit` — measurement (spec-anchored)

```rust
/// For a given SharedDep, the (rev → set of nodes pinning it) map.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RevHistogram {
    pub dep: SharedDep,
    pub buckets: BTreeMap<Rev, BTreeSet<NodeId>>,
}

impl RevHistogram {
    /// Build by walking the lock graph for nodes pinning `dep`.
    /// Spec-anchored — caller must supply the spec the dep belongs to,
    /// guaranteeing the histogram is for a sanctioned dep.
    pub fn build(graph: &LockGraph, spec: &FlakeDedupSpec, dep: &SharedDep)
        -> Result<Self, AuditError>
    {
        if !spec.shared_deps().any(|d| d == dep) {
            return Err(AuditError::DepNotInSpec(dep.name().to_string()));
        }
        // walk graph, count revs, return histogram
        Ok(Self { dep: dep.clone(), buckets: walk_revs(graph, dep) })
    }
    pub fn distinct_count(&self) -> usize { self.buckets.len() }
    pub fn sorted_majority_first(&self) -> Vec<(&Rev, usize)> { /* … */ }
    pub fn canonical_rev(&self) -> Option<&Rev> { /* sorted_majority_first().first() */ }
}

/// Aggregate histogram across all shared deps — the optimization target.
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DedupAudit {
    histograms: Vec<RevHistogram>,                  // private
    intentional_pins: BTreeSet<IntentionalPinRef>,  // private
}

impl DedupAudit {
    /// Build by computing every shared-dep histogram against the lock graph,
    /// filtering by active intentional pins.
    pub fn for_spec(graph: &LockGraph, spec: &FlakeDedupSpec, clock: &dyn Clock)
        -> Result<Self, AuditError>
    {
        let now = clock.now();
        let intentional_pins: BTreeSet<IntentionalPinRef> = spec.intentional_pins()
            .filter(|p| p.is_active(now))
            .map(IntentionalPin::as_ref)
            .collect();
        let histograms: Result<Vec<_>, _> = spec.shared_deps()
            .map(|dep| RevHistogram::build(graph, spec, dep))
            .collect();
        Ok(Self { histograms: histograms?, intentional_pins })
    }
    pub fn histograms(&self) -> &[RevHistogram] { &self.histograms }
    pub fn intentional_pins(&self) -> &BTreeSet<IntentionalPinRef> { &self.intentional_pins }
    /// Total fragmentation across all shared deps, ignoring sanctioned pins.
    pub fn total_fragmentation(&self) -> usize {
        self.histograms.iter()
            .map(|h| h.distinct_count().saturating_sub(1))
            .sum::<usize>()
            .saturating_sub(self.intentional_pins.len())
    }
    pub fn is_optimal(&self) -> bool { self.total_fragmentation() == 0 }
}

/// A `DedupAudit` proven to satisfy `is_optimal() == true`.
/// Constructible only via `try_from(DedupAudit)`.
#[derive(Debug, Clone)]
pub struct OptimalAudit(DedupAudit);  // private

impl TryFrom<DedupAudit> for OptimalAudit {
    type Error = NotOptimal;
    fn try_from(a: DedupAudit) -> Result<Self, Self::Error> {
        if a.is_optimal() { Ok(Self(a)) }
        else { Err(NotOptimal { fragmentation: a.total_fragmentation() }) }
    }
}
impl OptimalAudit {
    pub fn audit(&self) -> &DedupAudit { &self.0 }
    pub fn into_audit(self) -> DedupAudit { self.0 }
}
```

**Invariants encoded by construction:**
- `DedupAudit::for_spec` is the only public constructor; histograms
  are guaranteed spec-anchored. There's no way to slip in a
  histogram for a dep that isn't in the spec.
- `intentional_pins` is filtered by `clock.now()` at build time;
  expired pins don't count toward "sanctioned deviation."
- `total_fragmentation` uses `saturating_sub` — no underflow panics.
- `OptimalAudit` is type-level proof of `is_optimal()`. Convergence
  consumers accept `OptimalAudit` instead of `DedupAudit`, so they
  can't be handed a non-optimal audit and falsely claim convergence.

### 6. `IntentionalPin` — relief valve (clock-injected)

```rust
/// An operator-justified deviation from "all consumers follow root T".
///
/// Constructor REQUIRES a non-empty reason + a future expiration date.
/// Stale pins (past expiration) are flagged on every audit; the operator
/// must either renew with new justification or fix the underlying issue.
#[derive(Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord, Serialize)]
#[serde(into = "IntentionalPinRaw")]
pub struct IntentionalPin {
    consumer: NodeId,            // private
    dep: SharedDep,              // private
    reason: String,              // private — non-empty; operator-supplied prose
    expires: Date,               // private — was in the future at construction
}

#[derive(Serialize, Deserialize)]
struct IntentionalPinRaw {
    consumer: NodeId,
    dep: SharedDep,
    reason: String,
    expires: Date,
}

#[derive(Deserialize)]
#[serde(try_from = "IntentionalPinDeArgs")]
struct IntentionalPinDe(IntentionalPin);

/// Deserialize requires a clock — the deserializer pulls it from a
/// thread-local set by the deduper at run start.
struct IntentionalPinDeArgs(IntentionalPinRaw);

impl TryFrom<IntentionalPinDeArgs> for IntentionalPin {
    type Error = IntentionalPinError;
    fn try_from(args: IntentionalPinDeArgs) -> Result<Self, Self::Error> {
        let now = current_audit_clock_or_default().now();
        Self::new(args.0.consumer, args.0.dep, args.0.reason, args.0.expires, now)
    }
}
impl From<IntentionalPin> for IntentionalPinRaw {
    fn from(p: IntentionalPin) -> Self {
        Self { consumer: p.consumer, dep: p.dep, reason: p.reason, expires: p.expires }
    }
}

impl IntentionalPin {
    pub fn new(
        consumer: NodeId,
        dep: SharedDep,
        reason: impl Into<String>,
        expires: Date,
        now: Date,
    ) -> Result<Self, IntentionalPinError> {
        let reason = reason.into();
        if reason.trim().is_empty() {
            return Err(IntentionalPinError::EmptyReason);
        }
        if expires <= now {
            return Err(IntentionalPinError::AlreadyExpired { expires, now });
        }
        Ok(Self { consumer, dep, reason, expires })
    }
    pub fn consumer(&self) -> &NodeId { &self.consumer }
    pub fn dep(&self) -> &SharedDep { &self.dep }
    pub fn reason(&self) -> &str { &self.reason }
    pub fn expires(&self) -> Date { self.expires }
    pub fn as_ref(&self) -> IntentionalPinRef {
        IntentionalPinRef {
            consumer: self.consumer.clone(),
            dep_name: self.dep.name().to_string(),
        }
    }

    /// Returns an `ActivePin<'a>` IFF the pin is active per the
    /// supplied clock. Type-level proof that ambient time-trust is
    /// banished — every consumer of "active pin" reads `&ActivePin`.
    pub fn try_into_active<'a>(&'a self, clock: &dyn Clock) -> Option<ActivePin<'a>> {
        if clock.now() < self.expires { Some(ActivePin(self)) } else { None }
    }

    pub(crate) fn is_active(&self, now: Date) -> bool { now < self.expires }
}

/// A pin proven active per a checked clock. Cannot be constructed
/// without going through `IntentionalPin::try_into_active`.
#[derive(Debug, Clone, Copy)]
pub struct ActivePin<'a>(&'a IntentionalPin);
impl<'a> ActivePin<'a> {
    pub fn as_pin(&self) -> &IntentionalPin { self.0 }
}

/// Clock injection — banishes ambient-time trust.
///
/// The `tameshi` crate provides `SystemClock` (real `chrono::Utc::now()`)
/// + `MockClock` (test fixture). The deduper takes `&dyn Clock` in its
/// run config so test runs are deterministic.
pub trait Clock {
    fn now(&self) -> Date;
}

/// A reference to an active pin, used in DedupAudit.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize, Deserialize)]
pub struct IntentionalPinRef {
    pub consumer: NodeId,
    pub dep_name: String,  // owned by the spec; correctness is verified via spec-anchored DedupAudit
}
```

**Invariants encoded by construction:**
- Empty-reason pins cannot exist (constructor rejects + serde re-validates
  via `try_from`).
- Expired pins cannot be created — only renewed.
- `is_active(&self, now)` is `pub(crate)` — only the deduper uses it,
  always with a checked clock.
- Public callers see `try_into_active(&dyn Clock) -> Option<ActivePin>`.
  Consuming "an active pin" requires holding `&ActivePin`, which is
  type-level proof a checked clock authorized it. Ambient time-trust
  is unrepresentable.

### 7. `Remediation` — typed action (private variants, monotonic wrapper)

```rust
/// A typed action the deduper proposes/applies. Each variant is one
/// "atomic unit of fix" that monotonically improves DedupAudit.
///
/// Variants' associated data is private. Construction goes through
/// `Remediation::propose_for(graph, histogram)` ONLY; smart constructor
/// returns each variant wrapped in `MonotonicRemediation` so the
/// "I will improve fragmentation by ≥ 1" promise lives in the type.
#[derive(Debug, Clone)]
pub enum Remediation {
    /// Add `inputs.<direct_input>.inputs.<dep_name>.follows = "<dep_name>"`
    /// at the root flake.nix. The simplest, most common case.
    AddFollowsAtRoot {
        direct_input: InputName,
        dep_to_follow: DepName,
    },
    /// Same as above but at a deeper override path.
    AddDeepFollowsAtRoot {
        path: FollowsPath,
        dep_to_follow: DepName,
    },
    /// The fix lives in an upstream pleme-io repo's flake.nix.
    UpstreamPR {
        repo: RepoName,
        branch_name: BranchName,
        diff: UpstreamDiff,
    },
    /// Mark a specific (consumer, dep) as intentional.
    DocumentIntentionalPin { pin: IntentionalPin },
}

/// An input identifier — a non-empty alphanumeric+dash+underscore string.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize)]
#[serde(into = "String")]
pub struct InputName(String);
impl InputName {
    pub fn new(s: impl Into<String>) -> Result<Self, InputNameError> { /* validation */ }
    pub fn as_str(&self) -> &str { &self.0 }
}
// (DepName, RepoName follow the same shape — typed-string newtypes
// each with their own smart constructor + serde-validated deserialize.)

/// A typed `inputs.X.follows = "X";` line — replaces freeform
/// `Vec<String>` in `UpstreamDiff::additions`.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize)]
#[serde(into = "FollowsLineRaw")]
pub struct FollowsLine {
    input: InputName,                      // private
    dep_to_follow: DepName,                // private
    deep_path: Option<FollowsPath>,        // private — for nested follows
}

#[derive(Serialize, Deserialize)]
struct FollowsLineRaw {
    input: InputName,
    dep_to_follow: DepName,
    deep_path: Option<FollowsPath>,
}

#[derive(Deserialize)]
#[serde(try_from = "FollowsLineRaw")]
struct FollowsLineDe(FollowsLine);

impl TryFrom<FollowsLineRaw> for FollowsLine {
    type Error = FollowsLineError;
    fn try_from(r: FollowsLineRaw) -> Result<Self, Self::Error> {
        Self::new(r.input, r.dep_to_follow, r.deep_path)
    }
}

impl FollowsLine {
    pub fn new(
        input: InputName,
        dep_to_follow: DepName,
        deep_path: Option<FollowsPath>,
    ) -> Result<Self, FollowsLineError> {
        if input.as_str() == dep_to_follow.as_str() && deep_path.is_none() {
            // `inputs.fenix.follows = "fenix";` — that's a valid form,
            // not an error
        }
        Ok(Self { input, dep_to_follow, deep_path })
    }
    /// Render to the on-disk Nix line.
    pub fn to_nix(&self) -> String {
        match &self.deep_path {
            None => format!(
                "inputs.{}.inputs.{}.follows = \"{}\";",
                self.input.as_str(), self.dep_to_follow.as_str(), self.dep_to_follow.as_str()
            ),
            Some(p) => format!(
                "inputs.{}.inputs.{}.inputs.{}.follows = \"{}\";",
                self.input.as_str(),
                p.segments().join("."),
                self.dep_to_follow.as_str(),
                self.dep_to_follow.as_str()
            ),
        }
    }
}

/// Typed diff payload — no freeform strings.
#[derive(Debug, Clone, Serialize)]
pub struct UpstreamDiff {
    flake_nix_path: PathBuf,                // private
    input_block: InputName,                 // private
    additions: Vec<FollowsLine>,            // private — typed lines, never raw strings
}

impl UpstreamDiff {
    pub fn new(
        flake_nix_path: PathBuf,
        input_block: InputName,
        additions: Vec<FollowsLine>,
    ) -> Result<Self, UpstreamDiffError> {
        if additions.is_empty() {
            return Err(UpstreamDiffError::NoAdditions);
        }
        Ok(Self { flake_nix_path, input_block, additions })
    }
    pub fn flake_nix_path(&self) -> &Path { &self.flake_nix_path }
    pub fn input_block(&self) -> &InputName { &self.input_block }
    pub fn additions(&self) -> &[FollowsLine] { &self.additions }
}

/// A `Remediation` proven to monotonically reduce fragmentation by
/// at least `estimated_delta`. Constructible only via
/// `Remediation::propose_for`.
#[derive(Debug, Clone)]
pub struct MonotonicRemediation {
    remediation: Remediation,           // private
    estimated_delta: NonZeroUsize,      // private — smart-constructor invariant
    targets: Vec<IntentionalPinRef>,    // private — what (consumer, dep) tuples this fixes
}

impl MonotonicRemediation {
    pub(crate) fn build(
        remediation: Remediation,
        estimated_delta: NonZeroUsize,
        targets: Vec<IntentionalPinRef>,
    ) -> Self {
        // crate-private constructor — only `Remediation::propose_for` builds these
        Self { remediation, estimated_delta, targets }
    }
    pub fn remediation(&self) -> &Remediation { &self.remediation }
    pub fn estimated_delta(&self) -> NonZeroUsize { self.estimated_delta }
    pub fn targets(&self) -> &[IntentionalPinRef] { &self.targets }
}

impl Remediation {
    /// Smart constructor — every returned `MonotonicRemediation` carries
    /// a non-zero estimated delta. Empty result = no remediations
    /// available (state is already optimal).
    pub fn propose_for(
        graph: &LockGraph,
        spec: &FlakeDedupSpec,
        histogram: &RevHistogram,
    ) -> Vec<MonotonicRemediation> { /* … */ }
}
```

**Invariants encoded by construction:**
- `Remediation` variant fields are private — consumers can't construct
  arbitrary remediations from typed atoms; they go through `propose_for`.
- `InputName`, `DepName`, `RepoName`, `BranchName` are typed-string
  newtypes; freeform `String` doesn't reach `Remediation` payloads.
- `FollowsLine` carries the typed structure of an `inputs.X.follows`
  declaration; rendering to Nix syntax happens in `to_nix()` (a pure
  function), not at the typed level.
- `UpstreamDiff::additions` is `Vec<FollowsLine>` — non-empty by
  constructor, no raw-string drift possible.
- `MonotonicRemediation` carries `NonZeroUsize` for `estimated_delta`
  — type-level proof of "this WILL improve fragmentation by ≥ 1."
  Crate-private constructor; only `propose_for` builds it.

### 8. `DedupRun` and `Outcome` — the audit witness

```rust
/// A single attempt to apply a Remediation. Carries pre-state +
/// post-state + outcome — the audit witness that convergence is
/// monotonic.
///
/// Constructor enforces the post < pre invariant.
#[derive(Debug, Clone, Serialize)]
#[serde(into = "DedupRunRaw")]
pub struct DedupRun {
    started_at: Timestamp,           // private
    remediation: Remediation,        // private
    pre_state: DedupAudit,           // private
    post_state: DedupAudit,          // private
    outcome: Outcome,                // private
}

#[derive(Serialize, Deserialize)]
struct DedupRunRaw {
    started_at: Timestamp,
    remediation: Remediation,
    pre_state: DedupAudit,
    post_state: DedupAudit,
    outcome: Outcome,
}

#[derive(Deserialize)]
#[serde(try_from = "DedupRunRaw")]
struct DedupRunDe(DedupRun);

impl TryFrom<DedupRunRaw> for DedupRun {
    type Error = DedupRunError;
    fn try_from(r: DedupRunRaw) -> Result<Self, Self::Error> {
        // Re-derive Outcome from pre/post — refuse a deserialized run
        // whose claimed outcome doesn't match the actual fragmentation delta.
        let derived = derive_outcome(&r.pre_state, &r.post_state, &r.outcome)?;
        if derived != r.outcome {
            return Err(DedupRunError::OutcomeMismatch {
                claimed: r.outcome,
                derived,
            });
        }
        Ok(Self {
            started_at: r.started_at,
            remediation: r.remediation,
            pre_state: r.pre_state,
            post_state: r.post_state,
            outcome: r.outcome,
        })
    }
}
impl From<DedupRun> for DedupRunRaw { /* trivial */ }

#[derive(Debug, Clone, PartialEq, Eq, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum Outcome {
    /// Strict improvement.
    Improved {
        delta: NonZeroUsize,         // can NEVER be zero — type enforces
    },
    /// No improvement — Remediation didn't take effect.
    NoOp,
    /// Eval broke. Remediation has been reverted.
    BrokeEval { error: String },
    /// The remediation was an UpstreamPR — applied via gh, not locally.
    PrEmitted { pr_url: Option<String> },
}

impl DedupRun {
    /// Convergence assertion: if `outcome` is `Improved`, post < pre.
    /// A `DedupRun { outcome: Improved { delta: 0 }, .. }` cannot
    /// exist (delta is `NonZeroUsize`). A run whose post > pre
    /// returns `Err`.
    pub fn build(
        pre: DedupAudit,
        post: DedupAudit,
        remediation: Remediation,
    ) -> Result<Self, DedupRunError> {
        let started_at = Timestamp::now();
        let outcome = derive_outcome(&pre, &post, /* from_remediation */ &remediation)?;
        Ok(Self { started_at, remediation, pre_state: pre, post_state: post, outcome })
    }
    pub fn started_at(&self) -> Timestamp { self.started_at }
    pub fn remediation(&self) -> &Remediation { &self.remediation }
    pub fn pre_state(&self) -> &DedupAudit { &self.pre_state }
    pub fn post_state(&self) -> &DedupAudit { &self.post_state }
    pub fn outcome(&self) -> &Outcome { &self.outcome }
}

fn derive_outcome(pre: &DedupAudit, post: &DedupAudit, _r: &Remediation)
    -> Result<Outcome, DedupRunError>
{
    let pre_f = pre.total_fragmentation();
    let post_f = post.total_fragmentation();
    if post_f > pre_f {
        return Err(DedupRunError::FragmentationIncreased { pre: pre_f, post: post_f });
    }
    let delta = pre_f - post_f;
    Ok(if delta == 0 {
        Outcome::NoOp
    } else {
        Outcome::Improved { delta: NonZeroUsize::new(delta).unwrap() }
    })
}
```

**Invariants encoded by construction:**
- `Outcome::Improved { delta: NonZeroUsize }` — `Improved { delta: 0 }`
  is unrepresentable at the type level (`NonZeroUsize` is the
  standard library's non-zero-witnessing newtype).
- `DedupRun::build` returns `Err` if post > pre; the type cannot
  carry "I made things worse."
- Serde-deserialized `DedupRun`s re-derive their outcome and refuse if
  claimed ≠ actual — forged audit trails get rejected.
- All fields private; consumers read via accessors.

### 9. `FlakeDedupSpec` and `ConvergenceState` — top-level (proof-bearing)

```rust
/// The fleet's typed declaration: which deps to dedupe, which pins
/// are sanctioned, where to emit PRs.
///
/// Authored as `(defflakededup :shared-deps [...] :intentional-pins
/// [...] :upstream-pr-template ...)`.
#[derive(Debug, Clone, Serialize)]
#[serde(into = "FlakeDedupSpecRaw")]
pub struct FlakeDedupSpec {
    shared_deps: Vec<SharedDep>,            // private
    intentional_pins: Vec<IntentionalPin>,  // private
    upstream_pr_template: PrTemplate,       // private
    convergence_config: ConvergenceConfig,  // private
}

#[derive(Serialize, Deserialize)]
struct FlakeDedupSpecRaw { /* mirror of fields */ }

#[derive(Deserialize)]
#[serde(try_from = "FlakeDedupSpecRaw")]
struct FlakeDedupSpecDe(FlakeDedupSpec);

impl TryFrom<FlakeDedupSpecRaw> for FlakeDedupSpec {
    type Error = SpecError;
    fn try_from(r: FlakeDedupSpecRaw) -> Result<Self, Self::Error> {
        Self::new(r.shared_deps, r.intentional_pins, r.upstream_pr_template, r.convergence_config)
    }
}

impl FlakeDedupSpec {
    pub fn new(
        shared_deps: Vec<SharedDep>,
        intentional_pins: Vec<IntentionalPin>,
        upstream_pr_template: PrTemplate,
        convergence_config: ConvergenceConfig,
    ) -> Result<Self, SpecError> {
        if shared_deps.is_empty() {
            return Err(SpecError::NoSharedDeps);
        }
        // Reject duplicate dep names — would silently drop one
        let mut seen = BTreeSet::new();
        for d in &shared_deps {
            if !seen.insert(d.name()) {
                return Err(SpecError::DuplicateDep(d.name().to_string()));
            }
        }
        // Every intentional_pin must reference a dep in shared_deps
        for p in &intentional_pins {
            if !shared_deps.iter().any(|d| d == p.dep()) {
                return Err(SpecError::PinForUnknownDep {
                    dep: p.dep().name().to_string(),
                });
            }
        }
        Ok(Self { shared_deps, intentional_pins, upstream_pr_template, convergence_config })
    }
    pub fn shared_deps(&self) -> impl Iterator<Item = &SharedDep> { self.shared_deps.iter() }
    pub fn intentional_pins(&self) -> impl Iterator<Item = &IntentionalPin> { self.intentional_pins.iter() }
    pub fn convergence_config(&self) -> &ConvergenceConfig { &self.convergence_config }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct ConvergenceConfig {
    /// Max iterations before declaring "needs human review".
    pub max_iterations: u32,           // default 50
    /// Max consecutive `BrokeEval` outcomes before escalating.
    pub max_consecutive_failures: u8,  // default 3
    /// If true, automatically commit + push successful Improved runs.
    pub auto_commit: bool,             // default false
}

/// The witness emitted at run termination. Proof-bearing variants:
/// - `Converged` carries `OptimalAudit` (type-level proof is_optimal == true)
/// - `PendingUpstreamPRs` requires non-empty `emitted_prs`
/// - `Escalated` carries a typed reason
#[derive(Debug, Clone, Serialize)]
#[serde(into = "ConvergenceStateRaw")]
pub enum ConvergenceState {
    Converged {
        final_audit: OptimalAudit,                 // type-level proof
        runs: Vec<DedupRun>,                       // (possibly empty if pre-state was already optimal)
    },
    PendingUpstreamPRs {
        partial_audit: DedupAudit,
        emitted_prs: NonEmpty<EmittedPR>,          // type-level proof of non-empty
        runs: Vec<DedupRun>,
    },
    Escalated {
        final_audit: DedupAudit,
        reason: EscalationReason,
        runs: Vec<DedupRun>,
    },
}

/// Smart constructors enforce semantic invariants.
impl ConvergenceState {
    pub fn converged(audit: DedupAudit, runs: Vec<DedupRun>)
        -> Result<Self, ConvergenceStateError>
    {
        let optimal: OptimalAudit = audit.try_into()
            .map_err(ConvergenceStateError::NotOptimal)?;
        Ok(Self::Converged { final_audit: optimal, runs })
    }
    pub fn pending_prs(
        audit: DedupAudit,
        prs: Vec<EmittedPR>,
        runs: Vec<DedupRun>,
    ) -> Result<Self, ConvergenceStateError> {
        let prs = NonEmpty::from_vec(prs)
            .ok_or(ConvergenceStateError::EmptyPrList)?;
        Ok(Self::PendingUpstreamPRs { partial_audit: audit, emitted_prs: prs, runs })
    }
    pub fn escalated(
        audit: DedupAudit,
        reason: EscalationReason,
        runs: Vec<DedupRun>,
    ) -> Self {
        Self::Escalated { final_audit: audit, reason, runs }
    }
}

#[derive(Debug, Clone, Serialize, Deserialize)]
pub enum EscalationReason {
    MaxIterationsReached { iterations: u32 },
    TooManyConsecutiveFailures { count: u8 },
    AmbiguousFix {
        dep: SharedDep,
        candidate_paths: Vec<FollowsPath>,         // typed, not Vec<Vec<String>>
    },
}

#[derive(Debug, Clone)]
pub struct NonEmpty<T>(Vec<T>);
impl<T> NonEmpty<T> {
    pub fn from_vec(v: Vec<T>) -> Option<Self> {
        if v.is_empty() { None } else { Some(Self(v)) }
    }
    pub fn into_vec(self) -> Vec<T> { self.0 }
    pub fn iter(&self) -> impl Iterator<Item = &T> { self.0.iter() }
}
```

**Invariants encoded by construction:**
- `Converged` requires `OptimalAudit` — non-optimal `Converged` is
  unrepresentable.
- `PendingUpstreamPRs` requires `NonEmpty<EmittedPR>` — empty
  PR list is unrepresentable.
- `EscalationReason::AmbiguousFix.candidate_paths` is
  `Vec<FollowsPath>` (typed), not `Vec<Vec<String>>` (raw).
- All `ConvergenceState` variant fields are private; consumers
  pattern-match on the enum but cannot synthesize states by struct-literal.
- Smart-constructor functions are the only path to legitimate states.
- Serde deserialize round-trips the smart constructors.

### How they compose

```
        FlakeDedupSpec
              │
              ├─ shared_deps ─────────┐
              ├─ intentional_pins ────┤
              └─ convergence_config   │
                                      ▼
   LockGraph (parse) ────────► DedupAudit (measure, spec-anchored, clock-injected)
       │                            │
       │                            ▼
       │                      DedupAudit::is_optimal? ─── yes ──► OptimalAudit
       │                            │ no                              │
       │                            ▼                                 ▼
       │                Remediation::propose_for                Converged
       │                            │
       │                            ▼ (returns Vec<MonotonicRemediation>)
       │                MonotonicRemediation::apply
       │                            │
       └─────► (modified flake.nix + nix flake lock) ─┐
                                                       │
                            new LockGraph ◄────────────┘
                                  │
                                  ▼
                      new DedupAudit::for_spec
                                  │
                                  ▼
                            DedupRun::build
                                  │ (pre, post, remediation → derive Outcome)
                            ┌─────┴─────┐
                            ▼           ▼
                       Improved      NoOp/BrokeEval/PrEmitted
                          │             │
                          └──┬──────────┘
                             ▼
                        ConvergenceState (smart-constructed)
```

Every arrow is a typed function with a contract. Invalid intermediate
states cannot be constructed.

---

## The eight-phase convergence loop in this domain

Mirror of `theory/THEORY.md` §IV.3 (the fleet-wide eight-phase loop),
specialized to flake-dedup:

| Phase | Action | Type signature |
|---|---|---|
| 1. Declare | Operator authors `(defflakededup …)` | `FlakeDedupSpec::new` |
| 2. Simulate | Compute current audit | `LockGraph::from_file` → `DedupAudit::for_spec` |
| 3. Prove | Check `OptimalAudit::try_from(audit)` — if Ok, exit | `OptimalAudit::try_from` |
| 4. Remediate | Generate ordered Remediation queue | `Remediation::propose_for` |
| 5. Render | Apply one MonotonicRemediation; re-lock | `MonotonicRemediation::apply` + `nix flake lock` |
| 6. Deploy | Commit + push (if auto_commit) | `DedupRun` recorded |
| 7. Verify | Re-measure; assert post < pre | `DedupRun::build` (smart constructor) |
| 8. Reconverge | Iterate from Phase 2 | Loop until `ConvergenceState` |

Same shape as every other pleme-io convergence consumer
(forge-gen, FluxCD, pangea-architectures, …). Operators reading the
loop see the same eight phases.

---

## Caixa shape

```lisp
(defcaixa :name "tend-flake-deduper" :kind Binario
  :runtime Native
  :consumes [
    {:caixa "tend"          :version ">=0.5"}      ; workspace context
    {:caixa "cartorio"      :version ">=0.3"}      ; receipt recording
    {:caixa "pleme-actions" :version ">=0.1"}      ; gh-pr automation
    {:caixa "tameshi"       :version ">=0.2"}      ; Clock + try_from helpers
  ]
  :limits {:wall-clock "30m" :memory "1GiB"}
  :behavior {
    :on-init    (fn [ctx] (FlakeDedupSpec/load (:spec-path ctx)))
    :on-call    (fn [spec event] (run-convergence-loop spec event (Clock/system)))
    :on-info    (fn [spec event] (handle-pr-merged spec event))
  }
  :upgrade-from {
    "0.0.1" [(LoadModule "tend-flake-deduper-0.1.0")
             (StateChange "intentional-pins-schema-v2")]
  })
```

Two CI workflow gates added:

| Workflow | Behavior |
|---|---|
| `caixa-validate` | Runs `tend flake-deduper verify` on PR. Fails the build if the PR introduces new fragmentation. |
| `caixa-publish` | Records the post-state DedupAudit in cartorio as the build's "fragmentation receipt." |

---

## Cartorio attestation shape

Every build that the deduper graduates to "Converged" emits a typed
receipt:

```rust
ArtifactState::Bundle {
    digest: blake3(flake_lock_bytes),
    attestation: {
        compliance: {
            profile: "flake-dedup-optimal@1.0",
            result_hash: blake3({
                shared_deps_canonical_revs: [(name, rev), ...],
                intentional_pins:           [(consumer, dep, reason_hash, expires), ...],
                runs_summary:                { total: N, improved: M, ... }
            }),
        },
    },
}
```

Anyone with the public flake.lock + the typed FlakeDedupSpec can
re-derive the receipt and verify the lock is in optimal state. That's
the canonical compliant-artifact-provability shape, applied to a new
domain.

---

## Anti-patterns this design forbids

- **`String`-everywhere lock parsing.** The whole point is that
  `Rev`, `NodeId`, `SharedDep`, `Remediation` are typed primitives;
  the audit pipeline doesn't pass freeform strings around.
- **Silent rev mutation.** A `Rev` is constructed once, validated, and
  then opaque. No `&mut Rev` accessors.
- **Empty justification for an `IntentionalPin`.** Constructor refuses;
  silent waivers are unrepresentable.
- **`Outcome::Improved { delta: 0 }`.** `NonZeroUsize` makes it
  unrepresentable at the type level.
- **`ConvergenceState::Converged { final_audit }` where `final_audit`
  is non-optimal.** `OptimalAudit` is the witness; non-optimal
  audits cannot be wrapped.
- **`PendingUpstreamPRs { emitted_prs: vec![] }`.** `NonEmpty<EmittedPR>`
  rejects.
- **Hand-running `nix flake lock` mid-iteration without re-measuring.**
  The loop is closed; the only way to advance is through `DedupRun::build`,
  which forces re-measurement.
- **Ambient time-trust** (caller passes `now` to `is_active`). Banned —
  use `try_into_active(&dyn Clock) -> Option<ActivePin>`. Holding
  `&ActivePin` IS the proof a checked clock authorized the use.
- **Public enum-variant construction.** Variants' fields are private;
  the only path is the smart-constructor function.

---

## Inspirations + references

- **`tameshi`** (compliant-artifact-provability) — the
  `ComplianceProfile` + Merkle-leaf shape we mirror in the receipt;
  also the home for the canonical `Clock` trait.
- **`pangea-operator`'s reactive policies** — the `ConvergenceConfig`
  shape mirrors `SettlingPolicy` + `ReactivePolicy` from the operator's
  reconcile loop.
- **`forge-gen`'s typed code generation** — the renderer-from-typed-input
  pattern, applied to flake follows instead of OpenAPI specs.
- **`tend`'s existing workspace management** — the natural runtime
  home for the deduper; integrates as a subcommand.
- **The `ProviderKind` enum in pangea-operator** — exhaustive-match
  contract that prevents "added a variant, forgot to wire it" bugs
  (referenced in `reference_typed_exhaustive_provider_decisions.md`).
- **The `try_from`-via-`Raw` serde pattern** — a fleet-wide convention
  documented at the typescape verification methodology level; will be
  generalized as `tameshi::typed-primitive-audit` per
  [`FLAKE-DEDUP-VERIFICATION.md`](./FLAKE-DEDUP-VERIFICATION.md).

---

## What the typescape proves vs. doesn't

**Provable by construction:**
- Convergence is monotonic (`NonZeroUsize` delta + saturating-decrement).
- Loop termination (max_iterations + monotonic decrement).
- "Invalid intermediate state" is unrepresentable at every typed
  surface (smart constructors + private fields + serde-validated
  deserialize).
- Receipt determinism (typed Rev + sorted bucket walk).
- Idempotent (re-running on optimal state exits at Phase 3 via
  `OptimalAudit::try_from(Ok)`).
- Time-trust is type-level — `&ActivePin` proof of clock-checked use.
- "Forged audit trail" rejected on deserialize (`DedupRun`'s
  `try_from` re-derives `Outcome` from pre/post and rejects mismatches).

**Not (yet) provable, deferred to runtime:**
- That a generated `Remediation::AddFollowsAtRoot` actually causes
  `nix flake lock` to dedupe (requires running nix; verified by
  `DedupRun::build` post-condition).
- That an upstream PR will be accepted (out-of-band human process).
- That the operator will renew an expired `IntentionalPin`
  (organizational, not type-system; but expired pins fail audit and
  surface in the next deduper run, so the feedback loop is closed).

These are the boundaries between the typescape's domain and the
ambient reality. The typescape catches everything inside those
boundaries; runtime gates and operator review catch what crosses them.
