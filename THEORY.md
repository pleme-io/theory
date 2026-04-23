# THEORY — pleme-io's Unified Theory of Computing

> Rust owns types. Lisp owns flow. Nix owns images. Pangea owns
> infrastructure. OpenAPI owns APIs. Typescape owns structure. Convergence
> owns motion. Attestation owns trust. Generation owns authorship.
> Every pillar, every repo, every line of code rests on these nine columns.

This document is the single source of truth for how pleme-io thinks about
computing. It is what every repo's `CLAUDE.md` links to instead of
restating. It supersedes any prior scattered paragraph of theory elsewhere
in the fleet. If you're reading this for the first time, read it linearly;
you cannot skim the middle and follow the end.

Companion: [`VOCABULARY.md`](./VOCABULARY.md) defines every named term
used here exactly once.

---

## Part I — The Frame

### I.1 The core claim

Software is a _declaration of desired state_, a _proof that the declaration
is well-formed_, a _rendering of the declaration into an executable
artifact_, and a _convergence of the running world toward the declared
state_. All four are constructive. All four are checkpointed. All four are
typed. None of them is an exception.

This is not a slogan. It is a disciplinary commitment with concrete
consequences that recur in every pillar below. When you are confused about
whether some piece of the system is "well-designed" in pleme-io terms, ask
whether it makes each of the four steps visible and verifiable. If it
doesn't, it isn't.

### I.2 The twelve pillars

Every repo in pleme-io aligns with the twelve pillars of Blackmatter
Development. Each pillar names _what owns a concern_ — not merely a
preferred tool, but the one place where the concern is resolved. If you are
unsure which pillar a piece of code belongs under, the pillar list is a
forcing function: something owns it, and that thing is one of these twelve.

| # | Pillar | Owner (the one thing) | Realized in |
|---|---|---|---|
| 1 | Language | **Rust + tatara-lisp + WASM/WASI** | every crate |
| 2 | Configuration | **shikumi** (typed, hot-reloadable, YAML at `~/.config/{app}/{app}.yaml`) | every service |
| 3 | API generation | **forge-gen** (OpenAPI → SDKs + MCP + IaC + completions + docs) | every API |
| 4 | Data layer | **SeaORM + shinka migrations** | every persistence layer |
| 5 | Infrastructure declaration | **Pangea** (Ruby DSL → Terraform JSON) | every cloud resource |
| 6 | Typescape | **arch-synthesizer** (Rust types own every primitive and proof) | every generated artifact |
| 7 | Kubernetes control | **Helm + Kustomize + FluxCD**, all rendered from typescape | every cluster |
| 8 | Image building | **Nix only** (never Dockerfiles) | every container |
| 9 | SDLC | **substrate + repo-forge archetypes + `nix run .#app`** | every repo |
| 10 | Proof discipline | **`cargo test` = compliance verification; rspec + proptest; tameshi BLAKE3 chain** | every deployment |
| 11 | JIT Infrastructure | **spot + breathability + mandatory alert layer** | every compute unit |
| 12 | Generation over composition | **repo-forge / forge-gen / arch-synthesizer** (hand-writing is the fallback) | every recurring shape |

The canonical statement of these pillars lives at
`~/code/github/pleme-io/BLACKMATTER.md`; this document treats them as the
given starting point and develops their consequences. When the pillars
change, BLACKMATTER.md changes first and this document follows.

### I.3 What pleme-io believes about computing

These five beliefs are downstream of the pillars but worth stating
directly, because they explain why the pillars are shaped the way they are.

1. **Construction over composition over discovery.** We construct systems
   out of typed parts; we compose those constructions; we rely on discovery
   (grep, runtime reflection, dynamic dispatch) only as a last resort.
   Construction is provable, composition is provable, discovery is not.
2. **Types as the floor, Lisp as the ceiling.** Rust's type system is the
   ground beneath every value; Lisp's homoiconicity is the sky above every
   authoring surface. The boundary between them is a single proc macro —
   `#[derive(TataraDomain)]`.
3. **Every running system is a convergence process.** Not a batch job, not
   a state, not a snapshot. A process in the Unix sense, with a PID, a
   parent, a lifecycle, and a convergence loop that keeps declared and
   running in agreement.
4. **Every declaration has a proof; every proof has an attestation.** Proof
   is the type-level property (proptest, rspec, `cargo test`); attestation
   is its cryptographic seal (BLAKE3 Merkle, Ed25519). Both are required;
   neither substitutes for the other.
5. **Generation first, composition second, hand-authoring last.** Every
   recurring shape becomes a generator before it becomes a pattern; every
   pattern becomes a library before it becomes duplicated code. The
   duplication budget is zero. The generation budget is unlimited. Stop at
   diminishing returns.

---

## Part II — Language

### II.1 The Rust + Lisp pattern (the primary architectural concept)

> Rust-bordered, Lisp-mutable, self-optimizing environments. Rust owns
> types, invariants, memory safety, and the proof floor. Lisp owns
> declarative authoring, macro composition, and runtime IR rewriting. The
> boundary is a single proc macro. Every new typed domain joins the system
> with one line of `#[derive(TataraDomain)]`.

This is the primary pattern. Every typed domain in the fleet follows it.
The full cookbook is at `~/code/github/pleme-io/tatara/docs/rust-lisp.md`;
this document captures only the load-bearing invariants.

**The five invariants** (each holds by construction in
`tatara-lisp`/`tatara-nix` today):

1. **Typed entry.** The only way to produce a Rust value from Lisp is
   through `TataraDomain::compile_from_sexp`. Ill-typed input errors before
   the value exists.
2. **Free middle.** Inside the boundary, Lisp macros rewrite IR freely. No
   mid-rewrite type checks; no loss of flexibility.
3. **Typed exit.** Re-entering `compile_from_args` on any rewritten Sexp
   guarantees the output is well-typed. Any rewrite that type-checks is
   safe by construction.
4. **Deterministic identity.** Every value has a BLAKE3 content hash over
   its canonical serialization. Identical inputs produce identical
   identities.
5. **Composition preserves proofs.** `meet`/`join` of two values inherits
   the invariants of each. No proof repetition across composites.

**The six-line contract.** For every typed domain introduced in pleme-io:

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
#[tatara(keyword = "defmydomain")]
pub struct MyDomainSpec { /* fields */ }

pub fn register() {
    tatara_lisp::domain::register::<MyDomainSpec>();
}
```

Six lines buys Lisp authoring, YAML authoring, Nix authoring, Rust
programmatic authoring, typed compilation, macro composition,
self-optimization via `rewrite_typed`, deterministic identity via BLAKE3,
registry dispatch, and participation in the workspace-wide coherence
checker. If you're building a new domain and not using this pattern,
explain why in a design doc.

### II.2 The Four Lisps

This section names a structural observation that every pleme-io
contributor has felt but that has never been written down. It's elevated to
a first-class section because it is load-bearing for every design decision
in Parts III–VII.

**pleme-io authors typed domains through four surfaces, each of which is a
Lisp in the technical sense that matters: a homoiconic, macro-extensible,
composable, typed-or-typable authoring surface that reduces to a common
IR.** The four surfaces differ in strictness, not in kind.

| Surface | Typing | Homoiconicity | Authoring form | Compiles to |
|---|---|---|---|---|
| **tatara-lisp** | Strict: Rust typescape via `#[derive(TataraDomain)]` | Native S-expressions | `(defX :k v …)` | `MyDomainSpec` Rust value → canonical SExpr → BLAKE3 → attestation |
| **Nix** | Strict-ish: module system lattice, option types | Expression AST is the value | `{ imports = […]; config = { … }; }` | Store path, derivation, closure — content-addressed |
| **Pangea (Ruby DSL)** | Typed through `Dry::Struct`; blocks are s-expressions with syntactic sugar | Ruby's block + method_missing is a homoiconicity surrogate | `architecture(:name) { resource :aws_vpc, … }` | Typed resource graph → Terraform JSON |
| **OpenAPI (YAML spec)** | Typed through the JSON Schema system, `$ref`, component registry | YAML/JSON AST is the value | `openapi.yaml` paths + schemas + components | `sekkei` types → `takumi` IR → forge-gen's 51 outputs |

The four Lisps sit on a spectrum:

- **tatara-lisp** is the strict reference. Every invariant holds. Every
  value is typed at construction. Macros are pure term rewriting.
- **Nix** is stricter than it looks. Laziness is the eval discipline;
  the module system is a lattice algebra; options have types; overlays are
  functor composition. It is the infrastructure Lisp — typed through
  discipline, not through derive macros.
- **Pangea** is looser. Ruby metaprogramming is an evaluator, not a term
  rewriter, so some invariants hold only at Pangea build time. Type purity
  is enforced by `Dry::Struct` + rspec synthesis. It is the declarative
  infrastructure Lisp.
- **OpenAPI** is loosest. YAML has no macros; `$ref` is the only form of
  composition; the type system is extrinsic (JSON Schema) rather than
  intrinsic. But the schema system is rich enough that the typescape
  projection is total — `sekkei` + `takumi` give OpenAPI the same typed IR
  surface as the other three.

**Why this matters:**

- Any feature we add to one Lisp we eventually want in all four. A pattern
  that earns its keep in tatara-lisp (the strictest) is a forcing function
  for matching expressivity in Nix, Pangea, OpenAPI.
- Any feature we cannot express in the _loosest_ (OpenAPI) must be
  downgraded to a projection or rejected. OpenAPI is the floor of what
  every API surface must accept; anything richer is additive.
- The "sufficient looseness" of Nix, Ruby, and YAML means we do not need
  to rewrite them. We recognize them as Lisps-in-practice and commit to
  treating their ASTs as first-class IR. Ruby blocks are s-expressions;
  Nix expressions are s-expressions; YAML is a restricted s-expression
  encoding. The family is closed.
- **The IR is common.** All four compile to a canonical SExpr form
  (`iac_forge::sexpr::SExpr`) for attestation, and to the arch-synthesizer
  typescape for rendering. Authoring in any of the four is authoring in
  all four, modulo strictness.

We do not plan a fifth Lisp. If a new authoring surface is proposed, it
must reduce to one of these four — or prove that the canonical IR doesn't
cover it, which is a THEORY.md change.

### II.3 Brazilian × Japanese naming

Foundational crates and long-standing primitives take Japanese names —
precision, craft, discipline: `tatara`, `shikumi`, `sekkei`, `takumi`,
`forge`, `kenshi`, `shinka`, `sekiban`, `tameshi`, `kensa`, `sui`,
`blackmatter`.

New Tier 2+ primitives that evoke enclosed spaces, flows, growth, or craft
take Brazilian-Portuguese names: `terreiro` (arena / enclosed Lisp VM),
`forja` (compiler factory), `cordel` (bytecode stream), `cerrado` (typed
substrate), `jabuti` (slow-persistent bootstrap), `samba` (rhythmic
composition), `roça` (content-addressable field), `bordado` (overlay
pattern), `caderneta` (registry/notebook), `apurador` (validator).

The two traditions name orthogonal axes. They do not clash. Japanese names
the discipline layer; Portuguese names the ritual space, tropical
cultivation, rhythmic composition.

### II.4 Forbidden and permitted languages

- **Rust** for types, invariants, services, CLI tools, libraries, daemons,
  and proof code. Edition 2024. `rust-version = "1.89.0"`. Clippy pedantic.
- **tatara-lisp** for declarative authoring, macro composition, domain
  DSLs. Never for runtime logic that needs closures, recursion, or mutable
  state — that's Rust's job.
- **Nix** for build, composition, image authoring, system config. No
  Dockerfiles. No `writeShellScript` beyond three-line glue.
- **Ruby (Pangea DSL)** for infrastructure declaration. Not for services,
  not for tools, not for anything that isn't `pangea-*` gems.
- **OpenAPI YAML** for API specs. Every API has one.
- **YAML** for runtime config (shikumi-style). Never as a program, always
  as typed input to a Rust config loader.
- **TypeScript + React + MUI** for frontends at the edge only. Never
  server-side.
- **Frontend TypeScript** is permitted at `lilitu/` and equivalent product
  edges. Not anywhere else.
- **Production Python, server-side TypeScript, shell scripts** (beyond
  3-line glue) are forbidden. If you find yourself wanting one, stop and
  build the Rust/Lisp/Nix equivalent.

---

## Part III — Structure

### III.1 The typescape is the root data structure

The **typescape** is the universe of all types in the platform. It is
queryable, attestable, composable across eight dimensions. It lives as
typed Rust in `arch-synthesizer/`; every other concept in pleme-io is
either a dimension of the typescape or a projection from it.

**The eight dimensions:**

1. **Vocabulary** — 230+ canonical terms organized into five categories
   (Primitive, Composition, Lattice, Mutation, Meta). Every named concept
   in the platform appears here. Source: `arch-synthesizer/src/vocabulary.rs`.
2. **AST Domains** — 19 irreducible-primitive sets, one per infrastructure
   format (Ruby, YAML, HCL, Nix, Rust, Go, Python, TypeScript, OpenAPI,
   SQL, CRD, FluxCD, Markdown, etc.). Each domain has 3–20 primitives
   composable via six universal categories (Literal, Sequence, Mapping,
   Declaration, Annotation, Escape). Source: `arch-synthesizer/src/ast_domains.rs`.
3. **Morphisms** — 12+ structure-preserving maps between AST domains.
   Each is proven total and deterministic. Examples: `IacType → Ruby`,
   `ComplianceSuite → Helm`, `OpenAPI → Terraform`. Morphisms form a
   connected graph over the AST domain set. Source: the `Synthesizer` trait.
4. **Fixed points (controllers)** — 11 controllers mapped to four
   convergence strategies (Declarative, DiffAndPatch, FullRebuild,
   EventDriven). Each controller is a fixed-point operator: `f(x) = x` when
   the system has converged. FluxCD, Terraform, Nix, Pangea, kenshi,
   shinka, sekiban, sui, tatara-reconciler, convergence-controller are the
   controller instances.
5. **Compliance lattice** — meet/join/intersect over 50 compliance
   primitives organized into 6 baselines across 5 frameworks (NIST 800-53,
   CIS, FedRAMP, PCI DSS 4.0, SOC 2 Type II). Compliance is a dimension of
   the typescape, not a separate system bolted on.
6. **Workspace DAG** — typed cross-state references across Pangea
   workspaces. Parent workspace outputs flow into child workspace
   `RemoteState.output`. State keys `pangea/{parent}/{child}` are
   deterministic, not conventional. Workspace boundaries map to Unix
   process boundaries (see Part IV.4).
7. **Render state** — content-addressable index of every generated
   artifact. BLAKE3 hash of canonical input ↔ BLAKE3 hash of rendered
   output. The index is the cache and the attestation record at once.
8. **Module manifests** — per-repo `.typescape.yaml` files recording which
   dimensions contributed to which artifacts, content hashes, and the
   Merkle path back to the typescape root. Each repo is a leaf in the
   typescape Merkle tree.

**The key insight:** every artifact the platform produces — a Terraform
plan, a Helm chart, a Nix store path, a Rust crate, an OpenAPI spec — is
a projection of the typescape through a morphism. The artifact is never
the source. The typescape is the source; the artifact is how it looks from
a particular angle. Rewriting the artifact without going through the
typescape is drift.

### III.2 AST domains and morphisms

Every infrastructure domain has a finite set of irreducible primitives
(3–20 nodes). All expressiveness is composition. Dialect differences
(Ruby 3.3 vs 2.7, HCL 1.7 vs 1.2, Python 3.12 vs 3.8) are differences in
which compositions are valid, not in the primitive set. Absorb the dialect,
gain the expression space.

Morphisms are structure-preserving maps between AST domains. The
`Synthesizer` trait IS the morphism. The `Backend` trait has seven
implementations today (Terraform, Pulumi, Crossplane, Ansible, Pangea,
Steampipe, custom) — same types, same invariants, same compliance
controls. Adding a new backend inherits ALL existing proofs because proofs
attach to the typescape types, not to the rendering code.

### III.3 The compliance lattice

Compliance is a dimension of the typescape (III.1.5). It composes as a
lattice:

- **meet** (∩): the intersection of two baselines — controls both require.
- **join** (∪): the union of two baselines — controls either requires.
- **leq** (≤): one baseline is strictly weaker than another.

Every controller and every morphism carries a compliance binding. A
Terraform plan rendered from a typescape slice inherits the slice's
compliance lattice position; a plan cannot be rendered without meeting the
declared baseline. This is structural. Compliance cannot be added as an
afterthought because there is no place in the pipeline where uncompliant
state can exist — every intermediate representation carries the lattice.

Pre-deployment verification is done by `kensa` (OSCAL/NIST mapping, proof
of compliance at plan time). Runtime verification is done by `sekiban`
(K8s admission webhook refusing uncompliant resources). Continuous
verification is done by `tameshi` heartbeat chains. See Part V.

---

## Part IV — Motion

### IV.1 Convergence: lattice and process (the term drift, fixed)

The word **convergence** has historically been used for two distinct
things in pleme-io docs. They are the same phenomenon viewed statically
and dynamically, but they are not interchangeable. From this document
forward:

- **convergence-as-lattice** (static): the structure of all possible
  states the system can legitimately inhabit. This is a dimension of the
  typescape (III.1.5 + III.1.4). It names _what states exist_.
- **convergence-as-process** (dynamic): the motion of the running world
  toward a particular lattice point. This is driven by controllers. It
  names _how the system moves_.

When the unqualified word "convergence" appears in a pleme-io document, it
means convergence-as-process (the dynamic motion) by default — because
that's what operators do in the common case. When the static structure is
meant, say "the convergence lattice" or "the lattice."

Both views matter. The lattice is the static structure; convergence is the
dynamic motion across it. Same theory, two views.

### IV.2 Controllers as fixed-point operators

A controller is a function `f` from system state to system state such that,
given a declared desired state `d`, the controller runs `x_{n+1} = f(x_n,
d)` until `f(x_*, d) = x_*`. That point `x_*` is the fixed point. The
system has converged when `x_* = d`.

Four strategies, mapped to 11 controller instances:

| Strategy | What it does | Instances |
|---|---|---|
| **Declarative** | Re-declare desired state; controller reconciles the delta | FluxCD, Kubernetes (kube-controller-manager), Nix |
| **DiffAndPatch** | Compute diff between declared and running; apply minimum patch | Terraform, Pangea, ArgoCD |
| **FullRebuild** | Throw away running state; rebuild from scratch | Packer AMI bake, Nix image builds, OCI registry pushes |
| **EventDriven** | Wait for a signal; reconcile on fire | kenshi (test gate), shinka (db migration), sekiban (admission), tatara-reconciler (NATS signals), convergence-controller (cluster lifecycle) |

Every controller is content-addressable: its fixed point is determined by
the BLAKE3 hash of its declared input. Two controllers given the same
input reach the same fixed point. This is the basis for caching,
attestation, and provable idempotency.

### IV.3 The eight-phase universal loop

Every integration in pleme-io follows the same eight-phase enactment model.
If a phase is missing, the system has a convergence gap and the integration
is incomplete.

```
1. DECLARE    → Express desired state in types (IacType, CRD, OpenAPI, Nix module)
2. SIMULATE   → Run the full chain at zero cost (pangea-sim, ruby-synthesizer)
3. PROVE      → Verify invariants (proptest 10,000+ configs; rspec; kensa)
4. REMEDIATE  → Auto-fix violations before deployment (compliance-controls, kensa)
5. RENDER     → Generate artifacts (Backend trait, forge-gen, Helm, Nix)
6. DEPLOY     → Apply to infrastructure (FluxCD, Pangea→Terraform, kindling)
7. VERIFY     → Confirm convergence (InSpec, tameshi BLAKE3, health probes)
8. RECONVERGE → Detect drift; feed back to DECLARE (continuous loop)
```

The loop is not optional. Every new integration — a new IaC provider, a
new service type, a new customer onboarding flow, a new compute class — is
evaluated against the loop during design review. If a phase is implicit
(e.g., "we assume simulation is fine because we ran it by hand once"),
the integration is not done.

The `convergence-controller` repo is the reference implementation at the
cluster level: 12 MCP tools map to loop phases (`create` = declare,
`simulate` = simulate, `check_compliance` = prove, `deploy` = deploy,
etc.). Other integrations should mimic its shape.

### IV.4 Kubernetes clusters as Unix processes

Clusters are convergence processes — the Unix process model lifted from
node→pod to cluster→workload, enforced at every depth by the
`convergence-controller`.

| Unix concept | Convergence realization | Implementation |
|---|---|---|
| `fork()` | Cluster creation | `ConvergenceProcess` CRD → `InfrastructureFlow` |
| `exec()` | Bootstrap | `kindling-init` → K3s → FluxCD reconcile |
| `getpid()` / `getppid()` | Process identity | Hierarchical PID + `parent_pid` on status |
| `kill(SIGTERM)` | Graceful terminate | Controller SIGTERM handler drains direct children |
| `waitpid()` | Wait for child death | Finalizer blocks until child infra confirmed gone |
| `kill(SIGKILL)` | Force terminate | `grace_period_seconds: 0` after CRD timeout |
| `/proc` | Process table | `ProcessTable` CRD singleton (30s reconcile) |
| Orphan reaping | Init responsibility | `TableController` detects missing parent, terminates |
| Zombie detection | Stuck processes | `TableController` force-reaps after `zombieTimeoutSeconds` |
| `signal` | State change | Git commit → FluxCD reconciles |
| `stdout` | Observability | Grafana via LoadBalancer |

**DNS identity** is content-addressable:
`{service}.{name_or_hash}.{pid}.k8s.quero.lol`. The `name_or_hash` is a
BLAKE3 hash of the cluster spec (128-bit, base32, 26 chars) unless a name
is explicitly set. Examples:

| Process | PID | Role | DNS example |
|---|---|---|---|
| seph (init) | 1 | First process, spawns children | `grafana.seph.1.k8s.quero.lol` |
| lilitu | 2 | Product cluster | `api.lilitu.2.k8s.quero.lol` |
| drill | 3 | DR testing | `grafana.drill.3.k8s.quero.lol` |

**Self-hosting recursion.** The convergence controller's first expression
is creating itself. Each cluster past PID 1 contains its own controller,
which can spawn its own children. The tree is recursive; every node except
PID 1 is a schedulable node in the cluster matrix scheduler.

**Inception isolation.** PID 1 (seph) can manage itself with one
exception: its inception layer (VPC, ASG, NLB, IAM that created seph) is
protected from self-modification. This kernel/userspace distinction lets
us always destroy and recreate seph from the outside. The inception layer
is maintained by `pangea-operator` drift detection running independently.

### IV.5 Cluster matrix scheduler

K8s scheduling lifts from node→pod to cluster→workload. Every cluster past
PID 1 is a schedulable node.

| K8s (node level) | Convergence (cluster level) |
|---|---|
| `kube-apiserver` node list | `ProcessTable status.processes[]` |
| Node labels/taints | `ClusterQualities` (provider, region, compliance, operators) |
| Pod affinity/tolerations | `WorkloadRequirements` |
| `kube-scheduler` | `qualities_match()` + MCP `find_clusters()` / `schedule_workload()` |

Cluster config surface aligns with the CRD — same fields, snake_case YAML.

### IV.6 The seven questions

Before implementing any feature, a designer answers seven questions. If
they cannot answer all seven, the design is incomplete.

1. **What is the convergence declaration?** — What types express desired
   state?
2. **What are the invariants?** — What properties must hold at every
   checkpoint?
3. **What is the compliance baseline?** — Which frameworks apply (NIST,
   CIS, FedRAMP, SOC 2, PCI)?
4. **How does it render?** — What platform concretizes the declaration?
5. **How is it deployed?** — What agent drives declared→running?
6. **How is drift detected?** — How do you know running matches declared?
7. **How is it remediated?** — What happens automatically when drift
   occurs?

Review meetings that don't produce answers to all seven reschedule.

---

## Part V — Verification

### V.1 Construction guarantees — the knowable platform

A knowable platform is a computing environment where every capability is
proven by construction. You do not hope it works — you _know_ it works
because the proof exists.

**Programming model:** Types → Invariants → Proofs → Render Anywhere.

1. Define Rust types that make invalid states unrepresentable.
2. Write invariants as pure functions from state to pass/fail.
3. Prove with proptest over 10,000+ random configurations.
4. Map invariants to compliance controls (NIST/CIS/FedRAMP/PCI/SOC 2).
5. Render to any backend — proofs transfer because they attach to types.

**Domain coverage** (2,739 tests across 7 crates as of the current
checkpoint; updates as the fleet grows):

| Crate | Tests | Domain |
|---|---|---|
| `arch-synthesizer` | 1248 | Typescape, AST domains, morphisms, fixed points, 30 repo types, compliance lattice |
| `pangea-forge` | 548 | IacResource → Ruby + 8-artifact composition proofs |
| `pangea-sim` | 647 | All knowable domains (infrastructure, K8s, compliance, transitions, remediation, state machines, schemas, networks, security, business, composition, certification, self-proof) |
| `ruby-synthesizer` | 177 | Structural Ruby correctness by construction |
| `compliance-controls` | 14 | Type system for 5 compliance frameworks |
| `workspace-state-graph` | 62 | Typed DAG, cross-repo topology |
| `convergence-controller` | 43 | Cluster lifecycle, MCP tools |

**Theoretical foundations:** category theory (composition preserves
proofs), Curry-Howard (compiling programs ARE proofs), denotational
semantics (simulations map types to values; invariants are predicates over
values), lattice theory (compliance frameworks compose as lattice joins),
temporal logic (invariants hold at all checkpoints in a migration
sequence).

### V.2 Verification versus attestation (the term drift, fixed)

**Verification** and **attestation** are both colloquially called "proofs"
but they are distinct instruments. From this document forward:

- **Verification** is a type-level / property-level proof.
  `cargo test` is verification. proptest is verification. rspec synthesis
  is verification. InSpec post-deploy checks are verification. Mathematical
  in character; produces pass/fail, not evidence.
- **Attestation** is a cryptographic seal.
  BLAKE3 Merkle hashes are attestation. Ed25519 signatures are attestation.
  Akeyless Deterministic Signatures are attestation. Cryptographic in
  character; produces evidence that a specific declaration produced a
  specific artifact at a specific time.

Both are required. Neither substitutes for the other. A verified but
unattested artifact cannot be trusted in production (no proof of origin).
An attested but unverified artifact cannot be trusted either (proof of
origin but no proof of correctness).

### V.3 Three-pillar attestation

The canonical attestation artifact is a three-leaf BLAKE3 Merkle tree:

1. **artifact hash** — hash of the rendered output (the thing being
   attested).
2. **control hash** — hash of the configuration that produced it (the
   declaration).
3. **intent hash** — hash of the user-level intent that asked for it (the
   authorship context).

The three hashes compose into a root hash; the root is signed (Ed25519 or
Akeyless Deterministic). The signed root is the attestation.

`tameshi` implements the core. `sekiban` enforces it in-cluster (admission
webhook verifies the signature before admitting the resource). `kensa`
runs compliance checks and produces compliance attestations that feed into
the same chain. `inshou` applies the same discipline to Nix store paths
(pre-rebuild integrity verification).

**HeartbeatChain** records the ongoing audit trail — every attestation
event, every signature verification, every compliance check. Immutable.
Replayable.

### V.4 Two-phase signature composition

The attestation chain is two-phase by design:

1. **Phase 1 (untested):** the layer signature is produced the moment the
   artifact is rendered. It proves the artifact came from the declared
   inputs. It does not prove the artifact is safe to deploy.
2. **Phase 2 (secure):** after verification and compliance passes, a
   second signature is produced that composes the layer signature with
   the compliance attestation. Only Phase 2 signatures are admitted into
   production.

`sekiban` rejects Phase 1 signatures in production namespaces. Dev /
staging namespaces may accept Phase 1 under explicit policy. The phase is
part of the signature's metadata, not a separate registry.

---

## Part VI — Generation

### VI.1 Generation over composition

Hand-writing is the fallback. First question always: can this be
generated? If it can, it must be. If it can't, is that because the
generator doesn't cover it yet (fix the generator) or because the shape is
genuinely one-off (hand-write, once)?

Three-times rule: when a pattern repeats three times, extract an
archetype/backend/synthesizer and generate from it. Two occurrences is a
coincidence; three is a law.

Every generated artifact carries typed provenance:

- `.typescape.yaml` at the repo root — which dimensions contributed,
  content hashes, Merkle path.
- BLAKE3 attestation in `tameshi`'s chain.

Regenerating an artifact produces a byte-identical result given the same
inputs. Diffs across regenerations are drift and must be reconciled.

### VI.2 The canonical generation chains

Three generation chains cover the platform:

**API generation** (Pillar 3):
```
OpenAPI spec
  → sekkei (OpenAPI 3.0 typed load)
  → takumi (typed IR — FieldType, ResolvedSpec, CRUD grouping)
  → forge-gen (51 generators, one registry)
  → {SDKs (Rust/Go/Python/JS/Java/Swift), gRPC proto, GraphQL schema,
     REST server scaffold, MCP server, IaC providers, shell completions,
     docs}
```
Every downstream artifact reads the same IR. One spec change cascades to
every face.

**Infrastructure generation** (Pillars 5, 6, 7):
```
arch-synthesizer typescape slice
  → Pangea Ruby DSL (via pangea-forge)
  → Terraform JSON (via TerraformSynthesizer)
  → deployed resources

Same slice
  → Helm chart (via helm-synthesizer)
  → Kustomize overlay (via kustomize-synthesizer)
  → FluxCD HelmRelease / Kustomization
  → K8s manifests
  → deployed workloads

Same slice
  → Packer template (via packer-synthesizer)
  → AMI bake
  → EC2 launch template
```
All three outputs are provably consistent with the typescape slice.
Writing any of them by hand is drift.

**Repo generation** (Pillar 9):
```
repo-forge archetype (26 in catalog, 94.9% fleet coverage)
  → flake.nix (using substrate/lib/*)
  → Cargo.toml shape
  → .github/workflows/ci.yml
  → LICENSE, .gitignore, .envrc, CLAUDE.md scaffold
  → HM module wiring
```
Every new repo in the fleet goes through `repo-forge new`. Every existing
repo is periodically re-absorbed (`repo-forge absorb-org pleme-io`) to
detect drift. Migration (`repo-forge migrate --apply-boilerplate-only`)
fixes the drift without touching authored regions.

### VI.3 File taxonomy — Boilerplate / Hybrid / Authored

Every file in the fleet falls into exactly one class:

- **Boilerplate** — entirely generated, no authored content. `LICENSE`,
  `.gitignore`, `.envrc`, CI workflows, initial flake structure, initial
  `Cargo.toml` shape. Owned by `repo-forge`. Rewritten on every migration.
- **Hybrid** — partly generated, partly authored. A `flake.nix` where the
  outer structure is archetype-rendered but `devShells.default.buildInputs`
  is hand-curated. A `Cargo.toml` where the `[package]` section is
  archetype but `[dependencies]` is authored. Owned by the repo; migration
  reports drift but does not overwrite.
- **Authored** — entirely hand-written. Source code in `src/`, tests in
  `tests/`, docs in `docs/`, member `Cargo.toml`s, custom `devShells`
  bodies, richer HM options beyond the archetype default set. Owned by the
  author. `repo-forge` never touches these.

The taxonomy is what makes the generation loop safe. A migration run can
overwrite Boilerplate freely; must be conservative with Hybrid; must never
touch Authored.

### VI.4 The three construction questions (before any generator runs)

1. **What typed source declares this shape?** (the typescape slice)
2. **What morphism renders it?** (the Backend / Synthesizer impl)
3. **What attestation seals it?** (the three-pillar BLAKE3 chain)

A generator without answers to all three is not a generator; it is a
script. Scripts are forbidden (see Part II.4).

---

## Part VII — Operation

### VII.1 Attestation-gated deployments

Every deployment to production is gated on attestation. The gate is
structural, not a policy overlay:

- `sekiban` admission webhook runs in every cluster. A resource without a
  valid Phase 2 signature (V.4) is refused at the K8s API server.
- `kensa` runs pre-deploy compliance checks as part of the Terraform apply
  path. A Pangea architecture that fails compliance does not produce a
  plan, let alone apply.
- `inshou` runs pre-rebuild integrity checks for Nix store paths. A
  closure without a valid attestation does not activate.

There is no "exceptions" flag. There is a dev/staging/prod policy
distinction (V.4): prod requires Phase 2; dev/staging may accept Phase 1
under explicit policy. The distinction is declarative — part of the
`sekiban` `CompliancePolicy` CRD, not a CLI argument.

### VII.2 GitOps through FluxCD

FluxCD is the process manager for every cluster. Git commit is the
signal; reconciliation is the fixed-point controller; `Kustomization` and
`HelmRelease` are the process units; `ResourceSet` is the process group.

The shape is invariant across all clusters:

- One `FluxCD` instance per cluster, at a well-known namespace.
- Cluster config lives in a single repo (`k8s/`, `plo/`, `zek/`, cluster-
  specific as appropriate).
- Per-cluster overlays layer over shared bases using Kustomize. Sealed
  secrets use SOPS with age; `encrypted_regex: "^(data|stringData)$"` in
  `.sops.yaml` (NOT `unencrypted_suffix`).
- Reconciliation timeouts, health checks, and drift detection are declared,
  not configured at runtime.

Any manual `kubectl apply` to a FluxCD-managed cluster is drift. It will
be reconciled away within the FluxCD interval. If drift is what you want,
it goes through Git.

### VII.3 JIT Infrastructure (Pillar 11)

Every compute unit is a typed composition:

```
auction substrate ⊕ breathability ⊕ state externalization
   ⊕ interruption policy ⊕ substrate provider
```

Default: spot + breathable. Only structural exemption: EKS managed control
planes (which spot can't reach).

**Mandatory alert layer.** Every spot workload carries a typed
`AlertLayer` (source + forwarder + sinks) with `required: true` by
default. Renderers refuse to emit without a validated alert layer. This is
not policy; it is a rendering invariant.

**JIT fleet pattern** (formalized under `pangea-jit-builders` skill): a
scale-to-zero ASG woken on demand by an SSH ProxyCommand (`cordel
builder-wake`), scaled back down by a client-side watchdog and a
server-side CloudWatch alarm. The pattern generalizes to any cloud
capacity that must be available on demand but idle otherwise: build
fleets, GPU bursts, specialized k8s-builder pools, one-off staging
environments.

Cost model (reference): ~$3/month idle, ~$0.001 per 2-minute build.

### VII.4 Convergence processes and their operators

The running platform is a tree of convergence processes. At the root:

- **seph** (PID 1) — init. Inception layer: VPC, ASG, NLB, IAM.
  Manages its children but not itself (IV.4 inception isolation).
- **lilitu** (PID 2) — primary product cluster. API, services, user
  data.
- **drill** (PID 3) — DR testing cluster. Identical shape to lilitu;
  exists to verify DR recovery.

Every process is managed by its parent except PID 1, which is managed by
`pangea-operator` drift detection running outside the process tree.

### VII.5 MCP as the operator surface

The ultimate operator interface is MCP (Model Context Protocol). An LLM
agent manages the creation engine through typed tools. No `kubectl`. No
SSH. No manual operations.

```
LLM (Claude) ──MCP──→ convergence-controller MCP server
                        ├── create_platform(spec)
                        ├── create_process(name, config)
                        ├── get_process_table()
                        └── query_observability(sql)  ← shinryu-mcp
```

The LLM is the operator. The system is fully compliant, fully observable,
and closed to direct human manipulation beyond the MCP surface.

### VII.6 Live convergence mutation

Convergence is continuous. While a system is converging, new elements
merge in — new services, new compliance requirements, new infrastructure
— gated by testing and integrity verification. FluxCD's reconciliation
loop handles this naturally: Git commit adds new resources → FluxCD
applies alongside existing → new Phase 2 signatures are produced as
compliance checks pass → the new resources admit.

The mutation ingress system validates before merging: schema validation
→ compliance check (kensa) → test gate (kenshi) → integrity verification
(sekiban) → canary analysis → merge.

---

## Part VIII — The Connecting Threads

This part names the structural observations that tie the pillars into a
coherent theory. Each thread has been implicit across Parts I–VII; Part
VIII makes them explicit and unbreakable.

### VIII.1 The four Lisps collapse to one typed IR

Formalized in Part II.2. Expanded here:

All four authoring surfaces (tatara-lisp, Nix, Pangea Ruby, OpenAPI YAML)
produce values in the same canonical SExpr IR
(`iac_forge::sexpr::SExpr`). The IR is content-addressable (BLAKE3). The
IR feeds into the typescape as a typed projection. The typescape renders
through morphisms into every artifact.

The consequence: anything authored in any of the four Lisps is
attestable, verifiable, composable, and renderable by the same machinery.
The "four Lisps" are not four systems; they are four authoring surfaces
of one system.

### VIII.2 Typescape dimensions ≅ composition artifacts

The typescape has eight dimensions (III.1). The composition-testing
framework (`composition-testing` skill) verifies generation across eight
artifact types: Pangea workspaces, Packer, compliance architecture, Helm,
Kustomize, FluxCD, NixOS, repo scaffold.

**Claim:** the 8 dimensions and the 8 artifact types are isomorphic. They
are the same structure viewed from inside the typescape (as dimensions)
versus outside the typescape (as artifacts).

| Typescape dimension | Composition artifact |
|---|---|
| Vocabulary | repo scaffold |
| AST domains | Packer |
| Morphisms | NixOS |
| Fixed points | FluxCD |
| Compliance lattice | compliance architecture |
| Workspace DAG | Pangea workspaces |
| Render state | Helm |
| Module manifests | Kustomize |

This mapping is not arbitrary; it is a consequence of the pillars. When a
new dimension or artifact is proposed, it must either map into an
existing row or force a new row on both sides simultaneously.

### VIII.3 AST domains ↔ tameshi LayerTypes

19 AST domains (III.2). 14 `tameshi::LayerType` variants. The two lists
are not coincident; they are related.

- AST domains partition the _grammar_ space: one per infrastructure
  format.
- LayerTypes partition the _trust_ space: one per attestation
  layer type (Nix, OCI, Helm, Tofu, Kubernetes, Kindling, Tatara, FluxCD,
  ArgoCD, Akeyless, AkeylessTarget, + 3 math/ML domains).

Every deployable artifact has an AST domain (what syntax it's in) and a
LayerType (what trust layer it belongs to). The join is structural: a
Helm chart is in AST domain `YAML` (sub-dialect: Helm template) and
LayerType `Helm`. A Terraform plan is in AST domain `HCL` and LayerType
`Tofu`.

**Claim:** every LayerType has an AST domain; not every AST domain has a
LayerType (Markdown doesn't, for instance). The projection LayerType → AST
domain is total; the reverse is partial.

### VIII.4 Unix processes ↔ cluster scheduling ↔ agent lifecycles

The Unix process model governs three distinct levels in the platform:

1. **OS-level processes** — the actual `pid_t` values inside a node.
2. **Cluster-level processes** — `ConvergenceProcess` CRDs with
   hierarchical PIDs (IV.4).
3. **Agent-level lifecycles** — MCP agent sessions managed by kurage,
   fleet, or direct invocation.

All three use the same vocabulary: fork, exec, wait, kill, orphan,
zombie, reap. The discipline at every level is Unix's: every process has
a parent, every process has a lifecycle, every process is reapable.

The consequence: infrastructure reasoning, cluster reasoning, and agent
reasoning use the same mental model. An operator moves between them
without learning a new vocabulary.

### VIII.5 Nix ↔ sui convergence duality

| Property | Nix (single-node) | sui (distributed) |
|---|---|---|
| Evaluation | `nix eval` (lazy, pure) | `sui eval` (bytecode VM, 3× faster than CppNix on 45/48 benchmarks) |
| Build | `nix build` (sandbox) | `sui build` (NATS-triggered) |
| Cache | Local store + Attic | Attic (shared convergence memory) |
| Activation | `switch-to-configuration` | NATS publish (downstream convergence) |
| Proof | Store path (content hash) | Store path (same proof, distributed) |

The same discipline operates at both scales. `sui` is what Nix looks like
when extended to a NATS-coordinated fleet; Nix is what `sui` looks like
when restricted to a single node.

### VIII.6 The lattice is the structure; convergence is the motion

Named here because it's the most-missed connection. The lattice (III.1.5
+ III.1.4) and the convergence process (IV) are not two systems; they are
two views of one system.

- The lattice defines _what states exist_ and _how they compose_.
- Convergence defines _which state the system moves toward_ and _how
  fast_.

Static and dynamic. Always both. A system with a lattice but no
convergence is a static specification (incomplete). A system with
convergence but no lattice is a dynamic simulation (unbounded).

### VIII.7 NixOS / darwin / HM module = universal lattice primitive

Every system in the stack has an equivalent composable unit with typed
interfaces. The NixOS/darwin/HM module is the reference primitive. Other
systems use a structural analog:

| System | Lattice primitive | Join (compose) | Meet (constrain) |
|---|---|---|---|
| NixOS / darwin / HM | Module | `imports = [a b]` | `lib.mkIf cond` |
| Pangea | Architecture | `.build(synth, config)` | `Dry::Struct` validation |
| Helm | Chart values | `-f a.yaml -f b.yaml` | `{{- if .Values.x }}` |
| FluxCD | Kustomization | `dependsOn: [a, b]` | `spec.prune` |
| Kubernetes | Manifest | `kustomize build` | Admission webhooks |
| tameshi | AttestationLayer | Merkle composition | Phase gates |

**Design rule:** `lib.mkForce` (and its equivalents in other systems) is a
smell, not a tool. If you're using it, the lattice structure is wrong.
Restructure the modules so that bare values resolve through natural
priority ordering.

### VIII.8 Every business is a convergence declaration

A business IS a convergence declaration rendered into running software:

```
OpenAPI Spec (intent)
  → forge-gen (SDKs, MCP, IaC, Terraform, Helm, shell completions)
    → Pangea architectures (infrastructure compositions)
      → FluxCD (continuous reconciliation)
        → Running services on every platform (business realized)
```

One spec change cascades through the entire generation + deployment
chain. The running system IS the business, converged from a single
declaration. There is no "infrastructure team" separate from the
"product team"; there is one declaration and one convergence process.

---

## Part IX — Quick Reference

### IX.1 The one-line summary

> **Rust owns types. Lisp owns flow. shikumi owns config. forge-gen owns
> APIs. Pangea owns infrastructure. arch-synthesizer owns the typescape.
> Nix owns images. Helm+Kustomize+FluxCD own deploys. substrate owns the
> SDLC. tameshi owns the proof chain. JIT Infrastructure owns compute.
> Generation over composition, every time.**

### IX.2 The six-line contract (for every new typed domain)

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
#[tatara(keyword = "defmydomain")]
pub struct MyDomainSpec { /* fields */ }

pub fn register() {
    tatara_lisp::domain::register::<MyDomainSpec>();
}
```

### IX.3 The seven questions (before any design review)

1. What is the convergence declaration?
2. What are the invariants?
3. What is the compliance baseline?
4. How does it render?
5. How is it deployed?
6. How is drift detected?
7. How is it remediated?

### IX.4 The eight-phase loop (for every integration)

```
DECLARE → SIMULATE → PROVE → REMEDIATE → RENDER → DEPLOY → VERIFY → RECONVERGE
```

### IX.5 The three attestation hashes (for every artifact)

```
artifact hash ⊕ control hash ⊕ intent hash
  → BLAKE3 Merkle root
  → Ed25519 (or Akeyless Deterministic) signature
  → Phase 1 → verification+compliance → Phase 2
```

### IX.6 Where each topic is deeply treated

| Topic | Primary source |
|---|---|
| Rust+Lisp pattern | `pleme-io/tatara/docs/rust-lisp.md` |
| Twelve pillars | `pleme-io/BLACKMATTER.md` |
| Typescape | `pleme-io/arch-synthesizer/src/typescape.rs` + `skills/typescape/` |
| AST domains | `pleme-io/arch-synthesizer/src/ast_domains.rs` + `skills/ast-domains/` |
| Convergence flow | `skills/convergence-flow/` (the 8-phase loop) |
| Convergence processes | `pleme-io/convergence-controller/` |
| Knowable platform | `pleme-io/pangea-sim/` + `skills/knowable-platform/` |
| Compliance controls | `pleme-io/compliance-controls/` + `skills/compliant-systems/` |
| Attestation | `pleme-io/tameshi/` + `skills/attestation/` |
| JIT capacity | `skills/pangea-jit-builders/` |
| Generation pipeline | `pleme-io/forge-gen/` + `pleme-io/iac-forge/` + `skills/generation-process/` |
| Repo scaffolding | `pleme-io/repo-forge/docs/principles.md` + `skills/repo-forge/` |
| Pangea DSL | `pleme-io/pangea-core/` + `skills/pangea-architecture/` + `skills/pangea-resource/` |
| SDLC (Nix) | `pleme-io/substrate/` + `skills/build/` |
| MCP operator surface | `pleme-io/convergence-controller/mcp/` + `skills/convergence-flow/` |

### IX.7 Forbidden/permitted quick check

- Bash beyond 3-line glue: **forbidden**
- Dockerfile: **forbidden**
- Hand-written Terraform HCL: **forbidden** (use Pangea)
- Hand-written Kustomize overlay outside arch-synthesizer output:
  **forbidden**
- Production Python: **forbidden**
- Server-side TypeScript: **forbidden** (frontend edge only)
- Amending a published commit: **forbidden** without explicit request
- `--no-verify`, `--no-gpg-sign`: **forbidden** without explicit request
- `lib.mkForce` and equivalents: **smell** (restructure instead)

---

## Naming

- **Japanese** names the discipline layer: tatara, shikumi, sekkei,
  takumi, forge, kenshi, shinka, sekiban, tameshi, kensa, sui,
  blackmatter.
- **Brazilian-Portuguese** names enclosed spaces, flows, growth, craft:
  terreiro, forja, cordel, cerrado, jabuti, samba, roça, bordado,
  caderneta, apurador.
- When in doubt: foundational crate or pillar primitive → Japanese;
  Tier 2+ primitive evoking space/flow/growth → Brazilian-Portuguese.

---

## Closing

Every pillar is a claim about ownership. Every thread in Part VIII is a
claim about structure. Every part of this document is a claim pleme-io
has committed to — not a preference, not a suggestion.

If you are writing new code, it either fits the pillars or it doesn't. If
it fits, the six-line contract / the eight-phase loop / the seven
questions tell you where to go next. If it doesn't, you either found a
gap worth extending the theory for or you're about to drift.

The duplication budget is zero. The standardization budget is unlimited.
Stop at diminishing returns.

---

> Rust is the kernel. Lisp is the interface. The derive is the bridge.
> The typescape is the source. The morphism is the rendering. The
> controller is the motion. The attestation is the seal. The lattice is
> the structure. The process is the life. Everything else compounds on
> top.
