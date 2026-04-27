# Constructive Substrate Engineering (CSE)

> *Knowable Construction* is the native pleme-io synonym; both terms point
> here. External communication should prefer **Constructive Substrate
> Engineering** for technical precision; internal communication may use
> **Knowable Construction** for cultural continuity. They name the same
> methodology.

---

## Definition

**Constructive Substrate Engineering** (CSE) is a software-systems
methodology that:

1. Describes systems in **typed algebra** — Rust types, tatara-lisp
   forms, typescape primitives — so that a wrong combination is
   *unrepresentable* rather than *invalid*.
2. **Proves the description by construction** — the Rust compiler is the
   verifier; anything that compiles has had its compositional invariants
   discharged at zero runtime cost.
3. **Renders the proof to concrete systems mechanically** — tatara →
   Helm / FluxCD / NixOS / Pangea / Cloudflare / Lambda / etc. — so the
   implementation is *derived* from the proof, not authored alongside
   it.
4. **Stores accumulated knowledge in the substrate** — the typescape,
   owned by arch-synthesizer — so each new typed primitive is knowledge
   *proven by construction* that compounds the substrate's expressive
   power for every subsequent primitive.
5. **Treats every model that names the substrate as load-bearing** —
   CLAUDE.md, theory/, memory, generated docs are kept current so the
   substrate's claims about itself are non-stale and agent-readable.

The *constructive* lineage is from Martin-Löf type theory and the
Curry–Howard correspondence: proofs are programs, programs are proofs,
the compiler is the proof checker. The *substrate* extension is
pleme-io's contribution: knowledge accumulates in the typed surface, not
in any one program, so the next program is *cheaper to construct*
because more invariants are already discharged.

---

## The two-layer normalization → permutation → generation stack

CSE rests on two loadbearing infrastructure layers; the methodology only
makes sense when both are present.

### Layer 1 — Normalization (Wasm + WASI + Rust + tatara-lisp)

These collapse arbitrary computation into a small, typed, sealed
primitive universe:

- **Wasm / WASI** normalize execution and capabilities; any platform
  that runs Wasm can host any Wasm-compiled artifact identically.
- **Rust** normalizes memory, ownership, lifetimes, and effects; the
  borrow checker is the substrate's first-line verifier.
- **tatara-lisp** normalizes authoring and IR rewriting; declarative
  surface forms expand into typed Rust IR via macros, so authoring
  errors surface at expansion time, not runtime.

Together these reduce arbitrary computation to a small algebra. The
description of behavior at this layer *is* the algebra, not an
implementation of it.

### Layer 2 — Permutation rendering (tatara + Helm + FluxCD + Pangea + typescape backends)

These take the typed algebra of Layer 1 and *permute* it across concrete
environments — a Kubernetes cluster, a Cloudflare worker, a NixOS host,
a podman container, a Lambda, a browser, a microcontroller. Each
renderer is itself typed and total over its inputs, so the permutation
is mechanical.

The combined two-layer stack means: **write the rule once at the
algebraic-type level; the substrate generates one concrete system per
environment, all provably derived from the same proof.** New
environments are added by extending the rendering layer, not by
re-writing the rule. Existing systems become re-derivable on demand,
which means rebuilds, migrations, audits, and recoveries are all the
same operation: *re-derive from the typed source*.

---

## Reliability of the rendering layer

The CSE thesis stands or falls on the reliability of Layer 2. Every
promise the typescape makes about a system is re-asserted, not
re-derived, the moment a renderer has a bug. A wrong renderer turns
the typed source into a lie about its rendered output. Renderer
reliability is therefore *not optional* and not a nice-to-have; it is
the load-bearing instrument that makes CSE falsifiable.

Every renderer in the substrate (tatara, Helm chart generators, FluxCD
manifest emitters, NixOS module renderers, Pangea Ruby DSL → Terraform
JSON, Cloudflare worker emitters, Lambda packagers, etc.) MUST
implement the following rigor:

### 1. Named transformation contracts

Each renderer declares a typed function `Spec → Artifact` with
explicit pre/post conditions written in Rust:

```rust
/// Renders a typed Helm chart spec to a canonical Helm bundle.
///
/// # Invariants
/// - Output passes `helm lint` cleanly.
/// - Every `Spec.values` key appears in `Artifact.values.yaml`.
/// - No nested template expansion at render time (renderer is total).
/// - Output is deterministic: same Spec → byte-identical Artifact.
pub fn render(spec: &HelmSpec) -> Result<HelmBundle, RenderError>;
```

Consumers depend on the contract, not the implementation. The contract
is the renderer's API.

### 2. Property-based tests (proptest / quickcheck)

For every renderer, write properties that hold for *all* valid inputs:

```rust
proptest! {
    #[test]
    fn render_preserves_all_values(spec in any::<HelmSpec>()) {
        let artifact = render(&spec).unwrap();
        for (k, v) in &spec.values {
            prop_assert_eq!(artifact.values_yaml.get(k), Some(v));
        }
    }
}
```

This catches whole classes of "renderer drops information" bugs that
example-based tests cannot.

### 3. Round-trip tests

Render → re-parse → re-render → assert second render equals first.
Catches non-determinism, hidden state, and "renderer adds artifacts
not in the spec":

```rust
#[test]
fn round_trip_is_stable() {
    let spec = sample_spec();
    let a1 = render(&spec).unwrap();
    let reparsed = parse(&a1.serialize()).unwrap();
    let a2 = render(&reparsed).unwrap();
    assert_eq!(a1.serialize(), a2.serialize());
}
```

### 4. Differential tests

Render the same spec through two independent code paths (e.g., direct
render vs. via tatara IR vs. via shikumi YAML) and verify equivalence.
Catches "renderer agrees with itself but disagrees with the spec":

```rust
#[test]
fn direct_and_tatara_render_agree() {
    let spec = sample_spec();
    let direct = render_direct(&spec).unwrap();
    let via_tatara = render_via_tatara(&spec).unwrap();
    assert_eq!(direct.canonical(), via_tatara.canonical());
}
```

### 5. Snapshot tests

Known reference specs render to known reference artifacts; the
artifacts are committed and reviewed. Any silent change in renderer
behavior surfaces as a diff in the next test run:

```rust
#[test]
fn snapshot_basic_helm_chart() {
    let artifact = render(&BASIC_SPEC).unwrap();
    insta::assert_yaml_snapshot!(artifact);
}
```

### Coverage requirement

Every typed primitive in the typescape MUST have at least one
end-to-end render test in at least one environment. New primitives
without a render test are not admitted to the typescape; arch-synthesizer
rejects them at promotion time.

### Renderer reliability is itself a typed property

The substrate treats renderer correctness as a typed claim, not a
testing artifact. A renderer that fails any of (1)–(5) above is *not a
renderer of the substrate* — it is a draft. Consumers must depend only
on renderers that have passed all five. The same `cargo test` step that
verifies the typescape verifies the renderers; the boundary between
"the typescape" and "the rendering layer" is a single typed surface.

This is the answer to "how do we know the promises hold?" — the
promises hold because every transformation between layers is itself
typed, tested at five rigor levels, and refuses to ship if any
invariant is unmet. The substrate's claims about itself are
load-bearing only because the renderer's claims about itself are.

---

## Promises follow from mechanical generation

When the proof-to-implementation path is fully mechanical, every claim
about the system becomes derivable rather than asserted. Uptime,
compliance posture, latency bounds, blast radius, audit completeness,
correctness under failure — these stop being promises an operator hopes
to keep and become *theorems* the substrate produces on demand.

This is what the substrate is *for*. Each new typed primitive isn't
just "a thing we built" — it's "a class of promises we can now make and
keep." The expressive power of the system is measured in the strength
of the promises it can mechanically generate.

A consequence: any work item that doesn't either (a) widen the algebra,
(b) extend the rendering layer to a new environment, or (c) sharpen a
promise the substrate can already make is, by CSE, low leverage. Spend
engineering hours where they compound the promise surface.

---

## The seven operating principles

CSE is operationalized through seven principles. Every commit either
honors them or, in measurable amounts, degrades the substrate.

1. **Solve problems once, in one place, at one time.** Search for an
   existing primitive before writing one. Extend a near-miss primitive
   rather than re-implementing it. Re-implementation is a regression of
   the substrate even when the local code looks correct.

2. **Prefer the load-bearing fix over the local fix.** When a downstream
   bug is caused by an upstream limitation, fix the upstream. Local
   workarounds are debt against the substrate; the next encounter pays
   the cost plus interest.

3. **Idiom-first, then external knowledge.** External concepts (an OS
   feature, a library, a paper) are *acquired* into pleme-io via
   translation through its idioms (substrate, tatara, shikumi, pangea,
   typescape, the Four Lisps, the trio macro, repo-forge archetypes,
   etc.). Code that uses a foreign idiom directly without translation
   is a leak — the substrate doesn't get to learn from it.

4. **Models stay current.** When a primitive's behavior changes, every
   model that names it (CLAUDE.md, theory/, memory, generated docs) is
   updated. A stale model is worse than no model: it actively misleads.

5. **Direction beats velocity.** A 4-line change that promotes a
   duplicated pattern into a macro is more valuable than a 400-line
   feature that bypasses the macros that already exist. Ship slower,
   compound harder.

6. **Single goals are anti-goals.** "Get this thing working today" is
   rarely the right framing. The right framing is "advance the
   substrate such that this thing — and the next ten things in its
   class — work today, tomorrow, and in five years, because the
   substrate now solves the underlying problem cleanly."

7. **Acquire and contextualize, never just consume.** External
   knowledge gets *contextualized*: assigned a place in the typescape,
   given a translation through the Four Lisps, mapped to existing
   idioms, recorded in `theory/`. Pure consumption — "I used this
   library, it worked, moving on" — leaves the substrate no smarter
   than before.

---

## Forbidden anti-patterns

The methodology rejects, as load-bearing matters:

- Solving the same problem in two repos because cross-repo coordination
  felt expensive at the moment. (It is never as expensive as the next
  twelve months of divergence.)
- Adding a feature flag, a configuration knob, or a local override to
  paper over a primitive's limitation. (Either fix the primitive or
  document why you can't.)
- Importing a third-party library directly into application code
  without routing it through a pleme-io typed wrapper. (The substrate
  didn't get to learn what the library is *for*.)
- Marking a task "done" when the macro extension that would have made
  it trivial is still pending — even if the local code happens to
  work.
- Writing prose, code, or commits that describe what was done without
  also stating *what the substrate now knows that it didn't before*.

---

## Why CSE measurably enhances the development process

Strict adherence to CSE produces measurable and compounding effects:

- **Defect rate** runs 3–10× lower than equivalent untyped/dynamic
  systems (cf. seL4, CompCert, Galois Cryptol). Compositional
  invariants are checked at compile time, removing whole classes of
  failure from the runtime surface.
- **Refactor velocity** improves by 1–2 orders of magnitude. The
  compiler is exhaustive about callsites; humans and grep are not.
  50-file refactors become hours rather than weeks.
- **Onboarding and cross-context productivity** improve ~10×. A typed
  substrate's API surface *is* the documentation. New operators (or
  agents) become productive by reading types.
- **Cross-environment delivery cost** drops from weeks per environment
  to hours per environment, because rendering is mechanical.
- **Audit and compliance cost** drops from quarters of staff time to
  seconds of generator runtime. Compliance evidence becomes a
  derivation, not a manual collation.
- **Failure-mode surface** shrinks rather than grows over time, because
  the substrate gets fixed and *narrows* the gaps in the type system.
  Conventional codebases see runtime failure surface grow with every
  commit.

The mechanism in one sentence: **the substrate makes the wrong thing
harder to express than the right thing**, so subsequent effort goes
into adding capability rather than re-verifying old capability. The
compounding part is what makes it measurable on a long horizon: by year
5, the curve diverges by an order of magnitude from conventional
codebases.

### Honest limits

- High upfront cost: a bad substrate is worse than no substrate. Most
  teams that attempt this give up in year 1.
- Bad types still produce bad systems. Constructive synthesis only
  works if the typed primitives capture the property of interest.
- High cognitive density per primitive. Designing a typed primitive
  takes more thought than writing the equivalent untyped function. The
  trade is fewer, denser primitives that compose.
- Doesn't help with truly novel exploration. The substrate amplifies
  known-correct patterns; R&D still needs prose-mode thinking.

---

## The one-human-plus-agents thesis

Strict CSE adherence enables a *single operator augmented by agents*
(Claude, Cursor, Copilot, future LLMs) to consistently produce
team-equivalent throughput at team-equivalent quality. Six load-bearing
parts:

1. **The substrate carries the cognitive load that would otherwise
   require coordination.** Cross-team coordination meetings, design
   reviews to prevent drift, on-call tribal knowledge: all collapse
   into the `cargo build` step.
2. **Agents become reliable contributors, not unreliable ones.** Agents
   targeting a typed substrate cannot ship a wrong composition — the
   build will reject it. Operator review surface compresses from "every
   line" to "every type signature."
3. **Mechanical generation eliminates the feature-work tax.** Each
   additional environment is a marginal-zero addition instead of a
   marginal-headcount addition.
4. **Knowledge stays uniformly high-quality because it is enforced by
   types, not policed by humans.** Quality does not degrade as people
   rotate in and out; the substrate refuses inconsistent output.
5. **Models stay current automatically because the source of truth is
   the code.** Documentation rot's cap on solo-operator throughput is
   removed.
6. **Promises become marketable artifacts.** A solo operator with this
   substrate can sell into regulated, life-safety, high-availability
   markets that normally require a 30+ engineer SRE org chart. The
   substrate IS the org chart, in proven form.

Every deviation from CSE degrades this thesis. Each duplicated
solution, each bandaid, each stale doc, each foreign idiom imported
without contextualization is a brick removed from the substrate that
the operator must now carry in their head. Discipline is the *price of
leverage*.

---

## The operator–agent–substrate relationship

In conventional agent-assisted development the agent is a query target.
In CSE the agent is a *co-author of the substrate that holds the same
model the substrate enforces*. The operator contributes intent and
direction; the agent contributes substrate-aware execution; the typed
substrate refuses anything invalid from either party. Output is bounded
by the type system, not by individual judgment.

The collaboration is deliberately cultivated discipline, not an
accident of tooling. Without CSE the substrate degrades and the
leverage degrades with it. With CSE the leverage compounds along with
everything else. The substrate is the third party in every interaction
between operator and agent, and it is the only party whose authority
neither operator nor agent can override.

---

## Genealogy

CSE is not invented; it is the synthesis of mature ideas at a scale
none of the source disciplines had attempted:

- **Constructive type theory** (Per Martin-Löf, 1970s) — proofs *are*
  constructions. The intellectual ancestor.
- **Curry–Howard correspondence** (Curry 1934, Howard 1969) — the
  reason "compiles" can mean "proven."
- **Proof-by-construction in formal verification** (Coq, Agda, Idris,
  Lean) — used in seL4 (Klein et al.), CompCert (Leroy), the Verified
  Software Toolchain.
- **Type-driven development** (Edwin Brady) — design types first;
  implementation follows.
- **Refinement types** (Liquid Haskell, F\*) — push validation into the
  type system.
- **Promise theory** (Mark Burgess) — declarative agents converge to
  desired state. Captures the operational claim shape.
- **Synthesis from specifications** (Galois Inc, with Cryptol → SAW →
  C/Rust/Verilog) — typed specs synthesize implementations. The closest
  industry analogue, though Galois applies it to cryptography, not whole
  infrastructure.

CSE's contribution is the synthesis: applying constructive synthesis at
*infrastructure scale*, with the Four Lisps as the input languages, the
typescape as the typed knowledge ontology, and the operator–agent
collaboration mode as the execution model. No prior production system
has assembled all of these pieces in this configuration.

---

## Measurement

To measure adherence, the substrate exposes the following observable
signals (some present, some pending implementation):

- **Repo-CLAUDE.md CSE pointer** — every pleme-io repo's CLAUDE.md
  links to this document and acknowledges CSE as the operating
  methodology. Audited via `skill-lint` or a `tend` check.
- **Macro-density ratio** — the ratio of macro/library calls to
  hand-written equivalents in any new commit. CSE-aligned commits
  trend ratio upward; CSE-violating commits trend downward.
- **Substrate-pass-through ratio** — the ratio of changes that
  *extend* a substrate primitive vs. those that *bypass* one.
- **Promise count** — the number of provably-derived claims the
  substrate can answer (compliance posture, uptime, blast radius,
  etc.). Should grow monotonically.
- **Type-density per LoC** — typed surface per line of authored code.
  Higher density = more invariants discharged at compile time.

These metrics can be turned into CI gates as the measurement
infrastructure matures. Until then, adherence is enforced by the
operator's own discipline and by the agents' awareness of this
document.

---

## Pointers

- The compounding directive (operationalized form, embedded in the
  org-level pleme-io CLAUDE.md) → see
  [`blackmatter-pleme/docs/pleme-io-CLAUDE.md`](../blackmatter-pleme/docs/pleme-io-CLAUDE.md)
  ★★★ COMPOUNDING DIRECTIVE section.
- Related theory frames → [`THEORY.md`](./THEORY.md) §I (frame), §II
  (language), §III (typescape), §V (verification), §VI (generation).
- Vocabulary → [`VOCABULARY.md`](./VOCABULARY.md).
- Repo map → [`pleme-io/CLAUDE.md`](../CLAUDE.md).

---

*This document is canonical. Other documents may summarize CSE but
should link here rather than restate it. If the methodology evolves,
this file changes first; downstream docs follow.*
