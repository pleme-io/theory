# Scripting in pleme-io — tatara-lisp as the standard

> **Frame.** [Pillar 1](THEORY.md#pillar-1-language) makes Rust + tatara-lisp + WASM/WASI the
> language stack. [Pillar 12](THEORY.md#pillar-12-generation-over-composition) makes
> generation the default and hand-writing the fallback. This document
> binds those two pillars together for the scripting layer: **`tatara-lisp`
> is the canonical scripting language for pleme-io. Bash beyond 3-line
> glue requires explicit dispensation.**

---

## I. The rule

> Every operational script in the pleme-io fleet — chart publishers,
> migration runners, fleet sweepers, image promotion, ad-hoc reconcilers
> — is authored in **tatara-lisp** and evaluated by the `tatara` runner.
> Bash is allowed only for ≤3-line glue inside Nix derivations
> (`writeShellApplication`, devshell hooks, kustomize post-render hooks)
> where the alternative would be a 1:1 transliteration. Anything longer
> is drift; convert to tatara-lisp.

This is **stricter** than [`~/.claude/CLAUDE.md`](https://github.com/pleme-io/blackmatter-pleme/blob/main/docs/pleme-io-CLAUDE.md)'s
"no bash beyond 3-line glue". That earlier rule said *what not to do*;
this document says *what to do instead*.

## II. Why tatara-lisp

Three forces converge:

1. **Existing typed domains compose for free.** `#[derive(TataraDomain)]`
   already makes every Rust crate's domain types directly authorable in
   tatara-lisp via 6 lines of ceremony. A script is composition of
   typed operations; tatara-lisp is the one place those compositions
   are typed *and* concise.

2. **Generation, not composition** ([Pillar 12](THEORY.md#pillar-12-generation-over-composition)).
   A bash script encodes implementation. A tatara-lisp expression
   encodes intent. The former is "how"; the latter is "what". The fleet
   wants what.

3. **Macro reuse.** Across charts, scripts repeat the same shapes —
   "render → push to OCI", "list resources → patch by predicate",
   "watch metric → emit alert". Each repetition in bash is a leak;
   each becomes a tatara-lisp macro the rest of the fleet imports.

## III. The 3-line glue dispensation

These contexts get a free pass for bash:

- `writeShellApplication` / `writeShellScriptBin` in a flake when the
  body is ≤3 lines of straight-line invocation (no branches, no loops,
  no string-mangling beyond a single substitution).
- `shellHook` in `mkShellNoCC` (devshell prompt setup).
- Kustomize post-render hooks (`set -e; kubectl apply -f -`).
- `direnv`'s `.envrc` (≤3 `use_*` lines).

Anything else — push scripts, migration runners, log-analysis
one-shots, image promotion pipelines — is a tatara-lisp file living
under `<repo>/scripts/<name>.tatara`, executed via `tatara run`.

## IV. Conversion priority

The fleet has a substantial backlog of scripted automation written in
bash. Convert in this priority order:

| # | Class | Examples | Why first |
|---|---|---|---|
| 1 | Multi-step infra publishers | helmworks `nix run .#helm:release`, image push wrappers, OCI manifest signing scripts | Run frequently; high blast radius if they fail; small enough to type-port cleanly. |
| 2 | Migration runners | shinka, repo-forge migration / absorption flows (`scripts/migrate-repo.sh` patterns) | Run rarely but need correctness — Rust+tatara give exhaustive error handling. |
| 3 | Cluster sweepers | `kubectl … | xargs … | sed …` one-shots, fleet-state extraction | Exposed as MCP tools for agent use; tatara-lisp gives them a typed callable surface. |
| 4 | CI scripts | `.github/workflows/*.yml` `run:` blocks, `bin/ci-*` | Deferred — GitHub Actions YAML is mostly thin glue, and pleme-io is moving toward `nix run .#<flow>` + `fleet`-driven CI anyway. |
| 5 | Setup / bootstrap glue | `setup.sh`, `bootstrap.sh` files in fresh repos | Also deferred — repo-forge owns these and will template-replace them when the upstream changes. |

Class 1 is the highest-leverage starting point because:

- Many small scripts (≥30 across helmworks alone) all do the same shape — `skopeo copy …` after a `nix build`. One tatara-lisp publisher consumed by all of them is a 90% reduction.
- Each conversion is independent + reversible.
- The benefit shows up immediately: typed errors, retries, idempotency, attestation hooks.

## V. The standard publisher (target shape)

```
;; helmworks/scripts/publish.tatara — fleet-wide image publisher.

(defpublish chart-image
  :registry  "ghcr.io/pleme-io"
  :auth      (auth/file (env "GHCR_AUTH_FILE"
                             :default (path :home ".config/containers/auth.json")))
  :tags      [:semver :latest]
  :on-success [(attest/sign :type :container)
               (announce :slack "#fleet-publishes")]
  :on-failure [(announce :slack "#fleet-failures")
               (retry 3 :backoff :exponential)])

;; Per-chart invocation — 1 line in the chart's flake.nix:
(publish chart-image :artifact (nix-build :attr "image") :name "pleme-storage-elastic")
```

The DSL emits a typed `publish` action. The `tatara` runner schedules
it, captures stdout/stderr as structured `Event` records, signs the
artifact, and notifies the fleet — all from one declarative
expression.

A bash script doing the same job:

```bash
set -euo pipefail
IMAGE_TAR=$(nix build --print-out-paths .#image)
skopeo copy --authfile "$GHCR_AUTH_FILE" \
  "docker-archive:$IMAGE_TAR" \
  "docker://ghcr.io/pleme-io/pleme-storage-elastic:$VERSION"
skopeo copy --authfile "$GHCR_AUTH_FILE" \
  "docker-archive:$IMAGE_TAR" \
  "docker://ghcr.io/pleme-io/pleme-storage-elastic:latest"
# … plus error handling, retry, signing, notification:  +50 lines.
```

The bash version is shorter at first glance but loses the typed
attestation hook, the Slack announcement, the retry policy, and the
typed error surface. The tatara-lisp version is the **whole truth** in
one place; the bash version is a **starting point** that grows over
time as each requirement (retry, attest, announce) gets bolted on.

## VI. Migration backlog (initial snapshot, 2026-04-26)

Locations of operational bash scripts identified across the fleet,
ordered by Class (§IV) and by ease-of-conversion:

### Class 1 — multi-step infra publishers

| Repo | Script | Job |
|---|---|---|
| `helmworks` | `nix run .#helm:release` flake apps | Build + push every chart to ghcr.io/pleme-io/charts |
| `forge` | `nix run .#publish` flake apps | Build + push every CI image |
| `nexus` | `nix run .#release` | Build + push every product image |
| `pleme-io/sui` | various `scripts/build*.sh` | Bootstrap + publish sui binaries |
| `pleme-io/substrate` | `lib/*.nix` writeShellApplication usage | Helper scripts inside Nix derivations |
| `pleme-storage-elastic/image/flake.nix` | (removed — manual skopeo now) | Will be replaced by the standard publisher above |

### Class 2 — migration runners

| Repo | Script | Job |
|---|---|---|
| `shinka` | `bin/migrate.sh` (if present) | Apply DB migrations; emit migration CR |
| `repo-forge` | `bin/absorb.sh` / `bin/migrate.sh` | Absorb / migrate repos to archetype |
| `kindling` | various flake `nix run .#vpn-*` apps | VPN keygen + setup |

### Class 3 — cluster sweepers

| Repo / location | Script | Job |
|---|---|---|
| `pleme-io/k8s` | `bin/cluster-state.sh` (if present) | Snapshot cluster state for offline analysis |
| `pleme-io/dev-tools` | `scripts/k8s-*.sh` | Various reconcile / drain / cordon helpers |

(Detailed conversion tracker lives in
[`pangea-architectures/CLAUDE.md` — "Helm-first authoring" section](../pangea-architectures/CLAUDE.md),
mirrored here as the parallel scripting backlog. As tatara-lisp's
runtime gains domains the conversions can land independently.)

## VII. Renderer enforcement (TODO)

Add to `pangea-core` (or to `repo-forge`'s lint rules) a
`ScriptingFirstValidator` that:

1. Walks every repo in the fleet.
2. Flags every `*.sh` / `*.bash` whose line count > 3 OR whose body
   contains a control-flow keyword (`if`, `for`, `while`, `case`).
3. Verifies a peer `.tatara` file exists, OR an explicit
   `.scripting-exception: <reason>` file annotates the directory.
4. Emits a typed report (count by class, repo, file).

This is the same shape as the `HelmFirstValidator` proposed in
`BREATHABILITY.md §VII.7` for in-cluster workloads — same idea applied
to a different layer.

## VIII. See also

- [`THEORY.md` Pillar 1 — Language](THEORY.md#pillar-1-language)
- [`THEORY.md` Pillar 12 — Generation over composition](THEORY.md#pillar-12-generation-over-composition)
- [`BREATHABILITY.md` §VII.7 — Helm-first authoring](BREATHABILITY.md)
  — the in-cluster analog of this rule.
- [`tatara/docs/rust-lisp.md`](https://github.com/pleme-io/tatara/blob/main/docs/rust-lisp.md)
  — the canonical Rust+Lisp cookbook.
