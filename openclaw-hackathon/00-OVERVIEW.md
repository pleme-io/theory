# 00 — Overview

> **Status:** under review (00 first pass landed). No code touched until 00–08
> are signed off.
> **Repo:** `pleme-io/theory/openclaw-hackathon/` on `main` (no branches —
> direct-to-main per operator preference).

## Mission

Demonstrate, end-to-end on `pleme-dev`, that the **tameshi pattern produces
a provably-secure openclaw**. The demo's centerpiece is **cartorio's
double-merkle-tree** as the operative proof primitive — every state
transition appended to the event tree, every artifact bound into the
state tree, both composed into the single ledger root that signs the
posture of the whole substrate.

**Thesis (the line every other doc serves):**

> *Compliance becomes a theorem of the type system, not a claim in a
> document. Tampering breaks a hash; admission gates fire by construction;
> the operator never has to "trust" anything because every promise is
> mechanically verifiable.*

## Audience

- Akeyless engineering crowd + crypto/security people at the hackathon.
- Operator (the user) delivers the talk; the demo runs *for* the audience.
- Audience-mode: technical, skeptical, will scrutinize crypto details.
  Wave-handing or vibes-based claims will lose them. The framework's
  defensibility is the asset.

## Demo shape (locked)

| Dimension | Value |
|---|---|
| Slot length | 5-minute lightning |
| Delivery medium | Operator runs the talk; tooling lands the substance |
| Live target | `pleme-dev` K3s cluster on `akeyless-development` AWS |
| Talk content | Operator handles narrative + slot length |
| Our scope | Make the substrate self-evidently work; the operator picks what to show |

The 5-min slot means the substrate must answer **"what's the most
visceral 30-second wow?"** — and have everything else *available* if
the operator wants to extend in Q&A.

## Core values to demonstrate

Every demo asset (live API, UI, CLI, slide) must serve at least one of:

1. **Mechanical compliance.** A `CompliantListing<K>` is constructible
   only via `CompliantListingBuilder::finalize()`. Invariants are
   checked at build-time; no public constructor exists. (See
   `arch-synthesizer/src/compliant_store.rs:720`.)

2. **Tamper-evidence by construction.** `CertificationArtifact` is a
   3-leaf Merkle tree (artifact + control + intent → composed_root).
   Mutate one byte → root changes → signature fails → admission denied.
   The audience sees `VerificationError::CertificationRoot` fire on a
   live tampered listing.

3. **Cartorio double-merkle-tree.**
   - **Event tree** — append-only log of every admission / revocation /
     quarantine / re-attestation event.
   - **State tree** — current attested state of every artifact, indexed
     by listing id.
   - **Ledger root** — Blake3 over (event_root, state_root). Single
     signed handle that proves the entire substrate posture at a moment
     in time. (`cartorio/src/api.rs:84-108`,
     `cartorio/docs/ARCHITECTURE.md`.)

4. **Continuous re-attestation.** `openclaw-scanner` re-hashes every
   attested artifact on a configurable cadence and gates drift.
   (`openclaw-scanner/src/daemon.rs`.)

5. **Admission control end-to-end.**
   - `lacre` gates `PUT /v2/{name}/manifests/{ref}` on cartorio's
     allow-decision (only Active artifacts admitted to the registry).
   - `sekiban` ValidatingAdmissionWebhook gates Pod creation on
     `sekiban.pleme.io/signature` annotation against a SignatureGate CRD.
   The same proof gates push and run.

6. **Formal proofs underwrite the substrate.** 30 Kani harnesses across
   9 files cover certification-root determinism, ledger tamper
   detection, listing-state safety, install-gate logic, cycle detection.
   (`arch-synthesizer/proofs/kani/`.)

## In scope

- Deploy `tameshi/cartorio/lacre + openclaw triad` to `pleme-dev` with
  FedRAMP-high overlays.
- Land cartorio's double-merkle-tree as a fully exercised primitive
  (every endpoint that emits a proof returns a verifiable one; every
  admission writes to both trees and re-derives the ledger root).
- Build a Leptos 0.7 + TailwindCSS web UI in the lilitu-web stack
  shape. Routes: live merkle root, artifact list, artifact detail
  (event timeline + inclusion proof tree), admission outcomes, live
  tamper-detection demo.
- Make the demo runnable end-to-end against `pleme-dev` from the
  operator's laptop with one command (or one URL).
- Capture a hand-runnable demo storyboard (`07`) the operator can
  rehearse + extend.

## Out of scope

- **Akeyless integration.** No DFC signer demo, no Akeyless CLI in the
  flow, no AkeylessTarget proof. (Operator's call. We focus purely on
  the merkle / attestation side.)
- **Sekiban runtime probes for kanshi (eBPF LSM).** kanshi requires
  Linux nodes + privileged DaemonSet + special kernel config; it's
  worth its own demo. M0 scope is the publish→admit→verify→drift loop,
  not the runtime-LSM gate.
- **Hackathon talk content / slide deck.** Operator handles narrative
  + 5-min pacing. Our deliverable is the *substrate* that makes the
  story credible.
- **Multi-cluster federation.** Single cluster (`pleme-dev`).
- **Lacre's FedRAMP-high chart authoring.** Lacre has no chart today;
  for M0 we run lacre as a CLI/binary on the operator workstation
  pointed at the deployed cartorio. Chart authoring → M1.

## Success criteria

The demo lands if **every one of these is true at hackathon time**:

1. `pleme-dev` is running tameshi/cartorio + openclaw triad with
   FedRAMP-high overlays. Cluster comes up clean from cold sleep
   in <10 minutes via the existing `nix run .#wake` flow.

2. The operator can publish an artifact through `tabeliao` against
   the live cluster, see it admitted to cartorio, and watch the
   double-merkle-tree's ledger root advance — visible in the web UI
   in <5 seconds.

3. The operator can mutate one byte of a published listing and
   trigger a deterministic, named verification failure in <5 seconds.
   The error path is observable in the web UI (highlighted offending
   leaf) and in the cartorio audit log.

4. `openclaw-scanner` is running continuously against the deployed
   cartorio and produces a drift-detection event the operator can
   surface live (e.g. by tampering with a cluster-side artifact).

5. The web UI surfaces the **double-merkle-tree** structurally — not
   as text. State tree on one side, event tree on the other, ledger
   root joining them. New events animate. New artifacts animate.
   Tamper events flash red.

6. Every CLI invocation in the demo is captured in `07-DEMO-STORYBOARD.md`,
   verified to work in dry-run, and rehearsable in <5 min total.

7. The substrate is in a **known-clean state** before the talk —
   pleme-dev has been brought up, exercised, and brought back down
   at least once with the demo workflow, with all evidence captured.

## Risks (overview only — see `09-RISK-REGISTER.md` for mitigations)

- **Lacre has no Helm chart.** Will run as workstation binary; this is
  fine for M0 but means lacre is local-to-operator, not on cluster.
- **No web UI exists today.** Building one. Lilitu-web stack mitigates
  this — but it's still net-new code.
- **`pleme-dev` cordel cluster-wait phase param is sticky.** Known
  workaround (probe API directly). Mitigation: replace the wait with
  an API-probe-until-ready loop in the demo runbook.
- **DNS A-record lag** when the cluster's public IP changes per wake.
  Workaround: use `--server=<ip> --tls-server-name=<host>` in
  kubectl. Mitigation: bake the workaround into the runbook;
  verify Route53 sync timing as part of phase-0 confirmation.

## Document map

This plan is split across files in `theory/openclaw-hackathon/`. Each
file is reviewed independently before moving to the next:

| File | Contents | Status |
|---|---|---|
| `00-OVERVIEW.md` | This file. Mission, scope, success criteria. | **Draft — review** |
| `01-FINDINGS.md` | Synthesized recon: what exists today, ready/missing. | Pending |
| `02-ARCHITECTURE.md` | Target end-state diagram + component map (deployed shape). | Pending |
| `03-PHASES.md` | Phased execution plan, file-by-file change list. | Pending |
| `04-API-SURFACE.md` | Exact REST endpoints + request/response shapes. | Pending |
| `05-VISUAL-PLAN.md` | Web UI scope, lilitu-web stack consumption, route map. | Pending |
| `06-DEMO-NARRATIVE.md` | The "story" supporting the operator's 5-min talk. | Pending |
| `07-DEMO-STORYBOARD.md` | Run-of-show, time-coded; CLI + UI interactions. | Pending |
| `08-DECISION-LOG.md` | Open questions, resolutions, dated. | Pending |
| `09-RISK-REGISTER.md` | What can go wrong, mitigations, recovery paths. | Pending |

(Note: `06` was relabeled from "Akeyless narrative" — the operator's
call to drop Akeyless from the demo means this file becomes the
*tameshi pattern* narrative instead.)

## Decisions captured

These are locked and won't be reopened unless the operator says otherwise:

- **Hackathon date / timeline:** indefinite. Quality > speed.
- **Demo slot:** 5-min lightning. Operator owns pacing.
- **Demo runs from:** `pleme-dev` K3s cluster on `akeyless-development` AWS.
  The `pleme-` prefix is a legacy artifact of the `PLATFORM=pleme` workspace
  dimension; renaming infrastructure (ASG, SSM keys, env var, DNS) is
  deferred. Docs use `pleme-dev` verbatim.
- **Domain:** `quero.lol` (the `akeyless-development` AWS account hosts
  this zone). `quero.cloud` is the operator's homelab fleet domain — out
  of scope. Demo hostname: `openclaw.dev.use1.quero.lol`.
- **Branch policy:** direct-to-main on every pleme-io repo touched. No
  feature branches, no PRs.
- **Relationship goal:** show the value of the tameshi pattern; no
  commercial pressure.
- **Constraints:** none flagged.
- **UI ambition:** lilitu / lilitu-web stack patterns wholesale.
  - Frontend: Leptos 0.7 + TailwindCSS + pleme-mui (CSR WASM bundle).
  - Backend: Axum + SeaORM + Postgres (cartorio mirrors lilitu's crate
    layout where applicable).
  - **Frontend bundling:** the WASM bundle is baked into a single
    Docker image **alongside hanabi** (the BFF — GraphQL federation
    gateway + WebSocket relay), with a hanabi configuration that
    wires hanabi → cartorio + serves the frontend assets. **No
    external static hosting.** One image, one HelmRelease, one
    Ingress, all on the K3s cluster.
  - Result: full dashboard, deployed as a normal cluster workload,
    not an HTML-page workaround.
- **Akeyless component:** out. Pure merkle / attestation story.
- **Cartorio double-merkle-tree:** in scope, **fully realized + visualized**
  — including the three M0 additions:
  - **Consistency-proof endpoint** (proves ledger root N+1 extends N).
  - **Viz-friendly tree-fragment endpoint** (parents + siblings ready for
    UI render, not raw inclusion paths).
  - **`cartorio-verify` CLI** so the operator can run "audit the substrate"
    against the live cluster from the laptop.
- **Cluster namespace strategy:** `pleme-attestation` umbrella (matches
  tameshi-watch overlay). plo's `sekiban-system` standalone PoC remains as
  legacy; pleme-dev starts clean on the umbrella shape.
- **Cartorio persistence:** Postgres + SeaORM, mirroring lilitu's backend
  pattern (migrations, connection management, repository layer). Adopted
  verbatim where applicable.
- **Tamper-event sources:** both — `tabeliao publish` with mutated bytes
  rejected at admission **(shift-left prevention is the headline)** and
  `openclaw-scanner` flagging cluster-side drift as the secondary path.
  The demo emphasizes prevention; runtime drift is the safety net.

## Open questions for the operator

All M0 design questions are resolved. The remaining items are
implementation details the lilitu / hanabi recon (`01-FINDINGS.md`)
fills in:

- **Hanabi auth posture for the demo** — hanabi is the BFF in
  lilitu's pattern (GraphQL federation gateway + WebSocket relay).
  Whatever auth posture hanabi runs in lilitu, we inherit. If it bakes
  in auth that's needless friction for a 5-min lightning slot against
  synthetic data, we flag it before adopting.
- **Specific hanabi configuration shape for openclaw** — the lilitu
  recon will surface hanabi's existing config schema; we author one
  for openclaw that wires hanabi → cartorio (REST + GraphQL relay) →
  Leptos frontend bundle.

If anything below is wrong, flag now so `01–02` reflect the right
shape from the start.
