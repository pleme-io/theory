# Runtime patterns enabled by the caixa substrate

> **Frame.** [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) describes the
> compute hierarchy. [`LIVE-DEPLOYMENT.md`](./LIVE-DEPLOYMENT.md) shows
> the canonical hot-replace path. [`WASM-RUNTIME-COMPLETE.md`](./WASM-RUNTIME-COMPLETE.md)
> closes the runtime loop. **This doc catalogs the patterns the
> substrate enables once those primitives are load-bearing** — what
> becomes *possible*, organized by pattern family, with status flags
> and substrate references. Audience: future operators / agents who
> want to know what they can ask the substrate to do.
>
> Status legend:
>
> - **★ load-bearing** — works today, exercised in the fleet
> - **◐ partial** — primitive exists but pattern is one-step-removed
> - **○ aspirational** — substrate has the bones, code missing
>
> Every pattern below cites the substrate primitive(s) it relies on —
> if the primitive's path looks like `caixa-X` or `lib/Y` you can read
> it. None of these patterns require new architecture; they recombine
> what's already typed.

---

## I. Why these patterns exist

Traditional containers are *opaque processes*: code + state coupled at
build time, deploy = image swap + restart, no introspection. The
caixa substrate decouples all four:

| concern | container-shaped | caixa-shaped |
|---|---|---|
| code | bytes in image | typed IR in caixa.lisp + module URL |
| state | pod-local memory | explicit (PVC, KV, capability tokens) |
| capabilities | unix uid/gid + filesystem | wasi:* component imports + token list |
| identity | container hash | BLAKE3 closure root via lacre |
| boundaries | os process | wasm component (`wasi:http/proxy`, etc.) |

Because each axis is named and typed, you can change one without
disturbing the others. That's what makes the patterns below tractable.

---

## II. Pattern catalog

### A. Live-deploy patterns — V → V+1 without disturbance

#### A1. Per-request fresh wasm instantiation ★

wasmtime's modern instantiate cost is sub-millisecond (Pulley-mode
< 0.1ms; full JIT 1-3ms). `wasm-engine` can spawn a new component
instance per request. Deploy = swap the source bytes; the *next*
request runs new code. No drain, no restart, no in-flight breakage.

- substrate: wasi-service-flake (Rust→wasm), tatara/program-flake (Lisp→wasm)
- runtime: wasm-engine instantiates per-request from a content-addressed cache

#### A2. Component-model swap (wasi:http/proxy) ◐

The wasi:http/proxy component model exposes named imports/exports.
wasm-engine can keep `wasi:io/streams@0.2.x` mounted but swap the
specific `wasi:http/incoming-handler` — i.e., update only the request
handler while preserving connection state, in-memory caches, prepared
DB statements. Pure WIT-typed swap.

- substrate: caixa-flux's bundle path (per-component HelmReleases)
- runtime: needs wasm-engine's mounted-imports cache (M2 deliverable)

#### A3. Tatara-script live function redefine (Lisp morph) ○

Lisp's classical hot-code-reload (Erlang OTP / Common Lisp `defun` /
Clojure REPL): `(eval '(defun handle-hello (req) ...))` redefines the
top-level binding in place. The next call to `handle-hello` sees the
new code. No process restart; in-flight closures carry old refs but
new calls hit the new binding.

- substrate: tatara-script's eval surface
- pattern: see [`LIVE-DEPLOYMENT.md`](./LIVE-DEPLOYMENT.md) §III
- prerequisite: tatara-script must expose an authenticated `(reload :caixa "<nome>")` form, gated by capability tokens

#### A4. Blue/green at the IR level ◐

Two ComputeUnits running in parallel (`hello-world-blue` + `hello-world-green`),
KEDA HTTPScaledObject splits traffic by `weight`. Once green is
healthy, blue's `weight: 0` makes it scale to zero, then GC. No DNS
flip, no L7 LB reconfigure — KEDA does it inside the cluster.

- substrate: lareira-fleet-programs (one programs.yaml entry per color)
- runtime: KEDA HTTPScaledObject `weight` field (already supported)

#### A5. Differential update via tatara-lisp form diff ○

Two caixa.lisp / `.tlisp` versions diffed at the **form level** (not
text level), since tatara-lisp parses to a typed Sexp tree. Only
changed `(defun …)` / `(defmacro …)` forms ship over the wire. The
runtime evals just those, leaving everything else unchanged.

- substrate: caixa-ast (Sexp diff), caixa-resolver (BLAKE3 of unchanged forms)
- pattern: candidate for a follow-up theory doc + caixa-diff crate

---

### B. Live-migration patterns — moving running compute

#### B1. Wasm component snapshot + resume ◐

wasmtime supports `wasmtime::Store` snapshot via `serialize` (linear-
memory + globals + table state). Snapshot a running pod, ship the
bytes to another node, resume. In-flight HTTP requests get a 502;
new connections land on the resumed instance. Useful for cordon /
drain operations that today require pod recreation.

- substrate: wasm-engine + a checkpointing controller
- prerequisite: wasm-engine API for checkpoint/resume (M2)

#### B2. Cross-cluster live migration via FluxCD ◐

Two clusters (rio + mar), same caixa, different `programs:` weights.
Bump weight on mar from 0→100 over a cooldown window; rio's pod
scales to zero. Stateful migration via PVC re-attach (cross-AZ if
storage class supports it).

- substrate: lareira-fleet-programs across clusters, caixa-flux's GitOps drives the weight delta
- runtime: external traffic distribution (Cloudflare load balancer or DNS weight)

#### B3. CRIU-checkpoint of the wasmtime parent ○

For non-wasm-native state (e.g. open file descriptors held by the
wasmtime process itself, kernel-level connections), CRIU dumps the
linux process and resumes it elsewhere. wasmtime is CRIU-compatible
when not using JIT-shared-mmap; Pulley mode preserves this property.

- substrate: would need a `wasm-engine-checkpoint` operator companion
- runtime: CRIU + container-checkpoint K8s feature gate (1.30+)

---

### C. Edge-update patterns — ship only the diff

#### C1. Library hot-reload propagation ○

A Biblioteca caixa releases v0.2.1 with a single bug-fix `(defun foo …)`.
Every Servico that depends on it gets the diff: the runtime's lacre
notices the BLAKE3 closure root changed, pulls only the changed forms
(via caixa-ast IR diff), evals them. No Servico rebuild; no Servico
restart.

- substrate: caixa-lacre (closure root), caixa-resolver, caixa-ast
- prerequisite: lacre's BLAKE3 chain becomes the wasm-engine's source-of-truth for "what's loaded right now"

#### C2. Capability-only update (no code change) ★

Only `caixa.lisp`'s `:capabilities` slot changes (e.g. add `prom-query@…`).
wasm-operator detects the spec drift, mints the new RBAC role, binds
it. Running pod reads new capabilities via downward API on next request;
no restart needed.

- substrate: wasm-operator's reconciler (today reconciles RBAC alongside ComputeUnit)
- runtime: K8s downward API (works today)

#### C3. Edge-only OCI patch via ostree-style overlay ○

Treat the ghcr.io OCI image as a layered FS; bump only the top layer
(the .wasm bytes, ~50KB) without re-downloading wasmtime + cacert
(~50MB). Modern OCI distribution supports this via partial fetch.

- substrate: substrate's wasi-service.nix already builds layered images
  (wasmtime + cacert as a heavy-cached layer; .wasm as a tiny top layer)
- consumer: kubelet's image GC + dockerd layer cache

---

### D. Resource-morphing patterns — shape changes without restart

#### D1. In-place pod resize (K8s 1.27+) ◐

`kubectl edit cu/hello-rio` → bump `spec.resources.limits.memory: 256Mi → 512Mi`.
wasm-operator notices, calls k8s in-place resize API, pod's cgroup
expands without restart. Useful for traffic spikes.

- substrate: wasm-operator's reconciler (M2 — needs `Patch` SSA against pod's `resources` subresource)
- runtime: K8s 1.27+ InPlacePodVerticalScaling feature gate

#### D2. Capability mutation at runtime ○

A wasm-engine process holds a capability table mapping token → backend
(e.g. `kube-secret-read@photos/s3-credentials` → a kube client w/ that
RBAC). Operator updates `:capabilities`; engine receives a signed
update message via control socket; revokes/grants entries in the
table. No restart, even for credential rotation.

- substrate: wasm-engine's capability table (M2 deliverable)
- pattern: matches Erlang's "code:purge / load_module" semantics

#### D3. Schema migration co-deploy with shinka ◐

A Servico's caixa.lisp declares `:schema-migrations ("migrations/")`.
The shinka migration-operator (separate CRD) sees the new
`Migration` CR rendered alongside the ComputeUnit, applies migrations
*before* the new pod takes traffic. Failed migration = ComputeUnit
stays at the old version.

- substrate: pleme-io/shinka, lareira-fleet-programs's defaults block
- runtime: ordering via FluxCD `dependsOn`

---

### E. Bootstrap / meta-programming patterns

#### E1. Self-modifying caixas ○

A Servico can author other Servicos. `(feira deploy …)` invoked from
inside a tatara-script process emits a new ComputeUnit. Useful for
fleet automations: a "tenant onboarding" Servico authors a per-tenant
caixa on signup.

- substrate: caixa-feira's deploy command, caixa-flux's HelmRelease
  upsert — both invokable from tatara-script via a typed wrapper
- runtime: needs a `kube-deploy@<cluster>` capability token

#### E2. Tatara-lisp REPL-on-pod ○

`kubectl exec -it cu/hello-rio -- feira repl` drops into a tatara-script
REPL connected to the live process state. Operator can `(inspect *)`,
`(eval '(...))`, redefine functions in place, then run `(feira save)`
which writes the diff back to caixa.lisp and opens a PR. Productionize
your debugging session.

- substrate: caixa-feira repl (NEW subcommand), tatara-script's eval
- pattern: matches CL's SLIME / Clojure's nrepl

#### E3. Speculative pre-deploy on PR ◐

When a PR is opened against a caixa repo, caixa-validate.yml triggers
caixa-publish to a *staging* namespace under a hash-suffix tag
(v0.2.0-pr42). The PR's CI comment links to the live URL. Reviewer
sees behavior before merge.

- substrate: caixa-publish.yml (small change: tag selection from PR ref)
- runtime: ephemeral namespace cleanup via TTL controller

---

### F. Schema / migration / replay patterns

#### F1. Replay-based rolling forward via tameshi BLAKE3 chain ○

Every ComputeUnit reconciliation emits a typed event (LacreEvent),
hash-chained via tameshi. Roll forward to a target hash = replay events
up to that hash; the operator deterministically rebuilds the cluster
state. Disaster recovery becomes "replay from genesis to <hash>".

- substrate: pleme-io/tameshi (BLAKE3 chain), caixa-lacre (typed events)
- runtime: needs a `replay` flag on the operator

#### F2. Time-travel debugging ○

Given the LacreEvent log, replay any historical state in a sandbox
cluster. Reproduce an incident with the *exact* ComputeUnit configs
and lacre-pinned source bytes from when it happened.

- substrate: tameshi + caixa-lacre + lareira-tatara-stack (mount lacre store as the source resolver)
- runtime: ephemeral cluster bringup via kindling / kasou

#### F3. Cross-version state migration via lacre-typed delta ○

Two lacre-pinned versions of a caixa carry their `:state-schema` (a
tatara-lisp form). The operator computes a typed delta and runs it as
a one-shot migration job before the new pod takes traffic.
Round-trippable: rollback = inverse delta.

- substrate: caixa-lacre (typed state schema), shinka migration-operator
- consumer: caixa.lisp authors `:state-schema` slot

---

### G. Multi-tenant / sharded patterns

#### G1. Program-per-tenant ◐

Each tenant gets a dedicated ComputeUnit with their config + capability
tokens. Spinning up a new tenant = adding one programs.yaml entry. KEDA
scales each tenant independently; idle tenants are zero pods. Hard
isolation (separate wasm components, separate capability sets).

- substrate: lareira-fleet-programs, KEDA HTTPScaledObject
- runtime: tenant onboarding via E1 (self-modifying caixa)

#### G2. Hash-sharded request routing ◐

Multiple ComputeUnits with the same `module.source` but different
`config.shard` (`0..N`). Ingress L7 routes by hash of request key.
Each shard scales independently; data locality preserved per-shard.
Shard split = bump N + redistribute hash range.

- substrate: caixa-flux entry with explicit `shard` config field
- runtime: KEDA HTTPScaledObject with hash-based selector (M2)

#### G3. Cluster-aware module resolution ○

wasm-engine's module resolver checks: in-pod cache → in-node cache →
in-cluster cache → peer-cluster cache → OCI registry / git. Network-
partition tolerant; cold-start improved by serving from peer cluster
instead of going to ghcr.

- substrate: wasm-engine resolver layer + a per-cluster lacre mirror
- pattern: matches Bazel's remote cache hierarchy

---

### H. Observability / debug patterns

#### H1. Live IR dump / replay ○

Dump a running tatara-script process's IR + heap state to a content-
addressed blob. Re-load locally with `feira repl --restore <hash>` to
reproduce a prod issue *exactly* as it was at the moment of dump.

- substrate: tatara-script's image-dump primitive (CL-style world dump)
- consumer: feira repl --restore

#### H2. Cluster-wide attestation sweep (already a Servico) ★

`programs/fleet-attestation-sweep` (from this session's bulk migration)
walks every cluster, asks tameshi for the BLAKE3 root, verifies
signatures, emits a typed report. Audits the substrate against the
canonical chain.

- substrate: pleme-io/programs/fleet-attestation-sweep, pleme-io/tameshi
- runtime: oneShot ComputeUnit, scheduled via cron or manual invocation

#### H3. Differential trace replay ○

Capture a request's wasi:http/incoming-request as typed event, replay
locally against the same caixa version (lacre-pinned). Bug reproduction
becomes "replay event log + run `feira test --from <event>`".

- substrate: wasi:http event capture (M2), caixa-lacre version pinning
- runtime: tatara-script test harness

---

## III. Pattern composition rules

Patterns aren't independent — many compose. Key composition axes:

### Time

- **A1 (per-request) ∘ G1 (per-tenant)** = fresh wasm component per
  tenant per request. Hard isolation; trivial to reason about.
- **A3 (Lisp eval) ∘ E2 (REPL-on-pod)** = exec into pod, redefine,
  validate, ship. Closes the inner-loop dev cycle.

### Space

- **B2 (cross-cluster migration) ∘ G3 (cluster-aware resolution)** =
  caixas migrate without disturbing module-loading; warm cache follows
  the workload.
- **D1 (in-place resize) ∘ G2 (hash-sharded)** = scale a hot shard
  without restart, while cold shards stay zero-pod.

### Trust

- **C2 (capability-only update) ∘ F2 (time-travel debug)** = revoke a
  leaked credential, audit *exactly* what it could access between
  issue and revoke. Compliance becomes a `feira audit --since <hash>`.

### Identity

- **F1 (replay-based roll forward) ∘ F3 (typed state migration)** =
  full disaster recovery from genesis hash + state delta log.
  RPO = 0 (event log + lacre store mirrored cross-region).

---

## IV. The substrate's "you can ask for these"

A operator-facing checklist of what to ask the substrate to do, organized
by the language an operator already uses:

| operator wants… | pattern | status |
|---|---|---|
| "deploy without restart" | A1 | ★ today (per-request instantiation) |
| "swap one component" | A2 | ◐ needs wasm-engine M2 |
| "redefine a function while it runs" | A3 | ○ tatara-script eval surface |
| "blue/green this rollout" | A4 | ◐ KEDA weight ready, needs feira deploy --weight |
| "ship only the changed Lisp form" | A5 | ○ caixa-ast IR diff |
| "snapshot + resume on another node" | B1 | ◐ needs wasm-engine checkpoint API |
| "drain one cluster onto another" | B2 | ◐ today via weight + manual flux |
| "checkpoint at the OS level" | B3 | ○ CRIU + container-checkpoint |
| "hot-reload library v0.2.1 across all dependents" | C1 | ○ lacre + tatara-script eval |
| "rotate this credential without restart" | C2 | ★ today (RBAC + downward API) |
| "ship just the .wasm bytes, not the wasmtime base" | C3 | ★ today (layered OCI) |
| "resize this pod's memory without restart" | D1 | ◐ needs wasm-operator M2 |
| "revoke a capability live" | D2 | ○ wasm-engine capability table |
| "migrate the schema before the new pod takes traffic" | D3 | ◐ via shinka |
| "let one program author another" | E1 | ○ caixa-feira-from-tatara |
| "REPL into a running pod and edit it live" | E2 | ○ feira repl |
| "preview this PR before merge" | E3 | ◐ small caixa-publish change |
| "replay the cluster from genesis to hash X" | F1 | ○ tameshi-driven replay |
| "reproduce yesterday's prod bug exactly" | F2 | ○ time-travel via lacre |
| "migrate state across versions automatically" | F3 | ○ typed state schema |
| "give every tenant their own runtime" | G1 | ◐ via per-tenant programs.yaml |
| "shard requests by hash" | G2 | ◐ KEDA hash selector |
| "pull modules from peer cluster on cold start" | G3 | ○ caixa-resolver hierarchy |
| "dump live state for offline debug" | H1 | ○ tatara-script image dump |
| "audit attestation across the fleet" | H2 | ★ today |
| "replay a captured request locally" | H3 | ○ wasi:http event capture |

**11 ★ load-bearing today, 12 ◐ partial, 16 ○ aspirational.**

---

## V. Adjacent ideas worth considering

The patterns above are the substrate's *direct* surface. Several
adjacent ideas from the broader runtime literature inform what's
worth doing next:

- **Erlang OTP**: `code:purge / load_module / suspend / resume`. The
  caixa equivalent is A3 + D2. Erlang's `gen_server:code_change/3`
  callback (state migration during upgrade) maps to F3.
- **Smalltalk image-based dev**: dump live world to a file, restart
  from the file. Maps to H1. tatara-script could expose
  `(image-save "myfile.image")` + `tatara-script --image=myfile.image`.
- **K8s `kubectl debug --copy-to`**: make a debug copy of a running
  pod with extra capabilities. Maps to E2 (REPL-on-pod) + a
  capability override.
- **Dapr building blocks**: actors, state stores, pubsub abstractions.
  Maps to caixa's `:capabilities` (typed runtime services).
- **Rust async runtime + scheduler**: tokio's per-task scheduler maps
  cleanly to wasmtime's per-instance scheduler; same patterns scale.
- **CRDT state sync**: Yjs / Automerge as a runtime capability.
  Maps to a `:state` slot of caixa.lisp + a CRDT-typed sync layer.
  Useful for offline-first edge deployments.
- **Unison's content-addressed runtime**: every term is identified by
  hash of its AST, automatic dedup across versions. Tatara-lisp can
  adopt this via caixa-lacre's BLAKE3 chain over Sexp.
- **WebAssembly Component Model + WIT linking**: A2 is exactly this.
  As wit-bindgen matures, more of the stdlib becomes
  hot-swappable individually.

---

## VI. What's load-bearing for the dailyreader

If you're an operator running the substrate today, the patterns
that already work are A1 (per-request fresh instances), C2 (capability
updates without restart), C3 (layered OCI updates), H2 (attestation
sweeps). Everything else needs to wait for either wasm-operator M2
(real reconciler) or wasm-engine M2 (capability table + checkpoint).

If you're an agent doing repo-level work, the patterns that compose
into your daily flow are E2 (REPL-on-pod for debugging) and F2
(time-travel for incident reproduction) — these become possible the
moment tatara-script grows the eval-from-control-socket primitive,
which is one M2 deliverable away.

The substrate's value compounds as patterns move from ○ → ◐ → ★. Each
move unlocks composition with every other pattern at the same level
or above. Pillar-12 (generation over composition) means we get them
*together* once the underlying primitive lands, not one at a time.
