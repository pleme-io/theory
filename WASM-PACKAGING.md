# Packaging tatara-lisp like Nix flakes

> **Frame.** [`WASM-STACK.md`](WASM-STACK.md) packages the runtime;
> [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) names the
> authoring shape; [`WASM-PATTERNS.md`](WASM-PATTERNS.md) catalogues
> 49 program shapes. **This document closes the loop:** every program
> on the cluster is just a tatara-lisp package referenced by a Nix-
> flake-style URL (`github:owner/repo/path?ref=tag`). The cluster
> becomes **platform + tatara-lisp packages**. Every workload — jobs,
> async functions, warmed services, one-shots — runs through the same
> `ComputeUnit` abstraction. Lisp/WASM/WASI is the consistent runtime.

---

## I. The realization

The existing tatara-lisp ecosystem already has the packaging primitives:

1. `tatara-lisp` is a git-based Nix flake. Consumers pull it via
   `inputs.tatara-lisp.url = "github:pleme-io/tatara-lisp"`.
2. `tatara-script` takes a path and runs it; `(require "path.tlisp")`
   already does file-relative imports.
3. The interpreter has a rich stdlib (http, json, yaml, sops, sha256,
   regex, time, etc.) — it's already a *runtime*, not just an evaluator.

**What's missing:** the source-resolver step. Today `tatara-script
my.tlisp` requires a local path. Add the same URL grammar Nix flakes
use, and every cluster workload becomes a typed URL.

```
nix run github:owner/repo/path?ref=v1
              ↑
              same grammar
              ↓
tatara-script github:owner/repo/path?ref=v1.tlisp
```

## II. The source URL grammar

Same shape as Nix flake refs. Parsed by a single typed `Source`
struct:

```rust
pub enum Source {
    Local  { path: PathBuf },                    // ./foo.tlisp
    Git    { url: String, rev_or_ref: GitRev },  // git://github.com/...
    GitHub { owner: String, repo: String, path: PathBuf, rev: Option<String> },
    GitLab { owner: String, repo: String, path: PathBuf, rev: Option<String> },
    Codeberg { owner: String, repo: String, path: PathBuf, rev: Option<String> },
    Url    { url: String, hash: Option<Blake3Hash> },  // direct fetch, content-pinned
    Oci    { repo: String, tag: String, blake3: Option<Blake3Hash> },
    Flakehub { project: String, version: VersionConstraint },
}
```

URL forms recognized:

```
./local/path.tlisp                                      file path
github:owner/repo/path/to/program.tlisp                 latest main
github:owner/repo/path/to/program.tlisp?ref=v0.1.0      tagged
github:owner/repo/path/to/program.tlisp?ref=abc123      commit
github:owner/repo/path/to/program.tlisp?dir=programs    relative dir
gitlab:owner/repo/path.tlisp                            same shape
codeberg:owner/repo/path.tlisp                          same shape
git+https://example.com/repo.git?dir=programs           generic git
https://example.com/program.tlisp#blake3=abc123…       direct fetch + content pin
oci://ghcr.io/pleme-io/programs:dns-reconciler-v0.1.0   pre-compiled WASM
flakehub:pleme-io/programs/0.1                          flakehub-published lisp pkg
```

## III. The fetch + cache pipeline

```
┌───────────────────────────────────────────────────────────────────┐
│ author runs:  tatara-script github:pleme-io/programs/dns-reconciler/main.tlisp
│       OR:     ComputeUnit.spec.module.source =
│                     github:pleme-io/programs/dns-reconciler/main.tlisp?ref=v0.1.0
└──────────────────────────────────┬────────────────────────────────┘
                                   ▼
                     ┌──────────────────────────────┐
                     │ Source::parse_url(...)       │
                     └──────────────┬───────────────┘
                                    │
                     ┌──────────────▼───────────────┐
                     │ resolve_to_blake3(...)        │
                     │   - fetch tarball / file      │
                     │   - hash content              │
                     │   - return blake3 + bytes     │
                     └──────────────┬───────────────┘
                                    │
                ┌───────────────────┼───────────────────┐
                ▼                   ▼                   ▼
         ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
         │  module-store │  │ openclaw-side│  │ Nix store         │
         │  (PVC, RWX)   │  │ attestation   │  │ (host-side caching│
         │  blake3-keyed │  │ (optional)    │  │ for tatara-script)│
         └──────────────┘  └──────────────┘  └──────────────────┘
                                    │
                                    ▼
                  ┌─────────────────────────────────────┐
                  │ compile to WASM (one-time per blake3)│
                  │   tatara-lisp + wasi-builder         │
                  └──────────────┬──────────────────────┘
                                 ▼
                       ┌─────────────────────┐
                       │ wasm-engine instance │
                       │  + capability tokens │
                       └─────────────────────┘
```

The key design property: **the source URL determines the BLAKE3
hash, and the BLAKE3 hash is the cache key everywhere downstream.**
Two clusters fetching the same URL share the same compile cache. A
program pinned to a git revision is content-addressed without the
operator having to compute hashes manually.

## IV. The four runtime representations

The user observed that `program / job / service / controller` from
[`WASM-STACK.md` §I](WASM-STACK.md) maps onto serverless / containers
analogies:

```
┌────────────┬──────────────────────────┬───────────────────────────┐
│  shape     │ cloud-native analogy      │ lifecycle                  │
├────────────┼──────────────────────────┼───────────────────────────┤
│ program    │ AWS Lambda one-shot       │ run + exit                 │
│ function   │ AWS Lambda async event    │ wake on event, exit on done│
│ service    │ Cloud Run "warmed"        │ scale 0→N→0 with cooldown  │
│ controller │ Knative eventing operator │ watch loop, scale-to-zero  │
│ job        │ K8s CronJob               │ scheduled batch            │
└────────────┴──────────────────────────┴───────────────────────────┘
```

`ComputeUnit.spec.trigger` is the sum-type that selects:

```yaml
# program (one-shot)
trigger:
  oneShot: { args: [...] }

# function (event-driven async)
trigger:
  event:
    source: nats:rio.events.user.created    # or: kafka, sqs, redis-stream
    batch_size: 100                          # max events per invocation
    cooldown: 30s                            # tear down if no events

# service (warmed HTTP/gRPC)
trigger:
  service:
    port: 8080
    paths: [/v1/*]
    breathability:                           # KEDA HTTP add-on
      minReplicas: 0                         # → "warmed serverless"
      maxReplicas: 5
      cooldownPeriod: 600

# controller (CRD reconcile loop)
trigger:
  watch:
    group: dns.pleme.io
    kind:  DnsReconciler

# job (scheduled batch)
trigger:
  cron: "*/5 * * * *"
```

The wasm-operator dispatches each shape to the appropriate kubelet
primitive (Pod, Deployment, CronJob, …) but the *user-facing
contract is one CR* — `ComputeUnit`. From the author's perspective
these are not separate runtimes; they're separate `:trigger` types
on the same module shape.

### IV.1 The new `function` shape — async event-driven

This is the one not yet in `WASM-STACK.md`. It deserves its own
treatment because it's where the WASM cold-start advantage shines:

```yaml
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata:
  name: thumbnail-on-upload
spec:
  module:
    source: github:pleme-io/programs/thumbnail-fn/main.tlisp?ref=v1.2.0
  trigger:
    event:
      source: nats:rio.events.photo.uploaded
      batch_size: 10
      cooldown: 60s
  capabilities:
    - s3-read@photos-originals
    - s3-write@photos-thumbnails
    - http-out:imagemagick.svc.cluster.local
  resources:
    requests: { cpu: 100m, memory: 256Mi }
```

Behavior:

- NATS subject `rio.events.photo.uploaded` has 0 messages → no pods.
- One photo uploaded → one wasm-engine pod boots in ~3s, processes
  the event, exits.
- 100 photos uploaded → operator dispatches in batches of 10; could
  parallelize across N pods up to a configured max.
- Idle 60s → next event boots cold again.

Equivalent to AWS Lambda async invocations, but:
- Runs in your cluster.
- 3s cold start, not 30s.
- Capability-bounded (no IAM-style ambient authority).
- Same runtime as everything else; no separate Lambda execution model.

### IV.2 The `service` shape's "warmed" property

Plain K8s Deployments are always-on. KEDA HTTP scales 0→N. WASM
makes the cold-start so cheap (~3s vs ~30s containers) that the
practical user experience approaches "always-on serverless" — but
the resident cost is zero between idle periods.

`cooldownPeriod` is the dial. `cooldown=10s` means the pod stays
warm for 10s after the last request — basically equivalent to "warm"
for any continuous traffic; cold once per drift period. `cooldown=600s`
keeps the pod warm for 10 minutes — equivalent to "always on" for
business-hours use cases.

## V. The cluster topology

The end state is a cluster where every program is a typed URL:

```yaml
# clusters/rio/programs/*.yaml — all 30 of rio's lareira services
# rendered as ComputeUnit CRs pointing at github: URLs.

apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata: { name: pvc-autoresizer }
spec:
  module:  { source: github:pleme-io/programs/pvc-autoresizer/main.tlisp?ref=v0.1.0 }
  trigger: { cron: "*/5 * * * *" }
---
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata: { name: dns-reconciler }
spec:
  module:  { source: github:pleme-io/tatara-lisp-controllers/examples/dns-reconciler/main.tatara?ref=v0.1.0 }
  trigger: { watch: { crd: dns.pleme.io/DnsReconciler } }
---
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata: { name: thumbnail-fn }
spec:
  module:  { source: github:pleme-io/programs/thumbnail-fn/main.tlisp?ref=v1.2.0 }
  trigger: { event: { source: nats:rio.events.photo.uploaded } }
---
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata: { name: rate-limit-svc }
spec:
  module:  { source: github:pleme-io/programs/rate-limit/main.tlisp?ref=v2.0.0 }
  trigger: { service: { port: 8080 } }
```

What's left in the cluster:

- **The platform** — the lareira-* charts that the rest of this
  session built (cert-manager, ingress-nginx, KEDA, vm-stack,
  authentik, etc.).
- **The runtime** — `lareira-wasm-platform` that runs the WASM.
- **A pile of `ComputeUnit` CRs** — each pointing at a tatara-lisp
  package by URL.

That's it. No bespoke Go controllers, no per-tool container images,
no kustomize trees per program. Adding a program to the cluster is
publishing a `.tatara` file to GitHub + applying a 12-line CR.

## VI. Implementation sketch

### VI.1 Source-resolver crate

```
tatara-lisp/tatara-lisp-source/
├── Cargo.toml
└── src/
    ├── lib.rs               Source enum + parse_url + resolve fns
    ├── github.rs            GitHub tarball API + auth via GH_TOKEN
    ├── gitlab.rs            GitLab equivalent
    ├── git.rs               generic git (libgit2 or shelled to git)
    ├── http.rs              direct URL fetch + content-pin verify
    └── oci.rs               OCI registry pull (skopeo / oci-distribution crate)
```

Used by both `tatara-script` (host-side) and the wasm-operator
(cluster-side) — same code, different deployment target.

### VI.2 Cache layout

Host-side (`tatara-script`):

```
~/.cache/tatara/
├── sources/
│   └── <blake3>/                    raw .tlisp source bytes
├── compiled/
│   └── <blake3>/<wasi-target>.wasm  compiled WASM modules
└── manifest.json                    URL → blake3 mapping (lookups + GC)
```

Cluster-side (`wasm-operator`):

```
/var/lib/wasm-store/
├── sources/<blake3>/...
├── compiled/<blake3>/...wasm
└── manifest.json
```

Shared blake3 → the host's `tatara-script` and the cluster's
`wasm-operator` cache the same way; if a developer publishes a
program locally to test, the cluster's first fetch + compile is the
only network round-trip.

### VI.3 The `flake.lock`-equivalent

`tatara-lisp-script` learns to write `tatara.lock`:

```json
{
  "version": 1,
  "imports": {
    "github:pleme-io/programs/dns-reconciler/main.tlisp?ref=v0.1.0": {
      "blake3": "abc123…",
      "fetched_at": "2026-04-26T03:14:15Z",
      "size_bytes": 4096
    }
  }
}
```

A program that `(require ...)` other programs by URL pins the
transitive set in `tatara.lock` — same shape Nix flakes use. CI
detects when the lock doesn't match upstream and fails the build.

### VI.4 The wasm-operator's source field

```rust
// wasm-types/src/compute.rs — already has `module.source: ModuleSource`.
// Extend the enum:
pub enum ModuleSource {
    Local      { path: PathBuf },
    Inline     { bytes: Vec<u8> },
    Oci        { reference: String, blake3: Option<String> },
    GitHub     { owner: String, repo: String, path: PathBuf, rev: Option<String> },
    GitLab     { owner: String, repo: String, path: PathBuf, rev: Option<String> },
    HttpDirect { url: String, blake3: String },        // requires hash for pin
    Url        { url: String },                         // generic; fall through to git/http
}
```

The operator's `resolve_module` walks the enum and dispatches to the
matching fetcher. New source types are additive (`AzureRepos`,
`Flakehub`, an internal corp git server) — the operator's
reconciliation loop doesn't change.

## VII. Authentication

Pleme-io's existing patterns:

- **Public GitHub** — anonymous fetch.
- **Private GitHub** — `GITHUB_TOKEN` mounted via Akeyless / sops-nix
  Secret. Operator namespaces declare a `pull-secret-ref:` field.
- **Internal git** — SSH key from `kube-secret-read@<ns>/git-key`.
- **OCI registries** — `imagePullSecrets` already supported by
  the operator's pod spec generation; pre-existing path.

The capability model already covers this — a `ComputeUnit` whose
`source` requires GitHub auth needs `kube-secret-read@<ns>/<gh-secret>`
in its `capabilities` list. The operator refuses to fetch otherwise.

## VIII. Migration: from local-path to github:URL

The `pleme-storage-elastic` Rust watcher (built earlier this session)
becomes the first dogfood:

### Before

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata: { name: pleme-storage-elastic }
spec:
  chart:
    spec:
      chart: pleme-storage-elastic
      version: "0.1.x"
  values:
    enabled: true
    image:
      repository: ghcr.io/pleme-io/pleme-storage-elastic
      tag: 0.1.0
```

### After

```yaml
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata: { name: pvc-autoresizer }
spec:
  module:
    source: github:pleme-io/programs/pvc-autoresizer/main.tlisp?ref=v0.1.0
  trigger: { cron: "*/5 * * * *" }
  capabilities:
    - kube-pvc-list
    - kube-pvc-patch
    - prom-query@vmsingle-vm.monitoring.svc:8429
```

The `pleme-storage-elastic` Helm chart goes away entirely. The
`image/` Cargo crate also goes away. The 280-line Rust binary
becomes a ~50-line tatara-lisp file.

## IX. Why this is a generalization, not a rewrite

The existing pleme-io repos cleanly absorb this:

| Repo | What it gains |
|---|---|
| `tatara-lisp` | new `tatara-lisp-source` crate; `tatara-script` learns to fetch URLs |
| `wasm-platform` | `wasm-types::ModuleSource` enum extends additively |
| `repo-forge` | new `tatara-lisp-program` archetype with the right boilerplate |
| `helmworks` | `lareira-wasm-platform` chart already accepts module URLs in `ComputeUnit.spec.module.source` — no chart change needed |
| `tatara-infra` | becomes a flat list of `ComputeUnit` CRs pointing at github URLs |
| `forge` | image publish pipeline shrinks to "publish to GitHub repo, push tag" — no per-tool image build |

Six repos benefit; none change their core abstraction. The runtime
*becomes* "platform + tatara-lisp packages" not by rewriting the
platform but by fully exercising the module-source contract that's
already in `wasm-types`.

## X. Phasing

- **Phase A (designed — this doc):** all four representations + URL
  grammar + cache layout + auth.
- **Phase B (Rust work):**
    1. `tatara-lisp/tatara-lisp-source` crate (new).
    2. `tatara-script` learns to accept URLs; writes `tatara.lock`.
    3. `wasm-types::ModuleSource` enum extension.
    4. `wasm-operator` resolver step using the new crate.
- **Phase C (first dogfood):**
    1. `pleme-io/programs` repo (new) housing `pvc-autoresizer`,
       `dns-reconciler`, `thumbnail-fn`, `rate-limit-svc`.
    2. Convert `pleme-storage-elastic` Rust watcher → tatara-lisp
       program — first end-to-end github URL CR on rio.
    3. Delete `helmworks/charts/pleme-storage-elastic` once the CR
       is healthy.
- **Phase D (the rest):** every cookbook pattern that has a Rust /
  bash counterpart in the fleet gets migrated. `defwebhook-handler`
  patterns next; then the saga / etl / slo controllers.
- **Phase E (cluster-as-package-manifest):** every cluster's
  workload list is just a `tatara-infra/clusters/<name>/programs.yaml`
  declaring URL-addressed ComputeUnits. The whole cluster is
  reproducible from one git URL.

## XI. The end state

```
            cluster
                │
                ▼
┌─────────────────────────────────────────────────────────┐
│  PLATFORM (lareira-* + wasm-platform + KEDA + ESO + …)  │
│  the always-on substrate; <500 MiB resident at idle      │
└─────────────────────────────────────────────────────────┘
                                                          ▲
                                                          │  applies
                                                          │
┌─────────────────────────────────────────────────────────┘
│  ComputeUnit CRs pointing at github:* URLs
│
│  Each CR is 10–20 lines of YAML. Each program is 30–100 lines
│  of tatara-lisp. Each compiled module is 1–5 MiB of WASM.
│  Idle, none of them consume any resources.
│
│  Adding a program: push to GitHub, apply the CR.
│  Removing a program: delete the CR (or kustomize patch).
│  Upgrading a program: bump the ?ref=<version> in the CR.
└─────────────────────────────────────────────────────────
```

The cluster is just **platform + tatara-lisp packages**. Every
program is a typed URL. Every workload is breathable. Lisp/WASM/WASI
is the consistent abstraction.

## XII. See also

- [`WASM-STACK.md`](WASM-STACK.md) — the runtime
- [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49 program shapes
- [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) — Lisp + YAML
  authoring
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`BREATHABILITY.md`](BREATHABILITY.md) — fleet-wide use-causes-spin-up
- [Nix flake URL grammar](https://nixos.org/manual/nix/stable/command-ref/new-cli/nix3-flake.html#flake-references) — the grammar to mirror
- [`tatara-lisp`](https://github.com/pleme-io/tatara-lisp) — current packaging
