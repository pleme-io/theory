# CASCADE-RULE — the cache-fragmentation principle

> **Status:** canonical (2026-05-08). Lifted from
> [`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md) when the underlying rule
> proved to apply far beyond Nix flake-lock. Companion:
> [`CASCADE-RULE-VERIFICATION.md`](./CASCADE-RULE-VERIFICATION.md)
> verifies the typed primitives below against the methodology in
> [`TYPESCAPE-METHODOLOGY.md`](./TYPESCAPE-METHODOLOGY.md).
>
> This doc states the rule, formalizes its typed primitives, applies
> it to seven distinct domains pleme-io already touches, and
> specifies the canonical observation/measurement primitives
> consumers can reuse.

---

## The rule (one sentence)

> **Cache-fragmentation cost is proportional to `(number of distinct
> revs of any shared transitive dep) × (number of consumers)`. Any
> consumer that fails to canonicalize a shared transitive dep adds
> 1 to the fragmentation factor for that dep.**

The cost manifests in two ways:

1. **Initial materialization** — when a shared dep has N distinct
   revs, every artifact built against it has up to N distinct
   store-equivalent paths. The cache hit rate is at most 1/N.
2. **Cascade after a single bump** — when one consumer's pinned rev
   changes, every other consumer that *should* have shared that rev
   but pinned its own copy is unaffected (good for them) — but every
   downstream of the bumped consumer rebuilds whether the rev change
   actually mattered.

The rule's two consequences:

- **Lower-bound cache hit rate is `1/distinct_revs(T)`** for any
  artifact whose closure includes T.
- **Upper-bound rebuild blast radius is `|consumers(T)|`** when the
  shared dep T bumps and consumers don't follow.

These bounds are tight when consumers don't share any other inputs;
in practice they improve when fan-in correlates.

---

## Why it generalizes beyond Nix

The rule is **content-addressed-store-shape independent**. It applies
anywhere that:

1. There is a graph of artifacts where each artifact's identity is a
   function of its inputs.
2. Multiple top-level consumers transitively depend on the same
   logical dep T.
3. Any consumer can independently pin a different version of T.
4. Cache reuse depends on identity equality of the materialized
   artifact (not source equality).

This shape recurs across the substrate. Seven domains where the rule
applies (with status flags):

| Domain | Shared dep T | Consumer | Cache surface | Status |
|---|---|---|---|---|
| Nix flake.lock | nixpkgs/fenix/devenv | direct flake input | `/nix/store` | LIVE — see FLAKE-DEDUP |
| OCI image layers | base image (debian:bookworm) | every Dockerfile FROM | layer cache | partial — substrate's Nix base images are shared, but `forge`-built service images sometimes pin non-canonical bases |
| Helm chart deps | shared chart (cnpg, common) | every umbrella chart | OCI artifact cache | partial — `helmworks` registry should declare canonical |
| Terraform module versions | shared module (common-vpc) | every workspace | provider cache | partial — pangea-architectures has the shape |
| Cargo crate revs (workspace) | shared crate (kaname, shikumi) | every member of the workspace | `target/` | LIVE — substrate's rust-workspace-release-flake forces uniform crate revs |
| GitHub Actions composite refs | shared action (pleme-io/checkout-cached@v1) | every workflow | actions cache | LIVE — pleme-io/pleme-actions library hits this; mostly canonicalized |
| LLM context windows | shared knowledge file (CLAUDE.md, theory/) | every agent / skill | context-window working set | partial — agent skills sometimes restate theory inline instead of importing |

The two domains marked LIVE explicitly canonicalize. The ones marked
partial are the next remediation targets.

---

## The typed primitive set

Eight typed primitives. Each one's invariants encoded by construction
per [`TYPESCAPE-METHODOLOGY.md`](./TYPESCAPE-METHODOLOGY.md).

### 1. `SharedDependency` — a logical dep we want canonicalized

```rust
/// A logical dependency identified by name. Distinct from any
/// specific rev of it.
///
/// Example values: nixpkgs / fenix / debian-bookworm / cnpg-common /
/// pleme-io-checkout-cached.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize)]
#[serde(into = "SharedDependencyRaw")]
pub struct SharedDependency {
    name: String,                    // private — validated
    domain: Domain,                  // private — which artifact graph this lives in
    canonicalization: CanonicalizationPolicy,  // private
}

#[derive(Serialize, Deserialize)]
struct SharedDependencyRaw {
    name: String,
    domain: Domain,
    canonicalization: CanonicalizationPolicy,
}

impl TryFrom<SharedDependencyRaw> for SharedDependency {
    type Error = SharedDependencyError;
    fn try_from(r: SharedDependencyRaw) -> Result<Self, Self::Error> {
        Self::new(r.name, r.domain, r.canonicalization)
    }
}

impl SharedDependency {
    pub fn new(
        name: impl Into<String>,
        domain: Domain,
        canonicalization: CanonicalizationPolicy,
    ) -> Result<Self, SharedDependencyError> {
        let name = name.into();
        if name.trim() != name { return Err(SharedDependencyError::Whitespace); }
        if name.is_empty() { return Err(SharedDependencyError::Empty); }
        Ok(Self { name, domain, canonicalization })
    }
    pub fn name(&self) -> &str { &self.name }
    pub fn domain(&self) -> Domain { self.domain }
    pub fn canonicalization(&self) -> &CanonicalizationPolicy { &self.canonicalization }
}

/// Which artifact graph this dep lives in. Exhaustive — adding a
/// variant forces compile-time review of every consumer.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(rename_all = "kebab-case")]
pub enum Domain {
    NixFlakeLock,
    OciImageLayers,
    HelmChartDeps,
    TerraformModules,
    CargoCrates,
    GithubActions,
    LlmContextWindow,
}

/// How aggressively this dep should be canonicalized.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Serialize, Deserialize)]
#[serde(tag = "kind", rename_all = "kebab-case")]
pub enum CanonicalizationPolicy {
    /// All consumers must pin the EXACT same rev.
    StrictExact,
    /// All consumers must use revs from a "compatible band"
    /// (e.g., nixpkgs nixos-25.11 branch — different commits OK
    /// if same branch and ABI-compatible).
    BranchCompatible(BranchName),
    /// Source-only ref or third-party we can't influence —
    /// document deviation but don't enforce.
    Tolerated,
}
```

### 2. `Rev` — a specific version pinned by a consumer

```rust
/// A specific revision of a shared dep, in whatever ID scheme the
/// domain uses.
///
/// - NixFlakeLock: 40-hex Git sha
/// - OciImageLayers: 64-hex sha256 digest
/// - HelmChartDeps: semver string
/// - TerraformModules: semver / git tag
/// - CargoCrates: semver
/// - GithubActions: 40-hex Git sha or semver tag
/// - LlmContextWindow: content hash of the knowledge file
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize)]
#[serde(into = "RevRaw")]
pub struct Rev {
    domain: Domain,           // private — disambiguates ID scheme
    raw: String,              // private — validated per-domain
}

#[derive(Serialize, Deserialize)]
struct RevRaw { domain: Domain, raw: String }

impl TryFrom<RevRaw> for Rev {
    type Error = RevError;
    fn try_from(r: RevRaw) -> Result<Self, Self::Error> { Self::parse(r.domain, r.raw) }
}

impl Rev {
    pub fn parse(domain: Domain, raw: impl Into<String>) -> Result<Self, RevError> {
        let raw = raw.into();
        match domain {
            Domain::NixFlakeLock | Domain::GithubActions => {
                if raw.len() != 40 || !raw.chars().all(|c| c.is_ascii_hexdigit()) {
                    return Err(RevError::InvalidGitSha(raw));
                }
                Ok(Self { domain, raw: raw.to_ascii_lowercase() })
            }
            Domain::OciImageLayers => {
                // sha256:<64-hex> — accept with or without prefix
                let stripped = raw.strip_prefix("sha256:").unwrap_or(&raw);
                if stripped.len() != 64 || !stripped.chars().all(|c| c.is_ascii_hexdigit()) {
                    return Err(RevError::InvalidOciDigest(raw));
                }
                Ok(Self { domain, raw: format!("sha256:{}", stripped.to_ascii_lowercase()) })
            }
            Domain::HelmChartDeps | Domain::TerraformModules | Domain::CargoCrates => {
                if !is_valid_semver(&raw) {
                    return Err(RevError::InvalidSemver(raw));
                }
                Ok(Self { domain, raw })
            }
            Domain::LlmContextWindow => {
                // blake3 hex (64 chars) of the knowledge file body
                if raw.len() != 64 || !raw.chars().all(|c| c.is_ascii_hexdigit()) {
                    return Err(RevError::InvalidContentHash(raw));
                }
                Ok(Self { domain, raw: raw.to_ascii_lowercase() })
            }
        }
    }
    pub fn domain(&self) -> Domain { self.domain }
    pub fn raw(&self) -> &str { &self.raw }
}
```

### 3. `Consumer` — a graph node that pins a Rev

```rust
/// One consumer in the artifact graph that depends on at least one
/// SharedDependency.
///
/// - NixFlakeLock: a flake.nix that declares `inputs.X = ...`
/// - OciImageLayers: a Dockerfile / nix2container `FROM` clause
/// - HelmChartDeps: a Chart.yaml dependencies entry
/// - GithubActions: a .github/workflows/*.yml `uses:` line
/// - LlmContextWindow: an agent skill or CLAUDE.md
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize)]
#[serde(into = "ConsumerRaw")]
pub struct Consumer {
    domain: Domain,           // private
    identifier: String,       // private — validated per-domain (path / repo+ref / etc)
}

#[derive(Serialize, Deserialize)]
struct ConsumerRaw { domain: Domain, identifier: String }

impl TryFrom<ConsumerRaw> for Consumer {
    type Error = ConsumerError;
    fn try_from(r: ConsumerRaw) -> Result<Self, Self::Error> {
        Self::new(r.domain, r.identifier)
    }
}

impl Consumer {
    pub fn new(domain: Domain, identifier: impl Into<String>) -> Result<Self, ConsumerError> {
        let identifier = identifier.into();
        if identifier.trim() != identifier { return Err(ConsumerError::Whitespace); }
        if identifier.is_empty() { return Err(ConsumerError::Empty); }
        Ok(Self { domain, identifier })
    }
    pub fn domain(&self) -> Domain { self.domain }
    pub fn identifier(&self) -> &str { &self.identifier }
}
```

### 4. `Pinning` — observed (Consumer, SharedDependency) → Rev mapping

```rust
/// Observed: this consumer, in this domain, pins this rev of this dep.
#[derive(Debug, Clone, PartialEq, Eq, Hash, Ord, PartialOrd, Serialize, Deserialize)]
pub struct Pinning {
    pub consumer: Consumer,
    pub dependency: SharedDependency,
    pub rev: Rev,
}

impl Pinning {
    pub fn new(consumer: Consumer, dependency: SharedDependency, rev: Rev)
        -> Result<Self, PinningError>
    {
        // All three must be in the same Domain.
        if consumer.domain() != dependency.domain() {
            return Err(PinningError::DomainMismatch {
                consumer: consumer.domain(),
                dependency: dependency.domain(),
            });
        }
        if dependency.domain() != rev.domain() {
            return Err(PinningError::DomainMismatch {
                consumer: dependency.domain(),
                dependency: rev.domain(),
            });
        }
        Ok(Self { consumer, dependency, rev })
    }
}
```

### 5. `FragmentationGraph` — the parsed reality

```rust
/// All Pinnings for a single domain. The optimization target is the
/// FragmentationGraph; the canonical state is the one with the
/// minimum total fragmentation.
#[derive(Debug, Clone, Serialize)]
#[serde(into = "FragmentationGraphRaw")]
pub struct FragmentationGraph {
    domain: Domain,                       // private
    pinnings: BTreeSet<Pinning>,          // private
}

impl FragmentationGraph {
    /// Smart constructor — every Pinning must be in the declared domain.
    pub fn new(domain: Domain, pinnings: Vec<Pinning>) -> Result<Self, FragmentationGraphError> {
        for p in &pinnings {
            if p.dependency.domain() != domain {
                return Err(FragmentationGraphError::CrossDomainPinning {
                    expected: domain,
                    found: p.dependency.domain(),
                });
            }
        }
        Ok(Self { domain, pinnings: pinnings.into_iter().collect() })
    }

    /// Build the histogram of distinct revs per shared dependency.
    pub fn histogram(&self) -> Histogram {
        let mut buckets: BTreeMap<SharedDependency, BTreeMap<Rev, BTreeSet<Consumer>>> = BTreeMap::new();
        for p in &self.pinnings {
            buckets.entry(p.dependency.clone())
                .or_default()
                .entry(p.rev.clone())
                .or_default()
                .insert(p.consumer.clone());
        }
        Histogram { domain: self.domain, buckets }
    }
}
```

### 6. `Histogram` — measured fragmentation

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Histogram {
    pub domain: Domain,
    pub buckets: BTreeMap<SharedDependency, BTreeMap<Rev, BTreeSet<Consumer>>>,
}

impl Histogram {
    pub fn distinct_revs(&self, dep: &SharedDependency) -> usize {
        self.buckets.get(dep).map(|m| m.len()).unwrap_or(0)
    }
    pub fn fragmentation(&self, dep: &SharedDependency) -> usize {
        self.distinct_revs(dep).saturating_sub(1)
    }
    pub fn total_fragmentation(&self) -> usize {
        self.buckets.keys().map(|d| self.fragmentation(d)).sum()
    }
    pub fn canonical_rev(&self, dep: &SharedDependency) -> Option<&Rev> {
        // Majority-rev wins; in tie, lex-smallest rev.
        self.buckets.get(dep)?.iter()
            .max_by_key(|(rev, consumers)| (consumers.len(), std::cmp::Reverse((*rev).clone())))
            .map(|(rev, _)| rev)
    }
}

/// A `Histogram` proven to satisfy `total_fragmentation() == 0`.
#[derive(Debug, Clone)]
pub struct CanonicalHistogram(Histogram);

impl TryFrom<Histogram> for CanonicalHistogram {
    type Error = NotCanonical;
    fn try_from(h: Histogram) -> Result<Self, Self::Error> {
        if h.total_fragmentation() == 0 { Ok(Self(h)) }
        else { Err(NotCanonical { fragmentation: h.total_fragmentation() }) }
    }
}
```

### 7. `Remediation` — typed action that monotonically decreases fragmentation

```rust
/// A typed action that, when applied, strictly decreases
/// total_fragmentation by at least estimated_delta.
///
/// Variants exhaustively cover the seven domains. Adding a domain
/// forces compile-time review.
#[derive(Debug, Clone)]
pub enum Remediation {
    /// Edit a Nix flake.nix to add `inputs.X.inputs.Y.follows = "Y"`.
    NixAddFollows(NixAddFollowsParams),
    /// Edit a Dockerfile / nix2container manifest to use the canonical base image.
    OciCanonicalizeBase(OciCanonicalizeBaseParams),
    /// Edit a Chart.yaml to bump a dependency to the canonical chart rev.
    HelmCanonicalizeDep(HelmCanonicalizeDepParams),
    /// Edit a workspace.rb to bump a Pangea Ruby module to canonical version.
    TerraformCanonicalizeModule(TerraformCanonicalizeModuleParams),
    /// Edit a Cargo.toml to use the workspace-pinned crate.
    CargoCanonicalizeCrate(CargoCanonicalizeCrateParams),
    /// Edit a workflow.yml `uses:` ref to the canonical pleme-io action sha.
    GhaCanonicalizeAction(GhaCanonicalizeActionParams),
    /// Update a CLAUDE.md / skill to import canonical theory instead of restating.
    LlmCanonicalizeContext(LlmCanonicalizeContextParams),
}

// Each *Params struct privatizes its fields per Pattern 3.
pub struct NixAddFollowsParams {
    flake_path: PathBuf,             // private
    direct_input: InputName,         // private
    dep_to_follow: DepName,          // private
}
// ... and similarly for the other six.

/// A `Remediation` proven to monotonically reduce fragmentation by
/// `estimated_delta`. Crate-private constructor; only producer is
/// `Remediation::propose_for`.
#[derive(Debug, Clone)]
pub struct MonotonicRemediation {
    remediation: Remediation,             // private
    estimated_delta: NonZeroUsize,        // private
    targets: Vec<Pinning>,                // private — what specific pinnings this remediates
}

impl Remediation {
    pub fn propose_for(graph: &FragmentationGraph, histogram: &Histogram)
        -> Vec<MonotonicRemediation>
    { /* … */ }
}
```

### 8. `ConvergenceState` — terminal state after iteration

```rust
/// Terminal state. Smart-constructed.
#[derive(Debug, Clone)]
pub enum ConvergenceState {
    Canonical {
        final_histogram: CanonicalHistogram,        // type-level proof
        runs: Vec<RemediationRun>,
    },
    PendingExternalActions {
        partial_histogram: Histogram,
        emitted_actions: NonEmpty<EmittedExternalAction>,
        runs: Vec<RemediationRun>,
    },
    Escalated {
        final_histogram: Histogram,
        reason: EscalationReason,
        runs: Vec<RemediationRun>,
    },
}

impl ConvergenceState {
    pub fn canonical(h: Histogram, runs: Vec<RemediationRun>)
        -> Result<Self, ConvergenceStateError>
    {
        let canonical: CanonicalHistogram = h.try_into()
            .map_err(ConvergenceStateError::NotCanonical)?;
        Ok(Self::Canonical { final_histogram: canonical, runs })
    }
    pub fn pending_actions(
        h: Histogram,
        actions: Vec<EmittedExternalAction>,
        runs: Vec<RemediationRun>,
    ) -> Result<Self, ConvergenceStateError> {
        let actions = NonEmpty::from_vec(actions)
            .ok_or(ConvergenceStateError::EmptyActionList)?;
        Ok(Self::PendingExternalActions {
            partial_histogram: h, emitted_actions: actions, runs,
        })
    }
}
```

---

## Applications across pleme-io's existing domains

### 1. NixFlakeLock — the originating example

Already covered comprehensively in [`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md).
This `CASCADE-RULE.md` lifts the underlying primitives so other
domains can reuse them.

### 2. OciImageLayers

**Shared deps:** `nixos/nix:2.21.4`, `debian:bookworm-slim`,
`gcr.io/distroless/static`, `pleme-io/blackmatter-debug:v1.x`,
`alpine:3.20`.

**Consumers:** every `Dockerfile`, every `nix2container` manifest,
every `forge image build` invocation in pleme-io.

**Status:** Substrate's Nix-based images mostly canonicalize via
`forge` defaults. The remaining fragmentation lives in legacy
hand-written Dockerfiles (a few left in `pleme-io/{forge,
blackmatter-profiles, ...}`). Apply Remediation::OciCanonicalizeBase.

**Cache surface:** ghcr.io OCI layer cache. Cache hit happens when
the layer's digest matches; pinning a non-canonical base means a
unique chain of layers per consumer.

### 3. HelmChartDeps

**Shared deps:** `cnpg/cloudnative-pg`, `pleme-lib`,
`external-secrets/external-secrets`.

**Consumers:** every `helmworks` chart that has a `dependencies:`
block.

**Status:** `helmworks` registry already has a canonical chart for
`pleme-lib` (every chart depends on a single version per
`helmworks/charts/pleme-lib`). External chart deps (cnpg,
external-secrets) are sometimes pinned per-chart instead of via the
shared `pleme-lib` dep.

**Remediation:** lift the external-chart pin into `pleme-lib` itself
so every consumer inherits the canonical version. Pattern: same shape
as `inputs.X.follows = "X"` in flake-land.

### 4. TerraformModules

**Shared deps:** `pangea-aws::common-vpc`, `pangea-aws::eks-cluster`,
`pangea-akeyless::v3`.

**Consumers:** every `pangea-architectures/workspaces/*/<ws>.rb`
that requires the module.

**Status:** Pangea Ruby DSL gem version is pinned at top-level
(`Gemfile.lock`), which canonicalizes most module versions
transitively. Drift exists where individual workspaces require
specific gem versions inline.

**Remediation:** lift inline gem requires into the central
Gemfile.

### 5. CargoCrates

**Shared deps:** `kaname` (MCP server scaffold), `shikumi` (config),
`tatara-lisp`, `forge`, etc.

**Consumers:** every Rust workspace that pulls these as deps.

**Status:** substrate's `rust-workspace-release-flake` enforces
uniform crate revs per workspace. The fragmentation surface is
**across** workspaces — when `tatara` and `escriba` both depend on
`tatara-lisp` via different cargo paths.

**Remediation:** publish to crates.io with tight SemVer; consumers
follow caret bounds. Or use Cargo's `workspace.dependencies` from a
shared root.

### 6. GithubActions

**Shared deps:** `pleme-io/checkout-cached@v1`, `actions/checkout@v4`,
`pleme-io/terragrunt-apply@v1`, `pleme-io/argocd-app-sync@v1`.

**Consumers:** every `.github/workflows/*.yml`.

**Status:** `pleme-actions` library publishes canonical refs for
the 11 actions. Fragmentation = workflows pinning to specific shas
instead of the moving `@v1` tag.

**Remediation:** sweep workflows to use `@v1`; document SHA pins
only when explicitly needed (security-critical).

### 7. LlmContextWindow

**Shared deps:** `theory/THEORY.md`, `theory/CASCADE-RULE.md` (this
doc), `pleme-io/CLAUDE.md`, `blackmatter-pleme/skills/*/SKILL.md`.

**Consumers:** every Claude Code agent invocation, every skill
descriptor that quotes theory inline.

**Status:** Some skills restate theory inline instead of importing
("fragmentation"). When `THEORY.md` updates, the inline copies drift
and agents have inconsistent context.

**Remediation:** every skill descriptor should import canonical theory
by reference (e.g., `Read theory/THEORY.md before acting`) rather
than embed it. Substrate addition: a `tameshi context-audit` that
walks SKILL.md files, detects inline-theory restatement, and emits
remediation.

This is the most pleme-io-natural application of the rule — context-
window space is itself a cache surface; restating theory across N
skills is the same shape as pinning N copies of nixpkgs.

---

## The eight-phase convergence loop in this domain

Same shape as every other pleme-io convergence consumer:

| Phase | Action | Type signature |
|---|---|---|
| 1. Declare | Operator authors `(defcascadespec :domain X :shared-deps [...])` | `CascadeSpec::new` |
| 2. Simulate | Compute current `FragmentationGraph` | `FragmentationGraph::from_domain(...)` |
| 3. Prove | Check `CanonicalHistogram::try_from(histogram)` | succeeds → exit |
| 4. Remediate | Generate `Vec<MonotonicRemediation>` | `Remediation::propose_for` |
| 5. Render | Apply one remediation | mutate flake.nix / Dockerfile / Chart.yaml / ... |
| 6. Deploy | Commit + push | git history |
| 7. Verify | Re-measure | `RemediationRun::build` |
| 8. Reconverge | Iterate | until `ConvergenceState` |

---

## Caixa shape

```lisp
(defcaixa :name "tend-cascade-deduper" :kind Binario
  :runtime Native
  :consumes [
    {:caixa "tend"               :version ">=0.5"}
    {:caixa "cartorio"           :version ">=0.3"}
    {:caixa "pleme-actions"      :version ">=0.1"}
    {:caixa "tameshi"            :version ">=0.2"}
  ]
  :limits {:wall-clock "30m" :memory "1GiB"}
  :behavior {
    :on-init (fn [ctx] (CascadeSpec/load (:spec-path ctx)))
    :on-call (fn [spec event]
               (run-convergence-loop spec event (Clock/system) (:domain event)))
  }
  :upgrade-from { ; v0.0 was flake-only, v0.1 is multi-domain
    "0.0.1" [(LoadModule "tend-cascade-deduper-0.1.0")
             (StateChange "domain-enum-extension")]
  })
```

Generalizes `tend flake-deduper` (the Nix-only helper from
[`FLAKE-DEDUP.md`](./FLAKE-DEDUP.md)) into a multi-domain
deduplicator. The Nix subcommand becomes one of seven supported
domains.

---

## Cartorio attestation shape

```rust
ArtifactState::Bundle {
    digest: blake3(domain_artifact_bytes),     // flake.lock OR Dockerfile OR Chart.yaml
    attestation: {
        compliance: {
            profile: format!("cascade-canonical@{}", domain.as_kebab()),
            result_hash: blake3({
                domain: domain,
                shared_deps_canonical_revs: [(name, rev), ...],
                runs_summary: { total: N, monotonic: M, ... }
            }),
        },
    },
}
```

---

## Anti-patterns this rule forbids

- **"This is just one consumer, doesn't matter if it pins its own rev"**
  — every consumer adds 1 to the fragmentation factor; the cumulative
  effect across N consumers IS the cascade.
- **"We'll dedupe later"** — refactor-later debt almost never gets
  paid. Cardinal-sin per Operating Principle #0.
- **Ad-hoc per-domain canonicalization scripts** — every domain that
  doesn't go through `tend cascade-deduper` accrues its own
  fragmentation tooling, which itself fragments. Generation over
  composition (Pillar 12).
- **Pinning by SHA when a moving tag is canonical** — eliminates the
  benefit of canonicalization. Document SHA pins only for security-
  critical artifacts.
- **Restating theory in skills / CLAUDE.md** — context-window
  fragmentation. Import the theory by reference.

---

## Composition with other pleme-io theory

| Theory | Composition |
|---|---|
| `THEORY.md` Pillar 12 | The deduper IS generation-over-composition for cache fragmentation |
| `FLAKE-DEDUP.md` | The Nix-domain instance of CASCADE-RULE; this doc generalizes the rule, FLAKE-DEDUP.md instantiates it |
| `TYPESCAPE-METHODOLOGY.md` | The rule's typed primitives go through that doc's verification methodology |
| `BASE-PRIMITIVES.md` | `Histogram`, `CanonicalHistogram`, `MonotonicRemediation`, `NonEmpty<T>` lift to the canonical base-primitive catalog |
| `CAIXA-SDLC.md` | `tend cascade-deduper` ships as a `:kind Binario` caixa |
| `INSPIRATIONS.md` | Borrows from content-addressed storage theory (Bazel, Buck2) and from refinement-typed cache invalidation (Liquid Haskell `RefinementType` → store path correspondence) |

---

## What the typescape proves vs. doesn't

**Provable by construction:**
- A `Pinning` cannot mix domains (constructor checks).
- A `FragmentationGraph` can only contain `Pinning`s in its declared
  domain.
- A `CanonicalHistogram` has zero fragmentation.
- A `MonotonicRemediation` has `estimated_delta ≥ 1`.
- `ConvergenceState::Canonical` carries a `CanonicalHistogram` —
  non-canonical "convergence" is unrepresentable.
- Every `Rev` is well-formed for its declared domain.

**Not provable, deferred to runtime:**
- That a generated remediation actually causes the upstream tooling
  (Nix, Docker, helm) to dedupe. Verified by `RemediationRun::build`'s
  post-condition (`post_fragmentation < pre_fragmentation`).
- That the domain enum captures every relevant cache-fragmentation
  surface in pleme-io. New domains may emerge; the enum grows.
- That the operator's chosen `CanonicalizationPolicy` is the right
  policy. Operator judgment.

These boundaries are explicit. The typescape proves what it can prove;
runtime gates and operator review carry the rest.
