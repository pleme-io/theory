# 00 — Overview

> **Status:** under review (00 first pass landed). No code touched until 00–08
> are signed off.
> **Repo:** `pleme-io/theory/openclaw-hackathon/` on `main` (no branches —
> direct-to-main per operator preference).

## Mission

Demonstrate, end-to-end on `pleme-dev`, that **the tameshi pattern can
attest an entire environment as a single signed statement**.

The demo's example is **openclaw**. Four kinds of artifacts come together
into cartorio's double-merkle-tree, get signed, and produce one
ledger-root claim over the whole running system:

1. **The runtime — `OciImage` listings** for each openclaw component
   (publisher-pki, skill-store, scanner). The bytes that execute.
2. **The architecture — `HelmChart` listings** for each component's
   chart. The composed shape: RBAC, network policies, env vars, resource
   limits, service identities. Attesting the chart attests how the bytes
   *run*.
3. **The compliance proofs — `ComplianceRun` listings** for each pair
   above: `fedramp_high_openclaw_image@2` against each image,
   `fedramp_high_openclaw_helm_content@1` against each chart. Six runs,
   each itself a typed leaf in the same state tree.
4. **The graph — `cross_reference_edge` records** binding chart→image
   (deploys), image→skill (embeds), chart→sibling (depends-on), each
   component→its publisher enrollment. The edges are what turn a flat
   set of leaves into a *system claim*.

These don't sit beside each other in separate registries. They are *all
leaves of the same merkle tree*, hashed with the same domain-separated
math, signed by one publisher, and composed into **one `ledger_root`**.
That hex string underwrites every claim above simultaneously, and any
byte-level tampering anywhere in the graph deterministically produces a
different root.

**Thesis (the line every other doc serves):**

> *Compliance becomes a theorem of the type system, not a claim in a
> document. The substrate makes a single signed statement about the
> entire environment — runtime + architecture + tests + provenance —
> and tampering with any component invalidates the root. Admission
> gates fire by construction; the operator never has to "trust"
> anything because every promise is mechanically verifiable.*

**Demo lede:**

> *If openclaw is running on `pleme-dev`, that is the proof. The merkle
> tree has bound 6 listings (3 images + 3 charts) plus 6 compliance
> runs plus the cross-reference graph plus the publisher enrollment
> into one signed `ledger_root`. Every gate in the chain — publisher
> identity, certification root, signature, compliance pack
> reproducibility, framework coverage, ledger discipline — fired and
> passed for every leaf. Then the substrate keeps proving it: scanner
> re-attestation every 10 seconds, ledger root advancing on every
> verified posture tick. Compliance is no longer an audit you survive
> once a year; it is a continuous mechanically-verifiable property of
> the running system.*

**Regulatory framing for the audience:**

Recent compliance regimes (EU AI Act, SEC cyber-disclosure rule, FedRAMP
Rev 5 update, NIS2, DORA) all push toward continuous attestation rather
than point-in-time audit. The substrate doesn't *implement* those
regimes — it implements the underlying primitive (continuous,
mechanically-verifiable, content-addressed compliance) that every
modern regime is converging on. Pointing at any one regulation is less
load-bearing than the pattern itself: a substrate that gates
non-compliant changes *before* they deploy and continuously verifies
the running posture.

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

## What the merkle tree binds (the openclaw example, concretely)

Per the Mission, when openclaw is admitted into cartorio, the state tree
holds — *simultaneously, under one `ledger_root`* — the following
13 leaves plus their cross-reference graph:

| Kind | Leaf | What it attests |
|---|---|---|
| `OciImage` | `pleme-io/openclaw-publisher-pki@0.1.0` | the publisher-pki binary |
| `OciImage` | `pleme-io/openclaw-skill-store@0.1.0` | the skill-store binary |
| `OciImage` | `pleme-io/openclaw-scanner@0.1.0` | the scanner binary |
| `HelmChart` | `pleme-io/charts/openclaw-publisher-pki@0.1.0` | how publisher-pki deploys (RBAC, ports, env, NetworkPolicy, PSS) |
| `HelmChart` | `pleme-io/charts/openclaw-skill-store@0.1.0` | how skill-store deploys |
| `HelmChart` | `pleme-io/charts/openclaw-scanner@0.1.0` | how scanner deploys |
| `ComplianceRun` | `fedramp_high_openclaw_image@2 / openclaw-publisher-pki@0.1.0` | publisher-pki image passes FedRAMP-High image pack |
| `ComplianceRun` | `fedramp_high_openclaw_image@2 / openclaw-skill-store@0.1.0` | skill-store image passes |
| `ComplianceRun` | `fedramp_high_openclaw_image@2 / openclaw-scanner@0.1.0` | scanner image passes |
| `ComplianceRun` | `fedramp_high_openclaw_helm_content@1 / charts/openclaw-publisher-pki@0.1.0` | publisher-pki chart passes FedRAMP-High helm pack |
| `ComplianceRun` | `fedramp_high_openclaw_helm_content@1 / charts/openclaw-skill-store@0.1.0` | skill-store chart passes |
| `ComplianceRun` | `fedramp_high_openclaw_helm_content@1 / charts/openclaw-scanner@0.1.0` | scanner chart passes |
| `PublisherEnrollment` | `operator@pleme.io` | operator's cert chain to org-root, currently unrevoked |

Plus cross-reference edges (each is a `cross_reference_edge` row,
itself attested through the state tree's hashing math):

| Edge | Asserts |
|---|---|
| `chart/openclaw-publisher-pki → image/openclaw-publisher-pki` | the chart deploys *this* image (digest-pinned) |
| `chart/openclaw-skill-store → image/openclaw-skill-store` | same shape |
| `chart/openclaw-scanner → image/openclaw-scanner` | same shape |
| `chart/openclaw-skill-store → chart/openclaw-publisher-pki` | depends-on (skill-store calls into PKI for cert validation) |
| `chart/openclaw-scanner → chart/openclaw-publisher-pki` | depends-on (scanner pulls CRL from PKI) |
| `chart/openclaw-scanner → chart/openclaw-skill-store` | depends-on (scanner observes skill-store listings) |
| each listing → `operator@pleme.io` | publisher relation |
| each ComplianceRun → its tested artifact | tests relation |

That's the *whole environment*, captured as one merkle structure. A
single signed `ledger_root` underwrites all of it. **No separate
compliance database. No separate provenance ledger. No
"propagation step" from policy to enforcement. One tree, one root, one
verifiable claim.**

## Core values to demonstrate

Every demo asset (live API, UI, CLI, slide) must serve at least one of:

1. **Mechanical compliance — by construction.** A `CompliantListing<K>`
   is constructible only via `CompliantListingBuilder::finalize()`.
   Invariants are checked at build-time; no public constructor exists.
   (`arch-synthesizer/src/compliant_store.rs:720`.)

2. **Tamper-evidence — by construction.** `CertificationArtifact` is a
   3-leaf Merkle tree (artifact + control + intent → composed_root).
   Mutate one byte → root changes → signature fails → admission denied.
   `VerificationError::CertificationRoot` fires deterministically on a
   live tampered listing.

3. **Joint claim under one root.** Cartorio's double-merkle-tree —
   state tree (current posture, all kinds together) + event tree
   (append-only history) → `ledger_root = blake3(state_root,
   event_root)`. The 13 leaves + N cross-reference edges from the
   openclaw example all live in this one structure. The signed root
   IS the joint claim. (`cartorio/src/api.rs:84-108`.)

4. **Continuous re-attestation.** `openclaw-scanner` re-hashes every
   Active artifact every 10 seconds (demo mode). Each successful tick
   is itself an event in the event tree — the substrate's
   *history-of-having-stayed-compliant* is itself attested. Every
   advance of `ledger_root` carries the joint claim forward in time.

5. **Admission control end-to-end — gated by the same root.**
   - `tabeliao publish` cannot register a non-conforming listing
     (the 6 admission gates).
   - `lacre` cannot let a non-Active digest through registry push.
   - `sekiban` cannot let a non-Active digest run as a Pod.
   - `openclaw-scanner` cannot let drift persist past 10 seconds.
   *Every gate consults the same `ledger_root`.* No "did the policy
   propagate?" — the state tree IS the policy.

6. **Formal proofs underwrite the substrate.** 30 Kani harnesses across
   9 files cover certification-root determinism, ledger tamper
   detection, listing-state safety, install-gate logic, cycle detection.
   The two ledger-tamper proofs are the formal floor for the entire
   joint-claim story. (`arch-synthesizer/proofs/kani/`.)

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

1. **Substrate is up.** `pleme-dev` is running cartorio + cartorio-web +
   cartorio-postgres + sekiban + kensa with FedRAMP-high overlays.
   Cluster comes up clean from cold sleep in <10 minutes via the
   existing `nix run .#wake` flow.

2. **Openclaw is the workload under attestation.** The openclaw triad
   (`openclaw-publisher-pki`, `openclaw-skill-store`,
   `openclaw-scanner`) is published into cartorio via `tabeliao` —
   each image + chart attested through provas FedRAMP-High packs —
   and only then deployed onto the cluster. **If openclaw is running,
   every gate passed: that's success 1.** Untampered openclaw deploying
   on cluster *is* the proof.

3. **Continuous verification at 10-second cadence.** `openclaw-scanner`
   re-hashes every Active artifact every 10 seconds (demo-mode
   override; production default is 300s). The web UI shows the
   heartbeat: each successful re-attestation is a visible event on
   the timeline, the ledger root advancing in lockstep.

4. **Shift-left rejection is visible.** The operator can publish a
   tampered openclaw artifact and trigger a deterministic, named
   verification failure (`AdmissionError::CertificationRoot`) in <5
   seconds. The web UI highlights the offending leaf; the artifact
   never reaches the state tree; sekiban never sees an attestation
   for it; **the bad change never deploys.** That's success 2.

5. **Runtime drift is caught.** If a deployed openclaw artifact's
   bytes drift on cluster, the scanner detects within 10 seconds,
   posts a quarantine event, the web UI shows the artifact's card
   flip green→orange, and the next sekiban admission for that digest
   denies. That's the safety net.

6. **The double-merkle-tree is structurally visible.** The web UI
   surfaces it not as text. State tree on one side (every openclaw
   artifact, its compliance runs, its SBOM, the publisher
   enrollment), event tree on the other (every admission, re-attest,
   quarantine), ledger root joining them. New events animate. Tamper
   events flash red. Cross-reference edges show the joint claim.

7. **The demo is rehearsable.** Every CLI invocation in the demo is
   captured in `07-DEMO-STORYBOARD.md`, verified to work in dry-run,
   and rehearsable in <5 min total.

8. **The substrate is in a known-clean state before the talk.**
   pleme-dev has been brought up, exercised, and brought back down at
   least once with the demo workflow, with all evidence captured.

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
- **Workload under test:** **openclaw triad** (publisher-pki +
  skill-store + scanner) — *not* hello-rio. The demo's success-criterion
  is "openclaw deploys" because that means every gate passed.
- **Scanner cadence:** 10 seconds in demo mode (vs production default
  of 300s). Tight enough to be visibly demonstrative; tunable per
  HelmRelease values.
- **Sekiban gate scope:** label-based selector, not namespace-wide. Pods
  carrying `attestation.pleme.io/required: "true"` are gated; cartorio
  + cartorio-web + the supporting controllers don't carry the label
  (they're the substrate, not the workload). Only openclaw triad
  carries it. This avoids the bootstrap chicken-and-egg of "cartorio
  needs to attest itself to deploy."

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
