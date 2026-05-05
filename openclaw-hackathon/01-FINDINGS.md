# 01 — Findings

> Decision-grade synthesis of six parallel recons (openclaw triad, attestation
> core API surface, compliant_store typescape, visual / UI surface, Akeyless
> integration, lilitu backend stack). Sources cited at `file:line`. No
> proposals here — those land in `02-ARCHITECTURE.md`.

## TL;DR (the four findings that shape everything else)

1. **The end-to-end backbone already exists as a passing test.**
   `tabeliao/tests/real_openclaw_e2e.rs` runs the full FedRAMP-High proof
   over a real `lareira-openclaw-pki` Helm chart + a real openclaw-publisher-pki
   OCI manifest. Not a mock. **The demo's spine is "promote that test to a
   live walkthrough."**

2. **The "wow" moment is structural, not theatrical.**
   `CompliantListing<K>` has no public constructor — only
   `CompliantListingBuilder::finalize()` (`compliant_store.rs:720, 862`).
   Mutate one byte of `artifact_hash` and `verify()` fires
   `VerificationError::CertificationRoot` at `compliant_store.rs:823`. The
   substrate refuses tampered artifacts by construction; the audience watches
   the type system enforce it.

3. **Cartorio's double-merkle-tree already runs;** three additions complete
   the M0 scope (consistency-proof endpoint, viz-friendly tree-fragment
   endpoint, `cartorio-verify` CLI).

4. **Lilitu's stack is exactly the deploy shape we adopt.** Axum + SeaORM
   2.0-rc.30 + Postgres (CloudNative-PG) + shinka migration CRD + Hanabi BFF
   bundling Leptos WASM frontend + pleme-lib Helm chart. We mirror it
   verbatim, swap entities for cartorio's domain, drop the lilitu-specific
   quirks.

## 1. The attestation pipeline (what runs today)

### 1.1 Repos + their shape

| Repo | Role | Shape | Tests | Demo-ready? |
|---|---|---|---|---|
| `tameshi` | Core attestation: BLAKE3, Merkle, signers, layer types | Lib | 1260+ #[test] | ✓ via consumers |
| `cartorio` | Merkle ledger HTTP API (port 8082) | Bin + lib | 1260+ #[test] | ✓ |
| `lacre` | OCI Distribution Spec proxy w/ cartorio gate (port 8083) | Bin | — | ✓ workstation only (no chart) |
| `tabeliao` | Publisher CLI: digest + publish | Bin | e2e + real_openclaw_e2e | ✓ |
| `provas` | Compliance test framework (FedRAMP-High packs) | Lib + CLI | — | ✓ via kensa/tabeliao |
| `kensa` | Compliance engine (REST + GraphQL); pack runner | Bin | 362+ #[test] | ✓ CLI; daemon partial |
| `sekiban` | K8s ValidatingAdmissionWebhook + 3 CRDs | Bin | 454+ #[test] | ⚠ needs cluster + cert-manager |
| `inshou` | Nix integrity gate (BLAKE3 of closures) | Bin | 286+ #[test] | ✓ NixOS only |
| `openclaw-skill-store` | Listings API (port 8080) | Bin | 11 #[test] | ✓ basic; no UI |
| `openclaw-publisher-pki` | Cert + CRL HTTP server (port 8090) | Bin | 22 #[test] | ✓ inc. e2e smoke |
| `openclaw-scanner` | Continuous re-attestation daemon (port 9090) | Bin + lib | 5 #[test] | ✓ basic; assessors stubbed |

### 1.2 The exact pipeline (with citations)

```
operator artifact + attestations.yaml + signing-key (hex)
    │
    ├─▶ tabeliao publish               (tabeliao/src/main.rs:78-106)
    │   │
    │   ├─▶ POST cartorio/api/v1/artifacts {AdmitArtifactInput}
    │   │       (cartorio/src/api.rs:84-108)
    │   │   ├─ verify signed_root against publisher key
    │   │   ├─ compose 3-leaf cert (artifact, control, intent → root)
    │   │   ├─ append to event tree
    │   │   ├─ insert into state tree
    │   │   └─ recompute ledger_root = blake3(state_root, event_root)
    │   │
    │   └─▶ PUT lacre/v2/{image}/manifests/{ref}
    │           (lacre/src/routes.rs:64-131)
    │       └─ cartorio.lookup_by_digest() → GateDecision
    │           ├─ Allow → backend (real registry).put_manifest()
    │           └─ Reject → HTTP 403 (THE shift-left moment)
    │
    └─▶ Kubernetes apply
        └─▶ ValidatingAdmissionWebhook → sekiban /validate
                (sekiban/src/webhook/admission.rs:142-150)
            ├─ Read sekiban.pleme.io/signature annotation
            ├─ Look up SignatureGate CRD
            └─ AdmissionResponse {allowed: true|false}

(continuous, in parallel)
openclaw-scanner (every 300s, configurable)
        (openclaw-scanner/src/daemon.rs)
    └─ for each tracked artifact: re-hash → compare → drift?
        ├─ no drift: emit OK metric
        └─ drift: webhook + audit log + metrics
```

The pipeline is **whole** — operator publishes → cartorio admits → lacre
gates push → sekiban gates run → scanner re-verifies. Every arrow has a
real handler we can point at.

### 1.3 The real_openclaw_e2e test (the spine of the demo)

`tabeliao/tests/real_openclaw_e2e.rs:1-100` runs:

1. Real `lareira-openclaw-pki` Helm chart (from helmworks/)
2. Representative `openclaw-publisher-pki` OCI manifest
3. `fedramp_high_openclaw_image@2` + helm pack from provas
4. Admit image + chart + bundle to cartorio
5. Run full verifier procedure

**This is not a fixture.** It exercises the published Helm chart, the
published binary's manifest, the typed FedRAMP-High pack, and the
verifier code path. Every piece the audience cares about runs in this
test today.

## 2. The proof primitive: cartorio's double-merkle-tree

### 2.1 What's already exposed

| Endpoint | Purpose | Source |
|---|---|---|
| `POST /api/v1/artifacts` | Admit (+ verify signed_root, compose cert, append events, recompute roots) | `cartorio/src/api.rs:84-108` |
| `GET /api/v1/artifacts` | List all | same |
| `GET /api/v1/artifacts/{id}` | Fetch state | same |
| `GET /api/v1/artifacts/{id}/verify` | Re-verify state-leaf | same |
| `GET /api/v1/artifacts/{id}/history` | Event chain for one artifact | same |
| `GET /api/v1/artifacts/{id}/compliance-runs` | Pack results | same |
| `POST /api/v1/artifacts/{id}/revoke` | Revoke (PKI signs) | same |
| `POST /api/v1/artifacts/{id}/quarantine` | Quarantine (scanner signs) | same |
| `POST /api/v1/artifacts/{id}/reactivate` | Reactivate (operator + scanner cosign) | same |
| `POST /api/v1/artifacts/{id}/reattest` | Re-attest (scanner signs) | same |
| `POST /api/v1/artifacts/{id}/supersede` | Supersede (publisher signs) | same |
| `GET /api/v1/artifacts/by-digest/{digest}` | Lookup by content digest | same |
| `GET /api/v1/events` | Raw event log | same |
| `GET /api/v1/events/{event_id}` | One event | same |
| `GET /api/v1/artifacts/{id}/proof` | Merkle inclusion proof | same |
| `GET /api/v1/events/{event_id}/proof` | Event inclusion proof | same |
| `GET /api/v1/merkle/root` | `{state_root, event_root, ledger_root}` | same |
| `GET /health` + `GET /metrics` | Liveness + Prometheus | same |

### 2.2 The three M0 additions (locked per `00-OVERVIEW`)

- **Consistency-proof endpoint** — `GET /api/v1/merkle/consistency?from=<root_n>&to=<root_n+1>` returning a Merkle range proof that root N+1 extends root N (the standard tamper-evident-log invariant — without this, a malicious cartorio can rewrite history between snapshots).
- **Viz-friendly tree-fragment endpoint** — `GET /api/v1/artifacts/{id}/tree-fragment` returning the node + parents + siblings ready for d3-tree / cytoscape rendering, instead of forcing the UI to assemble inclusion paths into a tree by hand.
- **`cartorio-verify` CLI** — a small Rust binary that takes a `<listing-id>` (or "all"), pulls the proofs from cartorio over HTTP, and verifies them locally against a pinned org root. Operator runs this from the laptop to prove "the whole substrate is consistent right now." Lives in `cartorio/cartorio-verify/` or as a binary target in the cartorio crate.

### 2.3 Why this matters for the demo

The double-merkle structure gives the operator three distinct visualizable
artifacts:

- **Event tree** — append-only history, every admission/revocation event
  becomes a leaf. Visually: scrolling timeline.
- **State tree** — current attested state, indexed by listing id. Visually:
  artifact list/cards.
- **Ledger root** — single 64-char hex that signs the entire substrate posture
  at this moment. Visually: a banner that updates atomically as new events
  land.

When the operator tampers with anything, **all three change deterministically**
and the verifier reports exactly which leaf broke. That's the punchline.

## 3. The compliant_store typescape

### 3.1 Type set (in arch-synthesizer/src/compliant_store.rs)

| Type | Lines | Purpose |
|---|---|---|
| `ArtifactKind` (sealed trait) | 62-117 | 14 phantom-typed kinds: `SkillKind`, `OciImageKind`, `HelmChartKind`, … |
| `ComplianceFramework` | 156 | enum: NIST 800-53, FedRAMP, SOC2, HIPAA, PCI-DSS-v4, ISO 27001, EU AI Act, NIST AI RMF, CIS-K8s, SLSA-Build |
| `ComplianceVerdict` | 319 | framework + `NonEmpty<ControlVerdict>` + summary_hash |
| `ControlVerdict` | 306 | id, status (Passed/PassedWithWaiver/Failed), evidence_hash |
| `CertificationArtifact` | 381 | 3-leaf Merkle: artifact + control + intent → composed_root |
| `SignedRoot` | (re-export) | root + signature + algorithm + signer_id + signed_at |
| `SigstoreBundle` | 566 | Fulcio + Rekor commitment |
| `LedgerInclusionProof` | 532 | append-only log proof |
| `AttestedPublisher` | 475 | publisher_id + public_key + enrollment_chain + enrolled_root |
| `CompliantListing<K>` | 720 | **NO PUBLIC CONSTRUCTOR**; only via `CompliantListingBuilder::finalize()` |
| `ListingState<K>` | 696 | enum: `Active(_)` ‖ `Revoked(_)` ‖ `Quarantined(_)` |
| `CompliantListingBuilder<K>` | 862 | fluent builder, all required fields enforced at finalize |
| `NonEmpty<T>` | 242 | type-level non-emptiness |
| `Sbom` | 636 | sorted components + sbom_hash |
| `PillarEvidence` | 222 | binding to existing Pillar 10 chain |
| `CompliantStore<K>` (trait) | 1045 | publish / lookup / search / revoke / quarantine / count_active |

### 3.2 Formal underpinnings: 30 Kani harnesses across 9 files

`arch-synthesizer/proofs/kani/src/`:

- `cert_root_proofs.rs` — 3 (composition determinism, position-sensitivity)
- `cycle_detection_proofs.rs` — 5 (self-loop, 2/3-cycles, linear DAG, DFS agreement)
- `hash_proofs.rs` — 3 (determinism, leaf/internal disjoint, non-commutative)
- `install_gate_proofs.rs` — 5 (empty/all-allow/any-deny/short-circuit/passthrough)
- `ledger_proofs.rs` — 3 (append-verify roundtrip, **tampered-leaf-fails-verify**, **tampered-prior-fails-verify**)
- `listing_state_proofs.rs` — 3 (active iff Active variant — type-system-level safety)
- `nonempty_proofs.rs` — 4
- `verdict_proofs.rs` — 6 (empty rejected, single-failing rejected, single-passed succeeds, …)

The two ledger tamper proofs are the formal underwriters of the demo's
shift-left claim. We can cite them on screen.

### 3.3 Skill-evidence: 8 collectors, 5 frameworks, 19 controls

In `arch-synthesizer/skill-evidence/src/collectors/`:

| Collector | Output |
|---|---|
| `ArtifactHashCollector` | SLSA L1 / NIST SI-7 verdicts + artifact_hash hex |
| `CapabilityScopeCollector` | NIST AC-3 / AC-6 verdicts + drift findings |
| `SbomGenerator` | SBOM components + hashes |
| `SkillLintCollector` | NIST CM-6 metadata-quality |
| `GuardrailCollector` | safety verdicts |
| `AtlasTechniqueCollector` | MITRE ATLAS AML.T#### verdicts |
| `PromptInjectionCollector` | NIST-AI-RMF.Manage verdicts |
| `DeterministicTestRunCollector` | SLSA L3 / NIST-AI-RMF.Measure verdicts |

Frameworks shipping today (`skill-evidence/profiles/*.yaml`):

- NIST AI RMF (1.0) — 6 controls
- OWASP LLM Top 10 (2024) — 4 controls
- MITRE ATLAS (2024.10) — 4 controls
- SLSA Build L1 (1.0) — 2 controls
- EU AI Act Limited Risk (2024/1689) — 3 controls

Reference fixture: `arch-synthesizer/skill-evidence/tests/fixtures/hello-rio/`
round-trips through the full pipeline.

### 3.4 The `pangea-publish-skill` orchestrator

11-step flow, in `arch-synthesizer/skill-evidence/src/bin/pangea_publish_skill.rs`:

```
validate skill dir
  → run Phase 1 collectors → Vec<CollectorReport>
  → extract hashes + parse SKILL.md → SkillManifest
  → build pillar seal (Pillar 10 chain) → control_hash
  → compose 3-leaf cert: artifact_hash, control_hash, intent_hash
  → SignedRoot via LocalSigner.sign(seed, signer_id, root)
  → offline Sigstore bundle (Fulcio + Rekor)
  → group controls into verdicts by framework prefix
  → publish_morphism::run(...) → CompliantListing<SkillKind> | BuildError
  → render to JSON (60-80 lines, ~1.2-1.8 KB)
  → output to file or stdout
```

A finalized listing is the **single value** the demo hands to cartorio.
Tampering any field breaks `verify()` deterministically.

## 4. The openclaw triad

### 4.1 Per-service summary

**openclaw-skill-store** (port 8080, SeaORM/SQLite/Postgres):
- `GET /health`
- `GET /api/v1/skills` — list
- `POST /api/v1/skills` — publish (in-memory cut today; SeaORM scaffold ready)
- `GET /api/v1/skills/{id}/verify` — verify Active status

**openclaw-publisher-pki** (port 8090, in-memory BTreeMap):
- `GET /health` + `GET /metrics` (Prometheus)
- `POST /enroll` — issue cert
- `GET /org-root` — current signed org-root
- `POST /revoke` — mark revoked
- `GET /crl` — signed CRL
- **`examples/smoke.rs`** — runnable e2e script (255 lines) covering enroll → revoke → CRL roundtrip.

**openclaw-scanner** (port 9090, in-memory HashMap):
- `GET /health` + `GET /api/v1/status`
- Daemon: every 300s re-hashes tracked artifacts, emits Prometheus + webhook + audit log on drift.

### 4.2 What's wired vs. what's stubbed

| Working today | Stubbed for M0 |
|---|---|
| All three Axum servers boot | No docker-compose / single-command quickstart |
| Smoke test for PKI | No seed data for store |
| Independent metrics + logs | Scanner assessors (revocation fetch from PKI CRL) are placeholders |
| OCI image build via nix | No integrated revocation checking between store ↔ PKI |
| Health probes | No JWT/bearer auth between services |
| | No admission-control call-out from store to PKI before accept |

Demo-readiness estimate: **~60%**. Plumbing exists, glue + seed data missing.

## 5. Lilitu's stack — the canonical deployment shape we adopt

### 5.1 What lilitu is doing (verbatim, to mirror)

**Workspace + crates** at `lilitu/services/rust/`:
- `backend/` — main API server
- `migration/` — SeaORM Rust migrations
- Workspace dependencies pin shared versions

**Backend stack:**
- Axum 0.8 + tokio
- SeaORM 2.0-rc.30 + sqlx 0.8 (co-existing — sqlx for legacy, SeaORM forward)
- async-graphql 7 (GraphQL primary; REST for uploads)
- Hand-authored entities in `src/entities/` (45+ files; *not* sea-orm-cli generated)
- pleme-rbac for auth middleware
- Repository trait pattern: `BaseRepository<E>`, `InsertableRepository<E>`, `SoftDeletable`, `SoftDeleteExt::not_deleted()`

**Migrations** (dual-layer in lilitu; cartorio starts pure SeaORM):
- Legacy: `migrations/*.sql` (001-078 baseline)
- Current: `migration/src/m202601XX_*.rs` (30+ since baseline)
- **Manifest at `migration/src/migration-manifest.yaml`** classifying each as `schema_only | data_only | schema_and_data | noop` with forward/backward companions — the *safety gate against silent data corruption*.
- Run via **shinka** operator (Migration CRD in K8s) — init container waits, applies, gates the main pod.

**Postgres:**
- **CloudNative-PG (CNPG)** cluster — declared as `cnpgClusterRef: lilitu-postgres`. Not Bitnami. Cloud-native, no manual backups, declarative replicas + recovery.
- DB credentials from K8s Secret (`lilitu-postgres-connection`), env-injected as `DATABASE_URL`.

**Helm charts** at `lilitu/deploy/charts/`:
- `lilitu-backend` — Chart 0.1.1, depends on `pleme-lib v0.3.1`, 308-line values.yaml, init container = shinka migration wait, replicas configurable, network policy + PDB + topology spread, downward API for POD_NAME/NODE_NAME.
- `lilitu-web` — frontend + Hanabi BFF in a single image. Hanabi config in ConfigMap: `backend_url=http://lilitu-backend:8080/graphql`, session store = Redis.
- `lilitu-workers` — NATS JetStream consumers (out of cartorio's scope).

**API shape:**
- POST `/graphql` (mutations), GET `/graphql` (subscriptions over WebSocket)
- POST `/api/upload/*` (REST for files)
- `/health`, `/health/ready` (db + redis check), `/health/live`
- `/metrics` on port 9090

**Auth:**
- Hanabi BFF terminates session, validates against Redis, injects `x-user-*` headers
- Backend middleware reads headers, builds `AuthzContext`, passes to resolvers via `.data(authz)`

**Frontend:**
- **Leptos WASM** (Trunk.toml). Not React.
- Hosted *behind* Hanabi as static assets in the same container/pod.
- Hanabi reverse-proxies `/graphql` and serves the WASM bundle at `/`.

**Local dev + CI:**
- Nix flake at `lilitu/flake.nix` with inputs: forge, substrate, hanabi, libraries, pleme-linker, helmworks
- Commands: `nix run .#schema:lilitu`, `nix run .#migrate:lilitu`, `nix run .#test:lilitu`
- Cargo.nix via crate2nix; Nix builds OCI images directly
- testcontainers feature for in-process integration tests

### 5.2 What's lilitu-specific we don't copy

- **Domain entities** — campaigns, classifieds, profiles, listings → swap for cartorio's artifact/event/ledger/listing-state/compliance-run
- **Web Push (VAPID)** — no notifications in cartorio
- **Mercado Pago + Stripe webhooks** — cartorio doesn't take payments
- **Profile analytics** — cartorio uses Prometheus + audit log instead

### 5.3 Known limitations (don't propagate)

- SeaORM 2.0-rc.30 is pre-release — usable, but watch for stable 2.0 upgrade
- Seaography (GraphQL federation) is disabled pending 2.0 compatibility (`backend/Cargo.toml:146-147`)
- Legacy SQLx migrations (001-078) clutter the codebase — cartorio starts fresh, **pure SeaORM only**

## 6. Visual / UI surface

### 6.1 What exists today in pleme-io for visualization

| Asset | State | Verdict |
|---|---|---|
| `tameshi/README.md:25-36` Mermaid diagram | Written | **Use as cover slide** — already attestation-flow-shaped |
| `cartorio/docs/COMPLIANCE-PROOF.md:72-100` flow diagram | Written | **Lift directly into 06-DEMO-NARRATIVE** |
| `cartorio/docs/RUNBOOK.md` | Written | **Live curl crib sheet** |
| `lilitu-web` Leptos+Tailwind+pleme-mui | Production-ready | **Adopt the pattern** |
| `varanda` Yew PWA | Auth-portal-only | Skip — Leptos is what we're using |
| `narsil-mcp` React + Cytoscape | Production-ready | Skip — we're staying in Rust |
| `egaku-term` ratatui | Production-ready | Skip — web UI is the choice |

### 6.2 What doesn't exist (we author it)

- A compliance / attestation dashboard.
- A double-merkle-tree visualization (state tree + event tree + ledger root, animated on event arrival).
- A live tamper-detection demo surface (mutate one byte → flash red on the offending leaf).
- A consistency-proof viewer (root N → root N+1 with the proof structure displayed).
- A `cartorio-verify` CLI with rich terminal output for "audit the substrate" runs.

These are the M0 deliverables on the UI side.

## 7. What we inherit vs. what we author

### 7.1 Inherit wholesale (zero modification)

- `tameshi` core attestation library — types, Merkle, signers
- `cartorio` HTTP API (extend with the three M0 endpoints; don't rewrite)
- `lacre` OCI proxy (use as-is, run on workstation for M0)
- `tabeliao` CLI (use as-is — `real_openclaw_e2e.rs` already exercises the real flow)
- `provas` FedRAMP-High packs — `fedramp_high_openclaw_image@2`, `fedramp_high_openclaw_helm_content@1`
- `kensa` CLI for compliance-pack runs
- `arch-synthesizer/src/compliant_store.rs` typescape — the type system *is* the proof
- `arch-synthesizer/skill-evidence/` — 8 collectors, 5 frameworks, 19 controls, all shipping
- 30 Kani harnesses — cite on screen as the formal floor
- `pangea-publish-skill` orchestrator — the single command that turns a skill dir into a typed listing
- Lilitu's Cargo workspace shape, Helm chart shape, shinka migration pattern, Hanabi BFF pattern, pleme-lib v0.3.1 helm dep, CNPG cluster shape

### 7.2 Author or extend

| Artifact | Effort | Owner repo |
|---|---|---|
| Cartorio: consistency-proof endpoint | small (new handler + helper) | `cartorio` |
| Cartorio: viz-friendly tree-fragment endpoint | small (new handler + render helper) | `cartorio` |
| `cartorio-verify` CLI (new binary target) | small-medium | `cartorio` |
| Cartorio: SeaORM + CNPG persistence (replace in-memory) | medium (mirror lilitu's repository pattern) | `cartorio` |
| Cartorio: shinka-friendly migration crate | small (lift lilitu's migration manifest pattern) | `cartorio` |
| Cartorio: Helm chart `cartorio-backend` | medium (mirror `lilitu-backend` structure) | `helmworks` or `cartorio/deploy/charts/` |
| **Web UI:** Leptos+Tailwind+pleme-mui dashboard with double-merkle-tree visualization, artifact list, event timeline, tamper-detection live demo, consistency-proof viewer | medium-large (new frontend, but lilitu-web is the template) | new `cartorio-web` (or in `cartorio/services/rust/web/`) |
| **Hanabi configuration for openclaw** | small (mirror lilitu-web's hanabi configmap, swap backend_url to cartorio) | `cartorio/deploy/charts/cartorio-web/` |
| Helm chart `cartorio-web` (Hanabi BFF + Leptos bundle) | medium (mirror `lilitu-web` chart structure) | `helmworks` or `cartorio/deploy/charts/` |
| FedRAMP-high overlay opt-in on sekiban + kensa + cartorio + cartorio-web charts | small (one-line add per chart) | `helmworks/charts/*` |
| K3s cluster manifests for `pleme-dev`: flux-system, cert-manager, pleme-charts source, pleme-attestation namespace, all HelmReleases | small-medium | `k8s/clusters/pleme-dev/` |
| Demo seed: `tabeliao publish` script that runs against the live cluster + a script that intentionally publishes a tampered artifact | small | new `cartorio/demo/` or `theory/openclaw-hackathon/scripts/` |

### 7.3 Defer to M1+

- **Akeyless DFC signer integration** — operator's call to drop from this hackathon
- **kanshi (eBPF LSM) runtime probes** — separate demo; needs Linux nodes + privileged DaemonSet
- **Lacre Helm chart** — workstation-only is fine for M0
- **Multi-cluster federation** — single cluster is enough
- **lilitu-style NATS workers** — cartorio doesn't have async work to fan out yet

## 8. Pleme-dev cluster state (the deployment target)

### 8.1 Current state

- `apps/hello-world/` is the only declared workload.
- **No `flux-system/` manifests in repo.** Cluster IS bootstrapped (proven by hello-world reconciling), but the bootstrap isn't tracked in git.
- No HelmRepository source for `pleme-charts`.
- No cert-manager.
- No `pleme-attestation` namespace.

### 8.2 Sibling clusters (for pattern lift)

- **plo** runs sekiban v0.1.0 in `sekiban-system` namespace (PoC, **`failurePolicy: Ignore`**). One-off, not the umbrella shape.
- **`tameshi-watch/k8s-manifests/overlays/production/`** — already templated overlay with sekiban + kensa + kanshi + tameshi-watch wired into `pleme-attestation` umbrella. **Big win — copy this overlay shape.**

### 8.3 Known gotchas (carry into the runbook)

- **cordel cluster-wait phase param is sticky** — returns `ready` after 1s on second wake because the prior boot's value persists. Workaround: probe `kubectl get --raw /readyz` directly until it returns ok.
- **DNS A-record lag** — `api.dev.use1.quero.lol` may resolve to the prior wake's IP. Workaround: kubectl with `--server=https://<ip>:6443 --tls-server-name=api.dev.use1.quero.lol`.
- **Public IP changes per launch** — no Elastic IP. Operator's verify scripts must be regenerated on each wake.
- **Operator IP allowlist** — `operator_cidrs` already includes `24.158.175.41/32` (residential), `212.150.17.41/32` (secondary), `83.229.26.124/32` (tertiary). New locations need pangea apply.

## 9. The shape `02-ARCHITECTURE.md` will draw

(Preview only — full diagram + component map in `02`.)

```
                     ┌───────── pleme-dev K3s ─────────┐
                     │                                  │
operator laptop      │  pleme-attestation namespace    │
  │                  │  ┌─────────────────────────┐    │
  ├─ tabeliao ──┐    │  │ cartorio-backend (Pod)  │    │
  │             │    │  │  Axum + SeaORM          │    │
  ├─ lacre ─────┼───→│→ │  port 8082              │    │
  │             │    │  └─────────────────────────┘    │
  ├─ kensa ─────┘    │              │                  │
  │                  │              ▼                  │
  └─ cartorio-verify │  ┌─────────────────────────┐    │
                  ←──┼──│ cartorio-postgres (CNPG)│    │
                     │  │  state + event + ledger │    │
                     │  └─────────────────────────┘    │
                     │                                  │
                     │  ┌─────────────────────────┐    │
                     │  │ cartorio-web (Pod)      │    │
                     │  │  hanabi BFF + Leptos    │    │
                     │  │  port 8080              │    │
                     │  └─────────────────────────┘    │
                     │              │                  │
                     │              ▼ Ingress          │
                     │  openclaw.dev.use1.quero.lol   │
                     │                                  │
                     │  ┌─────────────────────────┐    │
                     │  │ openclaw-publisher-pki  │    │
                     │  │ port 8090               │    │
                     │  └─────────────────────────┘    │
                     │  ┌─────────────────────────┐    │
                     │  │ openclaw-scanner        │    │
                     │  │ daemon, every 300s      │    │
                     │  └─────────────────────────┘    │
                     │  ┌─────────────────────────┐    │
                     │  │ sekiban (admission webhook) │
                     │  │ + kensa (compliance daemon) │
                     │  │ FedRAMP-high overlays       │
                     │  └─────────────────────────┘    │
                     │                                  │
                     └──────────────────────────────────┘
```

That's the target. `02-ARCHITECTURE.md` formalizes this with:
- precise namespace + service map
- the saguão-shaped Ingress + cert-manager wiring
- the FluxCD Kustomization layout under `k8s/clusters/pleme-dev/`
- the SeaORM entity set cartorio needs (artifact, event, listing_state, compliance_run, ledger_snapshot, …)
- the Hanabi config for openclaw (with backend_url, session store, asset path)
- where each piece lives in code (existing crate vs. new file)

## 10. What's resolved + what 02 needs to decide

**Resolved by recon + operator answers:**
- Pipeline shape (`tabeliao → cartorio → lacre → sekiban → scanner`)
- Stack (Axum + SeaORM + CNPG + shinka + Hanabi + Leptos + pleme-lib)
- Deployment shape (single HelmRelease per service, cluster-only, Ingress at `openclaw.dev.use1.quero.lol`)
- Namespace (`pleme-attestation` umbrella)
- Persistence (Postgres via CNPG cluster)
- Tamper-event sources (both, prevention is headline)
- Double-merkle-tree scope (existing endpoints + 3 M0 additions)
- UI ambition (full Leptos dashboard)
- No Akeyless

**Decisions for `02-ARCHITECTURE.md`:**

1. **Cartorio repo layout** — does the SeaORM-backed cartorio live in:
   - **(A)** `pleme-io/cartorio/services/rust/{backend,migration,web}/` (mirror lilitu's monorepo shape inside the cartorio repo)
   - **(B)** Multiple repos: `cartorio` (current, becomes lib + bin), `cartorio-web` (frontend), `cartorio-backend` (API server)
   - **(C)** Extend the current `cartorio` Cargo workspace with new crates inline
   - I lean **(A)**, mirroring lilitu's pattern verbatim. Confirm in `02`.

2. **Where the new helm charts live** — `pleme-io/helmworks/charts/cartorio-backend` + `cartorio-web` (organization-canonical), or `pleme-io/cartorio/deploy/charts/` (lilitu pattern)?
   - I lean **`helmworks/`** — that's where every other org chart lives, and pleme-charts HelmRepository already serves them. Confirm in `02`.

3. **What Cargo workspace cartorio's web frontend lives in** — do we author `cartorio/services/rust/web/` as a Leptos crate inside cartorio's repo, or stand up a separate `pleme-io/cartorio-web/` repo following lilitu-web's shape? I lean **separate `cartorio-web` repo** for clean separation, but want operator's call. In either case, the *Hanabi BFF + Leptos in one Docker image* shape is locked.

4. **Which sigstore mode for the demo** — `pangea-publish-skill` produces an offline Sigstore bundle today. Online mode (`tameshi-sigstore` Fulcio + Rekor) requires real OIDC. I lean **offline for the demo** (deterministic, no external dep). Confirm.

These four are the only architecture-level decisions left. `02` lands as soon as they're called.
