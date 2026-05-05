# 02 — Architecture

> Target end-state for the openclaw / tameshi / cartorio demo on `pleme-dev`.
> Locks the four decisions from `01-FINDINGS.md` § 10 into precise file
> paths, entity definitions, API endpoints, helm chart shapes, and the
> FluxCD reconcile layout. `03-PHASES.md` turns this into a linear
> execution checklist.

## 0. The architecture in one sentence

A double-merkle-tree-backed attestation substrate (`cartorio-backend`,
Axum + SeaORM, port 8082) serves a single signed posture over a
CloudNative-PG cluster (`cartorio-postgres`) to four enforcement gates
(`lacre`, `sekiban`, `openclaw-scanner`, `cartorio-verify`) and one
Leptos+Hanabi visualization (`cartorio-web`), all deployed in the
`pleme-attestation` namespace on `pleme-dev` K3s, FluxCD-reconciled from
`pleme-io/k8s.git@main`, exposed at `openclaw.dev.use1.quero.lol`.

## 1. Layer model

```
─── operator workstation ──────────────────────────────────────────
   tabeliao publish ─┐                          ┌─ cartorio-verify
   lacre proxy   ────┤                          ├─ kensa CLI
   pangea-publish ───┤                          │
                     │                          │
                     ▼                          ▲
─── pleme-dev K3s — pleme-attestation ns ─────────────────────────
       Ingress  openclaw.dev.use1.quero.lol  (selfsigned-issuer TLS)
                              │
                              ▼
                   cartorio-web (Pod)
                   ├ hanabi BFF :80
                   └ Leptos WASM bundle (CSR)
                              │
                              ▼ HTTP + GraphQL + WS
                   cartorio-backend (Pod, :8082, :9090 metrics)
                   ├ Axum router
                   ├ SeaORM repository layer
                   ├ async-graphql 7
                   └ merkle math + 6-gate admission chain
                              │
                              ▼ pgwire
                   cartorio-postgres (CNPG Cluster, single instance)

       openclaw-skill-store    (Pod, :8080)
       openclaw-publisher-pki  (Pod, :8090)
       openclaw-scanner        (Pod, :9090 metrics; daemon timer 300s)
       sekiban                 (admission webhook + 3 CRDs)
       kensa                   (Pod, REST + GraphQL)
─── pleme-dev K3s — flux-system ns ────────────────────────────────
       FluxCD source/kustomize/helm/notification controllers
       HelmRepository pleme-charts → oci://ghcr.io/pleme-io/charts
       GitRepository pleme-io-k8s   → pleme-io/k8s.git@main
─── pleme-dev K3s — cert-manager-system ns ────────────────────────
       cert-manager + selfsigned-issuer (sekiban webhook TLS)
─── pleme-dev K3s — cnpg-system ns ────────────────────────────────
       CloudNative-PG operator (provides cartorio-postgres Cluster)
─── pleme-dev K3s — shinka-system ns ──────────────────────────────
       shinka migration operator (runs Migration CRDs on deploy)
```

## 2. Repository + file layout

### 2.1 `pleme-io/cartorio` — the monorepo (mirror lilitu wholesale)

```
cartorio/
├── README.md
├── CLAUDE.md
├── flake.nix                         # nix run .#dev|.#schema:cartorio|.#migrate:cartorio|.#test:cartorio
├── docs/
│   ├── ARCHITECTURE.md               # (existing — dual-tree merkle ledger design)
│   ├── COMPLIANCE-PROOF.md           # (existing — proof chain flow)
│   ├── RUNBOOK.md                    # (existing — live curl commands)
│   └── ENTITIES.md                   # NEW — SeaORM entity ref (per § 3)
└── services/rust/
    ├── Cargo.toml                    # workspace declaration; pin shared versions
    ├── backend/                      # Axum HTTP + GraphQL API server
    │   ├── Cargo.toml
    │   └── src/
    │       ├── main.rs               # bootstrap: db pool, migrations wait, axum router, shutdown
    │       ├── lib.rs
    │       ├── config.rs             # env-driven config (DATABASE_URL, ORG, PORT, METRICS_PORT, …)
    │       ├── entities/             # SeaORM types (per § 3); hand-authored, not generated
    │       │   ├── mod.rs
    │       │   ├── publisher_enrollment.rs
    │       │   ├── artifact_listing.rs
    │       │   ├── certification_artifact.rs
    │       │   ├── signed_root.rs
    │       │   ├── sigstore_bundle.rs
    │       │   ├── verdict.rs
    │       │   ├── control_verdict.rs
    │       │   ├── sbom.rs
    │       │   ├── sbom_component.rs
    │       │   ├── compliance_run.rs
    │       │   ├── cross_reference_edge.rs
    │       │   ├── state_leaf.rs
    │       │   ├── state_internal_node.rs
    │       │   ├── event_leaf.rs
    │       │   ├── event_internal_node.rs
    │       │   └── ledger_snapshot.rs
    │       ├── repository/           # repository pattern (lilitu shape)
    │       │   ├── mod.rs
    │       │   ├── traits.rs         # BaseRepository<E>, InsertableRepository<E>, SoftDeletable
    │       │   ├── publisher_repository.rs
    │       │   ├── artifact_repository.rs
    │       │   ├── compliance_run_repository.rs
    │       │   ├── cross_reference_repository.rs
    │       │   └── tree_repository.rs   # merkle reads + writes (state + event tree)
    │       ├── merkle/
    │       │   ├── mod.rs
    │       │   ├── hash.rs           # blake3 + domain separators
    │       │   ├── compose.rs        # 3-leaf cert composition + tree node hashing
    │       │   ├── inclusion.rs      # state/event inclusion proofs
    │       │   ├── consistency.rs    # NEW — log consistency proofs (root_n ⊑ root_n+1)
    │       │   └── fragment.rs       # NEW — viz-friendly tree-fragment builder
    │       ├── api/
    │       │   ├── mod.rs            # router composition
    │       │   ├── rest/             # 23 existing endpoints + 4 M0 additions
    │       │   │   ├── mod.rs
    │       │   │   ├── artifacts.rs
    │       │   │   ├── events.rs
    │       │   │   ├── merkle.rs
    │       │   │   ├── cross_references.rs   # NEW
    │       │   │   └── health.rs
    │       │   └── graphql/          # mirrors lilitu's graphql/ shape
    │       │       ├── mod.rs
    │       │       ├── schema.rs     # Query + Mutation + Subscription roots
    │       │       ├── queries.rs
    │       │       ├── mutations.rs
    │       │       └── subscriptions.rs   # WS push: ledger_root, event_added
    │       ├── auth/                 # x-user-* header middleware (lilitu pattern; demo: open)
    │       └── observability/        # /metrics + /health + /health/ready
    ├── migration/                    # SeaORM migrations crate (lilitu pattern)
    │   ├── Cargo.toml
    │   └── src/
    │       ├── lib.rs                # Migrator::migrations()
    │       ├── m20260505_000001_baseline.rs        # publishers, listings, certs, signed_roots
    │       ├── m20260506_000001_event_tree.rs
    │       ├── m20260506_000002_state_tree.rs
    │       ├── m20260506_000003_compliance_run.rs
    │       ├── m20260506_000004_cross_reference.rs
    │       ├── m20260506_000005_ledger_snapshot.rs
    │       └── migration-manifest.yaml             # safety classification per migration
    └── cartorio-verify/              # operator audit CLI
        ├── Cargo.toml
        └── src/
            ├── main.rs               # clap subcommands
            ├── proof.rs              # walk inclusion + consistency proofs locally
            └── output.rs             # human (colored) + json output modes
```

### 2.2 `pleme-io/cartorio-web` — separate repo (mirror lilitu-web)

```
cartorio-web/
├── README.md
├── flake.nix                         # nix run .#dev (trunk serve) | .#build (trunk build) | .#release
├── Trunk.toml                        # Leptos CSR build config
├── tailwind.config.js
├── styles/
│   └── input.css                     # tailwind directives + pleme-mui consumption
└── crates/cartorio-app/
    ├── Cargo.toml                    # leptos 0.7, leptos_router, leptos_meta, web-sys, pleme-mui
    └── src/
        ├── main.rs                   # leptos::mount_to_body
        ├── lib.rs
        ├── app.rs                    # <Router> with all routes
        ├── routes/
        │   ├── mod.rs
        │   ├── dashboard.rs          # state tree | event timeline | ledger banner
        │   ├── artifact.rs           # /artifact/:id — timeline + inclusion proof
        │   ├── tamper_demo.rs        # /demo/tamper — live mutate-and-fail
        │   ├── consistency.rs        # /consistency — root@N → root@N+1 viewer
        │   └── verify.rs             # /verify — operator-side verify panel
        ├── components/
        │   ├── tree_view.rs          # d3-tree-style render of merkle tree fragment
        │   ├── event_timeline.rs     # vertical scrolling event log w/ live additions
        │   ├── ledger_banner.rs      # current ledger_root display + WS subscribe
        │   ├── proof_inspector.rs    # walks an inclusion proof step by step
        │   ├── cross_ref_graph.rs    # cross-reference edges (joint claim viz)
        │   └── tamper_overlay.rs     # red-flash on offending leaf
        ├── api.rs                    # client to cartorio-backend (via hanabi)
        └── ws.rs                     # WebSocket subscription handler (graphql-ws protocol)
```

### 2.3 `pleme-io/helmworks/charts/` — new charts

```
helmworks/charts/
├── pleme-lib/                  v0.13+   (existing dependency)
├── pleme-compliance/                    (existing — namespace PSS, default-deny, quotas)
├── lareira-cartorio/           v0.2.0   (existing — kept as wrapper; cartorio-backend is the new
│                                          first-class chart for M0)
├── cartorio-backend/           NEW      lilitu-backend shape
├── cartorio-web/               NEW      lilitu-web shape (hanabi BFF + Leptos WASM)
├── cartorio-postgres/          NEW      CNPG Cluster wrapper
├── openclaw-skill-store/       NEW
├── openclaw-publisher-pki/     NEW
├── openclaw-scanner/           NEW
└── (sekiban/, kensa/, kanshi/, tameshi-watch/ already exist; M0 only adds
   `compliance.overlays: [fedramp-high]` to their values)
```

### 2.4 `pleme-io/k8s/clusters/pleme-dev/` — FluxCD reconcile target

```
k8s/clusters/pleme-dev/
├── kustomization.yaml                  # references infrastructure/ + apps/
├── flux-system/
│   ├── gotk-components.yaml            # FluxCD CRDs (one-time bootstrap, captured here for source-of-truth)
│   ├── gotk-sync.yaml                  # GitRepository → pleme-io/k8s.git@main + Kustomization root
│   └── kustomization.yaml
├── infrastructure/
│   ├── kustomization.yaml
│   ├── cert-manager/
│   │   ├── helmrelease.yaml            # cert-manager v1.16
│   │   ├── cluster-issuer.yaml         # selfsigned-issuer
│   │   └── kustomization.yaml
│   ├── pleme-charts/
│   │   ├── helmrepository.yaml         # oci://ghcr.io/pleme-io/charts
│   │   ├── ghcr-charts-secret.yaml     # SOPS-encrypted GHCR pull token
│   │   └── kustomization.yaml
│   ├── cnpg-operator/
│   │   ├── helmrelease.yaml            # cloudnative-pg operator
│   │   └── kustomization.yaml
│   ├── shinka/
│   │   ├── helmrelease.yaml            # shinka migration operator
│   │   └── kustomization.yaml
│   └── attestation/
│       ├── kustomization.yaml
│       ├── namespace.yaml              # pleme-attestation, PSS=restricted
│       ├── network-policy-default-deny.yaml
│       ├── sekiban-helmrelease.yaml    # adds compliance.overlays: [fedramp-high]
│       └── kensa-helmrelease.yaml
└── apps/
    ├── kustomization.yaml
    ├── hello-world/                    # (existing — kept as smoke test for cluster reconcile)
    └── openclaw/
        ├── kustomization.yaml
        ├── cartorio-postgres-cluster.yaml      # CNPG Cluster CR (non-helm; CR direct)
        ├── cartorio-backend-helmrelease.yaml
        ├── cartorio-web-helmrelease.yaml
        ├── openclaw-skill-store-helmrelease.yaml
        ├── openclaw-publisher-pki-helmrelease.yaml
        ├── openclaw-scanner-helmrelease.yaml
        └── ingress.yaml                # openclaw.dev.use1.quero.lol → cartorio-web:80
```

## 3. SeaORM entity set

All types live under `cartorio/services/rust/backend/src/entities/`,
hand-authored per lilitu pattern (not `sea-orm-cli generate`-d). Each
file declares one `Entity`, its `Model`, `Column` enum, `Relation` enum,
and `ActiveModelBehavior` impl.

### 3.1 Domain entities (the *what*)

| Entity | Key fields | Purpose |
|---|---|---|
| `publisher_enrollment` | `publisher_id` (PK str), `public_key` (bytea), `enrollment_chain` (jsonb), `enrolled_root` (str), `enrolled_at` (timestamp), `revoked_at` (nullable timestamp) | Who's allowed to publish; cert chain to org-root. |
| `artifact_listing` | `listing_id` (PK str, e.g. `alice@pleme.io/hello-rio@1.0.0`), `kind` (enum), `publisher_id` (FK), `name`, `version`, `listing_state` (enum: Active/Revoked/Quarantined), `state_metadata` (jsonb: revocation cert / quarantine entry), `created_at`, `updated_at` | The current `ListingState<K>` for an artifact. |
| `certification_artifact` | `listing_id` (FK,PK), `artifact_hash` (str), `control_hash` (str), `intent_hash` (str), `composed_root` (str) | The 3-leaf cert. |
| `signed_root` | `listing_id` (FK,PK), `root` (str), `signature` (bytea), `algorithm` (enum), `signer_id` (str), `signed_at` (timestamp) | Publisher's signature over `composed_root`. |
| `sigstore_bundle` | `listing_id` (FK,PK), `fulcio_cert_hash` (str), `rekor_entry_id` (str), `rekor_log_index` (i64), `bundle_hash` (str) | Sigstore commitment. Offline mode for demo. |
| `verdict` | `verdict_id` (PK uuid), `listing_id` (FK), `framework` (enum), `summary_hash` (str) | One per (listing, framework). |
| `control_verdict` | `id` (PK uuid), `verdict_id` (FK), `control_id` (str), `status` (enum: Passed/PassedWithWaiver/Failed), `evidence_hash` (str) | Per-control row inside a verdict. |
| `sbom` | `sbom_id` (PK uuid), `listing_id` (FK,unique), `sbom_hash` (str) | One SBOM per listing. |
| `sbom_component` | `id` (PK), `sbom_id` (FK), `name` (str), `version` (str), `hash` (str) | Individual deps. Sorted at insert. |
| `compliance_run` | `run_id` (PK uuid), `tested_artifact_listing_id` (FK), `pack_hash` (str), `run_hash` (str), `run_status` (enum), `executed_at` (timestamp), `run_data` (jsonb) | Result of a provas pack run. **Itself an attested testing artifact.** |
| `cross_reference_edge` | `id` (PK uuid), `source_listing_id` (FK), `target_listing_id` (FK), `reference_type` (enum: Embeds/Tests/HasSbom/Deploys/etc.), `source_field_path` (str), `recorded_at` (timestamp) | The joint-claim graph; explicit edges between leaves. |

### 3.2 Tree-state entities (the *how it's hashed*)

| Entity | Key fields | Purpose |
|---|---|---|
| `state_leaf` | `leaf_id` (PK uuid), `listing_id` (FK,unique), `leaf_hash` (str), `created_at_ledger_index` (i64), `last_updated_ledger_index` (i64) | Materialized state-tree leaf. One per listing_id. |
| `state_internal_node` | `id` (PK uuid), `parent_id` (FK self, nullable), `depth` (i32), `left_child_hash` (str, nullable), `right_child_hash` (str, nullable), `node_hash` (str), `ledger_index` (i64) | Internal nodes. Indexed by `ledger_index` for snapshot reconstruction. |
| `event_leaf` | `event_id` (PK uuid), `event_index` (i64, monotonic, unique), `listing_id` (FK), `kind` (enum), `prior_state_hash` (str, nullable), `posterior_state_hash` (str), `cosigners` (jsonb), `event_data` (jsonb), `leaf_hash` (str), `created_at` (timestamp) | Append-only events. |
| `event_internal_node` | same shape as `state_internal_node` | Append-only event tree internal nodes. |
| `ledger_snapshot` | `ledger_index` (PK i64), `state_root` (str), `event_root` (str), `ledger_root` (str), `signed_by` (str, nullable), `signed_at` (timestamp) | Per-event snapshot of all three roots. The substrate's audit trail. |

### 3.3 Indexes

| Index | Purpose |
|---|---|
| `idx_artifact_listing_publisher` (publisher_id) | "all artifacts by alice@pleme.io" |
| `idx_certification_artifact_hash` (artifact_hash) | lacre's `lookup_by_digest` (the registry-side gate) |
| `idx_compliance_run_tested` (tested_artifact_listing_id) | "all runs against artifact X" |
| `idx_compliance_run_pack` (pack_hash) | "every run that used pack v2" |
| `idx_event_leaf_listing_index` (listing_id, event_index) | per-artifact history walk |
| `idx_event_leaf_index` (event_index) | range-walk for consistency proofs |
| `idx_ledger_snapshot_index` (ledger_index) | snapshot retrieval by index |
| `idx_cross_ref_source` (source_listing_id) | "what does X reference?" |
| `idx_cross_ref_target` (target_listing_id) | "what references Y?" |
| `idx_state_leaf_listing` (listing_id) | UNIQUE — at most one state-leaf per listing |

### 3.4 Migration manifest (lilitu's safety-gate pattern)

`cartorio/services/rust/migration/src/migration-manifest.yaml`:

```yaml
migrations:
  - id: m20260505_000001_baseline
    classification: schema_only
    reversible: true
  - id: m20260506_000001_event_tree
    classification: schema_only
    reversible: true
  - id: m20260506_000002_state_tree
    classification: schema_only
    reversible: true
  - id: m20260506_000003_compliance_run
    classification: schema_only
    reversible: true
  - id: m20260506_000004_cross_reference
    classification: schema_only
    reversible: true
  - id: m20260506_000005_ledger_snapshot
    classification: schema_only
    reversible: true
```

shinka enforces classification: `data_only` and `schema_and_data` require
explicit forward/backward companions. Initial baseline is pure schema —
no data migrations until M1+.

## 4. cartorio-backend API surface

### 4.1 REST (port 8082, prefix `/api/v1/`)

| Method | Path | M0 status | Purpose |
|---|---|---|---|
| GET | `/health` | ✓ existing | liveness |
| GET | `/health/ready` | + extend | readiness (db check, lilitu pattern) |
| GET | `/metrics` | ✓ existing | Prometheus on port 9090 |
| POST | `/api/v1/artifacts` | ✓ existing | admit (the 6-gate chain — see § 4.4 below) |
| GET | `/api/v1/artifacts` | ✓ existing | list, paginated |
| GET | `/api/v1/artifacts/{id}` | ✓ existing | fetch ListingState |
| GET | `/api/v1/artifacts/{id}/verify` | ✓ existing | re-verify (recompute) |
| GET | `/api/v1/artifacts/{id}/history` | ✓ existing | event chain for one artifact |
| GET | `/api/v1/artifacts/{id}/compliance-runs` | ✓ existing | runs against this artifact |
| POST | `/api/v1/artifacts/{id}/revoke` | ✓ existing | publisher revokes |
| POST | `/api/v1/artifacts/{id}/quarantine` | ✓ existing | scanner quarantines |
| POST | `/api/v1/artifacts/{id}/reactivate` | ✓ existing | operator + scanner cosign reactivate |
| POST | `/api/v1/artifacts/{id}/reattest` | ✓ existing | scanner re-attests |
| POST | `/api/v1/artifacts/{id}/supersede` | ✓ existing | publisher supersedes |
| GET | `/api/v1/artifacts/by-digest/{digest}` | ✓ existing | lacre's lookup |
| GET | `/api/v1/events` | ✓ existing | raw event log, paginated |
| GET | `/api/v1/events/{event_id}` | ✓ existing | one event |
| GET | `/api/v1/artifacts/{id}/proof` | ✓ existing | inclusion proof in state tree |
| GET | `/api/v1/events/{event_id}/proof` | ✓ existing | inclusion proof in event tree |
| GET | `/api/v1/merkle/root` | ✓ existing | `{state_root, event_root, ledger_root, ledger_index}` |
| GET | `/api/v1/merkle/consistency?from=<r1>&to=<r2>` | **NEW M0** | range proof root@r2 ⊑ root@r1 |
| GET | `/api/v1/artifacts/{id}/tree-fragment` | **NEW M0** | parents + siblings ready for d3-tree rendering |
| GET | `/api/v1/cross-references/source/{listing_id}` | **NEW M0** | outbound edges (joint-claim graph) |
| GET | `/api/v1/cross-references/target/{listing_id}` | **NEW M0** | inbound edges |

### 4.2 GraphQL (port 8082, `/graphql`, async-graphql 7)

Mirrors lilitu's `src/api/graphql/` shape: Schema with Query, Mutation,
Subscription roots; auth via `.data(authz)` injection from middleware.

**Queries:**

```graphql
type Query {
  listing(id: String!): ListingState
  listings(filter: ListingFilter, page: PageInput): ListingPage!
  event(id: String!): Event
  events(filter: EventFilter, page: PageInput): EventPage!
  merkleRoot: MerkleRoot!
  consistencyProof(fromRoot: String!, toRoot: String!): ConsistencyProof!
  inclusionProof(listingId: String!): InclusionProof!
  treeFragment(listingId: String!): TreeFragment!
  crossReferences(listingId: String!): CrossReferences!
  complianceRun(id: String!): ComplianceRun
  complianceRunsForListing(listingId: String!): [ComplianceRun!]!
}
```

**Mutations:** mirror REST mutating endpoints — `admit(input)`, `revoke`,
`quarantine`, `reactivate`, `reattest`, `supersede`. GraphQL gives
cartorio-web the unified surface; REST stays for tabeliao/lacre/scanner.

**Subscriptions** (WebSocket only, graphql-ws protocol):

```graphql
type Subscription {
  ledgerRootChanged: MerkleRoot!
  eventAdded(filter: EventFilter): Event!
  listingStateChanged(listingId: String): ListingState!
}
```

These three drive cartorio-web's live updates: the ledger banner refreshes
on every `ledgerRootChanged`, the event timeline appends on every
`eventAdded`, and per-artifact views refresh on `listingStateChanged`.

### 4.3 cartorio-verify CLI

Binary at `cartorio/services/rust/cartorio-verify/src/main.rs`. clap
subcommands:

```
cartorio-verify root \
    --against https://openclaw.dev.use1.quero.lol
    # outputs current ledger_root + state_root + event_root + ledger_index

cartorio-verify proof <listing_id> \
    --against https://openclaw.dev.use1.quero.lol \
    [--json]
    # walks an inclusion proof locally; emits ✓ or ✗

cartorio-verify consistency \
    --pinned-root <hex> \
    --against https://openclaw.dev.use1.quero.lol \
    [--json]
    # GET /merkle/root → current
    # GET /merkle/consistency?from=<pinned>&to=<current>
    # verify range proof locally
    # emits ✓ consistent OR ✗ break-at-event-id

cartorio-verify audit \
    --pinned-root <hex> \
    --against https://openclaw.dev.use1.quero.lol \
    [--json]
    # consistency + every Active artifact's inclusion proof
    # the operator's "audit the whole substrate" command
```

Output modes: human (default, colored, tables) and `--json` (for piping
into other tooling). No deps on cartorio-backend's repository layer; pure
HTTP client + the merkle math copied from `cartorio/backend/src/merkle/`.

### 4.4 The 6-gate admission chain (recap from `01-FINDINGS` § 2)

`POST /api/v1/artifacts` runs these in order, single Postgres transaction:

```
incoming AdmitArtifactInput
   │
   ▼
Gate 1  verify_publisher        → 401  PublisherUnknown / Unenrolled / Revoked
Gate 2  verify_certification    → 422  CertificationRoot      ◀── THE TAMPER GATE
Gate 3  verify_signature        → 422  SignatureInvalid
Gate 4  recompute control_hash  → 422  ComplianceMismatch
Gate 5  framework coverage      → 422  InsufficientControls / FailedVerdict
Gate 6  listing collision       → 409  ListingExists
   │
   ▼  (all pass)
TX:
  - upsert state-leaf
  - append event-leaf (Admit)
  - recompute path nodes; commit new state_root + event_root + ledger_root
  - persist ledger_snapshot row
  - commit
   │
   ▼
200 OK + {listing_id, new_ledger_root, inclusion_proof}
```

Each error variant is a typed `AdmissionError` enum case, not a string.
Same chain runs for `revoke`, `quarantine`, `reactivate`, `reattest`,
`supersede` with the appropriate per-action gates.

## 5. Helm chart structure

All charts live in `pleme-io/helmworks/charts/<name>/`, depend on
`pleme-lib v0.13+`, build OCI artifacts at `oci://ghcr.io/pleme-io/charts`.

### 5.1 `cartorio-backend/`

Mirrors `lilitu-backend`:

```
cartorio-backend/
├── Chart.yaml                          # name, version 0.1.0, dep on pleme-lib v0.13+
├── values.yaml                         # ~250 lines
├── templates/
│   ├── deployment.yaml                 # uses pleme-lib._workload.tpl
│   ├── service.yaml                    # api 8082 + metrics 9090
│   ├── servicemonitor.yaml             # Prometheus scrape on /metrics
│   ├── networkpolicy.yaml              # default-deny + per-direction allow
│   ├── poddisruptionbudget.yaml
│   ├── shinka-migration.yaml           # Migration CRD; init container waits
│   └── _helpers.tpl
└── README.md
```

`values.yaml` essentials:

```yaml
image:
  repository: ghcr.io/pleme-io/cartorio-backend
  tag: ""                        # default → chart appVersion
replicaCount: 1
ports:
  api: 8082
  metrics: 9090

postgres:
  cnpgClusterRef: cartorio-postgres
  databaseName: cartorio
  secretRef: cartorio-postgres-connection

shinkaMigration:
  enabled: true
  type: seaorm
  image:
    repository: ghcr.io/pleme-io/cartorio-migration
  timeoutSecs: 300

probes:
  liveness:  { path: /health,       port: api }
  readiness: { path: /health/ready, port: api }

compliance:
  overlays: [fedramp-high]       # ← FedRAMP-High overlay opt-in (per helm-compliance-overlays)

networkPolicy:
  enabled: true

podDisruptionBudget:
  minAvailable: 0                # single-replica demo
```

### 5.2 `cartorio-web/`

Mirrors `lilitu-web`. Single image bundles hanabi BFF + Leptos WASM bundle
in one container, one HelmRelease, one Ingress.

```
cartorio-web/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml             # one container; hanabi serves :80, both static + proxy
    ├── service.yaml
    ├── ingress.yaml                # openclaw.dev.use1.quero.lol → cartorio-web:80
    ├── configmap-hanabi.yaml       # hanabi config (per § 6)
    ├── configmap-runtime.yaml      # frontend runtime config served at /_runtime.json
    ├── networkpolicy.yaml
    └── _helpers.tpl
```

`values.yaml` essentials:

```yaml
image:
  repository: ghcr.io/pleme-io/cartorio-web
  tag: ""
replicaCount: 1
ports:
  http: 80
  metrics: 9090

hanabi:
  backend_url: http://cartorio-backend:8082
  graphql_path: /graphql
  ws_path: /graphql
  static_root: /var/www/cartorio-app

ingress:
  enabled: true
  className: traefik             # K3s default
  host: openclaw.dev.use1.quero.lol
  tls:
    enabled: true
    issuerRef:
      name: selfsigned-issuer
      kind: ClusterIssuer

compliance:
  overlays: [fedramp-high]
```

### 5.3 `cartorio-postgres/`

Wraps a CNPG `Cluster` CR. Direct CR (not Helm) is also acceptable; we
choose chart wrapping for value override consistency.

```yaml
clusterName: cartorio-postgres
instances: 1                       # single-instance for demo
storage:
  size: 10Gi
  storageClass: local-path         # K3s default
backup:
  enabled: false                   # M0 demo; production later
bootstrap:
  initdb:
    database: cartorio
    owner: cartorio
    secret:
      name: cartorio-postgres-connection
```

The `cartorio-postgres-connection` Secret is created by CNPG's bootstrap
process and referenced by `cartorio-backend.values.postgres.secretRef`.

### 5.4 `openclaw-skill-store/`, `openclaw-publisher-pki/`, `openclaw-scanner/`

Each follows the same `pleme-microservice` shape (deployment + service +
servicemonitor + networkpolicy + pdb + FedRAMP-high overlay). Per-chart
specifics:

`openclaw-publisher-pki/values.yaml`:

```yaml
config:
  org: pleme-io
  org_seed:
    secretRef:
      name: openclaw-publisher-pki-seed
      key: seed-hex
ports:
  api: 8090
  metrics: 9090
compliance:
  overlays: [fedramp-high]
```

`openclaw-scanner/values.yaml`:

```yaml
config:
  cartorio_url: http://cartorio-backend:8082
  pki_url: http://openclaw-publisher-pki:8090
  scan_interval_secs: 300
ports:
  metrics: 9090
compliance:
  overlays: [fedramp-high]
```

`openclaw-skill-store/values.yaml`:

```yaml
config:
  cartorio_url: http://cartorio-backend:8082
ports:
  api: 8080
  metrics: 9090
postgres:
  cnpgClusterRef: cartorio-postgres
  databaseName: skill_store
compliance:
  overlays: [fedramp-high]
```

### 5.5 sekiban + kensa charts (existing — add overlay)

For both charts, the M0 change is one line in the HelmRelease values:

```yaml
spec:
  values:
    compliance:
      overlays: [fedramp-high]
```

(per `helmworks/CLAUDE.md` and `pleme-io/CLAUDE.md` ★★ Caixa section, the
overlay registry handles closure expansion + admission policy emission;
the chart itself doesn't need other changes for FedRAMP-High enrollment.)

## 6. Hanabi configuration for openclaw

`cartorio-web/templates/configmap-hanabi.yaml` mirrors lilitu-web's
`templates/configmap.yaml` shape:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cartorio-web-hanabi
data:
  hanabi.yaml: |
    server:
      bind: 0.0.0.0:80
    static:
      root: /var/www/cartorio-app
      index: index.html
      spa_fallback: true              # Leptos client-side router
    upstream:
      cartorio:
        url: http://cartorio-backend:8082
        rest_prefix: /api/v1
        graphql_path: /graphql
        graphql_ws_path: /graphql     # async-graphql serves both POST + WS upgrade on same path
    routes:
      - match: /api/*
        forward: cartorio
      - match: /graphql
        forward: cartorio
      - match: /_runtime.json
        serve: configmap
      - match: /
        serve: static
    auth:
      mode: open                      # demo only; pleme-io prod uses cracha SSO
    headers:
      inject_user_headers: false      # auth is open; no x-user-* injection
```

> **Note:** hanabi's exact config schema is what lilitu-web ships today;
> `03-PHASES` will reconcile this against the actual `lilitu-web/charts/`
> ConfigMap to make sure key names match. The shape above is the lilitu
> pattern adapted to cartorio's routes.

### 6.1 Frontend runtime config

`cartorio-web/templates/configmap-runtime.yaml` exposes a tiny JSON
config the WASM bundle reads at startup (avoids hardcoding URLs in WASM):

```json
{
  "graphql_endpoint": "/graphql",
  "graphql_ws_endpoint": "/graphql",
  "rest_endpoint": "/api/v1",
  "org": "pleme-io",
  "demo_seed_listings": ["alice@pleme.io/hello-rio@1.0.0",
                         "openclaw@v2",
                         "lareira-openclaw"]
}
```

## 7. FluxCD layout

### 7.1 Reconcile chain

```
GitRepository pleme-io-k8s          (in flux-system ns)
   ↓
Kustomization flux-system           (self-managed)
   ↓
Kustomization infrastructure        (cluster-wide prerequisites)
   ↓ dependsOn
   ├ cert-manager
   ├ pleme-charts (HelmRepository source)
   ├ cnpg-operator
   ├ shinka
   └ attestation (sekiban + kensa)
   ↓
Kustomization apps                  (workloads)
   ↓ dependsOn = [infrastructure]
   ├ hello-world  (existing)
   └ openclaw     (cartorio + openclaw triad)
       ↓
       ├ cartorio-postgres-cluster  (CNPG CR)
       ├ cartorio-backend           (depends on cartorio-postgres + pleme-charts)
       ├ cartorio-web               (depends on cartorio-backend)
       ├ openclaw-skill-store       (depends on cartorio-postgres)
       ├ openclaw-publisher-pki     (independent)
       ├ openclaw-scanner           (depends on cartorio-backend + openclaw-publisher-pki)
       └ ingress
```

### 7.2 OCI HelmRepository

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: pleme-charts
  namespace: flux-system
spec:
  type: oci
  url: oci://ghcr.io/pleme-io/charts
  secretRef:
    name: ghcr-charts-secret
  interval: 5m
```

### 7.3 Per-app HelmRelease shape (cartorio-backend example)

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: cartorio-backend
  namespace: pleme-attestation
spec:
  interval: 5m
  chart:
    spec:
      chart: cartorio-backend
      sourceRef:
        kind: HelmRepository
        name: pleme-charts
        namespace: flux-system
      version: ">=0.1.0 <0.2.0"
  values:
    compliance:
      overlays: [fedramp-high]
    postgres:
      cnpgClusterRef: cartorio-postgres
  dependsOn:
    - name: cartorio-postgres
    - name: cnpg-operator
      namespace: cnpg-system
```

## 8. Networking, observability, security

### 8.1 NetworkPolicy

Default-deny in `pleme-attestation` namespace + per-pod explicit allows:

| From | To | Port |
|---|---|---|
| cartorio-web | cartorio-backend | 8082 (HTTP+GraphQL+WS) |
| openclaw-scanner | cartorio-backend | 8082 |
| openclaw-scanner | openclaw-publisher-pki | 8090 |
| sekiban | cartorio-backend | 8082 (lookup_by_digest) |
| openclaw-skill-store | cartorio-backend | 8082 |
| cartorio-backend | cartorio-postgres | 5432 |
| openclaw-skill-store | cartorio-postgres | 5432 |
| Ingress controller | cartorio-web | 80 |
| Audience browsers (via Ingress) | cartorio-web | 80 |
| All pods | DNS service | 53 |
| Prometheus scraper | all `:9090` metrics | 9090 |

Generated from `pleme-lib._networkpolicy.tpl` template; declared via
`networkPolicy:` block in each chart's values.

### 8.2 ServiceMonitor

Every pod exposes `/metrics` on port 9090. Each chart emits a
ServiceMonitor (lilitu's `templates/servicemonitor.yaml` template).

Cartorio-specific metrics:

```
cartorio_admissions_total{kind, status}     # counter, status ∈ ok|gate1..gate6_failed
cartorio_admit_duration_seconds              # histogram
cartorio_ledger_root_advances_total          # counter
cartorio_active_listings                     # gauge
cartorio_quarantined_listings                # gauge
cartorio_revoked_listings                    # gauge
cartorio_consistency_proof_requests_total    # counter
cartorio_consistency_proof_size_bytes        # histogram
```

### 8.3 Ingress

Single Ingress at `openclaw.dev.use1.quero.lol`:

- `/api/*`, `/graphql` → cartorio-web → hanabi → cartorio-backend
- `/` → cartorio-web → hanabi → static Leptos bundle

DNS upsert: `pangea-architectures/workspaces/platform-k3s` first-boot
script publishes `api.dev.use1.quero.lol → <public_ip>`. We add an
additional record for `openclaw.dev.use1.quero.lol → <public_ip>` in
the same script (one-line addition to the bootstrap, lands as
infrastructure prerequisite in `03-PHASES`).

### 8.4 PSS = restricted

Namespace `pleme-attestation` carries:

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/audit:   restricted
  pod-security.kubernetes.io/warn:    restricted
```

All pods in this namespace must be non-root, read-only-root-fs, no host
mounts, no privileged. The `pleme-microservice` chart shape and
`compliance.overlays: [fedramp-high]` already configure this.

## 9. Local dev experience

Per lilitu's pattern. From `cartorio/`:

```
nix develop                      # dev shell with sea-orm-cli, sqlx-cli, leptosfmt, trunk
nix run .#dev                    # spins up postgres-via-testcontainers + cartorio-backend on :8082
nix run .#schema:cartorio        # regenerates GraphQL schema.graphql
nix run .#migrate:cartorio       # applies SeaORM migrations to local db
nix run .#test:cartorio          # cargo test (unit)
nix run .#test:cartorio:e2e      # spawn cartorio + lacre + scanner + tabeliao; run real_openclaw_e2e
nix run .#release                # builds OCI image
```

From `cartorio-web/`:

```
nix develop                      # dev shell with trunk
nix run .#dev                    # trunk serve on :3000 + reverse-proxy to cartorio-backend:8082
nix run .#build                  # trunk build (production WASM bundle)
nix run .#release                # builds OCI image bundling hanabi + WASM
```

End-to-end local stack (operator's "see the full picture before deploying"):

```
nix run github:pleme-io/cartorio#dev          # postgres + cartorio-backend
nix run github:pleme-io/cartorio-web#dev      # trunk serve, proxies to cartorio
# (in another terminal:)
nix run github:pleme-io/lacre -- ...
nix run github:pleme-io/openclaw-publisher-pki -- ...
nix run github:pleme-io/openclaw-scanner -- ...
# Now operator can:
nix run github:pleme-io/tabeliao -- publish --cartorio http://localhost:8082 ...
# And open http://localhost:3000 to watch.
```

## 10. Decision materialization recap

Per `01-FINDINGS.md` § 10:

| Decision | Materialization |
|---|---|
| **(1) Mirror lilitu** | `cartorio/services/rust/{backend,migration,cartorio-verify}/` workspace per § 2.1; SeaORM 2.0-rc.30 + Axum 0.8 + async-graphql 7 stack; repository trait pattern; shinka migration manifest |
| **(2) helmworks for charts** | All new charts under `pleme-io/helmworks/charts/` per § 2.3, § 5; consumed via `pleme-charts` HelmRepository |
| **(3) Separate `cartorio-web` repo** | `pleme-io/cartorio-web/` per § 2.2; bundled image (hanabi BFF + Leptos WASM in one container) per § 5.2 + § 6 |
| **(4) Offline sigstore** | `sigstore_bundle.fulcio_cert_hash` + `rekor_entry_id="offline-bundle-phase1"` per § 3.1 (matching `pangea-publish-skill`'s default) |

## 11. What `03-PHASES.md` will turn this into

A linear file-by-file execution plan, each step has explicit
artifacts + commands + verification:

1. **`pleme-dev` cluster prerequisites** — flux-system bootstrap committed
   to git, cert-manager + selfsigned-issuer, pleme-charts source +
   ghcr-charts-secret, cnpg-operator, shinka, DNS upsert for
   `openclaw.dev.use1.quero.lol`.
2. **`pleme-attestation` namespace + default-deny NetworkPolicy.**
3. **`cartorio-postgres` CNPG Cluster CR.**
4. **SeaORM entity types + baseline migration** — six migration files,
   migration-manifest.yaml, repository traits.
5. **cartorio-backend new endpoints** — consistency proof, tree-fragment,
   cross-references; Subscription wires.
6. **cartorio-verify CLI** — four subcommands, JSON + human output.
7. **cartorio-web Leptos app** — five routes, six components, runtime
   config plumbing.
8. **helmworks charts × 6** — cartorio-backend, cartorio-web,
   cartorio-postgres, openclaw-skill-store, openclaw-publisher-pki,
   openclaw-scanner. Each with FedRAMP-high overlay.
9. **Sekiban + kensa charts** — one-line overlay add to existing values.
10. **FluxCD HelmReleases under `apps/openclaw/`.**
11. **Smoke test** — operator runs `tabeliao publish` against the live
    cluster (clean artifact); cartorio-web banner advances; the inclusion
    proof is visible.
12. **Tamper test** — operator runs `tabeliao publish` with a mutated
    artifact_hash; cartorio rejects with `AdmissionError::CertificationRoot`;
    cartorio-web flashes red on the offending leaf.
13. **Drift test** — operator mutates a tracked artifact's bytes
    cluster-side; openclaw-scanner detects on next tick; quarantine event
    appended; UI flips the artifact's state from green to orange.
14. **Audit test** — operator runs `cartorio-verify audit --pinned-root
    <known-good>`; CLI walks the full substrate and emits ✓ consistent.
15. **Demo rehearsal** — full run-through, time-coded against the 5-min
    slot. Captured in `07-DEMO-STORYBOARD.md`.

Each step in `03` has:
- a list of files added or changed (with paths from § 2)
- the commands the operator runs to verify the step
- the cluster state expected after the step
- a rollback path if the step blows up

That's the architecture. `03-PHASES.md` ready to write next; do you want
to push back on any of the entity set / endpoints / chart values first?
