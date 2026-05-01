# Terreno — typed terrain for caixa workloads

> Status: design / M0. Companion track to
> [`PANGEA-WORKSPACE-RECONCILIATION.md`](./PANGEA-WORKSPACE-RECONCILIATION.md)
> — both register `ArchitectureGem` CRs, both produce IaC plans, both
> reconciled by `pangea-operator`. They are **parallel tracks**, not
> migration paths.

## Frame

caixa is the typed package + lifecycle primitive — the box. caixa
workloads run inside Kubernetes, podman, Lambda, browser, or any
typescape backend the renderer supports.

**Terreno is the typed terrain those boxes rest on.** Cloud accounts,
networks, secrets, tunnels, DNS records, IAM roles, stateful storage —
the substrate that exists *outside* the box but is required to host
it.

Where Pangea Ruby is the canonical DSL today (Pillar 5 of pleme-io),
terreno is the canonical *caixa-ecosystem-native* primitive: typed in
Rust via `#[derive(TataraDomain)]`, authored in tatara-lisp, rendered
to Terraform JSON / Crossplane CRs / Pulumi schema / NixOS modules
through one lattice of backends.

## Why parallel to Pangea, not replacement

Pangea Ruby is mature: `pangea-aws` has 448 typed resources;
`pangea-cloudflare` has 202; the entire fleet's infra goes through
it; rio's saguão Phase 1 reconciles via it.

Replacing Pangea would be (a) multi-month, (b) loses every existing
typed primitive's hand-tuned coverage, and (c) violates
[`THEORY.md` § "Solve problems once, in one place"](./THEORY.md) —
the substrate is strictly stronger when terreno + Pangea coexist than
when one displaces the other.

The two tracks compose at the operator boundary: every
`ArchitectureGem` CR declares `spec.source.kind: Ruby | Lisp | Wasm`,
the operator dispatches to the matching backend, and the same
reconciler state machine drives both to settled state. Pangea Ruby
keeps its ecosystem; terreno grows the substrate-native authoring
surface.

## Vocabulary

| Term | Meaning |
|---|---|
| **terreno** | Typed cloud-resource declaration. The author's surface. `(defterreno cloudflare-tunnel …)`. |
| **terreno-arquetipo** | Typed schema for a *kind* of terreno (analogous to `Pangea::Architectures::CloudflareTunnel`). |
| **terreno-renderer** | Backend that consumes a terreno IR and emits an IaC artifact (Terraform JSON, Crossplane CR, Pulumi schema, etc.). |
| **terreno-gem** | A git-published collection of terreno-arquetipos, packaged as a tatara-lisp library + a typed Rust crate. |
| **lavra** | A *workspace* — a tree of terreno declarations + variables + secret refs that get reconciled together (analogous to a Pangea workspace). Brazilian-Portuguese for "smallholding" / "homestead." |

## Architecture — four layers

### Layer 0 — Typed Rust primitives

Every terreno-arquetipo is a `#[derive(TataraDomain)]` struct in a
typed crate (canonical pleme-io Rust+Lisp pattern, see
[`THEORY.md` § II](./THEORY.md)).

```rust
#[derive(TataraDomain, Serialize, Deserialize, Debug, Clone)]
#[tatara(keyword = "defterreno")]
pub struct CloudflareTunnel {
    pub name: String,
    pub zone: String,
    pub ingress: Vec<TunnelIngress>,
    pub policy: Option<TunnelPolicy>,
}

pub fn register() {
    tatara_lisp::domain::register::<CloudflareTunnel>();
}
```

Six lines of ceremony per terreno-arquetipo, identical to every other
TataraDomain in the fleet.

### Layer 1 — Authoring via tatara-lisp

The operator consumes a `lavra` — a directory of `.lisp` files +
variables.yaml + secrets.yaml.

```clojure
(defterreno cloudflare-tunnel
  :name "rio"
  :zone "quero.cloud"
  :ingress [{:hostname "auth.quero.cloud"
             :service  "http://lareira-authentik-server.authentik.svc.cluster.local:80"}
            {:hostname "cracha.quero.cloud"
             :service  "http://cracha-lareira-cracha.cracha.svc.cluster.local:80"}]
  :policy {:destroy-protection true
           :drift-reaction     :require-approval})
```

Macro expansion is typed: invalid `:ingress` shapes fail at Rust
compile time (when the consuming repo regenerates its
`#[derive(TataraDomain)]` IR), not at runtime.

### Layer 2 — Renderers

A `Renderer` is a typed function `(IR) → ConcreteIaC`. The IR is the
canonical typescape form (typed Rust struct → BLAKE3-stable JSON).
Renderers ship as substrate primitives, one per backend:

| Renderer | Output | Status |
|---|---|---|
| `terreno-render-tofu` | Terraform JSON (current `tofu apply` workflow) | M2 — first concrete deliverable |
| `terreno-render-crossplane` | Crossplane Composition + XR CRDs | M3 — for clusters that prefer Crossplane to tofu |
| `terreno-render-pulumi` | Pulumi schema (Go/TS/Python) | M4 — for orgs that already standardize on Pulumi |
| `terreno-render-nixos` | NixOS module (e.g. `services.cloudflared.tunnels.<name>`) | M4 — for self-hosted bare-metal terrenos |

Multiple renderers per terreno is the typescape promise: write the
declaration once; render to whichever cluster substrate the workload
lands on.

### Layer 3 — Operator integration

`pangea-operator` (the existing reconciler) gains source-kind
dispatch on `ArchitectureGem` CRs:

```yaml
apiVersion: pangea.pleme.io/v1alpha1
kind: ArchitectureGem
metadata:
  name: terreno-cloudflare
spec:
  gemName: terreno-cloudflare
  source:
    kind: Lisp                          # NEW — was implicit Ruby
    gitRepository:
      url: https://github.com/pleme-io/terreno-cloudflare
      ref: main
      path: lib                         # tatara-lisp source dir
  # ...
```

Operator's `CompilerBackend::prepare_gem` branches on `source.kind`:

  - `Ruby`  → existing behavior: clone + prepend `lib/` to `$LOAD_PATH`
              for the embedded CRuby evaluator.
  - `Lisp`  → clone + load every `.lisp` file under `path/` into the
              embedded tatara-lisp evaluator's symbol table.
  - `Wasm`  → fetch the WIT-typed wasm component, register with
              wasmtime; instantiate per-template.

`InfrastructureTemplate` reconciles unchanged at the controller
level: the dispatch happens inside the backend impl. State machine
is identical: Discover → Verify → Plan → Apply → Drift-watch.

## Relationship to caixa

caixa workloads (`:kind Servico`) declare their **expected terreno**
as part of their typed slot (analogous to `:limits` and `:behavior`):

```clojure
(defcaixa :kind Servico
  :nome saguao-bff
  ;; existing slots …
  :terreno {:cloudflare-tunnel "rio"           ;; references a (defterreno cloudflare-tunnel :name "rio")
            :akeyless-target   "drz/saguao"})
```

`feira deploy` (the caixa author-to-cluster verb) refuses to deploy a
Servico whose declared `:terreno` references a terreno that hasn't
been reconciled to `Settled`. Same shape as today's
`ArchitectureGem` smoke gate, but at the workload-deployment layer.

This makes the terrenos a typed dependency edge from compute to
infrastructure — declared in Lisp, enforced by the operator,
auditable in `kubectl get architecturegem` + `kubectl get
infrastructuretemplate`.

## Milestones

### M0 (this iteration) — design + name

- ✅ Name decided: **terreno**.
- ✅ This document captures the four-layer architecture + Pangea
  parallel-track positioning.
- ✅ Cross-references in [`PANGEA-WORKSPACE-RECONCILIATION.md`](./PANGEA-WORKSPACE-RECONCILIATION.md)
  + org-level [`pleme-io/CLAUDE.md`](https://github.com/pleme-io/blackmatter-pleme/blob/main/docs/pleme-io-CLAUDE.md).

### M1 — typed primitive scaffold

- Create `pleme-io/terreno-core` (typed Rust, tatara-lisp authoring,
  WASM/WASI build target — canonical pleme-io trio).
- First terreno-arquetipo: `CloudflareTunnel` (mirrors
  `Pangea::Architectures::CloudflareTunnel` exactly so the
  parallel-track invariant is testable).
- `terreno-render-tofu` renderer crate that takes a typed
  `CloudflareTunnel` value and emits the same Terraform JSON shape
  Pangea Ruby produces today. Round-trip RSpec: a terreno IR rendered
  by both renderers (Pangea Ruby + terreno) must produce
  byte-equivalent (or semantically-equivalent) Terraform JSON for a
  test fixture.

### M2 — operator dispatch

- Extend `ArchitectureGem` CRD: `spec.source.kind: Ruby | Lisp | Wasm`
  with default `Ruby` for backward compat. Schema migration is
  pure-additive — existing CRs continue to validate.
- `CompilerBackend::prepare_gem` dispatches on kind. `Ruby` path
  unchanged (M8.4.2). `Lisp` path new: clone the gem, register
  `.lisp` files with the embedded tatara-lisp evaluator.
- New `terreno-eval` crate (analogous to `pangea-ruby-eval`)
  embeds tatara-lisp inside the operator. Smoke fixtures verify the
  eval path: a `CloudflareTunnel.build` invocation produces the
  expected typed IR.

### M3 — first terreno-gem migration

- Migrate `pangea-cloudflare` to a parallel `terreno-cloudflare` gem
  (~5 typed primitives). Both gems remain published; clusters opt in
  per `ArchitectureGem` CR. Saguão Phase 1 stays on Pangea Ruby
  through M3 — flipping rio is M4.

### M4 — multi-renderer + cluster flip

- `terreno-render-crossplane` lands. Same fixture round-trips against
  both tofu and Crossplane outputs.
- rio's `architecture-gems.yaml` flips one terreno-arquetipo from
  `kind: Ruby` (pangea-architectures) to `kind: Lisp` (terreno-cloudflare).
  Saguão DNS reconciles end-to-end through the Lisp path.

### M5 — caixa-Servico `:terreno` slot enforcement

- `caixa-core` adds the `:terreno` typed slot.
- `feira deploy` queries the operator's reconcile state for every
  declared terreno; refuses deployments whose terrenos are not
  Settled.
- First consumer: `pleme-io/programs/hello-rio` declares `:terreno
  {:cloudflare-tunnel "rio"}`. Deployment blocked unless rio's
  cloudflare-tunnel terreno is Settled. End-to-end typed dependency
  edge from compute to infra, enforced at deploy time.

## Naming rationale

caixa is the box. terreno is what the box rests on. Brazilian-
Portuguese, caixa-family. Compact: `(defterreno …)` reads cleanly.

Also-rans considered + rejected:

| Candidate | Why not |
|---|---|
| `alicerce` | Already taken by an existing pleme-io thing. |
| `canteiro` ("construction site") | Verb-shaped — implies action, not state. terreno is a noun. |
| `estaleiro` ("shipyard") | Strong flavor but too tied to "ships," misleading. |
| `arcabouço` ("framework") | Cedilla + 4 syllables; ergonomics suffer. |
| `chao` ("ground/floor") | Most compact but loses the typed-substrate connotation. |
| `fundacao` ("foundation") | Direct, but more architectural than terrain. |

## Anti-patterns terreno explicitly rejects

1. **Frame as Pangea replacement.** Pangea Ruby is canonical
   (Pillar 5). terreno is the caixa-ecosystem-native sibling. Both
   register `ArchitectureGem` CRs; both reconcile through the same
   operator; both produce typed IaC. Neither displaces the other.

2. **One renderer per gem.** Renderers are substrate primitives,
   shared across gems. Adding Crossplane output for one gem adds it
   for all (subject to typespace coverage).

3. **Hand-write the Lisp form.** `(defterreno …)` is a macro that
   expands to a typed Rust IR. Authors write the form; the IR is
   re-derived by `cargo build`. Hand-rolled IRs that bypass the
   macro are rejected by `cse-lint --strict`.

4. **Treat tatara-lisp as a "scripting language."** Lisp here is the
   typed authoring surface for Rust IRs (the canonical Rust+Lisp
   pattern, [`THEORY.md` § II](./THEORY.md)). It is not a runtime
   scripting interface.

## Cross-references

- [`THEORY.md`](./THEORY.md) — frame for typed substrate engineering;
  Four Lisps; typescape rendering.
- [`PANGEA-WORKSPACE-RECONCILIATION.md`](./PANGEA-WORKSPACE-RECONCILIATION.md)
  — Track A. Pangea Ruby + magnus + the eight-substep hollow-out.
  Read first; terreno parallels the operator-side machinery exactly.
- [`CAIXA-SDLC.md`](./CAIXA-SDLC.md) — caixa primitive that consumes
  terreno via the M5 `:terreno` slot.
- [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) — four-layer compute
  hierarchy that terreno's L0-L3 layering instantiates.
- [`RUST-LISP-EMBEDDING.md`](./RUST-LISP-EMBEDDING.md) — the
  canonical Rust+Lisp pattern terreno's Layer 0 + Layer 1 use.

## Maintained by

This file. Update on every milestone landing. Implementation lives
across `pleme-io/terreno-*` repos (M1+); operator-side dispatch
lives in `pangea-operator/src/ruby/` extension (M2+); CRD extension
lives in `pangea-operator/src/crd/architecture_gem.rs` (M2+).
