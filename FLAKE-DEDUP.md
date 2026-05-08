# FLAKE-DEDUP — typed cache-fragmentation deduplication

> **Status:** canonical (2026-05-08). Implementation deferred to
> `pleme-io/tend` as the `tend flake-deduper` subcommand. Theory leads
> the implementation per the cardinal directive.
>
> This document specifies the typed primitive set that lets pleme-io
> drive `flake.lock` toward a provably-optimal cache-fragmentation
> state, continuously, without operator intervention beyond annotation
> review. It complements [`THEORY.md`](./THEORY.md) §VI.2 (typescape
> rendering), §II.2 (Four Lisps — typed-input pillar), and Pillar 12
> (generation over composition).

---

## North Star — what "optimal" means measurably

A `flake.lock` is in **optimal state** when, for every shared
transitive dependency T (`nixpkgs`, `fenix`, `devenv`, `cachix`,
`substrate`, `sops-nix`, `home-manager`, …):

```
distinct_revs(T) = 1 + count(intentional_pins(T))
```

In words: every consumer either uses the **canonical rev** root pins,
or carries an explicit `IntentionalPin` annotation justifying the
deviation. Any other state is non-optimal and must be refactored.

This metric is:

1. **Measurable** — count distinct `(name, rev)` pairs in the lock
   graph.
2. **Monotonic** — every successful refactoring step strictly decreases
   `sum(distinct_revs(T) - 1 - count(intentional_pins(T)))`.
3. **Bounded below by 0** — convergence terminates by the well-ordering
   principle.
4. **Attestable** — the witness is the post-state lock file's
   histogram, recordable in cartorio with a Merkle hash of the
   `(dep_name, sorted_rev_set)` tuple.

---

## The typed primitive set

Eight types compose the FlakeDedup domain. Each type's constructor
encodes invariants the Rust compiler enforces; states the spec calls
"invalid" are unrepresentable.

### 1. `SharedDep` — a dep we deduplicate

```rust
/// A transitive dependency the fleet wants collapsed to a single rev.
///
/// Constructor refuses empty names + names containing `/` (which would
/// confuse the lock-graph walker that uses `/` as a path separator
/// for child input addressing).
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, PartialEq, Eq, Hash)]
#[tatara(keyword = "defshareddep")]
pub struct SharedDep {
    name: String,                        // private — see SharedDep::new
    priority: Priority,
    upstream_constraint: ConstraintKind,
}

impl SharedDep {
    /// Smart constructor — returns Err for invalid names.
    pub fn new(name: impl Into<String>, priority: Priority,
               upstream_constraint: ConstraintKind)
        -> Result<Self, SharedDepError>
    {
        let name = name.into();
        if name.is_empty() {
            return Err(SharedDepError::EmptyName);
        }
        if name.contains('/') {
            return Err(SharedDepError::InvalidName(name));
        }
        Ok(Self { name, priority, upstream_constraint })
    }
    pub fn name(&self) -> &str { &self.name }
    pub fn priority(&self) -> Priority { self.priority }
}

/// Higher = remediate first. Encodes the heuristic that
/// nixpkgs > fenix > devenv > cachix > sops-nix > home-manager
/// because earlier-listed deps cascade more (nixpkgs invalidates
/// stdenv → invalidates everything).
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, Copy, PartialEq, Eq, Hash, Ord, PartialOrd)]
pub struct Priority(u32);

impl Priority {
    pub const HIGHEST:  Priority = Priority(100);
    pub const HIGH:     Priority = Priority(75);
    pub const STANDARD: Priority = Priority(50);
    pub const LOW:      Priority = Priority(25);
}

/// What kind of upstream rev-equality is enforceable.
///
/// Exhaustive match — adding a variant forces every consumer to
/// explicitly handle it (the typed-substrate guarantee).
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum ConstraintKind {
    /// All consumers must pin the EXACT same rev (e.g. fenix —
    /// different revs produce different rust toolchains, no cache
    /// reuse possible).
    ExactRev,
    /// All consumers must pin a rev compatible with root's branch
    /// (e.g. nixpkgs nixos-25.11 — different commits OK if same branch
    /// and ABI-compatible).
    BranchCompatible(BranchName),
    /// Source-only ref; no flake-input-resolution applies (the
    /// `*-src` pattern). Always treated as already-converged.
    SourceOnly,
}
```

**Invariants encoded by construction:**
- `name` is non-empty and contains no path separators (validated in `new`)
- `priority` is `Copy`, can't be silently mutated
- `ConstraintKind` is exhaustive — no fallback `Other(String)` escape hatch

### 2. `Rev` — a locked revision

```rust
/// A specific Git rev pinned in flake.lock.
///
/// Constructor refuses non-hex / wrong-length values that would silently
/// produce false-positive "different rev" comparisons.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd)]
pub struct Rev(String);  // private — see Rev::parse

impl Rev {
    pub fn parse(s: impl Into<String>) -> Result<Self, RevError> {
        let s = s.into();
        if s.len() != 40 || !s.chars().all(|c| c.is_ascii_hexdigit()) {
            return Err(RevError::InvalidFormat(s));
        }
        Ok(Self(s.to_ascii_lowercase()))
    }
    pub fn short(&self) -> &str { &self.0[..7] }
    pub fn full(&self)  -> &str { &self.0 }
}
```

**Invariant:** every `Rev` is exactly 40 lowercase hex chars. Comparing
two `Rev`s is a string comparison and always meaningful.

### 3. `LockNode` and `LockGraph` — the parsed flake.lock

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct LockNode {
    /// Node id from flake.lock — e.g. "fenix" or "fenix_47" for dups.
    pub id: NodeId,
    /// What this node resolves to.
    pub locked: LockedRef,
    /// Inputs of this node: name → resolution.
    pub inputs: BTreeMap<String, InputResolution>,
}

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub enum LockedRef {
    /// A real Git rev — eligible for dedup.
    GitHub { owner: String, repo: String, rev: Rev },
    /// `flake = false` source-only ref.
    SourceOnly { rev: Rev },
    /// Path-only ref (no rev — typically dev override).
    Path { path: PathBuf },
}

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub enum InputResolution {
    /// Resolves to another node directly.
    DirectNode(NodeId),
    /// Follows a path from root: ["A", "B", "C"] = "A's B's C".
    Follows(Vec<String>),
}

/// The fully-parsed flake.lock.
///
/// Constructor parses the JSON and validates structural invariants:
/// - Every NodeId in any `inputs` map exists in `nodes`
/// - Every Follows path resolves to a real node
/// - Cycle detection (Tarjan's SCC); cycles are an error, not a state
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct LockGraph {
    nodes: BTreeMap<NodeId, LockNode>,  // private
    root: NodeId,                        // private — usually "root"
}

impl LockGraph {
    pub fn from_file(path: &Path) -> Result<Self, LockParseError> {
        let raw = std::fs::read_to_string(path)?;
        let parsed: LockFileJson = serde_json::from_str(&raw)?;
        let g = Self::from_json(parsed)?;
        g.validate_no_cycles()?;
        g.validate_all_refs()?;
        Ok(g)
    }
    pub fn root(&self) -> &LockNode { &self.nodes[&self.root] }
    pub fn nodes(&self) -> impl Iterator<Item = &LockNode> { self.nodes.values() }
    /// Walk inputs of `from`, resolving Follows chains, returning
    /// the eventual concrete node.
    pub fn resolve(&self, from: &NodeId, input_name: &str) -> Option<&LockNode> { /* ... */ }
}
```

**Invariants encoded by construction:**
- A `LockGraph` with dangling refs cannot exist (`from_file` errors)
- A `LockGraph` with cycles cannot exist
- `root` is always present (parser asserts the `.nodes.root` key exists)

### 4. `RevHistogram` — the measurement

```rust
/// For a given SharedDep, the (rev → set of nodes pinning it) map.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct RevHistogram {
    pub dep: SharedDep,
    pub buckets: BTreeMap<Rev, BTreeSet<NodeId>>,
}

impl RevHistogram {
    /// Build by walking the lock graph for nodes pinning `dep`.
    pub fn build(graph: &LockGraph, dep: &SharedDep) -> Self { /* ... */ }
    /// The number of distinct revs.
    pub fn distinct_count(&self) -> usize { self.buckets.len() }
    /// The (rev, count) pairs sorted by count descending.
    pub fn sorted_majority_first(&self) -> Vec<(&Rev, usize)> { /* ... */ }
    /// The "majority rev" — what root would canonicalize to.
    pub fn canonical_rev(&self) -> Option<&Rev> { /* sorted_majority_first().first() */ }
}

/// Aggregate histogram across all shared deps — the optimization target.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct DedupAudit {
    pub histograms: Vec<RevHistogram>,
    pub intentional_pins: BTreeSet<IntentionalPinRef>,
}

impl DedupAudit {
    /// Total fragmentation across all shared deps, ignoring sanctioned pins.
    /// Optimal state ⇔ this returns 0.
    pub fn total_fragmentation(&self) -> usize {
        self.histograms.iter()
            .map(|h| h.distinct_count().saturating_sub(1))
            .sum::<usize>()
            .saturating_sub(self.intentional_pins.len())
    }
    pub fn is_optimal(&self) -> bool { self.total_fragmentation() == 0 }
}
```

**Invariants encoded by construction:**
- `RevHistogram` is keyed by `Rev` (validated 40-hex), so duplicate keys are impossible
- `is_optimal()` is a pure function — convergence is testable, not asserted
- `total_fragmentation` saturates rather than panics on edge cases

### 5. `IntentionalPin` — the relief valve

```rust
/// An operator-justified deviation from "all consumers follow root T".
///
/// Constructor REQUIRES a non-empty reason + a future expiration date.
/// Stale pins (past expiration) are flagged on every audit; the operator
/// must either renew with new justification or fix the underlying issue.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct IntentionalPin {
    consumer: NodeId,            // private — see ::new
    dep: SharedDep,              // which shared dep is intentionally NOT followed
    reason: String,              // private — non-empty; operator-supplied prose
    expires: Date,               // private — must be in the future at construction
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
    pub fn is_active(&self, now: Date) -> bool { now < self.expires }
}

/// A reference to an active pin, used in DedupAudit.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd)]
pub struct IntentionalPinRef {
    pub consumer: NodeId,
    pub dep_name: String,
}
```

**Invariants encoded by construction:**
- Empty-reason pins cannot exist (cardinal directive: no silent waivers)
- Expired pins cannot be created — only renewed
- The expiration field forces periodic re-justification — debt has a clock

### 6. `Remediation` — a typed action

```rust
/// A typed action the deduper proposes/applies. Each variant is one
/// "atomic unit of fix" that monotonically improves DedupAudit.
///
/// Constructor must accept (LockGraph, RevHistogram) — i.e. cannot
/// construct a Remediation against a state where it would be a no-op.
/// The typed contract: applying a Remediation strictly decreases
/// fragmentation OR fails (never leaves state worse).
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub enum Remediation {
    /// Add `inputs.<direct_input>.inputs.<dep_name>.follows = "<dep_name>"`
    /// at the root flake.nix. The simplest, most common case.
    AddFollowsAtRoot {
        direct_input: String,
        dep_to_follow: String,
    },
    /// Same as above but at a deeper override path. Used when
    /// `direct_input` doesn't itself pin the dep — its transitive child
    /// does.
    AddDeepFollowsAtRoot {
        path: Vec<String>,         // e.g. ["devenv", "cachix"]
        dep_to_follow: String,
    },
    /// The fix lives in an upstream pleme-io repo's flake.nix. Emits
    /// PR description; does NOT modify root flake.nix locally.
    UpstreamPR {
        repo: String,              // pleme-io/<repo>
        branch_name: String,       // suggested branch
        diff: UpstreamDiff,        // typed, not freeform string
    },
    /// Mark a specific (consumer, dep) as intentional. Requires
    /// operator-supplied reason via the audit-review flow.
    DocumentIntentionalPin {
        pin: IntentionalPin,
    },
}

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct UpstreamDiff {
    pub flake_nix_path: PathBuf,
    pub input_block: String,       // the input name being modified
    pub additions: Vec<String>,    // typed list of `inputs.X.follows = "X";` lines
}

impl Remediation {
    /// Smart constructor — fails if the proposed action is a no-op
    /// against the current graph (e.g. adding follows for a dep that
    /// the input doesn't actually consume).
    pub fn propose(
        graph: &LockGraph,
        histogram: &RevHistogram,
    ) -> Result<Vec<Self>, RemediationError> { /* ... */ }
}
```

**Invariants encoded by construction:**
- A `Remediation` against a state where it would be a no-op cannot be constructed
- `UpstreamDiff` is typed (no freeform string diffs) — substrate validates well-formed
- Variants are exhaustive — every minority rev maps to exactly one Remediation kind

### 7. `DedupRun` — one execution of the loop

```rust
/// A single attempt to apply a Remediation. Carries pre-state +
/// post-state + outcome — the audit witness that convergence is
/// monotonic.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct DedupRun {
    pub started_at: Timestamp,
    pub remediation: Remediation,
    pub pre_state: DedupAudit,
    pub post_state: DedupAudit,
    pub outcome: Outcome,
}

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub enum Outcome {
    /// Strict improvement — post.fragmentation < pre.fragmentation.
    Improved {
        delta: usize,              // how many fewer fragments
    },
    /// No improvement — Remediation didn't take effect (deep follows
    /// can silently no-op; nix flake lock didn't pick it up).
    NoOp,
    /// Eval broke. The Remediation has been reverted; this is
    /// information for the operator queue.
    BrokeEval {
        error: String,
    },
    /// The remediation was an UpstreamPR — applied via gh, not locally.
    PrEmitted {
        pr_url: Option<String>,
    },
}

impl DedupRun {
    /// Convergence assertion: `outcome` is `Improved` ⟹ post < pre.
    /// Run construction asserts this; a `DedupRun { outcome: Improved
    /// { delta: 0 }, .. }` cannot exist.
    pub fn build(
        pre: DedupAudit,
        post: DedupAudit,
        remediation: Remediation,
    ) -> Result<Self, DedupRunError> {
        let started_at = Timestamp::now();
        let outcome = if post.total_fragmentation() < pre.total_fragmentation() {
            Outcome::Improved {
                delta: pre.total_fragmentation() - post.total_fragmentation(),
            }
        } else if post.total_fragmentation() == pre.total_fragmentation() {
            Outcome::NoOp
        } else {
            // This is impossible by the Remediation contract — if it
            // happens, the deduper's invariant is broken; bail loudly.
            return Err(DedupRunError::FragmentationIncreased {
                pre: pre.total_fragmentation(),
                post: post.total_fragmentation(),
            });
        };
        Ok(Self { started_at, remediation, pre_state: pre, post_state: post, outcome })
    }
}
```

**Invariants encoded by construction:**
- A `DedupRun` whose post-state is worse than pre-state cannot be constructed
- `Outcome::Improved { delta: 0 }` is unrepresentable
- Every successful run carries a witness (the delta) that termination is monotonic

### 8. `FlakeDedupSpec` + `ConvergenceState` — the top-level contract

```rust
/// The fleet's typed declaration: which deps to dedupe, which pins
/// are sanctioned, where to emit PRs.
///
/// Authored as `(defflakededup :shared-deps [...] :intentional-pins
/// [...] :upstream-pr-template ...)`.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
#[tatara(keyword = "defflakededup")]
pub struct FlakeDedupSpec {
    pub shared_deps: Vec<SharedDep>,
    pub intentional_pins: Vec<IntentionalPin>,
    pub upstream_pr_template: PrTemplate,
    pub convergence_config: ConvergenceConfig,
}

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub struct ConvergenceConfig {
    /// Max iterations before declaring "needs human review".
    pub max_iterations: u32,           // default 50
    /// Max consecutive `BrokeEval` outcomes before escalating.
    pub max_consecutive_failures: u8,  // default 3
    /// If true, automatically commit + push successful Improved runs.
    /// If false, emit a patch the operator reviews.
    pub auto_commit: bool,             // default false
}

/// The witness emitted at run termination. Either we converged or we
/// have a typed reason why not.
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub enum ConvergenceState {
    Converged {
        final_audit: DedupAudit,
        runs: Vec<DedupRun>,
    },
    PendingUpstreamPRs {
        partial_audit: DedupAudit,
        emitted_prs: Vec<EmittedPR>,
        runs: Vec<DedupRun>,
    },
    Escalated {
        final_audit: DedupAudit,
        reason: EscalationReason,
        runs: Vec<DedupRun>,
    },
}

#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
pub enum EscalationReason {
    MaxIterationsReached { iterations: u32 },
    TooManyConsecutiveFailures { count: u8 },
    AmbiguousFix {
        dep: SharedDep,
        candidate_paths: Vec<Vec<String>>,
    },
}
```

**Invariants encoded by construction:**
- The three terminal states are exhaustive — no "in-flight" state can leak out
- `Converged { final_audit }` requires `final_audit.is_optimal() == true`
  (asserted in the smart constructor — not shown but implied by typed-substrate convention)
- `PendingUpstreamPRs` requires at least one emitted PR (empty list rejected)

### How they compose

```
        FlakeDedupSpec
              │
              ├─ shared_deps ─────────┐
              ├─ intentional_pins ────┤
              └─ convergence_config   │
                                      ▼
   LockGraph (parse) ────────► DedupAudit (measure) ────► is_optimal?
       │                            │                          │
       │                            ▼                  yes ────┘
       │                      RevHistogram              │
       │                            │                   ▼
       │                            ▼              Converged
       │                     Remediation::propose
       │                            │
       │                            ▼ (apply one)
       └─────► (modified flake.nix + nix flake lock) ─┐
                                                       │
                            new LockGraph ◄────────────┘
                                  │
                                  ▼
                            new DedupAudit
                                  │
                                  ▼
                            DedupRun::build
                                  │
                            ┌─────┴─────┐
                            ▼           ▼
                        Improved      NoOp/BrokeEval
                          │             │
                          └──┬──────────┘
                             ▼
                        ConvergenceState
```

Every arrow is a typed function with a contract. Invalid intermediate
states cannot be constructed.

---

## The eight-phase convergence loop in this domain

Mirror of `theory/THEORY.md` §IV.3 (the fleet-wide eight-phase loop),
specialized to flake-dedup:

| Phase | Action | Type signature |
|---|---|---|
| 1. Declare | Operator authors `(defflakededup …)` | `FlakeDedupSpec` |
| 2. Simulate | Compute current audit | `LockGraph::from_file` → `DedupAudit::build` |
| 3. Prove | Check `is_optimal()` — if true, exit | `DedupAudit::is_optimal` |
| 4. Remediate | Generate ordered Remediation queue | `Remediation::propose` |
| 5. Render | Apply one Remediation; re-lock | `Remediation::apply` + `nix flake lock` |
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
  ]
  :limits {:wall-clock "30m" :memory "1GiB"}
  :behavior {
    :on-init    (fn [ctx] (FlakeDedupSpec/load (:spec-path ctx)))
    :on-call    (fn [spec event] (run-convergence-loop spec event))
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
- **`Remediation::Improved { delta: 0 }`.** Smart constructor rejects;
  every recorded run is meaningful.
- **`ConvergenceState::Converged { final_audit }` where `final_audit`
  is non-optimal.** Smart constructor refuses; the type is the proof.
- **Hand-running `nix flake lock` mid-iteration without re-measuring.**
  The loop is closed; the only way to advance is through `DedupRun::build`,
  which forces re-measurement.

---

## Inspirations + references

- **`tameshi`** (compliant-artifact-provability) — the
  `ComplianceProfile` + Merkle-leaf shape we mirror in the receipt
- **`pangea-operator`'s reactive policies** — the `ConvergenceConfig`
  shape mirrors `SettlingPolicy` + `ReactivePolicy` from the operator's
  reconcile loop
- **`forge-gen`'s typed code generation** — the renderer-from-typed-input
  pattern, applied to flake follows instead of OpenAPI specs
- **`tend`'s existing workspace management** — the natural runtime
  home for the deduper; integrates as a subcommand
- **The `ProviderKind` enum in pangea-operator** — exhaustive-match
  contract that prevents "added a variant, forgot to wire it" bugs
  (referenced in `reference_typed_exhaustive_provider_decisions.md`)

---

## What the typescape proves vs. doesn't

**Provable by construction:**
- Convergence is monotonic (well-orderedness via `usize` saturating-decrement)
- Loop termination (max_iterations + monotonic decrement)
- "Invalid intermediate state" is unrepresentable (smart constructors)
- Receipt determinism (typed Rev + sorted bucket walk)
- Idempotent (re-running on optimal state exits at Phase 3)

**Not (yet) provable, deferred to runtime:**
- That a generated `Remediation::AddFollowsAtRoot` actually causes
  `nix flake lock` to dedupe (requires running nix; verified by
  `DedupRun::build` post-condition)
- That an upstream PR will be accepted (out-of-band human process)
- That the operator will renew an expired `IntentionalPin`
  (organizational, not type-system)

These are the boundaries between the typescape's domain and the
ambient reality. The typescape catches everything inside those
boundaries; runtime gates and operator review catch what crosses them.
