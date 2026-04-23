# VOCABULARY — pleme-io canonical terms

Every named concept, verb, primitive, or idiosyncrasy used in pleme-io
documentation appears here exactly once. When a term has historically
been used with two distinct meanings, this document splits it into two
rows with distinct names and fixes the callers.

If you use a term in a `CLAUDE.md`, skill, or other document, it must
already be defined here. If it isn't, add the row first.

Companion: [`THEORY.md`](./THEORY.md) is where these terms are composed
into the unified theory.

---

## Index by category

- [Pillars](#pillars)
- [Language primitives](#language-primitives)
- [Structure](#structure)
- [Motion](#motion)
- [Verification](#verification)
- [Generation](#generation)
- [Operation](#operation)
- [Naming](#naming)
- [Term drift resolutions](#term-drift-resolutions)

---

## Pillars

| Term | Definition | Canonical source |
|---|---|---|
| **Blackmatter Development** | The official pleme-io software methodology. Twelve pillars, enforced on every repo. Deviation requires explicit `skip-blackmatter:` note. | `pleme-io/BLACKMATTER.md` |
| **Pillar** | One of the twelve load-bearing commitments of Blackmatter Development. Each pillar names what owns a concern. Not a preference — an exclusive owner. | `pleme-io/BLACKMATTER.md` §1–12 |
| **Pillar 1 — Language** | Rust + tatara-lisp + WASM/WASI. Never shell, production Python, or server-side TypeScript. | BLACKMATTER.md §1 |
| **Pillar 2 — Configuration** | `shikumi` as THE configuration library. Typed, hot-reloadable, YAML at `~/.config/{app}/{app}.yaml`. | BLACKMATTER.md §2 |
| **Pillar 3 — API generation** | `forge-gen`. OpenAPI spec → SDKs + gRPC + GraphQL + REST + MCP + IaC + completions + docs. One spec, N faces. | BLACKMATTER.md §3 |
| **Pillar 4 — Data layer** | `SeaORM` + `shinka` migrations. No raw SQL outside migration files. | BLACKMATTER.md §4 |
| **Pillar 5 — Infrastructure declaration** | Pangea Ruby DSL → Terraform JSON. Every cloud resource. | BLACKMATTER.md §5 |
| **Pillar 6 — Typescape** | `arch-synthesizer` owns every Rust primitive type and proof. `RenderBackend` trait is the one entry. | BLACKMATTER.md §6 |
| **Pillar 7 — Kubernetes control** | Helm + Kustomize + FluxCD rendered from typescape. Never hand-authored. | BLACKMATTER.md §7 |
| **Pillar 8 — Image building** | Nix only. Never Dockerfiles. | BLACKMATTER.md §8 |
| **Pillar 9 — SDLC** | `substrate` + `repo-forge` archetypes + `nix run .#app` uniform commands. | BLACKMATTER.md §9 |
| **Pillar 10 — Proof discipline** | `cargo test` = compliance verification. `rspec` + `proptest` + `tameshi` BLAKE3 chain. | BLACKMATTER.md §10 |
| **Pillar 11 — JIT Infrastructure** | Spot + breathability + mandatory alert layer. Every compute unit. | BLACKMATTER.md §11 |
| **Pillar 12 — Generation over composition** | Hand-writing is the fallback. Every recurring shape becomes a generator. | BLACKMATTER.md §12 |

---

## Language primitives

| Term | Definition | Canonical source |
|---|---|---|
| **Rust + Lisp pattern** | The primary architectural concept of pleme-io. Rust owns types and invariants; Lisp owns declarative authoring and macros; the boundary is `#[derive(TataraDomain)]`. | `tatara/docs/rust-lisp.md` |
| **TataraDomain** | The proc macro that makes a Rust struct authorable as a Lisp form `(defX :k v …)`. One line per new domain. | `tatara-lisp-derive/` |
| **Six-line contract** | The minimum ceremony to introduce a new typed domain: derive + keyword + struct + register fn. | THEORY.md §II.1 |
| **Five invariants (of Rust+Lisp)** | Typed entry, free middle, typed exit, deterministic identity, composition preserves proofs. | THEORY.md §II.1; rust-lisp.md §"The five invariants" |
| **tatara-lisp** | The strictest of the four Lisps. Native S-expressions. Macros as term rewriting. Typed via `#[derive(TataraDomain)]`. | `tatara-lisp/` |
| **Four Lisps** | tatara-lisp, Nix, Pangea Ruby DSL, OpenAPI YAML. Four authoring surfaces, all reducible to the same typed IR (`iac_forge::sexpr::SExpr`). | THEORY.md §II.2 |
| **Nix (as Lisp)** | The infrastructure Lisp. Lazy functional evaluation. Module system is a lattice algebra. Overlays are functor composition. | THEORY.md §II.2, §VIII.1, §VIII.7 |
| **Pangea (as Lisp)** | The declarative infrastructure Lisp. Ruby blocks are s-expressions with syntactic sugar. Typed via `Dry::Struct`. Method_missing is macro expansion. | THEORY.md §II.2 |
| **OpenAPI (as Lisp)** | The spec Lisp. YAML is a restricted s-expression encoding. Typed via JSON Schema + `sekkei` + `takumi`. Looser than the other three but still part of the family. | THEORY.md §II.2 |
| **Typed IR** | Canonical SExpr form (`iac_forge::sexpr::SExpr`). Every Lisp in the Four reduces to it. Content-addressable (BLAKE3). | THEORY.md §VIII.1 |
| **shikumi** | The configuration library. `ConfigDiscovery::new("app")`, `ConfigStore::<T>::load(…)`. ArcSwap hot-reload. | Pillar 2; `shikumi/` |
| **sekkei** | Canonical OpenAPI 3.0 serde types + spec loading. Used by `takumi`, `forge-gen`, every OpenAPI consumer. | `sekkei/` |
| **takumi** | OpenAPI → typed IR pipeline. `FieldType` enum, `ResolvedSpec`, CRUD grouping. Consumed by `forge-gen` and `iac-forge`. | `takumi/` |
| **WASM / WASI** | Portable execution substrate for Lisp-compiled runtime extensions. Default system interface for plugins. | Pillar 1 |

---

## Structure

| Term | Definition | Canonical source |
|---|---|---|
| **Typescape** | The root data structure of the platform. The universe of all types, queryable, attestable, composable across eight dimensions. Lives as typed Rust in `arch-synthesizer/`. | THEORY.md §III.1 |
| **SystemTypescape** | The concrete Rust type implementing the Typescape. `SystemTypescape::for_domain("x").summary()`. | `arch-synthesizer/src/typescape.rs` |
| **Typescape dimension** | One of eight: vocabulary, AST domains, morphisms, fixed points, compliance lattice, workspace DAG, render state, module manifests. _Distinct from_ "convergence classification dimension" and "loop phase" — see term drift below. | THEORY.md §III.1 |
| **Vocabulary (dimension)** | 230+ canonical terms in five categories (Primitive, Composition, Lattice, Mutation, Meta). | `arch-synthesizer/src/vocabulary.rs` |
| **AST domain** | A finite set of irreducible primitives for one infrastructure format (Ruby, YAML, HCL, Nix, Rust, Go, Python, TypeScript, OpenAPI, SQL, CRD, FluxCD, Markdown, etc.). 19 domains cataloged. Six universal primitive categories: Literal, Sequence, Mapping, Declaration, Annotation, Escape. | `arch-synthesizer/src/ast_domains.rs` |
| **Morphism** | A structure-preserving map between AST domains. Implemented by the `Synthesizer` trait. Proven total and deterministic. 12+ morphisms form a connected graph. | THEORY.md §III.2 |
| **Backend** | A morphism's rendering target. The `Backend` trait has 7 implementations (Terraform, Pulumi, Crossplane, Ansible, Pangea, Steampipe, custom). Adding a backend inherits all existing proofs. | `iac-forge/` |
| **Synthesizer** | The Rust trait that implements a morphism. `impl Synthesizer<Src, Tgt>`. | `arch-synthesizer/` |
| **Compliance lattice** | Meet/join/intersect over 50 compliance primitives in 6 baselines across 5 frameworks (NIST 800-53, CIS, FedRAMP, PCI DSS 4.0, SOC 2 Type II). A dimension of the typescape (III.1.5), not a separate system. | THEORY.md §III.3, §V.1 |
| **Compliance primitive** | One atomic control (e.g., "all EBS encrypted", "no public SSH"). 50 cataloged. | `compliance-controls/` |
| **Baseline** | A set of compliance primitives that together define a certification goal (e.g., FedRAMP Moderate, CIS AWS v3.0, SOC 2 Type II, PCI DSS 4.0). 6 cataloged. | `pangea-sim/src/compliance/` |
| **Compliance framework** | A vendor-defined certification scheme (NIST 800-53, CIS, FedRAMP, PCI DSS, SOC 2). 5 recognized. | `compliance-controls/` |
| **Workspace DAG** | The typed cross-state reference graph across Pangea workspaces. Parent outputs flow into child `RemoteState.output`. State keys `pangea/{parent}/{child}` are deterministic. | THEORY.md §III.1.6 |
| **RemoteState** | The Pangea mechanism for cross-workspace output references. Type-safe; failures at resolve time. | `pangea-core/lib/remote_state.rb` |
| **Render state (dimension)** | Content-addressable index of every generated artifact. BLAKE3(input) ↔ BLAKE3(output). The index is cache and attestation at once. | THEORY.md §III.1.7 |
| **TypescapeManifest** | `.typescape.yaml` at the root of every repo. Records which dimensions contributed, content hashes, Merkle path back to typescape root. | THEORY.md §III.1.8 |
| **IacType** | Platform-independent IaC type in the `iac-forge` IR. Morphism source for code generation. Variants: String, Integer, Float, Boolean, List, Set, Map, Object, Enum, Any. | `iac-forge/src/ir/` |
| **IacResource** | A concrete infrastructure resource definition in the IR. The output of `iac-forge` sync; input to the backend rendering step. | `iac-forge/src/ir/` |
| **IacAttribute** | A single field on an IacResource. Has a type, a sensitivity flag, a required flag, an immutable flag. | `iac-forge/src/ir/` |

---

## Motion

| Term | Definition | Canonical source |
|---|---|---|
| **Convergence (unqualified)** | By default, convergence-as-process (the dynamic motion toward a declared state). When the static structure is meant, say "convergence lattice" or "lattice". See [Term drift resolutions](#term-drift-resolutions). | THEORY.md §IV.1 |
| **Convergence-as-lattice** | The static structure of all possible states the system can legitimately inhabit. A dimension of the typescape. | THEORY.md §IV.1 |
| **Convergence-as-process** | The dynamic motion of the running world toward a lattice point. Driven by controllers. Default meaning of "convergence" in pleme-io docs. | THEORY.md §IV.1, §IV.2 |
| **Controller** | A function `f(x_n, d) → x_{n+1}` that runs until `f(x_*, d) = x_*` (the fixed point). 11 instances across the platform. | THEORY.md §IV.2 |
| **Fixed point** | The state `x_*` such that `f(x_*, d) = x_*` for a controller `f` and declared state `d`. Convergence complete when `x_* = d`. | THEORY.md §IV.2 |
| **Convergence strategy** | One of Declarative, DiffAndPatch, FullRebuild, EventDriven. Every controller uses exactly one. | THEORY.md §IV.2 |
| **Eight-phase loop** | DECLARE → SIMULATE → PROVE → REMEDIATE → RENDER → DEPLOY → VERIFY → RECONVERGE. Every integration must implement all eight. | THEORY.md §IV.3 |
| **Loop phase** | One step of the eight-phase loop. Distinct from "typescape dimension" and "classification dimension" — see term drift below. | THEORY.md §IV.3 |
| **ConvergenceController** | The Rust trait and operator that enforces the eight-phase loop at the cluster level. 12 MCP tools, 43 tests. | `convergence-controller/` |
| **ConvergenceProcess** | The K8s CRD representing a cluster as a Unix-style process. Has PID, ppid, state, qualities. | `convergence-controller/crates/` |
| **ProcessTable** | K8s CRD singleton implementing `/proc`. 30s reconcile. Orphan reaping. Zombie detection. | THEORY.md §IV.4 |
| **Hierarchical PID** | Integer identity for every `ConvergenceProcess`. PID 1 = seph (init). Child PID > parent PID. | THEORY.md §IV.4 |
| **DNS identity** | `{service}.{name_or_hash}.{pid}.k8s.quero.lol`. Content-addressable via BLAKE3 cluster spec hash. | THEORY.md §IV.4 |
| **seph** | PID 1. Init cluster. Manages inception layer (VPC, ASG, NLB, IAM) _except_ its own — see inception isolation. | THEORY.md §VII.4 |
| **lilitu** | PID 2. Primary product cluster (API, services, user data). | THEORY.md §VII.4 |
| **drill** | PID 3. DR testing cluster. Identical shape to lilitu; verifies DR. | THEORY.md §VII.4 |
| **Inception isolation** | PID 1 cannot modify its own inception layer. Managed externally by `pangea-operator` drift detection. | THEORY.md §IV.4 |
| **Cluster matrix scheduler** | K8s scheduling lifted from node→pod to cluster→workload. `find_clusters(requirements)` + `schedule_workload(name, requirements)` MCP tools. | THEORY.md §IV.5 |
| **ClusterQualities** | The set of labels/traits a cluster advertises (provider, region, compliance, operators). Analog of K8s node labels. | `convergence-controller/` |
| **WorkloadRequirements** | The set of constraints a workload imposes. Analog of K8s pod affinity/tolerations. | `convergence-controller/` |
| **Self-hosting recursion** | The convergence controller's first expression is creating itself. Every cluster past PID 1 contains its own controller. | THEORY.md §IV.4 |
| **Seven questions** | The design-review forcing function. Declaration, invariants, baseline, render, deploy, drift, remediate. | THEORY.md §IV.6 |

---

## Verification

| Term | Definition | Canonical source |
|---|---|---|
| **Knowable platform** | A computing environment where every capability is proven by construction. 2,739 tests across 7 crates prove the pleme-io platform. | THEORY.md §V.1 |
| **Construction guarantee** | An invariant enforced by the Rust type system — invalid states unrepresentable. Bugs prevented, not caught. | THEORY.md §V.1 |
| **Verification** | Type-level or property-level proof. `cargo test`, proptest, rspec synthesis, InSpec post-deploy. Mathematical. Pass/fail. _Distinct from_ attestation. | THEORY.md §V.2 |
| **Attestation** | Cryptographic seal. BLAKE3 Merkle + Ed25519 (or Akeyless Deterministic) signature. Evidence of origin + correctness. _Distinct from_ verification. | THEORY.md §V.2 |
| **Three-pillar attestation** | artifact hash ⊕ control hash ⊕ intent hash → BLAKE3 Merkle root → signature. The canonical attestation form. | THEORY.md §V.3 |
| **CertificationArtifact** | The tameshi Rust type holding a three-pillar attestation. | `tameshi/src/artifact/` |
| **LayerType** | One of 14 tameshi variants classifying what trust layer an artifact belongs to (Nix, OCI, Helm, Tofu, Kubernetes, Kindling, Tatara, FluxCD, ArgoCD, Akeyless, AkeylessTarget, + 3 ML/math). | `tameshi/src/layer_type.rs` |
| **HeartbeatChain** | Immutable audit trail of attestation events. Every signature verification, every compliance check. Replayable. | `tameshi/src/heartbeat/` |
| **Phase 1 (signature)** | Layer signature produced when artifact is rendered. Proves origin; does not prove safety to deploy. | THEORY.md §V.4 |
| **Phase 2 (signature)** | Signature composing Phase 1 with compliance attestation. Only Phase 2 admitted to production. | THEORY.md §V.4 |
| **sekiban** | K8s admission webhook + reconciler. Refuses resources without valid Phase 2 signatures. | `sekiban/` |
| **kensa** | Compliance engine. NIST 800-53 / OSCAL mapping. Pre-deploy gate via `terraform` resource type. | `kensa/` |
| **inshou** | Nix integrity gate. Pre-rebuild verification of store path BLAKE3 hashes. | `inshou/` |
| **tameshi** | The core attestation library. 11 layer types, 3-leaf Merkle, Ed25519 signing, HeartbeatChain. | `tameshi/` |
| **kanshi** | eBPF runtime verifier. Continuous attestation from inside the kernel. | `kanshi/` |
| **proptest** | Property-based testing framework. Used for invariant verification over 10,000+ random configs. | `pangea-sim/` |
| **rspec synthesis** | Zero-cost infrastructure testing. Rspec tests Ruby-generated Terraform JSON structure + compliance controls. | `pangea-architectures/` |
| **InSpec** | Real-API post-deploy compliance verification. 10 controls over SecureVpc baseline. | `inspec-secure-vpc/`, `inspec-akeyless/` |
| **Curry-Howard alignment** | Types are propositions; programs are proofs; compiling is proof-checking. Applied via `ProofWitness<P>` phantom types. | `arch-synthesizer/` |

---

## Generation

| Term | Definition | Canonical source |
|---|---|---|
| **Generation over composition** | Pillar 12. Hand-writing is the fallback. First question: can this be generated? | BLACKMATTER.md §12 |
| **Three-times rule** | When a pattern repeats three times, extract an archetype/backend/synthesizer. Two is coincidence; three is law. | THEORY.md §VI.1 |
| **forge-gen** | The API generation CLI. 51 generators in one registry. `--sdks --servers --iac --mcp --completions --schemas --docs`. | `forge-gen/` |
| **iac-forge** | Platform-independent IaC code generation. TOML+OpenAPI or Terraform schema → `IacResource` IR → backend. | `iac-forge/` |
| **terraform-forge / pulumi-forge / crossplane-forge / ansible-forge / pangea-forge / steampipe-forge** | Backend implementations of `iac-forge::Backend`. Each renders the same IR to a specific IaC platform. | `{terraform,pulumi,crossplane,ansible,pangea,steampipe}-forge/` |
| **mcp-forge** | OpenAPI 3.0.3 → Rust MCP server codegen (spec → IR → types + client + MCP tools + CLI + Nix). | `mcp-forge/` |
| **completion-forge** | OpenAPI 3.0.3 → shell completions (spec → IR → skim-tab YAML + fish). | `completion-forge/` |
| **arch-synthesizer** | Infrastructure synthesis from typescape slices. Owns the `Backend` trait, the `RenderBackend` trait, and every primitive type. | `arch-synthesizer/` |
| **repo-forge** | The typed-archetype repo renderer. 26 archetypes, 94.9% fleet coverage. `absorb` (read-only), `new`, `check`, `migrate`. | `repo-forge/` |
| **archetype** | A typed template that renders a complete repo scaffold. Identified by a name (e.g., `rust-substrate-tool`). | `repo-forge/` |
| **Boilerplate / Hybrid / Authored** | File taxonomy. Boilerplate = generated, overwritable. Hybrid = partly generated, drift-reported only. Authored = hand-written, never overwritten. | THEORY.md §VI.3 |
| **substrate** | The Nix build-pattern library. `rust-tool-release-flake.nix`, `rust-workspace-release-flake.nix`, `rust-library.nix`, `rust-service-flake.nix`, and the full `lib/` catalog of `mk<Thing>` builders and `hm-*-helpers`. | `substrate/` |
| **Three construction questions** | Before any generator runs: (1) typed source? (2) morphism? (3) attestation? | THEORY.md §VI.4 |
| **Canonical generation chain** | The three canonical generation flows: API (OpenAPI → forge-gen), Infrastructure (typescape → Pangea/Helm/Packer), Repo (archetype → flake + Cargo). | THEORY.md §VI.2 |

---

## Operation

| Term | Definition | Canonical source |
|---|---|---|
| **FluxCD** | The GitOps reconciler. Fixed-point controller. One instance per cluster. Git commit = signal; reconciliation loop = motion. | THEORY.md §VII.2 |
| **HelmRelease** | FluxCD CRD wrapping a Helm chart. Rendered from typescape, not hand-authored. | Pillar 7 |
| **Kustomization** | FluxCD CRD wrapping a Kustomize overlay. Rendered from typescape, not hand-authored. | Pillar 7 |
| **ResourceSet** | FluxCD CRD grouping related resources. The "process group" of the cluster-as-process model. | Pillar 7 |
| **AlertLayer** | The mandatory interruption alert layer on every spot workload. `source + forwarder + sinks`. `required: true` default. Renderers refuse to emit without it. | THEORY.md §VII.3 |
| **Breathability** | A property of compute units: the capacity to scale to zero and wake on demand. Default for every compute unit. | Pillar 11 |
| **Spot** | The default auction substrate for compute units. EKS managed control planes are the only structural exemption. | Pillar 11 |
| **JIT fleet pattern** | Scale-to-zero ASG woken via SSH ProxyCommand (`cordel builder-wake`), scaled down by client watchdog + server CloudWatch alarm. Generalizes to any on-demand cloud capacity. | `skills/pangea-jit-builders/` |
| **cordel** | The Rust builder-wake orchestrator. Named for Brazilian cordel folk poetry (linear sequence of typed units). | `cordel/` |
| **kindling** | Nix flake management CLI. K3s cluster lifecycle wrapper via `kikai`. | `kindling/` |
| **kikai** | K3s cluster lifecycle orchestrator — init, up, down, status, destroy, daemon. | `kikai/` |
| **kenshi** | GitOps ephemeral testing operator for K8s. Test gate in the eight-phase loop (phase 3: PROVE). | `kenshi/` |
| **shinka** | DB migration operator for K8s. Runs `sqlx` migrations via `Migration` CRD + FluxCD hooks. | `shinka/` |
| **pangea-operator** | Drift detection for PID 1's inception layer. Runs outside the process tree. | THEORY.md §IV.4 |
| **MCP (Model Context Protocol)** | The ultimate operator surface. LLM agents manage the platform through typed MCP tools. No `kubectl`, no SSH. | THEORY.md §VII.5 |
| **tend** | Workspace repo manager + version watch daemon. `tend sync`, `tend watch`, `tend daemon`. | `tend/` |
| **forge** | The CI/CD build platform. Nix pipeline → Attic cache → GHCR → deployments. | `forge/` |
| **Attic** | Shared Nix cache. The "shared convergence memory" in the Nix/sui duality. | External |
| **Live convergence mutation** | Merging new elements into a converging system without pausing. Gated by schema + compliance + test + integrity. | THEORY.md §VII.6 |

---

## Naming

| Term | Definition |
|---|---|
| **Japanese idiomatic names** | Applied to foundational crates and long-standing primitives. Connotation: precision, craft, discipline. Examples: tatara, shikumi, sekkei, takumi, forge, kenshi, shinka, sekiban, tameshi, kensa, sui, blackmatter, kikai, kindling, forja (shared Japanese–Portuguese: forge), kontena. |
| **Brazilian-Portuguese names** | Applied to new Tier 2+ primitives evoking enclosed spaces, flows, growth, craft. Examples: terreiro, forja, cordel, cerrado, jabuti, samba, roça, bordado, caderneta, apurador. |
| **Naming orthogonality** | Japanese names the discipline layer; Portuguese names the ritual/cultivation/rhythmic layer. The two do not clash. |
| **terreiro** | Arena / enclosed Lisp VM. From Candomblé's sacred compound. Bounded memory region with deterministic semantics. |
| **forja** | Compiler factory / furnace. Pairs with `tatara`. Produces specialized compilers. |
| **cordel** | Bytecode stream. From folk-poetry string-verse form. Linear sequence of typed units. |
| **cerrado** | Typed substrate ecosystem. Brazilian savanna. Deep-rooted, slow-growing, regeneratively persistent. |
| **jabuti** | Slow-persistent bootstrap. Tortoise. Builds slowly, keeps its form. |
| **samba** | Rhythmic composition. Layered rhythmic structure. Module composition. |
| **roça** | Content-addressable field. Cleared agricultural plot. Things grown and indexed by where. |
| **bordado** | Overlay / layered pattern. Embroidery. Stitch on top without disturbing the base. |
| **caderneta** | Registry / notebook. Small book of records. Operator's working memory. |
| **apurador** | Validator / inspector. Examiner. Runs checks. |

---

## Term drift resolutions

These terms have historically been used with multiple meanings across
pleme-io documentation. This section is the canonical resolution; older
uses in specific CLAUDE.mds must be updated to match.

### "Convergence"

**Before:** used for both the static lattice and the dynamic process.

**After:**
- **convergence-as-lattice** — static structure of possible states.
- **convergence-as-process** — dynamic motion across it. _This is the
  default meaning of unqualified "convergence" in pleme-io docs._

See [THEORY.md §IV.1](./THEORY.md#iv1-convergence-lattice-and-process-the-term-drift-fixed).

### "Proof"

**Before:** used for type-level checks, proptest results, cryptographic
seals, state-machine checkpoints — all called "proofs."

**After:**
- **verification** — mathematical proof (type checks, proptest, rspec,
  InSpec). Pass/fail. Unconditioned by time.
- **attestation** — cryptographic seal (BLAKE3 Merkle, Ed25519
  signature). Evidence. Time-stamped. Replayable.

Both required. Neither substitutes for the other.

See [THEORY.md §V.2](./THEORY.md#v2-verification-versus-attestation-the-term-drift-fixed).

### "Dimension"

**Before:** typescape had 8 dimensions; convergence-computing had 6
classification dimensions; iac-testing had 4 layers / 8 artifact types —
all called "dimensions."

**After:**
- **Typescape dimension** — one of the 8 orthogonal dimensions of the
  typescape (vocabulary, AST domains, morphisms, fixed points,
  compliance, workspace DAG, render, manifests).
- **Classification dimension** — one of the 6 axes along which a
  convergence point is classified (Horizon, Structure, Substrate,
  Coordination, Trust, Intelligence). Used only in
  `convergence-computing` skill context.
- **Loop phase** — one of the 8 steps of the eight-phase convergence
  loop (DECLARE → … → RECONVERGE). Never called "dimension" henceforth.
- **Testing layer** — one of the 4 layers of the iac-testing pyramid
  (kensa terraform resource, rspec synthesis, InSpec, iac-test-runner
  full-cycle). Never called "dimension" henceforth.

### "Control"

**Before:** used for AWS IAM resources, compliance controls, and
fixed-point controllers — all called "controls."

**After:**
- **IAM control** — an AWS IAM resource. Qualify with "IAM" or name the
  specific resource.
- **Compliance control** — a named compliance primitive (e.g., NIST
  SC-7). Qualify with the framework when ambiguous.
- **Controller** — a fixed-point operator (FluxCD, Terraform, kenshi,
  etc.). Never shortened to "control."

### "Rendering"

**Before:** used for morphism application and for the typescape's render
state — both called "rendering."

**After:**
- **Rendering (verb)** — applying a morphism to produce an artifact.
- **Render state (noun)** — the typescape dimension containing the
  content-addressable index of rendered artifacts.

The noun form always qualifies as "render state" to avoid collision.

### "Knowable platform" vs "Compliant computing" vs "Proven computing"

**Before:** these three phrases were used interchangeably across skills.

**After:** **knowable platform** is the canonical term. The other two
are deprecated; uses elsewhere should be updated to "knowable platform"
(the name) or "proof by construction" (the mechanism).

### "Process"

**Before:** used for OS processes, convergence processes, and agent
sessions — all called "processes."

**After:** "process" without qualification is context-dependent. When
the level matters, qualify:
- **OS process** — kernel-level `pid_t`.
- **Convergence process** — `ConvergenceProcess` CRD (cluster-level).
- **Agent session** — MCP agent lifecycle.

All three share Unix process semantics (fork, exec, wait, kill). That
shared vocabulary is the whole point of THEORY.md §VIII.4.

---

## Adding a new term

1. Decide which section it belongs in.
2. Write a one-line definition. If you need more than one line, the
   term is probably composite — split it.
3. Cite a canonical source. If there isn't one yet, write the primary
   source first, then add the row.
4. If the term conflicts with an existing one, either merge rows or add
   a "term drift resolution" entry and update callers.
5. Never add a term that's only used in one document. Glossary terms
   earn their keep by being used in ≥2 places.
