# Live deployment — shift the sand while it runs

> **The user, 2026-04-26**: *"with lisp we can actually host
> something and shift the sand under it and preserve it all probably
> while it runs"* + *"it can all be triggered by straight up git
> hashes."*
>
> Combine Lisp's classical hot-code-reload tradition (Erlang/Smalltalk
> lineage) with the WASM/WASI runtime + git-content-addressed packaging
> ([`TATARA-PACKAGING.md`](TATARA-PACKAGING.md)) and the
> deployment-as-git-push model becomes natural: **a git hash is the
> deployment trigger**, the running program **preserves state** across
> the swap, and **rollback is just resolving an older hash**.

---

## I. What hot-replacement means here

```
                       PROGRAM RUNNING (version A)
                          state: in-memory + PVC
                                │
                ┌───────────────┼────────────────┐
                ▼ git push      ▼ FluxCD detects ▼ wasm-operator hot-replace
        ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐
        │ git tag     │  │ ComputeUnit │  │ - instantiate version B     │
        │ v0.2.0 →    │  │ source: ... │  │ - migrate state A → B       │
        │ <hash>      │  │ ?ref=v0.2.0 │  │ - drain in-flight requests  │
        └─────────────┘  └─────────────┘  │ - swap dispatch routing     │
                                          │ - reap version A             │
                                          └─────────────────────────────┘
                                │
                                ▼
                       PROGRAM RUNNING (version B)
                          state: preserved + new schema applied
```

**Zero downtime** for in-flight requests. **State preserved** across
the swap. **Git hash is the trigger** — no separate deploy command.
**Rollback** is changing the `?ref=` qualifier back to the previous
hash; the operator hot-replaces in reverse.

## II. Four levels of preservation

The exact preservation guarantee depends on what state the program
keeps. Each level requires more cooperation between the old and new
modules:

### Level 0 — Fully stateless

The new module instantiates; KEDA HTTP starts routing new requests
to it; in-flight requests on the old module finish; old module
exits. **No state to migrate.** This is the easy case — most cookbook
programs (PVC autoresizer, dns-reconciler, github-webhook-flux) live
here.

### Level 1 — In-process state, schema-stable

Module keeps a hashmap or counter in memory. The new module exports
a `(receive-state old-state)` hook; the operator copies the
serialized state from the old module's `(snapshot)` output into the
new module's import. **Same schema, different code.** Most
controllers (where the state is "what CRs have I seen?") fit here.

```lisp
;; In version A (and later versions following the same protocol):

(defprogram my-controller
  :version "v0.2.0"

  ;; Called by the operator before instantiating B.
  :snapshot
  (lambda ()
    (json-stringify
      (list (list "seen-crs" *seen-crs*)
            (list "last-tick" *last-tick*))))

  ;; Called by the operator on B right after instantiating.
  :receive-state
  (lambda (json-state)
    (let* [parsed (json-parse json-state)]
      (set! *seen-crs*  (alist-get parsed "seen-crs"))
      (set! *last-tick* (alist-get parsed "last-tick"))))

  ;; Normal program body — uses *seen-crs* etc.
  ...)
```

### Level 2 — In-process state, schema-evolving

Version B has a different state shape than A. The new module's
`(receive-state)` hook must understand the old shape and migrate it.
Pattern: ship migrations alongside the code.

```lisp
;; In version B:
:receive-state
(lambda (json-state)
  (let* [parsed (json-parse json-state)
         migrated (migrate-from-version (alist-get parsed "version") parsed)]
    (set! *state* migrated)))

(defn migrate-from-version [version data]
  (cond
    (= version "v0.1") (migrate-v01-to-v02 data)
    (= version "v0.2") data
    :else (panic (string-append "unknown state version: " version))))
```

This is the same shape Erlang's `code_change/3` callback has. Lisp
makes it ergonomic because the state IS data; transformations are
function calls.

### Level 3 — Persistent state with continuations

For long-running operations that span the swap (e.g., a saga waiting
on an external API call), the program serializes its **continuation**
— what code to run next + what state to bring along — to a CR
status field or shared PVC. The new module reads the continuation
and resumes.

```lisp
(define-saga my-multi-step
  :step-1 (lambda () (cf-create-record ...))
  :step-2 (lambda (record-id) (wait-for-dns-propagation record-id))
  :step-3 (lambda (record-id) (notify-completion record-id)))

;; If swap happens while we're waiting in step-2, the operator
;; serializes (state, continuation) → CR status; the new module
;; reads it back + resumes step-2 with the same `record-id`.
```

This is the strongest preservation: **even mid-operation work
survives** a code swap. Requires the program to use the saga
primitives or explicit continuation-passing — not free, but
tractable.

## III. Git hash as the deployment trigger

Per [`TATARA-PACKAGING.md`](TATARA-PACKAGING.md), every program is
URL-addressed:

```
github:pleme-io/programs/<name>/main.tlisp?ref=<hash-or-tag>
```

The trigger surface:

```
1. Author pushes commit:    git push pleme-io/programs main
2. (Optionally tags a release: git tag v0.2.0 && git push --tags)
3. Operator's reference hash:
   - In FluxCD: clusters/<name>/programs/release.yaml has the ref pinned;
     bumping it = deploy the new commit.
   - In wasm-operator: ComputeUnit.spec.module.source's ?ref=
     resolves to the new BLAKE3.
4. wasm-operator detects the BLAKE3 change → hot-replaces.
5. Old module drains; new module takes over.
```

**No separate `kubectl rollout restart` step**, no Docker image
build CI, no version-bump-then-deploy. The hash IS the trigger; the
operator does the rest.

### III.1 Floating refs vs pinned refs

```yaml
# Floating — picks up every push to main:
source: github:pleme-io/programs/hello-world/main.tlisp?ref=main

# Pinned to a tag — operator deploys new ref by editing this:
source: github:pleme-io/programs/hello-world/main.tlisp?ref=v0.2.0

# Pinned to a commit — strongest reproducibility:
source: github:pleme-io/programs/hello-world/main.tlisp?ref=abc123def
```

Production clusters use pinned refs (tag or commit). Dev clusters
can float on `?ref=main` for continuous deploys.

### III.2 The cluster-level revision tracker

A cluster's `clusters/<name>/programs/release.yaml` lists every
program with its `?ref=`. **Rolling out a fleet upgrade** is bumping
the refs in this one file:

```diff
- - { name: hello-world, module: { source: github:.../hello-world/main.tlisp?ref=v0.1.0 }, ... }
+ - { name: hello-world, module: { source: github:.../hello-world/main.tlisp?ref=v0.2.0 }, ... }
- - { name: dns-reconciler, module: { source: github:.../dns-reconciler/main.tlisp?ref=v0.3.0 }, ... }
+ - { name: dns-reconciler, module: { source: github:.../dns-reconciler/main.tlisp?ref=v0.3.1 }, ... }
```

PR + merge + FluxCD reconciles + wasm-operator hot-replaces both.
**Two programs, one PR, zero downtime.**

### III.3 Rollback

Change the `?ref=` back to the previous hash. The operator
hot-replaces in reverse, copying the new module's snapshot back
into the old module's `receive-state`. **Rollback is the same
mechanism as forward-deploy** — by symmetry there's no separate
"oh no roll back" code path.

## IV. The wasm-operator's hot-replace algorithm

```
trigger:
  ComputeUnit.spec.module.source changed (resolver hits new BLAKE3)
  OR  spec.module.blake3 changed
  OR  spec.config changed and module.opts.restart-on-config-change

algorithm:
  let old = current running engine pod for this CU
  if old has snapshot capability:
    serialized_state = old.invoke('snapshot')
  else:
    serialized_state = nil

  let new = instantiate engine pod with new BLAKE3
  if new has receive-state capability AND serialized_state is not nil:
    new.invoke('receive-state', serialized_state)

  if shape == service:
    update Service selector to point at new pod
    drain old: stop accepting new requests, wait for in-flight to complete
              (max grace-period from CU spec)
    reap old
  if shape == controller:
    transfer leader-election lease from old to new
    reap old
  if shape == job:
    let next-cron-tick handle the swap (no in-flight to migrate)
    reap old after current tick
  if shape == function (event-driven):
    drain old: process inflight batch, no new dispatch
    new starts subscribing
    reap old
  if shape == oneshot:
    n/a — no swap during oneshot run; re-trigger needed
```

The operator owns this state machine. Programs only need to
implement the `:snapshot` + `:receive-state` hooks if they care
about state preservation; the no-state path requires zero program
cooperation.

## V. The Lisp advantage

Why this is much harder with containers + much easier with Lisp+WASM:

| Property | Container (Year-1 shape) | Lisp + WASM (Year-2) |
|---|---|---|
| Cold start | ~30 s (image pull + container init) | ~3 s (wasmtime instantiate) |
| Two versions running side-by-side | requires double-pod allocation; PVC-bound state can't share | wasm-engine pod runs both modules in same process; shared memory transparent |
| State serialization | requires the program to ship its own | `(snapshot)` is a Lisp value → built-in serialization |
| Schema migration | bespoke per service | `(migrate-from-version)` is just a Lisp function |
| Continuation save/restore | not available | natural in Lisp; saga macro exposes it |
| Rollback | reverse rolling restart | resolver hits older hash; operator hot-swaps in reverse |
| Audit trail | git log | git log (same — both are content-addressed) |

The Lisp tradition (Erlang's hot code load; Smalltalk's image
persistence; Common Lisp's runtime redefinition) was always
*architecturally cleaner* than the recompile-redeploy model. WASM
made it deployable; pleme-io combines them.

## VI. The full operator-side surface

```yaml
# Hypothetical ComputeUnit fields supporting hot-replace:

apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata: { name: my-program }
spec:
  module:
    source: github:pleme-io/programs/my-program/main.tlisp?ref=v0.2.0
    # Override: hot-replace policy.
    upgradePolicy:
      strategy: hot-replace        # or: recreate, manual, blue-green
      gracePeriod: 30s             # max time to drain in-flight on old
      preserveState: schema-evolving  # off, stable, schema-evolving, continuation
      rollbackOnSnapshotFailure: true # if (snapshot) fails, refuse to swap
      attestationRequired: true    # tameshi chain must verify before instantiating new
```

Each field has a mechanical default:
- New module: `upgradePolicy.strategy: hot-replace` for service/controller/function;
  `recreate` for program (one-shot can't swap mid-run).
- `gracePeriod: 30s`.
- `preserveState: stable` for stateless programs (most); operators
  upgrade to `schema-evolving` when their program needs it.
- `rollbackOnSnapshotFailure: true`.
- `attestationRequired: true` once Tameshi attestation is fleet-wide.

## VII. Trigger the deploy via git directly

Beyond FluxCD, programs can subscribe to git events themselves:

```lisp
;; A long-running controller that watches its own source repo:

(defprogram self-updating-program
  :version "v0.1.0"
  :git-watch
  {:repo "pleme-io/programs"
   :path "<name>/main.tlisp"
   :on-new-commit
   (lambda (new-hash)
     ;; Operator fires hot-replace via patching our own ComputeUnit:
     (k8s/patch-cr (k8s/own-cr)
                   {:spec {:module {:blake3 new-hash}}}))}
  ...)
```

The program self-watches; pushing a new commit upstream triggers a
self-replace. **GitOps-as-the-program**, not GitOps-as-an-operator.
Pattern reusable for any program that wants to mirror a git source.

## VIII. The sand under the running program — a concrete example

Stateful service: the OpenAPI codegen registry from
[`TATARA-CODEGEN.md`](TATARA-CODEGEN.md). It maintains a cache of
parsed specs. Operator publishes v0.2 with a different cache schema:

```
1. v0.1 running, in-memory cache populated.
2. operator pushes v0.2 with new cache schema.
3. wasm-operator:
   a. v0.1.snapshot() → JSON {"version":"v0.1", "cache": {...}}
   b. instantiate v0.2 module
   c. v0.2.receive-state(snapshot)
      → migrate-from-version detects "v0.1" → migrate-v01-to-v02
      → set new schema
   d. swap traffic
   e. reap v0.1
4. New requests served by v0.2 with the migrated cache.
   No re-fetch of the OpenAPI specs; cache hot-restored.
```

Operator-visible: a 30-second deploy where the cache stayed warm.
**Sand shifted; structure preserved.**

## IX. Phasing

- **Phase A (designed — this doc):** the hot-replace contract +
  4 preservation levels.
- **Phase B:** wasm-operator implements the algorithm in §IV.
  Requires:
  - Snapshot/receive-state hook protocol (defined in
    `wasm-types::ProgramHooks`).
  - Lease transfer for controller-shape.
  - Drain logic for service-shape.
- **Phase C:** tatara-lisp `(defprogram :snapshot ... :receive-state ...)`
  syntactic sugar — auto-marshalls to JSON via the existing serde stack.
- **Phase D:** continuation-aware saga macro (level 3 preservation).
- **Phase E:** self-watching git pattern (§VII).

## X. The closing principle

**Deployment is not separate from coding.** The act of `git push`
to a tagged ref is the deploy. The cluster's behavior changes when
hashes change. **Operators don't deploy; they edit the URL.** The
runtime preserves state across changes when the program declares
how. Rollback is symmetric.

This is the strongest possible expression of the user's principle:
*"we send things to GitHub and we run from GitHub and that's it."*
The verb "deploy" doesn't need to exist as a separate operation;
publishing IS deploying when content addressing carries through to
the running runtime.

## XI. See also

- [`TATARA-PACKAGING.md`](TATARA-PACKAGING.md) — git+nix native packaging
- [`WASM-PACKAGING.md`](WASM-PACKAGING.md) — URL grammar for sources
- [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — 4-layer compute hierarchy
- [`FLEET-DECLARATION.md`](FLEET-DECLARATION.md) — cluster's program list
- [`EXTENSIBILITY.md`](EXTENSIBILITY.md) — push-to-GitHub-and-run principle
- [`WASM-PATTERNS.md` Pattern #48](WASM-PATTERNS.md) — hot replacement
- Erlang OTP `code_change/3` — the Lisp-tradition prior art
- Smalltalk image persistence — the deepest prior art
