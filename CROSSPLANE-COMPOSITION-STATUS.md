# Crossplane Composition typed status reflection — the canonical pattern

> **Frame.** Every Crossplane Composition that wants its operator-facing
> XR `status` to reflect the actual outcome of the function it
> implements (verify-pass/fail, post-cleanup state, retrieved-data,
> etc.) — not just Crossplane's coarse `Ready=True/Synced=True` —
> follows the **two-step Composition pipeline**: Resources step emits
> composed resources; XR step reads a result-CM observer and writes
> typed status fields back to the XR. This document is the canonical
> spec. Read once; cite from repo CLAUDE.md.

---

## I. Why the gap exists

A naïve Crossplane Composition produces composed resources and lets
Crossplane manage two stock conditions on the XR:

- `Synced=True/False` — function pipeline ran without error
- `Ready=True/False` — every composed resource is itself Ready

Both are about the *substrate* (did Crossplane converge?), not about
the *function the Composition implements* (did the verify Job find the
canary in the restored env? did cleanup tear down all 8 resources?).

For a Composition representing a typed function — `(args) → result` —
the operator wants the result on `kubectl get <xr> -o yaml` without
having to dig into per-correlation ConfigMaps written by interior
Jobs. **That's the gap typed status reflection closes.**

---

## II. The pattern

```
                  ┌───────────────────────────────────┐
                  │ XR (e.g. PITRSession)             │
                  │   status:                         │
                  │     phase: Succeeded              │ ← what operator
                  │     retrievedSecrets: [...]       │   sees
                  │     correlationId: drill-...      │
                  │     cleanupStatus: Completed      │
                  └─────────────▲─────────────────────┘
                                │  (server-side apply)
                                │
        ┌───────────────────────┴────────────────────┐
        │ Composition pipeline (function-kcl × 2)    │
        │                                            │
        │  step orchestrate    target: Resources     │
        │    → emits items[]: RDS Instance MRs,      │
        │      Object MRs (CMs/Deploys/Jobs),        │
        │      result-cm-observer Object MR          │
        │                                            │
        │  step reflect-status target: XR            │
        │    → reads ocds[result-cm-observer-…]      │
        │    → emits items[0] = XR with new status   │
        └────────────────────────────────────────────┘
                                ▲
                                │  (CM data via ocds)
                                │
                  ┌─────────────┴─────────────────────┐
                  │ result-cm-observer ConfigMap      │
                  │  (in workload's namespace)        │
                  │  data:                            │
                  │    phase: Succeeded               │
                  │    retrievedSecrets: '["..."]'    │
                  │    cleanup_status: Completed      │
                  └─────────────▲─────────────────────┘
                                │  written by
                  ┌─────────────┴─────────────────────┐
                  │ verify Job, cleanup Job, etc.     │
                  │ (composed by orchestrate step)    │
                  └───────────────────────────────────┘
```

---

## III. Why two steps (not one)

function-kcl's `target: Resources` and `target: XR` are mutually
exclusive per pipeline step:

- `target: Resources` — KCL output `items` becomes the desired
  composed resource set; output `dxr` is **ignored**
- `target: XR` — KCL output `items[0]` becomes the desired XR
  (status patch source); output `dxr` is treated as a literal
  field on the XR (rejected by SSA: "field not declared in
  schema")

A Composition that wants both must use two steps. Sharing source
between them fails because the Resources-step KCL emits items[]
of composed resources, and the XR step's path traversal trips on
field names with leading dots (e.g., `.dockerconfigjson` in
image-pull Secrets).

---

## IV. The contract — six items

Every XR-mode KCL fragment MUST satisfy these six rules. Violating
any one wedges the Composition (Synced=False) and prevents any work.

### 1. Output shape: `items = [{XR-with-status}]`

```kcl
items = [{
    apiVersion = _oxr.apiVersion
    kind       = _oxr.kind
    metadata   = { name = _oxr.metadata.name }
    status     = _new_status
}]
```

NOT a top-level `dxr = {...}` keyword. function-kcl passes top-level
`dxr` through as a literal field name; SSA rejects with `.dxr: field
not declared in schema`.

### 2. No composed resources in items[]

The XR-step's KCL must NOT emit additional items[] (no RDS
Instances, Object MRs, etc.). Only items[0] = the XR. Otherwise
SSA path traversal trips on field names with leading dots.

### 3. Cherry-pick metadata.name only

```kcl
metadata = { name = _oxr.metadata.name }
```

NOT `metadata = _oxr.metadata`. Wholesale copy brings managedFields
along; SSA rejects with `metadata.managedFields must be nil`.

### 4. XRD declares every status field

For every key in your `status = {...}` block, the XRD must have a
matching entry under
`spec.versions[].schema.openAPIV3Schema.properties.status.properties`.
Missing fields fail with `.status.<field>: field not declared in
schema`.

### 5. Skip conditions[] (or pass time as param)

KCL has no stdlib `now()`. Kubernetes' built-in metav1.Condition
validator requires `lastTransitionTime` regardless of the XRD
schema. Either:
- **Skip** — let Crossplane manage Ready/Synced; expose domain
  outcome via `status.phase` instead
- **Inject time as KCLInput param** — operator must set this at
  chart-render time; not deterministic across reconciles

### 6. Result-CM observer pattern

The XR-step reads its inputs from `option("params").ocds[<name>]`
where `<name>` is the composed-resource-name of a
provider-kubernetes Object MR observing a ConfigMap. The
Resources-step emits that observer; the workload (verify Job,
cleanup Job, etc.) writes the CM. The CM data shape is the
**typed protocol** between the substrate-side function and the
operator-facing status.

---

## V. Worked example

`akeylesslabs/akeyless-environments/saas/kubernetes/helm/pitr-akeyless`
chart 0.9.21:

- `templates/composition.yaml` — both pipeline steps:
  - `orchestrate-drill` (target: Resources) — `kcl/pitr-composition.k`
  - `reflect-xr-status` (target: XR) — `kcl/pitr-status.k`
- `kcl/pitr-status.k` — 80 lines:
  - Reads `ocds["result-cm-observer-{correlation}"]`
  - Extracts `phase`, `retrievedSecrets`, `cleanup_status`
  - Computes `correlationId` + `restoreNamespace` from `oxr.metadata.uid`
  - Emits `items[0]` = PITRSession with new `status`
- `templates/xrd.yaml` — declares `phase`, `correlationId`,
  `restoreNamespace`, `retrievedSecrets`, `cleanupStatus`

**Validated:** drill #23 (correlation 2b70f82b) — `status.phase`
populated as `Restoring` within ~30s of PITRSession apply, before
any RDS instance came up; transitions to `Succeeded` once verify
writes its outcome.

**Iteration cost to derive the contract:** 5 chart bumps + 5
wedged drills (#18-#22). Captured in
`pleme-io/theory/CROSSPLANE-COMPOSITION-STATUS.md` (this doc) +
4 memory entries so the next Composition pays zero re-derivation
cost.

---

## VI. Substrate primitive

`pleme-io/helmworks/charts/pleme-lib` v0.14+ ships
`pleme-lib.composition.statusReflectStep` — a Helm helper that
emits the `target: XR` pipeline step with a consumer-supplied
KCL source string, encapsulating the YAML scaffold. Consumer
charts include it as one line:

```yaml
{{- include "pleme-lib.composition.statusReflectStep" (dict
      "step" "reflect-xr-status"
      "kclSource" (tpl (.Files.Get "kcl/status.k") .)) }}
```

The KCL fragment itself is consumer-authored (status fields are
domain-specific); the SCAFFOLD + the 6-item contract becomes
substrate.

---

## VII. Anti-patterns

- **One KCL file shared between Resources + XR steps.** Triggers
  the SSA path-parse error on any composed Object MR with a
  Secret-data key starting with `.`.
- **`dxr = {...}` at top level.** Function-kcl passes it through
  as a literal field name on the XR; rejected by SSA.
- **Wholesale `metadata = _oxr.metadata` copy.** Brings
  managedFields; rejected.
- **Adding status fields to KCL without the matching XRD bump.**
  Wedges Composition until both ship together.
- **Emitting custom conditions[] without lastTransitionTime.**
  K8s metav1.Condition validation rejects independent of XRD.

---

## VIII. Forward work

- **Future cse-lint check** (pleme-io/cse-lint) — parses every
  Composition's pipeline; for each `target: XR` step, validates
  the embedded KCL against rules 1-4 of the contract. Catches
  all five drill #18-22 errors at chart-render time instead of
  via wedged drills. Tracked.
- **Function-pleme-status-reflect** (Wasm Composition Function)
  — eventual replacement for the 2nd function-kcl step. Takes
  a typed config (result-CM-name + field-mapping), emits the
  XR status patch directly. Removes the need for hand-authored
  KCL fragments. Larger investment.

---

## IX. Citation

When a new Composition wants typed status reflection, cite this
doc + apply the pleme-lib helper. Repo CLAUDE.md should reference
this from a `## Status reflection` section if the chart uses the
pattern.
