# Pangea-Operator — IaC as a typed convergence process

> **Status:** AUDIT — this doc was authored greenfield-style on
> 2026-04-29 BEFORE discovery of the existing
> [`pleme-io/pangea-operator`](https://github.com/pleme-io/pangea-operator)
> workspace. Sections I-XII describe the desired SHAPE, but reality
> diverges from the proposal in important ways. **Read § XVI first**
> for the actual gap analysis between this doc and the live operator.
> The greenfield framing remains useful as a "north star" the live
> operator can be measured against.

---

## XVI. Reality vs this design (gap analysis)

The pangea-operator already exists at
`~/code/github/pleme-io/pangea-operator` (canonical:
`github.com/pleme-io/pangea-operator`). Recent commits show active
polish — six new metrics, per-resource policy engine, state-settling
tracker, structured drift detail, provider-credentials env injection.

### What this doc proposed vs what exists

| Concept (this doc) | Reality (live operator) | Gap |
|---|---|---|
| `PangeaWorkspace` CRD | `InfrastructureTemplate` (`infra` / `it`) | Naming difference only. Use the existing name. |
| `PangeaApplication` CRD | `InfrastructureFlow` | Same. |
| State backend: S3 + DynamoDB | Postgres schema-per-`pangeaNamespace` (per `src/backend/{postgres,state,lock,schema,config}.rs`) | Material divergence — the live operator centralizes state in postgres rather than per-workspace S3 buckets. Update §III to reflect this. |
| Drift detection design | Already implemented + recently polished (per-resource detail, settling tracker, six metrics on :9090) | No build needed; only consume. |
| Compliance webhook | `ComplianceBinding` + `ComplianceSchedule` CRDs already exist | Different shape than my admission-webhook proposal — these are CRs, not webhooks. Read existing controllers to learn the gating model. |
| Attestation gate (`Attested<RenderedOutput>`) | Not yet integrated with arch-synthesizer's `Attested<T>` | Genuine gap. **This is the load-bearing ask** — wire arch-synthesizer's compliance proof into the existing template/flow reconcilers. |
| Helm chart in helmworks | `pangea-operator/charts/pangea-operator/` already exists in-repo | Chart exists; integration with `lareira-*` umbrella convention may be a future polish. |
| FluxCD HelmRelease on rio | `deploy.yaml` says "FluxCD handles deployment via GitOps (pleme-io/k8s)" — likely already deployed or wired | Verify by inspecting `k8s/clusters/rio/`. |
| Auth via Akeyless → ESO → pod env | Per recent commit "inject providerCredentials secret keys as ENV at compile time" | Already shipped. Different shape (per-template providerCredentials secret) than my generic `secretRefs[]` proposal. Adopt existing shape. |
| Multi-cluster apply | Not addressed yet | Genuine gap, but Phase-6e scope. |

### What this doc got right (worth porting into operator docs)

- The operator IS the main interface (thesis carries forward).
- Reconcile state machine description (Pending → Synthesizing → Planning → Awaiting → Applying → Converged → Drifted) — likely matches what's already implemented; cross-check `src/controller/template_controller.rs` and `src/controller/settling.rs`.
- Drift remediation policy (alert vs auto-remediate vs import-then-apply) — useful framing for the live operator's policy engine.
- Operator UX section (`kubectl apply` → typed status visible via kubectl) — exactly what the live operator already provides.

### Remaining genuine work (revised Phase 6 plan)

| Phase | Was | Now |
|---|---|---|
| 6a | Design doc | DONE — but framed wrong; this audit corrects it. |
| 6b | Build operator from scratch | **AUDIT existing operator + integrate `Attested<RenderedOutput>` from arch-synthesizer.** Read `src/crd/infrastructure_template.rs`, `src/controller/template_controller.rs`, `src/executor/{tofu,plan,policy}.rs`. Identify the seam where the compliance proof plugs in. |
| 6c | Deploy to rio | **Verify operator is live on rio** (inspect `k8s/clusters/rio/`) + write the first `InfrastructureTemplate` CR for `pleme-io-opensource`. The operator's first reconcile of that CR is the smoke test — including auto-remediation of the standing label drift from the imperative deploy of 2026-04-29. |
| 6d | Migrate first workspace | Subsumed into 6c. |
| 6e | Roll out remaining workspaces | Same — author one `InfrastructureTemplate` CR per pangea workspace. Existing posture / cluster / DAG workspaces all flip to CR-driven. |

### Lessons recorded

This audit recovery is itself a substrate-strength signal: the prime
directive's "Before writing code, search pleme-io for the closest
primitive" rule was violated. The recovery is cheap (mostly prose), but
the lesson is: when the user redirects with "this should run on X",
inspect X first. The repo at `~/code/github/pleme-io/pangea-operator/`
has been in active development for months; missing it cost ~2 hours of
greenfield design that's now an audit reference rather than a build
spec.

---

> **Status:** design (Phase 6a, 2026-04-29 — see § XVI for the audit).
> **Canonical home:** this doc, with § XVI as the binding gap analysis
> and §§ I-XV as the north-star framing.

---

## ★ Thesis

**Pangea-operator is pleme-io's primary interface for infrastructure.**

Every change to AWS, GitHub, Cloudflare, the homelab clusters, or any
other typed-Pangea-rendered system goes through one path:

```
operator commits to git
       │
       ▼
FluxCD reconciles the cluster
       │
       ▼
pangea-operator picks up the new PangeaWorkspace generation
       │
       ▼
synth → plan → attestation gate → apply → status: Converged
```

Laptop-side `nix run .#flow-deploy-...` becomes a development affordance
for *new* workspaces only. Steady-state convergence is in-cluster, typed,
attested, and auditable. The operator's auth (Akeyless creds, AWS
profiles, GitHub tokens) leaves the laptop entirely.

This is the **convergence-as-process** thesis from
[`THEORY.md` §IV](./THEORY.md) made operational for IaC: a Kubernetes
cluster IS a convergence process; FluxCD is its process manager;
pangea-operator is the IaC controller running inside that process.

---

## I. CRDs

### `PangeaWorkspace` (singleton, one tofu state)

```yaml
apiVersion: pangea.pleme.io/v1
kind: PangeaWorkspace
metadata:
  name: pleme-io-opensource
  namespace: pangea-system
spec:
  source:
    git:
      repo: github:pleme-io/pangea-architectures
      ref: main
      path: workspaces/pleme-io-opensource
  auth:
    # All auth resolved from cluster secrets. No literals.
    secretRefs:
      - name: github-app-token
        backend: akeyless
        path: /pleme-io/pangea-operator/github-app-token
        envVar: GITHUB_TOKEN
      - name: aws-credentials
        backend: akeyless
        path: /pleme-io/pangea-operator/aws-akeyless-development
        envVar: AWS_PROFILE
  state:
    # Where tofu state lives. S3 for shared workspaces, configmap for
    # cluster-local. Operator refuses local-fs state in production.
    backend: s3
    bucket: pleme-dev-terraform-state
    key: pangea/pleme-io-opensource
    region: us-east-1
  reconcile:
    # How aggressive: drift-only (plan continuously, alert on drift),
    # apply-on-change (plan when source changes, apply if clean),
    # apply-on-merge (apply on every git ref bump).
    mode: apply-on-change
    interval: 5m
    timeout: 10m
  attestation:
    # The Attested<T> seal that arch-synthesizer emits is the apply gate.
    # Operator refuses to apply without a fresh, valid attestation.
    require: true
    maxAgeSeconds: 3600
  drift:
    detection: continuous          # plan every `interval`
    onDrift: alert                 # or: auto-remediate (re-apply spec)
    notify:
      - alertmanager
status:
  conditions:
    - type: Synthesized
      status: "True"
      reason: SynthSucceeded
      lastTransitionTime: 2026-04-29T03:14:15Z
    - type: PlanClean
      status: "False"
      reason: DriftDetected
      message: "5 label resources need creation"
    - type: Applied
      status: "Unknown"
    - type: Converged
      status: "False"
  lastSynthHash: blake3:abc123...
  lastPlan:
    summary: "Plan: 0 to add, 0 to change, 0 to destroy"
    timestamp: 2026-04-29T03:14:15Z
    attestation: "..."             # Attested<RenderedOutput> handle
  lastApply:
    timestamp: 2026-04-29T03:14:15Z
    revision: blake3:def456...
  drift:
    detected: false
    lastChecked: 2026-04-29T03:14:15Z
```

### `PangeaApplication` (multi-workspace DAG)

For pangea workspaces with cross-state dependencies (e.g.
`seph-vpc → seph-cluster → seph-helm`). Equivalent to the existing
`fleet.yaml` flow runner, expressed as a CR.

```yaml
apiVersion: pangea.pleme.io/v1
kind: PangeaApplication
metadata:
  name: kazoku
  namespace: pangea-system
spec:
  workspaces:
    - name: kazoku-state-backend
      dependsOn: []
    - name: kazoku-vpc
      dependsOn: [kazoku-state-backend]
    - name: kazoku-cluster
      dependsOn: [kazoku-vpc]
    - name: kazoku-helm
      dependsOn: [kazoku-cluster]
  reconcile:
    mode: apply-on-change
    interval: 10m
status:
  workspaceStates:
    kazoku-state-backend: Converged
    kazoku-vpc: Converged
    kazoku-cluster: Applying
    kazoku-helm: Pending
  overall: Applying
```

The `PangeaApplication` reconciler creates a child `PangeaWorkspace` CR
per declared workspace and orchestrates the DAG via `dependsOn` — an
upstream workspace's `Converged` is the downstream's gate.

---

## II. Reconcile state machine

```
       ┌────────────────────────────────────────────────────┐
       ▼                                                    │
   Pending ─────► Synthesizing ─────► Planning             │
                                          │                 │
                                  ┌───────┴────────┐        │
                                  ▼                ▼        │
                             PlanClean       PlanDrift      │
                                  │                │        │
                            (no apply           Awaiting    │
                             needed)          (manual or    │
                                  │           policy gate)  │
                                  │                │        │
                                  ▼                ▼        │
                             Converged ◄────── Applying     │
                                  │                │        │
                                  │                ▼        │
                                  │           ApplyFailed   │
                                  │                │        │
                                  ▼                ▼        │
                              (drift loop)     (alert +     │
                                              backoff)      │
                                                            │
       (next reconcile interval) ───────────────────────────┘
```

Every transition is attested (BLAKE3 hash of inputs+outputs into the
status field). Status conditions are first-class kubernetes
`metav1.Condition`s — `kubectl get pangeaworkspace -o yaml` shows the
full chain.

---

## III. Auth flow

### The problem

Each pangea workspace authenticates against different external systems:

| Workspace | Needs |
|-----------|-------|
| `pleme-io-opensource` | GitHub App token (write to pleme-io org) |
| `pleme-io-tailnet` | Tailscale OAuth client + AWS profile |
| `kazoku-vpc` | AWS profile (akeyless-development) |
| `cloudflare-pleme` | Cloudflare API token |
| `splunk-akeyless` | Splunk HEC token |

We can't have laptop-time `AWS_PROFILE=...` env var dances in the
operator. Auth flows differently:

### The flow

```
1. Operator secrets live in Akeyless under /pleme-io/pangea-operator/...
2. ExternalSecret (External Secrets Operator) syncs those into a
   namespace-scoped Kubernetes Secret on the rio cluster.
3. PangeaWorkspace.spec.auth.secretRefs[] names the Secret keys + which
   env var the operator should set when invoking tofu for that workspace.
4. The reconciler builds a per-workspace env from the Secrets, injects
   it into the tofu subprocess, and never logs the values.
```

The Akeyless creds the operator itself uses to talk to ESO are
bootstrap creds (a single privileged AccessKey rotated quarterly).
Everything else flows through that one trust anchor.

### Secret rotation through cofre

`cofre` (the typed secret materialization tool) generates the
operator's bootstrap creds and any per-workspace credentials we own.
`cofre apply --manifest pangea-operator-secrets.yaml` is the only
sanctioned path; rotation is `cofre apply --rotate <name>`.

---

## IV. Attestation gate

### What it gates

The operator's apply phase is conditional on a fresh
`Attested<RenderedOutput>` from the workspace's pangea synth:

```rust
// arch-synthesizer/src/attested.rs (already exists)
pub struct Attested<T> {
    inner: T,
    proof: ComplianceProof,
    attestation: RenderAttestation,
}
```

Pangea-operator's reconciler:

1. Runs `pangea synth` → produces `terraform.tf.json` + `attestation.json`.
2. Loads `attestation.json`, verifies its BLAKE3 chain (the same
   `verify_integrity` arch-synthesizer's `Attested::seal` requires).
3. Refuses to call `tofu apply` if attestation is missing, expired
   (`maxAgeSeconds`), or `verify_integrity` fails.
4. On successful apply, records the attestation hash in `status.lastApply.revision`.

### Why this matters

Without the gate, anyone with apply rights could push a hand-edited
`tf.json` into the workspace and the operator would converge it. With
the gate, the only path to a successful apply is through the typed
arch-synthesizer rendering pipeline — which is in turn gated by the
Rust compiler + `cargo test` + the `verify_compliance` constructor.

The chain is total: invalid Ruby that produces invalid `tf.json` →
arch-synthesizer's compliance gate fails → no `Attested<T>` is sealed
→ operator refuses to apply.

---

## V. Drift detection + remediation

### Detection

The reconcile loop runs `pangea plan` every `spec.reconcile.interval`.
When the plan output is non-empty and the workspace is in `Converged`
state, the operator transitions to `Drifted`.

### Causes (with assigned recovery)

| Cause | Recovery |
|-------|----------|
| Engineer pushed a change but FluxCD hasn't reconciled the source git repo yet | wait for next reconcile (no action) |
| External system mutated state behind tofu's back (e.g. someone clicked in the GitHub UI) | re-apply the spec OR re-import depending on `spec.drift.onDrift` |
| The workspace's auth credentials rotated and tofu can't refresh state | alert; reconcile fails fast |
| Provider-version drift (terraform-provider-github bumped from v6.10 to v6.11) | re-apply if version pin is permissive; alert if not |

### Remediation modes

- **`alert`** — log a status condition + fire alertmanager event;
  human investigates. Default for production-critical workspaces.
- **`auto-remediate`** — re-apply the spec to overwrite drift. Default
  for posture workspaces (org repos, branch protection) where the
  declared spec is the law.

### The pleme-io-opensource case (today's example)

The imperative `nix run .#flow-deploy-pleme-io-opensource` of
2026-04-29 left 5 GitHub-default labels in a "tofu wants to create
them but they exist already" loop. Once pangea-operator owns this
workspace:

- Its first reconcile detects the drift (5 labels need import).
- With `onDrift: auto-remediate`, the operator runs `tofu import`
  for each colliding label, then re-applies. State converges. No
  human action.

---

## VI. Compliance webhook

A validating admission webhook refuses `PangeaWorkspace` CRs whose
referenced workspace fails compliance gates:

- The workspace must be a known directory under
  `pangea-architectures/workspaces/`.
- Its `pangea synth` must succeed (compile-time validation passes).
- The `Attested<RenderedOutput>` produced by synth must seal cleanly.
- For `pleme-io-opensource` specifically: every public repo block
  must satisfy `OpenSourceRepo.compliance_gate` (OSI license,
  description, branch protection profile).

The webhook runs `pangea synth --check-only` against the source git
ref at admission time. Rejects on any failure with a structured error
the operator can fix.

---

## VII. RBAC

Pangea-operator is cluster-scoped (it manages CRs in any namespace
that maps to a workspace). RBAC:

- `ClusterRole/pangea-operator`:
  - read/write on `pangea.pleme.io/PangeaWorkspace`,
    `pangea.pleme.io/PangeaApplication`
  - read on `core/Secret` (for auth resolution)
  - write on `events.k8s.io/Event` (for status events)
- `ClusterRole/pangea-workspace-author` (granted to humans + bot
  service accounts that may create CRs):
  - create/update/get/list on PangeaWorkspace + PangeaApplication
  - no Secret access (auth lives in CRs as references, never inline)

---

## VIII. Helm chart shape

Following pleme-io conventions
([`pangea-architectures/CLAUDE.md` §Helm-first authoring](../pangea-architectures/CLAUDE.md)),
pangea-operator ships as a `lareira-pangea-operator` umbrella chart in
`pleme-io/helmworks` consuming `pleme-lib`:

```
helmworks/charts/lareira-pangea-operator/
  Chart.yaml              # depends on pleme-lib + pleme-operator
  values.yaml             # operator image, log level, reconcile defaults
  templates/
    deployment.yaml       # via pleme-operator's library template
    rbac.yaml             # cluster role + binding
    crds/                 # PangeaWorkspace + PangeaApplication CRDs
    webhook.yaml          # admission webhook for compliance gate
    servicemonitor.yaml   # via pleme-lib (mandatory observability)
    prometheusrule.yaml   # alert rules (mandatory per Pillar 11)
```

The chart's `values.yaml` exposes a minimal opinionated surface; the
rest stays inside the umbrella per the Helm-first authoring rule.

---

## IX. FluxCD integration on rio

A HelmRelease in `k8s/clusters/rio/infrastructure/pangea-operator/`
deploys the chart. The operator's own auth secrets land via a
`blackmatter.components.secrets` declaration on rio's NixOS config —
which then renders an ExternalSecret resource into the cluster, which
ESO syncs from Akeyless into the operator's namespace.

```
sops + akeyless-nix on rio
       │
       ▼
ExternalSecret (k8s manifest, FluxCD-managed)
       │
       ▼
External Secrets Operator
       │
       ▼
core/Secret (rio:pangea-system:akeyless-bootstrap)
       │
       ▼
pangea-operator pod (env)
```

---

## X. Migration path

### Phase 6c — first workspace (pleme-io-opensource)

1. Land the operator (Helm chart on rio).
2. Author a `PangeaWorkspace/pleme-io-opensource` CR.
3. The operator's first reconcile detects the standing label drift
   (from the imperative deploy of 2026-04-29) and auto-remediates.
4. Retire the `flow-deploy-pleme-io-opensource` flake app → it now
   only generates the CR YAML for human inspection.

### Phase 6d — single-state workspaces

Each pangea workspace under
`pangea-architectures/workspaces/<name>/` gets one CR. Order:
posture-only workspaces first (low blast radius), cluster workspaces
later (high blast radius, want more confidence first).

### Phase 6e — DAG workspaces

Convert `fleet.yaml` flows to `PangeaApplication` CRs. Cross-workspace
state references continue to use `RemoteState.output()` — that's a
synthesis-time read, doesn't need controller awareness.

### Phase 6f — retire laptop-side flows

Once every workspace has a CR, the only `nix run .#flow-deploy-...`
left is `flow-deploy-pangea-operator` itself (the bootstrap). That
one stays, since you can't deploy the controller through the
controller before the controller exists.

---

## XI. Failure modes + recovery

| Failure | Detection | Recovery |
|---------|-----------|----------|
| Operator pod crashes mid-apply | Reconcile loop on next pod start; tofu state lock detects unfinished apply | Operator retries; if state is locked by dead PID, force-unlock with explicit operator action (CR annotation `pangea.pleme.io/force-unlock: <reason>`) |
| Auth secret missing | Reconcile fails with `AuthMissing` condition | Human creates ExternalSecret; reconcile retries automatically |
| Workspace synth fails | Reconcile fails with `SynthFailed` condition; status shows error | Engineer fixes the workspace source; FluxCD picks up the fix; next reconcile recovers |
| Attestation expired (older than `maxAgeSeconds`) | Reconcile re-runs synth | Auto-recovery on next reconcile |
| GitHub provider 422 (label collision, etc.) | tofu apply error in status; condition `ApplyFailed` | If `auto-remediate`, operator runs `tofu import` for the collision then re-applies |
| Cross-workspace dependency stuck | `PangeaApplication.status.workspaceStates.<dep>` not Converged | Investigate the upstream workspace; downstream waits |

---

## XII. Operator UX

### Today (imperative)

```bash
$ cd ~/code/github/pleme-io/pangea-architectures
$ nix run .#flow-plan-pleme-io-opensource
$ nix run .#flow-deploy-pleme-io-opensource
```

### After Phase 6f

```bash
$ git push                        # via PR + merge to pleme-io/pangea-architectures
# (FluxCD picks up; operator reconciles; status visible via:)
$ kubectl get pangeaworkspace -A
NAMESPACE        NAME                    READY   STATUS      LAST APPLIED
pangea-system    pleme-io-opensource     True    Converged   2026-04-29T03:14:15Z
pangea-system    pleme-io-tailnet        True    Converged   2026-04-29T03:14:00Z
pangea-system    kazoku-vpc              True    Converged   2026-04-29T03:13:00Z

$ kubectl describe pangeaworkspace pleme-io-opensource
# full reconcile chain: source git ref, last synth hash, last plan,
# last apply, drift state, attestation hash, conditions ladder
```

The operator's auth never leaves the cluster. No tofu binary on the
laptop. No `AWS_PROFILE` dance. Engineer touches code, CR converges.

---

## XIII. Open questions (for Phase 6b/6c)

1. **Tofu execution model** — subprocess via `tatara-script` shellout, or
   embed via `tofu-rs` once that exists? Subprocess is simpler and
   matches the existing `Pangea::CLI::Operations.run_tofu` shape.
2. **Source git fetch** — operator clones via libgit2, or relies on
   FluxCD's GitRepository CR + Source controller? FluxCD path is
   cleaner (one source per pangea repo, multiple workspaces consume it).
3. **State locking for shared S3 backend** — DynamoDB lock table is the
   tofu-native answer; needs IAM access from the operator. Or, use
   the workspace's existing lock setup (most pleme-io workspaces
   already have DynamoDB locks).
4. **Rio vs other clusters** — operator runs on rio first; if a
   workspace needs cluster-local resources from a different cluster
   (e.g. `kazoku-helm` needs the kazoku cluster's API), the operator
   needs cross-cluster kubectl access. Bootstrap question: the
   workspace's `KUBECONFIG` is itself a Pangea-rendered output. Solve
   this at Phase 6e (DAG workspaces) — early phases dodge.
5. **Webhook in webhook-only mode vs full controller in same pod** —
   simpler to ship one Deployment with both responsibilities. Split
   in a future hardening phase.
6. **Compliance webhook auth to source repo** — webhook runs `pangea
   synth --check-only` against a git ref it doesn't have a checkout
   of. Either have the webhook fetch the source itself, or precompute
   the synth as part of the source repo's CI and let the webhook read
   a published artifact. Phase 6b decision.

---

## XIV. Theory linkage

- [`THEORY.md` §III](./THEORY.md) — render state dimension; pangea-operator
  IS the controller for the render-state lattice point.
- [`THEORY.md` §IV.2](./THEORY.md) — fixed-point controllers; pangea-operator
  is a new entry in the controller catalog (line up alongside FluxCD
  HelmRelease, Sui agent, kenshi TestGate, shinka Migration).
- [`THEORY.md` §V](./THEORY.md) — three-pillar attestation; pangea-operator
  consumes Phase-1 signatures (compile-time `Attested<T>`) and produces
  Phase-2 signatures (runtime apply success → status hash).
- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
  Pillar 11 (JIT Infrastructure) — pangea-operator IS pleme-io's
  generic IaC convergence machine; specializes the JIT primitives for
  the Ruby+tofu render chain.
- `arch-synthesizer/src/convergence_platform.rs` — the typed model the
  operator reifies.

---

## XV. What changes vs what stays

| Stays | Changes |
|-------|---------|
| `arch-synthesizer` is still where typescape lives | The synthesizer's output now feeds a controller, not a laptop |
| `pangea-architectures/workspaces/` is still the source of truth | A `PangeaWorkspace` CR per workspace lives alongside the workspace dir |
| `cofre` is still the way secrets are born | The operator's bootstrap creds are cofre-generated |
| `Attested<T>` is still the seal | The operator is the consumer of the seal |
| `repo-forge` + `org.yaml` still own posture | Their deploy path now goes through the operator |
| `tofu` is still the apply engine | It runs in-cluster, not on the laptop |
| FluxCD is still the cluster GitOps engine | It now reconciles the operator + the operator reconciles everything else |

The thesis is unchanged. The operator is just the missing controller
in the existing convergence-as-process model.
