# WASI service deploy chain — typed source to running pod

> **★★★ CSE / Knowable Construction.** This document is the canonical
> end-to-end deploy chain for tatara-lisp / Rust → WASI component →
> running pod on a pleme-io cluster (rio for Phase A; any cluster for
> Phase B). Read [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> first for the methodology; this doc is the operational specialization
> for the WASI deployment vertical.

---

## TL;DR

Two parallel deploy paths now coexist:

| Path | Status | Mechanism | Operator-mediated |
|------|--------|-----------|--------------------|
| **standalone** | Works today | `wasi-service-flux-flake` renders one HelmRelease per service using `bjw-s/app-template` | No — direct Helm |
| **lareira** | Phase B (suspended) | `wasi-service-flux-flake` contributes a `programs[]` entry to `lareira-fleet-programs`; the chart aggregates; `wasm-operator` reconciles each entry to a running pod via the `wasm-engine` runtime | Yes — `Program` CRD + operator |

Both paths share the same authoring surface (`module = { ... }` + `deploy = { cluster, ... }` in the service's `flake.nix`) and the same on-disk artifact tree (`k8s/clusters/<cluster>/services/<name>/`). Migrating from standalone to lareira is one-flag (`deploy.mode = "lareira"`) once Phase B prerequisites are met.

---

## The full chain (left to right)

```
┌─────────────────┐    ┌───────────────────┐    ┌───────────────────┐    ┌────────────────┐    ┌─────────────────┐    ┌──────────────────┐
│  tatara-lisp    │ → │ Rust crate +      │ → │ wasm32-wasip2     │ → │ Docker image   │ → │ FluxCD bundle   │ → │ K3s pod          │
│  defservice/    │    │ `module = { ... }`│    │ component         │    │ (wasmtime +    │    │ (HelmRelease +  │    │ (Deployment +    │
│  defjob form    │    │ `deploy = { ... }`│    │ (.wasm artifact)  │    │  .wasm)        │    │  kustomization) │    │  Service +       │
│                 │    │ in flake.nix      │    │                   │    │ pushed to GHCR │    │ committed to    │    │  Ingress, etc.)  │
│ pleme-io/       │    │ pleme-io/         │    │ substrate's       │    │ via forge      │    │ pleme-io/k8s    │    │ rio reconciles   │
│ programs/       │    │ <service>/        │    │ wasi-service-     │    │ image-push     │    │ via             │    │ via FluxCD       │
│ <name>/         │    │ flake.nix         │    │ flake.nix         │    │ workflow       │    │ wasi-service-   │    │                  │
│ main.tlisp      │    │                   │    │                   │    │                │    │ flux-flake.nix  │    │                  │
└─────────────────┘    └───────────────────┘    └───────────────────┘    └────────────────┘    └─────────────────┘    └──────────────────┘
                                                                                                       │
                                                                                                       │  (Phase B — lareira mode)
                                                                                                       ▼
                                                                                            ┌─────────────────┐    ┌──────────────────┐
                                                                                            │ Program CR      │ → │ wasm-operator    │
                                                                                            │ (one entry in   │    │ reconciles to    │
                                                                                            │ lareira-fleet-  │    │ Deployment +     │
                                                                                            │ programs values)│    │ Service +        │
                                                                                            │                 │    │ ScaledObject     │
                                                                                            └─────────────────┘    │ (KEDA-backed,    │
                                                                                                                   │  scale-to-zero,  │
                                                                                                                   │  using           │
                                                                                                                   │  ghcr.io/pleme-io│
                                                                                                                   │  /wasm-engine    │
                                                                                                                   │  as runtime)     │
                                                                                                                   └──────────────────┘
```

---

## Per-layer responsibilities

### 1. Authoring (tatara-lisp)

The service author writes a typed `(defservice ...)` / `(defjob ...)` /
`(defcontroller ...)` form in `pleme-io/programs/<name>/main.tlisp`.
The form declares: name, description, capabilities, port, paths,
config defaults. See [`SCRIPTING.md`](./SCRIPTING.md) for the canonical
DSL and [`LISP-YAML-CONTROLLERS.md`](./LISP-YAML-CONTROLLERS.md) for
the four authoring tiers.

The form is *data*. The Rust crate at `pleme-io/programs/<name>/` glues
the form to a runtime via `include_str!` / `tatara-lisp::eval`.

Standalone-mode services (today's MVP) skip the .tlisp layer and live
directly as a Rust crate at `pleme-io/<name>/`. They can author a
`.tlisp` form later without changing the deploy chain — that's exactly
the migration path standalone → lareira embodies.

### 2. Build (Rust + substrate)

The service's `flake.nix` calls
[`substrate/lib/wasi-service-flux-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/wasi-service-flux-flake.nix)
which wraps `wasi-service-flake.nix` (the underlying wasm + Docker image
builder).

The single typed spec:

```nix
module = {
  description = "...";
  hmNamespace = "blackmatter.components";
};
deploy = {
  cluster = "rio";
  namespace = "tatara-system";
  imageRepo = "ghcr.io/pleme-io";
  imageTag = "latest";
  port = 8080;
  healthPath = "/healthz";
  config = { GREETING = "Hello"; AUDIENCE = "rio"; };
  ingress = { enabled = true; host = "hello.quero.cloud"; };
  mode = "standalone";   # or "lareira"
  breathability = { enabled = true; minReplicas = 0; maxReplicas = 5; };
};
```

drives every artifact below. No hand-written HelmRelease YAML, no
per-service kustomize plumbing.

### 3. WASM compilation

`substrate/lib/build/wasm/wasi-service.nix` invokes the fenix Rust
toolchain with `wasm32-wasip2` target. Output: `target/wasm32-wasip2/
release/<name>.wasm`. Reproducibly built under Nix; cached via flake
inputs.

### 4. Docker image

`wasi-service.nix` packages the wasm into a minimal image: a `wasmtime`
binary as ENTRYPOINT, the .wasm as the program argument, capability
flags from `wasiCapabilities`. Image is published to
`ghcr.io/pleme-io/<name>:<tag>` via substrate's
[`image-push.yml`](https://github.com/pleme-io/substrate/blob/main/.github/workflows/image-push.yml)
reusable workflow.

### 5. FluxCD bundle rendering

`wasi-service-flux-flake.nix`'s `nix run .#render-deploy` writes:

```
pleme-io/k8s/clusters/<cluster>/services/<name>/
├── kustomization.yaml
└── release.yaml          # standalone HelmRelease (mode = "standalone")
                          # OR Program CR contribution (mode = "lareira")
```

The renderer is purely typed — no string templating, no shelling out.
Re-rendering is idempotent and produces deterministic YAML.

### 6. Git → cluster

`nix run .#deploy-<cluster>` adds + commits + pushes the bundle. FluxCD
on the target cluster reconciles within 5 minutes (the parent
Kustomization at `flux-kustomizations/services.yaml` polls the repo).

In **standalone mode**, FluxCD's `helm-controller` materializes the
HelmRelease using `bjw-s/app-template`, producing Deployment +
Service + Ingress.

In **lareira mode**, FluxCD's `helm-controller` materializes the
parent `lareira-fleet-programs` HelmRelease, which iterates the
aggregated `programs[]` and emits one `tatara.pleme.io/v1alpha1/Program`
CR per entry. The `wasm-operator` (deployed by `lareira-tatara-stack`)
watches `Program` CRs and reconciles each to a Deployment running
`wasm-engine` (image `ghcr.io/pleme-io/wasm-engine:0.1.0`) which
fetches the `.wasm` from the URL grammar and serves it.

### 7. Verify

`cse-lint audit --strict` includes a `deployment-coverage` checker
(pending implementation) that asserts every manifest entry with
`class = "wasi-service"` has a corresponding `services/<name>/`
directory in the target cluster's k8s overlay. Closes the loop:
typed source ↔ rendered deployment ↔ CSE adherence.

`shinryu` (the analytical query plane) observes runtime state — pod
health, latency, error rates — independent of the deploy chain. Both
substrate-engineered: each layer is typed and provably renders to
the next.

---

## Standalone mode — works today

Prerequisites (all met):
- `bjw-s` HelmRepository in `flux-system` namespace (may need adding).
- `forge push` workflow OR manual `docker push ghcr.io/pleme-io/<name>:<tag>`.
- Ingress controller (nginx) on rio (already present).
- DNS for the ingress host (Cloudflare Tunnel for `*.quero.cloud`).

To deploy a new standalone service:

1. Create `pleme-io/<name>/` repo with `flake.nix` consuming
   `wasi-service-flux-flake.nix`, `Cargo.toml`, `src/main.rs` (a thin
   Rust crate that builds to `wasm32-wasip2`).
2. Push the GHCR image once (via forge image-push workflow once the
   repo is created on github.com/pleme-io).
3. `nix run .#deploy-rio` — writes the bundle, commits, pushes.
4. Wait ~5 minutes for FluxCD; the pod schedules.

`hello-rio` (`pleme-io/hello-rio`) is the canonical reference. Read
its `flake.nix`; that's the authoring template.

---

## Lareira mode — Phase B

Prerequisites (pending):

| Prerequisite | Status | Where it lives |
|--------------|--------|----------------|
| `ghcr.io/pleme-io/wasm-operator:0.1.0` | Pending build | `pleme-io/wasm-operator` (scaffolded) |
| `ghcr.io/pleme-io/wasm-engine:0.1.0`   | Pending build | `pleme-io/wasm-engine` (scaffolded) |
| `lareira-tatara-stack` Helm chart      | Pending build | `pleme-io/lareira-charts` (scaffolded) |
| `lareira-fleet-programs` Helm chart    | Pending build | `pleme-io/lareira-charts` (scaffolded) |
| `pleme-charts` HelmRepository OCI publish | Pending      | `pleme-io/lareira-charts` GHCR publish |
| `lareira-tatara-stack` HelmRelease unsuspend | Pending     | `k8s/clusters/rio/infrastructure/wasm-stack/release.yaml` (currently `suspend: true`) |
| `lareira-fleet-programs` HelmRelease unsuspend | Pending   | `k8s/clusters/rio/programs/release.yaml` (currently `suspend: true`) |

To migrate a standalone service to lareira:

1. Implement Phase B prerequisites above (multi-day work; tracked in
   each scaffolded repo).
2. Flip `deploy.mode = "lareira"` in the service's `flake.nix`.
3. `nix run .#render-deploy` — re-renders into `services/<name>/program.yaml`
   (a single `Program` CR contribution to `lareira-fleet-programs`).
4. Drop the old `services/<name>/release.yaml`; the lareira-fleet-programs
   chart aggregates all `program.yaml` files via kustomize merge.

The substrate primitive handles the file-shape difference; the
deployed pod's runtime behavior is identical (wasmtime running the
same .wasm). Lareira mode adds: KEDA-backed scale-to-zero,
capability-token gating, the `Program` CR audit trail.

---

## Why two modes, not one

CSE principle 2 (load-bearing fix > local fix). The full lareira
architecture is the right end state — it provides scale-to-zero, capability
tokens, the typed `Program` CR audit surface, the recursive bootstrap.
But Phase B prerequisites are weeks of work across four repos.
Standalone mode is the *bridge*: every authoring decision (typed
spec, repo shape, Cargo crate, .tlisp form, file layout) carries
forward to lareira mode unchanged. The substrate macro absorbs the
mode switch. Operators get a working deploy chain *today* without
waiting for Phase B.

This is the same pattern the trio macro used: ship the canonical
primitive first, layer the operator on top later.

---

## Pointers

- Substrate primitive:
  [`substrate/lib/wasi-service-flux-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/wasi-service-flux-flake.nix)
- Underlying wasi-service builder:
  [`substrate/lib/wasi-service-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/wasi-service-flake.nix)
- Canonical reference service: `pleme-io/hello-rio`
- Phase B scaffolds:
  - `pleme-io/programs` — typed program inventory
  - `pleme-io/wasm-operator` — Program-CR reconciler
  - `pleme-io/wasm-engine` — wasmtime runtime image
  - `pleme-io/lareira-charts` — `lareira-tatara-stack` + `lareira-fleet-programs`
- k8s/cluster overlay: `pleme-io/k8s/clusters/rio/services/`
- FluxCD wiring: `pleme-io/k8s/clusters/rio/flux-kustomizations/services.yaml`
- Theory frame: [`THEORY.md`](./THEORY.md), [`WASM-STACK.md`](./WASM-STACK.md),
  [`FLEET-DECLARATION.md`](./FLEET-DECLARATION.md), [`LISP-YAML-CONTROLLERS.md`](./LISP-YAML-CONTROLLERS.md).
- Methodology: [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md).
