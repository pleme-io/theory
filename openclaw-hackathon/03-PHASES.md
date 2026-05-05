# 03 — Phases

> Linear, file-by-file execution plan for the openclaw / cartorio /
> tameshi demo on `pleme-dev`. Every step has explicit *files*, *commands*,
> *verification*, and *rollback*. Dependencies between steps are
> declared at the top of each phase. Phases B / C / D can largely run
> in parallel once Phase A is committed.
>
> **Branch policy** (from `00-OVERVIEW`): direct-to-main on every repo.
> No feature branches. Each step is one or more commits on `main`.

## Phase summary

| Phase | Scope | Depends on | Parallelizable with |
|---|---|---|---|
| **A — Cluster substrate** | flux-system, cert-manager, pleme-charts source, cnpg, shinka, namespace, postgres CR, DNS | nothing | — |
| **B — Cartorio code** | SeaORM entities, migrations, repository layer, new endpoints, GraphQL, cartorio-verify CLI | A.1 (DNS upsert can lag) | C, D.7 |
| **C — Cartorio-web code** | Leptos app, hanabi-ready bundle | B.5 (API surface stable) | B (after B.5), D |
| **D — Helm charts** | 6 new charts in helmworks + sekiban/kensa overlay add | B (image source), C (image source) | E.7 lands later |
| **E — FluxCD wiring** | apps/openclaw HelmReleases, Ingress, sekiban/kensa overlay | A, D | F |
| **F — Verification** | smoke + tamper + drift + audit live tests | A, B, C, D, E | G |
| **G — Rehearsal** | full demo run, time-coded | F | — |

---

# Phase A — Cluster substrate

> Brings `pleme-dev` from "hello-world only" to "ready to host the
> openclaw stack". One cluster wake/verify/sleep cycle at the end
> exercises the bootstrap. Everything lands as files in
> `pleme-io/k8s.git@main` for FluxCD reconcile.

### A.1 — DNS upsert: add `openclaw.dev.use1.quero.lol`

*Why:* Audience needs to reach the demo over the public internet without
Tailscale-fiddling. Cluster's first-boot script already publishes
`api.dev.use1.quero.lol`; we add a sibling record.

*Files:*
- `pleme-io/programs/k3s-bootstrap/k3s-bootstrap.tlisp` — append one
  Route53 upsert call alongside the existing `api.dev.use1.quero.lol`
  upsert. (The tlisp form is already typed; new record = same shape,
  different host.)

*Commands:*
```bash
cd ~/code/github/pleme-io/programs
$EDITOR k3s-bootstrap/k3s-bootstrap.tlisp
git -c commit.gpgsign=false commit -am "k3s-bootstrap: publish openclaw.dev.use1.quero.lol on first-boot"
git push origin main
```

*Verify:* Next cluster wake, after `phase=ready`, both DNS records
resolve to the same fresh public IP. `dig openclaw.dev.use1.quero.lol
+short` returns the IP within ~30s of phase=ready.

*Rollback:* `git revert <commit>` and re-push.

### A.2 — Bootstrap flux-system manifests in repo

*Why:* The cluster IS bootstrapped (proven last cycle), but the
bootstrap isn't tracked in git. We commit it so the bootstrap is
reproducible and reviewable.

*Files:*
- `k8s/clusters/pleme-dev/flux-system/gotk-components.yaml` (FluxCD CRDs,
  produced via `flux install --export`)
- `k8s/clusters/pleme-dev/flux-system/gotk-sync.yaml` (GitRepository →
  `pleme-io/k8s.git@main` + Kustomization root pointing at
  `clusters/pleme-dev/`)
- `k8s/clusters/pleme-dev/flux-system/kustomization.yaml`

*Commands:*
```bash
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml flux install --export \
  > k8s/clusters/pleme-dev/flux-system/gotk-components.yaml
# author gotk-sync.yaml + kustomization.yaml by hand (small, ~30 lines)
git add k8s/clusters/pleme-dev/flux-system/
git commit -m "k8s/pleme-dev: track flux bootstrap in git"
git push origin main
```

*Verify:* `kubectl -n flux-system get gitrepositories,kustomizations`
shows the source-of-truth GitRepository pointing at the committed
manifests. Reconcile is idempotent — the cluster is already running these
controllers.

*Rollback:* No revert needed; tracking-only change. If the manifests
diverge from running state, restore via `flux install --export` again.

### A.3 — cert-manager + selfsigned-issuer

*Why:* sekiban's ValidatingAdmissionWebhook needs TLS. Selfsigned is
sufficient for `pleme-dev`.

*Files:*
- `k8s/clusters/pleme-dev/infrastructure/cert-manager/helmrelease.yaml`
  (cert-manager v1.16+, namespace `cert-manager-system`, CRDs enabled)
- `k8s/clusters/pleme-dev/infrastructure/cert-manager/cluster-issuer.yaml`
  (`selfsigned-issuer` ClusterIssuer)
- `k8s/clusters/pleme-dev/infrastructure/cert-manager/kustomization.yaml`

*Commands:*
```bash
git add k8s/clusters/pleme-dev/infrastructure/cert-manager/
git commit -m "k8s/pleme-dev: cert-manager + selfsigned-issuer"
git push origin main
# wake cluster, watch reconcile
PLATFORM=pleme nix run .../platform-k3s#wake
# (verify below)
```

*Verify:*
```
kubectl -n cert-manager-system get pods                       # all Running
kubectl get clusterissuer selfsigned-issuer -o jsonpath='{.status.conditions[0].type}'
# → Ready
```

*Rollback:* `git revert` the commit; FluxCD will Suspend then Delete the
HelmRelease.

### A.4 — `pleme-charts` HelmRepository + GHCR pull secret

*Why:* All new helmworks charts publish to
`oci://ghcr.io/pleme-io/charts`. FluxCD needs an OCI HelmRepository source
plus a SOPS-encrypted GHCR pull token.

*Files:*
- `k8s/clusters/pleme-dev/infrastructure/pleme-charts/helmrepository.yaml`
  (OCI HelmRepository named `pleme-charts`, namespace `flux-system`)
- `k8s/clusters/pleme-dev/infrastructure/pleme-charts/ghcr-charts-secret.yaml`
  (SOPS-encrypted Secret with GHCR pull token)
- `k8s/clusters/pleme-dev/infrastructure/pleme-charts/kustomization.yaml`

*Commands:*
```bash
# generate GHCR token with read:packages scope (gh CLI):
gh auth token | gh api ... # or hand-create from web UI
kubectl create secret docker-registry ghcr-charts-secret \
  --docker-server=ghcr.io --docker-username=<user> \
  --docker-password=<token> --dry-run=client -o yaml \
  | sops -e --encrypted-regex '^(data|stringData)$' \
  > k8s/clusters/pleme-dev/infrastructure/pleme-charts/ghcr-charts-secret.yaml
git add k8s/clusters/pleme-dev/infrastructure/pleme-charts/
git commit -m "k8s/pleme-dev: pleme-charts OCI HelmRepository"
git push origin main
```

*Verify:* `kubectl -n flux-system get helmrepositories pleme-charts` shows
`READY=True`.

*Rollback:* git revert.

### A.5 — CloudNative-PG operator

*Why:* Cartorio (and openclaw-skill-store) need Postgres. Lilitu uses
CNPG; we mirror.

*Files:*
- `k8s/clusters/pleme-dev/infrastructure/cnpg-operator/helmrelease.yaml`
  (`cloudnative-pg` chart, namespace `cnpg-system`, latest stable)
- `k8s/clusters/pleme-dev/infrastructure/cnpg-operator/kustomization.yaml`

*Commands:*
```bash
git add k8s/clusters/pleme-dev/infrastructure/cnpg-operator/
git commit -m "k8s/pleme-dev: cloudnative-pg operator"
git push origin main
```

*Verify:* `kubectl -n cnpg-system get pods` shows the controller Running.

*Rollback:* git revert (CRDs persist; if needed, `kubectl delete crd
clusters.postgresql.cnpg.io` etc. — destroy on dev cluster only).

### A.6 — shinka migration operator

*Why:* SeaORM migrations run via shinka's Migration CRD on every deploy.
Lilitu uses this; we mirror.

*Files:*
- `k8s/clusters/pleme-dev/infrastructure/shinka/helmrelease.yaml`
  (`shinka` chart, namespace `shinka-system`)
- `k8s/clusters/pleme-dev/infrastructure/shinka/kustomization.yaml`

*Commands:*
```bash
git add k8s/clusters/pleme-dev/infrastructure/shinka/
git commit -m "k8s/pleme-dev: shinka migration operator"
git push origin main
```

*Verify:* `kubectl -n shinka-system get pods` shows the operator Running.
`kubectl get crd migrations.shinka.pleme.io` exists.

*Rollback:* git revert.

### A.7 — `pleme-attestation` namespace + default-deny NetworkPolicy

*Why:* All openclaw workloads land here. PSS=restricted enforced.
Default-deny baseline prevents accidental cross-namespace reach.

*Files:*
- `k8s/clusters/pleme-dev/apps/openclaw/namespace.yaml`
  (Namespace `pleme-attestation` with PSS labels:
  `pod-security.kubernetes.io/{enforce,audit,warn}: restricted`)
- `k8s/clusters/pleme-dev/apps/openclaw/network-policy-default-deny.yaml`
  (NetworkPolicy `default-deny-all` matching all pods in the namespace,
  zero ingress + zero egress; explicit allows added per-pod by helm
  charts.)
- `k8s/clusters/pleme-dev/apps/openclaw/kustomization.yaml`
  (initially just these two; HelmReleases added in Phase E.)
- `k8s/clusters/pleme-dev/apps/kustomization.yaml`
  (add `openclaw/` reference.)

*Commands:*
```bash
git add k8s/clusters/pleme-dev/apps/openclaw/{namespace,network-policy-default-deny,kustomization}.yaml
git add k8s/clusters/pleme-dev/apps/kustomization.yaml
git commit -m "k8s/pleme-dev: pleme-attestation namespace + default-deny"
git push origin main
```

*Verify:*
```
kubectl get ns pleme-attestation -o jsonpath='{.metadata.labels}' | grep restricted
kubectl -n pleme-attestation get netpol
# → default-deny-all
```

*Rollback:* git revert. Namespace deletion drops every workload; do
this only when the cluster is intentionally being torn down.

### A.8 — `cartorio-postgres` CNPG Cluster

*Why:* The Postgres database backing cartorio-backend.

*Files:*
- `k8s/clusters/pleme-dev/apps/openclaw/cartorio-postgres-cluster.yaml`
  (CNPG `Cluster` CR; instances=1; storage=10Gi local-path; bootstrap
  initdb with database=`cartorio`, owner=`cartorio`)
- update `k8s/clusters/pleme-dev/apps/openclaw/kustomization.yaml` to
  include it.

*Commands:*
```bash
git add k8s/clusters/pleme-dev/apps/openclaw/cartorio-postgres-cluster.yaml
git add k8s/clusters/pleme-dev/apps/openclaw/kustomization.yaml
git commit -m "k8s/pleme-dev: cartorio-postgres CNPG Cluster"
git push origin main
```

*Verify:*
```
kubectl -n pleme-attestation get cluster cartorio-postgres
# → Status=Cluster in healthy state
kubectl -n pleme-attestation get secret cartorio-postgres-connection
# → Type=kubernetes.io/basic-auth, contains DATABASE_URL
```

*Rollback:* git revert. Cluster deletion drops the PVC; data is lost
(M0 demo, no backup configured).

### A.9 — Phase A cluster cycle confirmation

*Why:* Validate the bootstrap reconciles cleanly from cold sleep.

*Commands:*
```bash
PLATFORM=pleme AWS_PROFILE=akeyless-development nix run .../platform-k3s#wake
# wait until /readyz returns ok (typically ~70s)
# build kubeconfig:
bash /tmp/build-pleme-kubeconfig.sh
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl get kustomizations -A
# → all READY=True
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl -n pleme-attestation \
  get cluster cartorio-postgres
# → healthy
PLATFORM=pleme nix run .../platform-k3s#sleep
```

*Pass criteria:*
- All Kustomizations READY=True
- cert-manager controller Running, selfsigned-issuer Ready
- cnpg-operator Running, cartorio-postgres Cluster healthy
- shinka-system controller Running
- `pleme-attestation` namespace exists with PSS labels + default-deny netpol

*Rollback:* If any step fails, fix the offending file; re-push; cluster
re-reconciles. No state to roll back beyond git.

---

# Phase B — Cartorio code

> Cartorio repo is restructured into a lilitu-shaped monorepo. SeaORM
> entities, migrations, repository layer, new endpoints, GraphQL surface,
> and `cartorio-verify` CLI all land in `pleme-io/cartorio`.

### B.1 — Workspace restructure: `services/rust/`

*Why:* The current `cartorio` is a single Cargo crate; we lift it into
the lilitu workspace shape (backend + migration + cartorio-verify).

*Files added:*
- `cartorio/services/rust/Cargo.toml` — workspace declaration
- `cartorio/services/rust/backend/Cargo.toml` — moved from existing crate
- `cartorio/services/rust/migration/Cargo.toml` — new
- `cartorio/services/rust/cartorio-verify/Cargo.toml` — new

*Files moved:*
- existing `cartorio/src/` → `cartorio/services/rust/backend/src/`

*Commands:*
```bash
cd ~/code/github/pleme-io/cartorio
mkdir -p services/rust/{backend,migration,cartorio-verify}
git mv src services/rust/backend/src
git mv Cargo.toml services/rust/backend/Cargo.toml
# author services/rust/Cargo.toml workspace + the two new crate Cargo.tomls
git -c commit.gpgsign=false commit -m "cartorio: lift to lilitu-shaped workspace at services/rust/"
git push origin main
```

*Verify:* `cargo check --workspace` from `cartorio/services/rust/`
succeeds. Existing tests still green.

*Rollback:* `git revert` the commit. Restoring the flat layout is a
single move-back.

### B.2 — SeaORM entities (16 entity types)

*Why:* The persistent shape of cartorio. Per `02-ARCHITECTURE` § 3.

*Files added* (`cartorio/services/rust/backend/src/entities/`):
- `mod.rs`
- Domain (11): `publisher_enrollment.rs`, `artifact_listing.rs`,
  `certification_artifact.rs`, `signed_root.rs`, `sigstore_bundle.rs`,
  `verdict.rs`, `control_verdict.rs`, `sbom.rs`, `sbom_component.rs`,
  `compliance_run.rs`, `cross_reference_edge.rs`
- Tree-state (5): `state_leaf.rs`, `state_internal_node.rs`,
  `event_leaf.rs`, `event_internal_node.rs`, `ledger_snapshot.rs`

*Commands:*
```bash
# author each entity hand-written, mirroring lilitu's entities/ shape
# (Entity, Model, Column, Relation, ActiveModelBehavior)
cargo check -p cartorio-backend
git add cartorio/services/rust/backend/src/entities/
git -c commit.gpgsign=false commit -m "cartorio: SeaORM entities (16 types — domain + tree state)"
git push origin main
```

*Verify:* `cargo check -p cartorio-backend` clean. `cargo test -p
cartorio-backend --no-run` builds.

*Rollback:* git revert. Pre-entity code paths still work because nothing
imports the new types yet.

### B.3 — Migration crate + manifest

*Why:* Lilitu pattern: explicit migrations crate, run via shinka.

*Files added* (`cartorio/services/rust/migration/src/`):
- `lib.rs` — `Migrator::migrations()` returns the ordered list.
- `m20260505_000001_baseline.rs` — publishers, listings, certs,
  signed_roots, verdicts, control_verdicts, sbom, sbom_components.
- `m20260506_000001_event_tree.rs` — event_leaf + event_internal_node.
- `m20260506_000002_state_tree.rs` — state_leaf + state_internal_node.
- `m20260506_000003_compliance_run.rs`.
- `m20260506_000004_cross_reference.rs`.
- `m20260506_000005_ledger_snapshot.rs`.
- `migration-manifest.yaml` — classifications per `02-ARCHITECTURE` § 3.4.

*Commands:*
```bash
cargo check -p cartorio-migration
# spin up local postgres + apply migrations:
nix run .#migrate:cartorio
git add cartorio/services/rust/migration/
git -c commit.gpgsign=false commit -m "cartorio: SeaORM migrations baseline + 5 follow-ons + manifest"
git push origin main
```

*Verify:* Local Postgres has the schema (every entity's table). `psql -d
cartorio -c '\dt'` lists all expected tables. shinka's
`migration-manifest.yaml` parses (lint via shinka CLI if available, or
ruby YAML.safe_load).

*Rollback:* `nix run .#migrate:cartorio -- down` reverses migrations
(reversible: true on every M0 migration).

### B.4 — Repository layer (BaseRepository, InsertableRepository, +5 specific)

*Why:* Clean DAO/service split, lilitu pattern.

*Files added* (`cartorio/services/rust/backend/src/repository/`):
- `mod.rs`
- `traits.rs` — `BaseRepository<E>`, `InsertableRepository<E>`,
  `SoftDeletable`, `SoftDeleteExt::not_deleted()`.
- `publisher_repository.rs`
- `artifact_repository.rs`
- `compliance_run_repository.rs`
- `cross_reference_repository.rs`
- `tree_repository.rs` — encapsulates merkle reads + writes (state and
  event trees, recompute roots, persist ledger_snapshot).

*Commands:*
```bash
cargo check -p cartorio-backend
cargo test -p cartorio-backend repository::
git add cartorio/services/rust/backend/src/repository/
git -c commit.gpgsign=false commit -m "cartorio: repository layer (lilitu trait pattern + 5 specific repos)"
git push origin main
```

*Verify:* Unit tests (against testcontainers Postgres) for each repo
pass. `tree_repository`'s round-trip tests confirm:
- inserting a state leaf updates `state_root` and `ledger_root`
- appending an event leaf updates `event_root` and `ledger_root`
- `ledger_snapshot` row is persisted with all three roots

*Rollback:* git revert; previous in-memory pathway still works.

### B.5 — New REST endpoints + GraphQL surface

*Why:* M0 deliverables — the four REST endpoints and the full GraphQL
schema (Query + Mutation + Subscription).

*Files added* (`cartorio/services/rust/backend/src/api/`):
- `rest/cross_references.rs` — handlers for the two new
  `/cross-references/*` endpoints.
- update `rest/merkle.rs` — add `/merkle/consistency` and
  `/artifacts/{id}/tree-fragment` handlers.
- `merkle/consistency.rs` — log-consistency-proof math (RFC 6962-ish).
- `merkle/fragment.rs` — viz-friendly tree-fragment builder.
- `graphql/mod.rs`, `graphql/schema.rs`, `graphql/queries.rs`,
  `graphql/mutations.rs`, `graphql/subscriptions.rs`.

*Files modified:*
- `api/mod.rs` — wire new routes + graphql schema.
- `main.rs` — mount async-graphql at `/graphql` (same path serves POST
  + WebSocket upgrade per async-graphql 7).

*Commands:*
```bash
cargo check -p cartorio-backend --features graphql
cargo test -p cartorio-backend api::
nix run .#schema:cartorio  # regenerate schema.graphql
git add cartorio/services/rust/backend/src/{api,merkle}/
git -c commit.gpgsign=false commit -m "cartorio: M0 endpoints (consistency, tree-fragment, cross-refs) + GraphQL"
git push origin main
```

*Verify:*
- Local server: `nix run .#dev` brings up cartorio-backend on :8082.
- `curl http://localhost:8082/api/v1/merkle/consistency?from=...&to=...`
  returns a structured proof.
- `curl http://localhost:8082/api/v1/artifacts/<id>/tree-fragment`
  returns parents + siblings.
- GraphQL playground (if exposed in dev) shows the full schema.
- WebSocket subscription test connects and receives a `ledgerRootChanged`
  event after a manual admit POST.

*Rollback:* git revert. Existing REST endpoints unaffected.

### B.6 — `cartorio-verify` CLI

*Why:* Operator-side audit binary, runs from laptop against the live
cluster.

*Files added* (`cartorio/services/rust/cartorio-verify/src/`):
- `main.rs` — clap with four subcommands (`root`, `proof`, `consistency`,
  `audit`).
- `proof.rs` — local proof verification.
- `output.rs` — human + json output modes.

*Commands:*
```bash
cargo check -p cartorio-verify
cargo build -p cartorio-verify --release
git add cartorio/services/rust/cartorio-verify/
git -c commit.gpgsign=false commit -m "cartorio-verify: operator audit CLI (root/proof/consistency/audit)"
git push origin main
```

*Verify:* Run against local dev server:
```
./target/release/cartorio-verify root --against http://localhost:8082
./target/release/cartorio-verify audit --pinned-root <hex> --against http://localhost:8082
# both emit ✓ on a clean local stack
```

*Rollback:* git revert; binary gone.

### B.7 — Phase B integration test (e2e against testcontainers)

*Why:* Full pipeline test before deploying anything.

*Files modified:*
- `cartorio/tests/e2e_with_postgres.rs` — new integration test that
  spins up Postgres via testcontainers, applies migrations, exercises
  every endpoint including the new ones + WebSocket subscription, and
  asserts the `ledger_root` math is consistent.

*Commands:*
```bash
nix run .#test:cartorio:e2e
```

*Verify:* All endpoints respond as expected; `cartorio-verify audit`
against the in-test server emits ✓.

*Rollback:* git revert.

---

# Phase C — Cartorio-web code

> New repo `pleme-io/cartorio-web` mirrors `lilitu-web` exactly. Five
> Leptos routes, six components, hanabi-ready bundle.
>
> Can begin once **B.5** lands (the API surface is stable enough to
> code against). API client lives in `cartorio-web/.../api.rs` and uses
> the schema generated in B.5.

### C.1 — Repo creation via pangea

*Why:* New repo on GitHub, registered in pleme-io-opensource.

*Files:*
- `pleme-io/pangea-architectures/workspaces/pleme-io-opensource/org.yaml`
  — append `cartorio-web` block (visibility public, license MIT, topics
  rust + leptos + pleme-io + attestation).
- `pleme-io/repo-forge/repos.lisp` — append `(defrepo cartorio-web …)`
  with `:kind :rust-leptos-app`.

*Commands:*
```bash
cd ~/code/github/pleme-io/pangea-architectures/workspaces/pleme-io-opensource
$EDITOR org.yaml
cd ~/code/github/pleme-io/repo-forge
$EDITOR repos.lisp
cd ~/code/github/pleme-io/pangea-architectures
nix run .#plan-pleme-io-opensource    # confirm only the new repo appears in the plan
nix run .#deploy-pleme-io-opensource  # tofu apply — creates github.com/pleme-io/cartorio-web
# scaffold via repo-forge:
mkdir ~/code/github/pleme-io/cartorio-web && cd ~/code/github/pleme-io/cartorio-web
nix run github:pleme-io/repo-forge#default -- migrate . \
  --catalog ../repo-forge/repos.lisp --only cartorio-web --apply
git init -b main
git add -A
git -c commit.gpgsign=false commit -m "init cartorio-web (Leptos+Tailwind+pleme-mui shell)"
git remote add origin git@github.com:pleme-io/cartorio-web.git
git push -u origin main
```

*Verify:* Repo exists at `github.com/pleme-io/cartorio-web`. Local clone
builds via `nix build`.

*Rollback:* Two-phase per `pleme-io-github-posture` skill:
1. Set `archived: true` in `org.yaml`, `nix run .#deploy`.
2. After cooldown, remove block and re-deploy.

### C.2 — Leptos app skeleton

*Why:* Trunk + Tailwind + pleme-mui wiring, identical to lilitu-web.

*Files added:*
- `Trunk.toml` — output to `dist/`, post-build hook for tailwind.
- `tailwind.config.js`
- `styles/input.css`
- `crates/cartorio-app/Cargo.toml`
- `crates/cartorio-app/src/{main,lib,app}.rs`
- `flake.nix` with outputs `dev`, `build`, `release`.

*Commands:*
```bash
cd cartorio-web
nix run .#dev  # trunk serve on :3000
# verify the skeleton renders "Hello cartorio" or similar
git add Trunk.toml tailwind.config.js styles/ crates/ flake.nix
git -c commit.gpgsign=false commit -m "cartorio-web: Leptos skeleton (Trunk + Tailwind + pleme-mui)"
git push origin main
```

*Verify:* Local `nix run .#dev` serves on `:3000`, browser shows the
skeleton page styled by tailwind.

*Rollback:* `git reset --hard <prior>` — fresh-init repo.

### C.3 — Five routes

*Why:* Per `02-ARCHITECTURE` § 2.2.

*Files added* (`crates/cartorio-app/src/routes/`):
- `mod.rs`
- `dashboard.rs` — main view (state tree | event timeline | ledger banner)
- `artifact.rs` — `/artifact/:id`
- `tamper_demo.rs` — `/demo/tamper`
- `consistency.rs` — `/consistency`
- `verify.rs` — `/verify`

*Files modified:*
- `app.rs` — `<Router>` with all routes.

*Commands:*
```bash
nix run .#dev
# manual smoke-test each route in the browser
git add crates/cartorio-app/src/routes/ crates/cartorio-app/src/app.rs
git -c commit.gpgsign=false commit -m "cartorio-web: five routes (dashboard, artifact, tamper, consistency, verify)"
git push origin main
```

*Verify:* Each route loads without error; routing transitions are
smooth.

*Rollback:* git revert.

### C.4 — Six components

*Why:* The actual UI primitives.

*Files added* (`crates/cartorio-app/src/components/`):
- `mod.rs`
- `tree_view.rs` — d3-tree-style render
- `event_timeline.rs` — vertical scrolling event log
- `ledger_banner.rs` — current ledger_root + WS subscribe
- `proof_inspector.rs` — walks an inclusion proof
- `cross_ref_graph.rs` — joint-claim graph
- `tamper_overlay.rs` — red-flash on offending leaf

*Commands:*
```bash
git add crates/cartorio-app/src/components/
git -c commit.gpgsign=false commit -m "cartorio-web: six components (tree, timeline, banner, proof, cross-ref, tamper)"
git push origin main
```

*Verify:* Each component renders against fixture data in the relevant
route. Live updates fire via WebSocket subscription on dashboard +
artifact routes.

*Rollback:* git revert.

### C.5 — API client + WebSocket subscription

*Why:* Everything talks to `/api/v1/*` and `/graphql` (via hanabi proxy
in deploy; direct in dev).

*Files added:*
- `crates/cartorio-app/src/api.rs` — typed client over reqwest-wasm.
- `crates/cartorio-app/src/ws.rs` — graphql-ws protocol handler.

*Commands:*
```bash
git add crates/cartorio-app/src/api.rs crates/cartorio-app/src/ws.rs
git -c commit.gpgsign=false commit -m "cartorio-web: API client + WS subscription handler"
git push origin main
```

*Verify:* End-to-end against local cartorio-backend:
- Dashboard loads listings + ledger root via REST.
- Subscribe to `ledgerRootChanged` → WS connects.
- Manual admit POST → banner advances within ~100ms.

*Rollback:* git revert.

---

# Phase D — Helm charts

> All new charts in `pleme-io/helmworks/charts/<name>/`. Build OCI
> artifacts to `oci://ghcr.io/pleme-io/charts`. Existing sekiban + kensa
> get one-line FedRAMP-high overlay add.

### D.1 — `helmworks/charts/cartorio-backend/`

*Why:* Per `02-ARCHITECTURE` § 5.1.

*Files added:*
- `Chart.yaml`, `values.yaml`, `README.md`
- `templates/_helpers.tpl`, `deployment.yaml`, `service.yaml`,
  `servicemonitor.yaml`, `networkpolicy.yaml`,
  `poddisruptionbudget.yaml`, `shinka-migration.yaml`

*Commands:*
```bash
cd ~/code/github/pleme-io/helmworks
helm lint charts/cartorio-backend
helm template charts/cartorio-backend
git add charts/cartorio-backend/
git -c commit.gpgsign=false commit -m "helmworks/cartorio-backend: lilitu-shape chart with FedRAMP-high overlay"
git push origin main
# CI publishes to ghcr.io/pleme-io/charts
```

*Verify:* `helm lint` and `helm template` succeed. Chart appears at
`oci://ghcr.io/pleme-io/charts/cartorio-backend:0.1.0` after CI.

*Rollback:* git revert; OCI registry tag stays (M0 demo, no harm).

### D.2 — `helmworks/charts/cartorio-web/`

*Why:* Per `02-ARCHITECTURE` § 5.2; bundles hanabi BFF + Leptos WASM in
one Docker image.

*Files added:*
- `Chart.yaml`, `values.yaml`
- `templates/{deployment,service,ingress,configmap-hanabi,configmap-runtime,networkpolicy}.yaml`
- `templates/_helpers.tpl`

*Commands:*
```bash
helm lint charts/cartorio-web
helm template charts/cartorio-web
git add charts/cartorio-web/
git -c commit.gpgsign=false commit -m "helmworks/cartorio-web: hanabi BFF + Leptos bundle chart"
git push origin main
```

*Verify:* Same as D.1.

*Rollback:* git revert.

### D.3 — `helmworks/charts/cartorio-postgres/`

*Why:* CNPG Cluster wrapper.

*Files added:*
- `Chart.yaml`, `values.yaml`
- `templates/cluster.yaml` (CNPG `Cluster` CR)

*Commands:*
```bash
helm lint charts/cartorio-postgres
git add charts/cartorio-postgres/
git -c commit.gpgsign=false commit -m "helmworks/cartorio-postgres: CNPG Cluster wrapper"
git push origin main
```

*Verify:* `helm template` produces a valid CNPG Cluster CR.

*Rollback:* git revert.

### D.4 — `helmworks/charts/openclaw-skill-store/`

*Why:* Per `02-ARCHITECTURE` § 5.4.

*Files added:* same shape as cartorio-backend.

*Commands + Verify + Rollback:* same as D.1.

### D.5 — `helmworks/charts/openclaw-publisher-pki/`

*Why:* Per `02-ARCHITECTURE` § 5.4.

*Files added:* same shape; values include `org_seed.secretRef`.

*Commands + Verify + Rollback:* same as D.1.

### D.6 — `helmworks/charts/openclaw-scanner/`

*Why:* Per `02-ARCHITECTURE` § 5.4.

*Files added:* same shape; values include `cartorio_url`, `pki_url`,
`scan_interval_secs`.

*Commands + Verify + Rollback:* same as D.1.

### D.7 — sekiban + kensa: one-line FedRAMP-high overlay add

*Why:* Existing charts; only the values change.

*Files modified:*
- `helmworks/charts/sekiban/values.yaml` — add `compliance.overlays:
  [fedramp-high]` (or set as the chart's default if other clusters
  already opt in).
- `helmworks/charts/kensa/values.yaml` — same.

(If the existing chart values already have an overlay slot, this is
purely additive in the consuming HelmRelease's `.spec.values` rather
than the chart defaults; coordinate with `helm-compliance-overlays`
skill conventions.)

*Commands:*
```bash
git add charts/sekiban/values.yaml charts/kensa/values.yaml
git -c commit.gpgsign=false commit -m "helmworks/sekiban+kensa: opt into FedRAMP-high overlay"
git push origin main
```

*Verify:* `helm template charts/sekiban --set
compliance.overlays={fedramp-high}` emits the expected validators +
admission policies.

*Rollback:* git revert.

### D.8 — Publish OCI charts (CI handles)

*Why:* Each chart commit triggers helmworks's existing CI workflow that
publishes to `oci://ghcr.io/pleme-io/charts`.

*Verify:* `flux pull artifact oci://ghcr.io/pleme-io/charts/cartorio-backend:0.1.0
--output /tmp/test-pull` succeeds.

---

# Phase E — FluxCD wiring

> All new HelmReleases under `k8s/clusters/pleme-dev/apps/openclaw/`,
> plus the sekiban + kensa overlay-add HelmReleases under
> `infrastructure/attestation/`. FluxCD reconciles them into the cluster.

### E.1 — `cartorio-backend` HelmRelease

*Files added:*
- `k8s/clusters/pleme-dev/apps/openclaw/cartorio-backend-helmrelease.yaml`
  per `02-ARCHITECTURE` § 7.3 example.

*Commands:*
```bash
git add k8s/clusters/pleme-dev/apps/openclaw/cartorio-backend-helmrelease.yaml
git -c commit.gpgsign=false commit -m "k8s/pleme-dev: cartorio-backend HelmRelease"
git push origin main
```

*Verify:*
- `kubectl -n pleme-attestation get helmrelease cartorio-backend` →
  READY=True.
- `kubectl -n pleme-attestation get pods -l app=cartorio-backend` →
  Running.
- `kubectl -n pleme-attestation logs deploy/cartorio-backend | grep
  "listening"` → port 8082 bound.
- shinka migration init container completed with status Successful.

*Rollback:* git revert; HelmRelease + pods deleted.

### E.2 — `cartorio-web` HelmRelease + Ingress

*Files added:*
- `k8s/clusters/pleme-dev/apps/openclaw/cartorio-web-helmrelease.yaml`
- `k8s/clusters/pleme-dev/apps/openclaw/ingress.yaml`

*Commands:*
```bash
git add k8s/clusters/pleme-dev/apps/openclaw/{cartorio-web-helmrelease,ingress}.yaml
git -c commit.gpgsign=false commit -m "k8s/pleme-dev: cartorio-web HelmRelease + Ingress"
git push origin main
```

*Verify:* `curl https://openclaw.dev.use1.quero.lol/` returns the
Leptos bundle's `index.html`. Browser to that URL renders the dashboard.
Ingress TLS cert from selfsigned-issuer.

*Rollback:* git revert.

### E.3 — `openclaw-skill-store` HelmRelease

*Files added:*
- `k8s/clusters/pleme-dev/apps/openclaw/openclaw-skill-store-helmrelease.yaml`,
  with **every Pod label seekiban gates on**:
  ```yaml
  spec:
    values:
      podLabels:
        attestation.pleme.io/required: "true"
  ```
  This is what makes openclaw the workload-under-test. Cartorio +
  cartorio-web do NOT carry this label — they're the substrate.

*Commands + Verify + Rollback:* analogous to E.1, but **expect the pod
to fail to start until F.1 publishes its attestation.** This is the
test of the gate.

### E.4 — `openclaw-publisher-pki` HelmRelease

*Files added:*
- `k8s/clusters/pleme-dev/apps/openclaw/openclaw-publisher-pki-helmrelease.yaml`
- a SOPS-encrypted `Secret` named `openclaw-publisher-pki-seed`
  containing a 64-char hex random seed.

*Commands:*
```bash
SEED=$(openssl rand -hex 32)
kubectl create secret generic openclaw-publisher-pki-seed \
  --from-literal=seed-hex="$SEED" -n pleme-attestation \
  --dry-run=client -o yaml | sops -e ... > .../secret.yaml
git add ...
git -c commit.gpgsign=false commit -m "k8s/pleme-dev: openclaw-publisher-pki HelmRelease + seed"
git push origin main
```

*Verify:* `curl http://openclaw-publisher-pki.pleme-attestation:8090/health`
from inside the cluster returns ok. `/org-root` returns the signed root.

*Rollback:* git revert.

### E.5 — `openclaw-scanner` HelmRelease (10s cadence in demo mode)

*Files added:*
- `k8s/clusters/pleme-dev/apps/openclaw/openclaw-scanner-helmrelease.yaml`
  with `.spec.values.config.scan_interval_secs: 10` (override from the
  chart default of 300; values.yaml documents this is demo-mode).

*Commands + Verify:* `kubectl logs -n pleme-attestation
deploy/openclaw-scanner` shows scan ticks every 10s; corresponding
`Reattest` events appear on the cartorio event timeline.

### E.6 — sekiban + kensa overlay HelmReleases

*Files added:*
- `k8s/clusters/pleme-dev/infrastructure/attestation/sekiban-helmrelease.yaml`
  with `.spec.values.compliance.overlays: [fedramp-high]`.
- `k8s/clusters/pleme-dev/infrastructure/attestation/kensa-helmrelease.yaml`
  with the same overlay.
- update `k8s/clusters/pleme-dev/infrastructure/attestation/kustomization.yaml`.

*Commands:*
```bash
git add k8s/clusters/pleme-dev/infrastructure/attestation/
git -c commit.gpgsign=false commit -m "k8s/pleme-dev: sekiban + kensa with FedRAMP-high overlay"
git push origin main
```

*Verify:* sekiban admission webhook reachable; ValidatingWebhookConfig
present. `kubectl -n pleme-attestation get pods -l app=kensa` →
Running.

### E.7 — Phase E cluster cycle

*Why:* The whole deployment reconciles cleanly from cold sleep.

*Commands:*
```bash
PLATFORM=pleme nix run .../platform-k3s#wake
# wait for /readyz
# rebuild kubeconfig
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl get kustomizations -A
# all READY=True
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl -n pleme-attestation get pods
# every workload Running
curl -k https://openclaw.dev.use1.quero.lol/api/v1/merkle/root
# returns {state_root, event_root, ledger_root, ledger_index}
PLATFORM=pleme nix run .../platform-k3s#sleep
```

*Pass criteria:*
- All HelmReleases READY=True.
- cartorio-backend, cartorio-web, openclaw-skill-store,
  openclaw-publisher-pki, openclaw-scanner, sekiban, kensa pods Running.
- Ingress serves the dashboard over HTTPS.
- `cartorio-verify root --against https://openclaw.dev.use1.quero.lol`
  emits a fresh ledger root.

---

# Phase F — Verification

> Four named live tests against the deployed cluster. Each becomes a
> beat in the demo storyboard.

### F.0 — Pre-publish: try to deploy openclaw without attestation

*Why:* Establish the gate. **Sekiban must refuse a Pod with the
`attestation.pleme.io/required: "true"` label whose digest is not
Active in cartorio.** If this step succeeds (i.e. the deploy goes
through despite no attestation), the gate is misconfigured and
nothing else in F is meaningful.

*Setup:* The openclaw triad HelmReleases land in Phase E.3–E.5 with
the `attestation.pleme.io/required: "true"` label on every Pod they
create. Cartorio is empty of openclaw listings at this point.

*Verify:*
```
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml \
  kubectl -n pleme-attestation get events --sort-by=.lastTimestamp | tail -20
```
- Sekiban admission webhook denies the Pod create with reason
  `no Active attestation found in cartorio for digest <sha>`.
- `kubectl -n pleme-attestation get pods` shows openclaw triad pods in
  state `0/1 Pending` or `0/1 ImagePullBackOff` (lacre also denying).
- The substrate is doing its job: the unattested workload doesn't run.

*Rollback:* Not applicable; this is the expected pre-state.

### F.1 — Publish openclaw triad attestations; watch them deploy

*Why:* The headline of the demo. **If openclaw deploys, every gate in
the chain fired and passed.** That's the proof.

*Commands:*
```bash
# operator's laptop:
cd ~/code/github/pleme-io

# enroll the operator-side publisher first (one-time):
SEED_HEX="$(openssl rand -hex 32)"
curl -k -XPOST https://pki.dev.use1.quero.lol/enroll \
  -H 'content-type: application/json' \
  -d "{\"publisher_id\":\"operator@pleme.io\",
       \"public_key\":\"$(echo -n $SEED_HEX | xxd -r -p | base64)\",
       \"org\":\"pleme-io\"}"

# publish each openclaw component (image + chart). For each artifact:
#   tabeliao publish --manifest <ociManifest>
#                    --config <attestations.yaml that names
#                              fedramp_high_openclaw_image@2
#                              or fedramp_high_openclaw_helm_content@1>
#                    --cartorio https://openclaw.dev.use1.quero.lol
#                    --lacre   https://lacre.dev.use1.quero.lol  (optional)
#                    --image   ghcr.io/pleme-io/openclaw-publisher-pki
#                    --reference 0.1.0
#                    --signing-key $SEED_HEX
#                    --pack    fedramp_high_openclaw_image@2

for component in openclaw-publisher-pki openclaw-skill-store openclaw-scanner; do
  # publish the OciImage attestation
  tabeliao publish \
    --manifest    /path/to/${component}-manifest.json \
    --cartorio    https://openclaw.dev.use1.quero.lol \
    --image       ghcr.io/pleme-io/${component} \
    --reference   0.1.0 \
    --signing-key $SEED_HEX \
    --pack        fedramp_high_openclaw_image@2

  # publish the HelmChart attestation
  tabeliao publish \
    --manifest    /path/to/${component}-chart-manifest.json \
    --cartorio    https://openclaw.dev.use1.quero.lol \
    --image       oci://ghcr.io/pleme-io/charts/${component} \
    --reference   0.1.0 \
    --signing-key $SEED_HEX \
    --pack        fedramp_high_openclaw_helm_content@1
done

# Now flux + sekiban see the listings and let the pods run.
```

*Verify:*
- Each `tabeliao publish` returns 200 with a fresh `ledger_root`.
- Browser at the dashboard URL: 6 new artifact cards appear (3 images +
  3 charts), 6 cross-reference edges materialize (chart→image deploys),
  6 ComplianceRun cards appear (3 FedRAMP-image + 3 FedRAMP-helm), plus
  the openclaw triad's pods now visibly deploy:
  ```
  kubectl -n pleme-attestation get pods -l app.kubernetes.io/component=openclaw
  # → all Running within ~30s of the last attestation publish
  ```
- `cartorio-verify audit --pinned-root <root_before_demo>
  --against https://openclaw.dev.use1.quero.lol` emits ✓ over all 12
  artifacts (6 listings + 6 compliance-runs).

*Rollback:* If a publish fails, the gate that fired names the issue
(see `01-FINDINGS` § 2.2 for the 6 gates). Common causes in priority
order:
1. Publisher not enrolled — re-run the `/enroll` step.
2. Compliance pack hash mismatch — pack version drift between
   tabeliao-side and cartorio-side; align provas versions.
3. Missing migration — shinka init container; check status on
   cartorio-backend deployment.
4. NetworkPolicy blocking publisher → cartorio ingress — confirm
   pleme-charts overlay rendered allow-publish rule.

### F.2 — Tamper test — shift-left rejection (headline beat #2)

*Why:* The substrate refuses tampered artifacts deterministically.
Nothing about openclaw can be slipped past the gate.

*Commands:*
```bash
# Take the openclaw-publisher-pki image listing JSON we built in F.1,
# mutate one byte of artifact_hash, try to admit:
jq '.certification.artifact_hash = "DEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEFDEADBEEF"' \
  /tmp/openclaw-publisher-pki-listing.json > /tmp/tampered.json
curl -k -XPOST https://openclaw.dev.use1.quero.lol/api/v1/artifacts \
  -H 'content-type: application/json' \
  --data @/tmp/tampered.json
```

*Verify:*
- Response is 422 with body containing
  `"error": "AdmissionError::CertificationRoot"` (or the equivalent
  named variant).
- Body specifies `expected_root`, `recomputed_root`, and
  `offending_leaf: "artifact_hash"`.
- Browser dashboard: tamper-overlay flashes red on the offending leaf
  in the cert tree; the listing does NOT enter the state tree (no new
  card, no ledger advance).
- `cartorio-verify root` shows `ledger_root` unchanged.
- *Most importantly:* nothing in the cluster moves. The bad openclaw
  never gets a pod. Audience sees: the substrate refused — not a
  policy decision, not a runtime block, but a cryptographic
  impossibility. *That bad version of openclaw cannot run.*

*Rollback:* No state to roll back; the substrate refused the write.

### F.3 — Continuous verification (10-second cadence — the safety net)

*Why:* Once openclaw is running, the substrate keeps proving it stays
compliant. Every 10 seconds the scanner re-hashes; every successful
tick is an event the operator can point at as "the substrate is
asserting *right now* that openclaw is unchanged from what was
attested." If something drifts, the scanner catches it within 10
seconds and quarantines.

*Setup:* `openclaw-scanner` HelmRelease values include
`scan_interval_secs: 10` (overridden from the production default of
300s for demo-mode visibility — see § E.5 / `02-ARCHITECTURE` § 5.4).

*Commands (continuous tick — no operator action needed):*
```
# nothing to run; the scanner ticks autonomously every 10s.
# operator can show the heartbeat in the dashboard:
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl -n pleme-attestation \
  logs deploy/openclaw-scanner --tail=30 -f
# every 10s, lines emit: "tick: re-attested 6 artifacts; ledger_root <hex>"
```

*Verify (continuous):*
- Browser dashboard shows a `Reattest` event animating onto the timeline
  every 10 seconds; ledger root banner advances on each tick (it's
  expected to advance — every successful re-attestation is itself an
  event in the event tree, by design, so the substrate's
  history-of-having-stayed-compliant is itself attested).
- `cartorio_admissions_total{status="ok",kind="reattest"}` Prometheus
  counter incrementing.

*Commands (drift simulation — optional Q&A material):*
```
# Force a drift on one artifact:
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl -n pleme-attestation \
  exec -it deploy/cartorio-postgres-1 -- \
  psql -U cartorio -c "UPDATE certification_artifact \
    SET artifact_hash = 'BROKEN' WHERE listing_id LIKE '%openclaw-skill-store%';"
# next scanner tick (≤ 10s later) catches it.
```

*Verify (drift):*
- Scanner logs show "drift detected" within ≤ 10s.
- `curl https://openclaw.dev.use1.quero.lol/api/v1/artifacts/<id>` shows
  the listing in `Quarantined` state.
- Dashboard: the artifact's card flips green → orange; new `Quarantine`
  event animates onto timeline; ledger root advances.
- *Crucial:* sekiban's next admission for that digest will *deny*. If
  the openclaw-skill-store pod is restarted or rescheduled, it will fail
  to come back up. **The substrate has propagated the verdict to the
  runtime gate without any explicit propagation step.**

*Rollback:* Restore the row in postgres, then `POST /reactivate` (with
operator + scanner cosignatures) to clear the Quarantined state.

### F.3.1 — Demo framing for F.3

The continuous-verification beat is the *response* to the framing in
`00-OVERVIEW`'s **Regulatory framing**: every modern regime is
converging on continuous attestation, and this is what continuous
attestation looks like as a primitive. Three points to land for the
audience:

1. **Heartbeat is a signal.** The 10-second tick isn't "for the demo,
   we tuned it." It's a knob — production can run hourly, dev can run
   per-second. The substrate doesn't care; the proof structure is the
   same.
2. **Compliance is a property, not a snapshot.** Every successful tick
   re-affirms; every failed tick quarantines. The audit handle
   (`ledger_root@T`) is a *running* claim, not a ribbon-cut.
3. **Drift propagates without policy push.** If a workload drifts, the
   *next admission* for its digest denies, automatically. There is no
   "rollout the policy update" step. The state tree is the policy.

### F.4 — Audit test

*Why:* Operator-side `cartorio-verify audit` exercises every endpoint.

*Commands:*
```bash
ROOT_NOW=$(cartorio-verify root --against https://openclaw.dev.use1.quero.lol --json | jq -r .ledger_root)
cartorio-verify audit --pinned-root "$ROOT_NOW" \
  --against https://openclaw.dev.use1.quero.lol
```

*Verify:* Output includes:
- ✓ root retrieved
- ✓ consistency proof verified
- ✓ N inclusion proofs verified (one per Active artifact)
- ✓ substrate consistent

*Rollback:* No state changes.

---

# Phase G — Demo rehearsal

> Full demo run-through, time-coded against the 5-min slot. Captured as
> `07-DEMO-STORYBOARD.md` (separate doc).

### G.1 — End-to-end timed run

*Why:* Confirm the demo fits in 5 minutes with margin for narration.

*Commands:* Scripted run covering F.1 → F.2 → F.4, with the dashboard
URL open in the browser the whole time.

*Verify:* Total wall-clock from "operator runs `tabeliao publish`" to
"audit emits ✓" is < 4 minutes, leaving > 1 minute of narrative
margin in the 5-min slot.

### G.2 — Resilience checks

*Why:* Demo must survive the cluster being asleep at the start of the
talk; openclaw must come up freshly attested each rehearsal.

*Commands:*
```bash
# Right before the demo:
PLATFORM=pleme nix run .../platform-k3s#wake
# Wipe prior demo state cleanly:
KUBECONFIG=/tmp/pleme-dev-kubeconfig.yaml kubectl -n pleme-attestation \
  delete pvc -l postgres.cnpg.io/cluster=cartorio-postgres
# (cartorio-postgres re-bootstraps with empty schema; migrations re-apply.)
# Then F.0 (verify pods can't start because attestations missing)
# Then F.1 (operator publishes openclaw triad → pods come up)
# Skip F.2 in rehearsals (deterministic; just verify it works once)
# F.3's continuous tick is automatic; no manual setup
```

*Verify:* Operator can rehearse the demo cleanly each time:
- Phase F.0 always shows the gate working.
- Phase F.1 publishes openclaw fresh and watches it deploy.
- F.3 ticks visibly during the rehearsal slot (10-second cadence makes
  this near-immediate; not waiting 5 minutes).

### G.3 — Capture artifacts

*Why:* If the live demo fails, fall back to a captured run.

*Commands:* Record a screencast of the rehearsal (any tool).

*Output:* a `.mp4` (kept locally; not committed to git).

---

## What `04` through `09` will land

Per `00-OVERVIEW.md`'s document map:

- **`04-API-SURFACE.md`** — exact request/response JSON schemas for every
  cartorio-backend endpoint (REST + GraphQL), plus the WebSocket
  subscription wire format. Used as a contract for cartorio-web.
- **`05-VISUAL-PLAN.md`** — wireframes / sketches for each cartorio-web
  route + component. Defines what each visual primitive shows and how it
  reacts to live updates.
- **`06-DEMO-NARRATIVE.md`** — the supporting narrative (operator's call
  on the actual delivery). Ours is the substrate-side story arc the
  technical demo will speak to.
- **`07-DEMO-STORYBOARD.md`** — the time-coded run-of-show. Captures
  what the operator says, types, and clicks at each of the demo's
  beats.
- **`08-DECISION-LOG.md`** — open questions, resolutions, dates. Updated
  as decisions are made through `03`–`07`.
- **`09-RISK-REGISTER.md`** — what could go wrong (cluster cold-start
  fail; DNS lag; postgres data loss; API drift; audience can't reach
  Ingress); mitigations; recovery paths.

The execution order is `03` → land everything in Phases A–G → confirm
verification → write `07` based on the rehearsed run.

`04`–`09` ready to write next, or do you want to push back / refine `03`
first?
