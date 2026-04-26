# Extensibility — boundless mechanical extension of existing patterns

> **The user, 2026-04-26**: *"the architecture is boundlessly
> extensible by mechanical extension of existing patterns"* +
> *"basically, we send things to GitHub and we run from GitHub and
> that's it."*
>
> This document locks that principle in. It is the **rule against
> bespoke architecture**. Every future capability is added by
> following an existing pattern; new patterns are added only when
> the existing pattern provably can't carry the new shape, and even
> then they are added *as patterns* — codified, generalized,
> reusable.

---

## I. The principle

```
              ┌──────────────────────────────────────────────────┐
              │  PUSH TO GITHUB                                  │
              │      git tag v0.x.y && git push --tags           │
              └────────────────────────┬─────────────────────────┘
                                       │
                                       ▼
              ┌──────────────────────────────────────────────────┐
              │  REFERENCE BY URL                                │
              │      github:owner/repo/path?ref=v0.x.y          │
              └────────────────────────┬─────────────────────────┘
                                       │
                                       ▼
              ┌──────────────────────────────────────────────────┐
              │  RUN FROM ANYWHERE                               │
              │      tatara-script <url>                          │
              │  OR  ComputeUnit.spec.module.source = <url>      │
              │  OR  use-domain (in another tlisp)               │
              │  OR  nix run github:owner/repo                   │
              └──────────────────────────────────────────────────┘
```

That's the whole loop. Every pleme-io capability funnels through it:

- **A new program**: write `.tlisp`, push to `pleme-io/programs/<name>/`,
  reference as `github:pleme-io/programs/<name>/main.tlisp?ref=v0.1.0`.
- **A new controller**: same, plus `(use-library github:pleme-io/tatara-lisp-controllers/lib/...)`.
- **A new API client**: spec → codegen → push to `pleme-io/tatara-codegen/domains/<provider>/`,
  reference as `(use-domain github:pleme-io/tatara-codegen/domains/<provider>/api.tlisp?ref=v1)`.
- **A new shared library**: push to a new pleme-io repo,
  `(use-library github:pleme-io/<name>/lib/...)`.
- **A new image**: `flake.nix` + substrate's `tool-image-flake.nix`,
  built by `nix build github:pleme-io/<name>#image`.
- **A new helm chart**: push to `helmworks/charts/<name>/`,
  consumed via `oci://ghcr.io/pleme-io/charts/<name>:<version>`.

**One mechanism**, applied at every layer.

## II. The five existing patterns and what each carries

The architecture is closed under these five patterns. Anything new
is mechanically a member of one of them:

### II.1 Tatara-lisp program (the fundamental unit)

```
pleme-io/programs/<name>/
├── main.tlisp        ≤ 200 lines, stdlib + (use-domain ...)
├── computeunit.yaml  reference CR (kubectl-friendly)
├── flake.nix         delegates to substrate/lib/build/tatara/program-flake.nix
└── README.md
```

Add a program → `git tag` → reference by URL. Cookbook:
[`WASM-PATTERNS.md`](WASM-PATTERNS.md). Authoring tier:
[`LISP-YAML-CONTROLLERS.md` §V](LISP-YAML-CONTROLLERS.md).

### II.2 Helm chart (the deployment unit)

```
helmworks/charts/lareira-<name>/
├── Chart.yaml        depends on pleme-computeunit (or pleme-microservice for non-WASM)
├── values.yaml       cluster-wide defaults (~30 lines)
└── README.md
```

Add a chart → `nix run .#helm:release` → consumed via
`oci://ghcr.io/pleme-io/charts/lareira-<name>:<version>`. Library
chart catalog: [`META-FRAMEWORK.md` §IV](META-FRAMEWORK.md).

### II.3 Codegen domain (the API/IaC/CRD/etc. unit)

```
pleme-io/tatara-codegen/domains/<provider>/<service>/
├── service.tlisp     auto-generated (defdomain ...)
├── spec-version      git-tag of the source spec
└── README.md
```

Add an external integration → drop a spec file in any pleme-io repo
+ run `tatara-codegen auto-discover` → domain lands in
`tatara-codegen/domains/`. Catalog:
[`TATARA-CODEGEN-MATRIX.md`](TATARA-CODEGEN-MATRIX.md).

### II.4 Container image (the always-running-Rust unit)

```
pleme-io/<service>/
├── flake.nix         delegates to substrate/lib/build/rust/tool-image-flake.nix
├── Cargo.toml
└── src/
```

Add a Rust service → `nix run .#release` → image at
`ghcr.io/pleme-io/<service>:<version>`. Pattern:
[`substrate/lib/build/rust/tool-image-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/build/rust/tool-image-flake.nix).

### II.5 Cluster declaration (the operator unit)

```
pleme-io/k8s/clusters/<name>/programs/release.yaml   one cluster's program inventory
pleme-io/k8s/clusters/<name>/infrastructure/         lareira-* HelmReleases
```

Add a cluster → 4 directories of YAML + a flux-system bootstrap →
self-reconciles. Pattern: [`FLEET-DECLARATION.md`](FLEET-DECLARATION.md).

## III. The N-times-mechanical extensions

Each pattern has a generalization rule. Following the rule is
mechanical work — no design judgment needed:

| Want N more of... | Do | Effort |
|---|---|---|
| Programs | one new dir per program in `pleme-io/programs/` | ~30 min per program |
| Helm charts | one new dir per chart in `helmworks/charts/` | ~30 min per chart |
| Codegen domains | one new spec source dropped in a repo | ~5 min per domain |
| Rust services | one new repo from `repo-forge new --archetype rust-service-tool` | ~10 min per service |
| Stdlib primitives | one new `<domain>.rs` in `tatara-lisp-script/src/stdlib/` | ~30 min per primitive |
| Clusters | one new `clusters/<name>/` in `pleme-io/k8s/` | ~1 hour per cluster |
| Workspace flakes | one new `pangea-architectures/workspaces/<name>/` | ~30 min per workspace |
| Programs running on a cluster | one new entry in `clusters/<name>/programs/release.yaml` | ~5 min per addition |

The work *scales linearly* in count and is bounded in time. There
is no scaling penalty — adding the 100th program takes the same
~30 minutes as adding the 1st.

## IV. The closure rule

**Every capability the fleet needs maps to one of the five patterns.**
This is verifiable: walk through what real services need, match each
against §II.

| Need | Pattern |
|---|---|
| HTTP service that returns JSON | II.1 program (service shape) |
| Cron job that resizes PVCs | II.1 program (job shape) |
| Reconciler watching CRs | II.1 program (controller shape) |
| Static frontend served from CDN | II.4 (Rust) — until WASI-component fully serves the frontend |
| Database with schema migrations | II.4 (Rust runtime) + II.3 (SQL DDL → defmodel domain) + II.1 (migration runner program) |
| Monitoring dashboard | II.2 helm chart with pangea-grafana auto-render |
| Secrets rotation | II.1 program (cron / event shape) calling external API via II.3 domain |
| Edge auth | II.5 cluster declaration including authentik HelmRelease |
| Multi-cluster sync | II.1 program (controller shape) using II.3 K8s domain |
| AI-driven controller | II.1 program (controller shape) using II.3 anthropic domain |
| GraphQL API serving | II.1 program (service shape) with II.3 GraphQL → server domain |
| gRPC service | II.4 Rust service (until tatara-lisp gains tonic bindings) |
| K8s controller (formerly kube-rs Rust) | II.1 program (controller shape) using II.3 K8s domain + tatara-lisp-controllers macros |
| **anything else** | → **map to one of the five before adding architecture** |

If a capability genuinely doesn't fit, that's a signal to **extend the
existing pattern, not add a sixth.** Examples:

- "I need HTTP server with dynamic handlers" → not a new pattern;
  extend `http-serve-static` to `http-serve` (closure-aware).
- "I need a CRD reconciler in Lisp" → not a new pattern; add macros
  to `tatara-lisp-controllers/lib/`.
- "I need protobuf RPCs" → not a new pattern; add `proto.tlisp` parser
  to `tatara-codegen/lib/`.

## V. The maintenance budget

The architecture's maintenance scales with **number of patterns**, not
with number of consumers of each pattern.

```
maintenance load ≈ Σ (per-pattern complexity)  — bounded constant
                NOT
                ≈ Σ (per-program complexity)   — would be unbounded
```

Today's pattern count: **5**. Each pattern's complexity is
documented in 1-2 theory documents. The fleet has dozens of
programs, hundreds of charts, tens of clusters — and the
maintenance load is what those 5 patterns ask of one engineer.

The constraint that prevents architecture sprawl: **a sixth pattern
needs an explicit document explaining why an existing pattern can't
carry it.** No sixth pattern has been needed; all needed capabilities
have been mechanical extensions of the five.

## VI. Anti-patterns this rule prevents

By committing to the five patterns above, we exclude the
shape-explosions that bloat most multi-language projects:

| Anti-pattern | Why we don't have it |
|---|---|
| Per-service config DSL | tatara-lisp + (defschema ...) is the universal schema |
| Per-API client library | tatara-codegen consumes any spec into a defdomain |
| Per-cluster bespoke YAML | lareira-fleet-programs + lareira-* charts are universal |
| Per-program Dockerfile | substrate's tool-image-flake handles every Rust image; WASM doesn't need one |
| Per-program packaging tool | git+nix+content-hash *is* the package format |
| Per-program CI pipeline | nix run .#release is universal |
| Multiple in-house ORMs | one defmodel emitter from SQL DDL |
| Multiple in-house IaC DSLs | pangea-* is the synthesis target; tatara-codegen emits its inputs |
| Multiple workflow engines | saga reconciler from WASM-PATTERNS #31 covers it |
| Multiple secret-handling conventions | sops + akeyless-nix + blackmatter-secrets per [the existing rule] |

When tempted to add a domain-specific abstraction, **first try to
mechanically extend a pattern**. The constraint is the architecture's
backbone.

## VII. The proof from this session

This session demonstrated the principle by accumulation:

```
Day 1 capabilities      → 5 patterns
Day N capabilities      → still 5 patterns
                              hello-world program       (instance of II.1)
                              lareira-hello-world       (instance of II.2)
                              http-server stdlib        (mechanical extension of II.1's stdlib)
                              kube + uuid stdlib        (mechanical extension)
                              github URL fetcher        (mechanical extension)
                              tlisp2nix builders        (mechanical extension of substrate)
                              fleet-programs chart       (instance of II.2)
                              meta-framework theory     (formalization, no new pattern)
                              hello-world live + curl   (proof, no new pattern)
                              codegen-matrix theory     (formalization of II.3)
```

Every commit added an instance of an existing pattern OR extended
one in a documented way. **No new pattern was needed.** The proof
that the principle holds is that we built a working, deployable,
content-addressed, breathable, capability-bounded service runtime
without inventing one new architectural concept — only by
composing the five.

## VIII. The mandate

Anyone (human or AI agent) adding to pleme-io:

1. **Identify which of the five patterns** the new capability fits.
2. **Extend that pattern mechanically** — no bespoke architecture.
3. **If the pattern truly can't carry the capability**, write a
   theory document explaining why. The bar is high; it has not
   yet been cleared.
4. **Push to GitHub** under the right path. URL resolution is the
   distribution mechanism; that's the entire publishing story.
5. **Reference by URL** from consumers. Content-pinning via
   `?ref=<tag>` is the version contract.
6. **Run from GitHub.** Nothing else is the integration story.

## IX. See also

Every other theory doc in this directory is a specialization of this
rule. Read them in this order to see the principle compound:

1. [`THEORY.md`](THEORY.md) — the 12 pillars (the architecture's axioms)
2. [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — the 4 compute layers
3. [`TATARA-PACKAGING.md`](TATARA-PACKAGING.md) — git+nix native
4. [`WASM-STACK.md`](WASM-STACK.md) — runtime
5. [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49 patterns (each an instance of II.1)
6. [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) — controller authoring
7. [`TATARA-CODEGEN.md`](TATARA-CODEGEN.md) + [`TATARA-CODEGEN-MATRIX.md`](TATARA-CODEGEN-MATRIX.md) — codegen surface
8. [`FLEET-DECLARATION.md`](FLEET-DECLARATION.md) — cluster-as-package-list
9. [`BREATHABILITY.md`](BREATHABILITY.md) — fleet breathability
10. [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
11. [`HELLO-WORLD-LIVE.md`](HELLO-WORLD-LIVE.md) — the canonical proof
12. **`EXTENSIBILITY.md` (this doc)** — the closing-the-loop principle

This document is the **last one that needs to exist** in this
generation of pleme-io theory. Future architecture work either
extends an existing pattern (most cases) or, rarely, demonstrates
why a sixth pattern is necessary — and then becomes the 13th theory
document.
