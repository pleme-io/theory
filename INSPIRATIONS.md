# Best-in-class inspirations for the caixa substrate

> **Frame.** The caixa substrate is a deliberate cousin to Erlang/OTP,
> Smalltalk, Common Lisp, Unison, Pony, Lunatic, Akka, and Orleans —
> each of which solved a piece of "running typed software live"
> decades before us. This document studies each tradition for the
> patterns we should *absorb verbatim*, the ones we should *adapt*,
> and the ones we should *deliberately reject*. Audience: anyone
> reaching for a runtime feature who wants to know which prior art
> already nailed it.
>
> Companion to [`RUNTIME-PATTERNS.md`](./RUNTIME-PATTERNS.md), which
> catalogs the patterns *the substrate already enables*. This doc is
> the supply side: where the patterns came from, what we still owe.

---

## I. The unifying frame — *introspectable typed substrate*

Every system below shares one premise: **the running program is itself
a first-class data structure** the runtime can introspect, transform,
and rebuild without restarting. The caixa substrate inherits this
premise; it differs only in the *shape* of the data:

| tradition | the running program is… | introspected as |
|---|---|---|
| Erlang/OTP | a tree of supervised processes | `process_info/1`, `code:get_object_code/1` |
| Smalltalk | a *world image* of objects | the Image itself; objects can `become:` |
| Common Lisp | a tree of bindings + a class graph | symbols, packages, MOP, `find-class` |
| Unison | a graph of hash-addressed terms | UCM (codebase manager), `view`, `dependencies` |
| Pony | a typed actor graph | compile-time-only — runtime is opaque |
| Lunatic | wasm processes + supervisor types | wit-typed; preempted at fuel boundaries |
| Akka | cluster-sharded persistent entities | `PersistenceId`, ddata replicators |
| caixa (us) | typed `(defcaixa …)` form + lacre closure | `caixa-core::Caixa`, lacre BLAKE3 chain |

All of them treat *deployment* as "swap the program in place" rather
than "tear it down and bring up a new one". That's the inheritance.

---

## II. Erlang/OTP — the central inspiration

Erlang has been doing this since 1986. Telephony switches with 99.9999999%
uptime (~31ms downtime/year). The patterns below are the load-bearing
parts of OTP we should *absolutely* take.

### II.1. BEAM-as-hypervisor

One OS process (`beam.smp`) hosts **millions of lightweight Erlang
processes**, each with:

- its own heap (no shared memory; messages are copied)
- its own GC (generational young/old; collected only when *that*
  process is full; **no stop-the-world** across processes)
- its own scheduler slot (preemptive at reductions boundary)
- its own mailbox (selective receive)

Crashes stay process-local; the BEAM is the fault-isolation boundary.

> **Why this is best-in-class:** the BEAM is effectively a hypervisor
> *inside one Linux process*. You get container-like isolation
> (per-process state) without container-like overhead (per-process
> kernel namespace). 30 years of telecom production has hardened it.

**Take it verbatim:** the wasm-engine in pleme-io is morally a BEAM —
one OS process holds many wasm component instances. We should call this
out by name and adopt the same vocabulary (process, mailbox, link,
monitor).

### II.2. Supervisor trees + restart strategies

OTP applications are trees:

```
my-app  (application)
└── my-sup       (supervisor: one_for_one)
    ├── my-worker-1   (gen_server)
    ├── my-worker-2   (gen_server)
    └── my-cache-sup  (supervisor: rest_for_one)
        ├── cache-loader
        └── cache-server
```

Each supervisor declares a *restart strategy*:

- `one_for_one` — restart only the failed child
- `one_for_all` — restart every child when one fails
- `rest_for_one` — restart the failed child and every child started after it
- `simple_one_for_one` — dynamic children, one shape, restart-on-demand

Plus per-child policies: `permanent` (restart unconditionally),
`temporary` (never restart), `transient` (restart only on abnormal exit).

> **Why this is best-in-class:** every fault becomes "did the wrong
> piece of code crash? OK, the supervisor knows what to do." The tree
> shape is the typed answer to "what's my dependency graph and what
> should happen if X dies."

**Take it verbatim** — *but at the ComputeUnit level*. The wasm-operator
should reconcile a `tree` of ComputeUnits with declared restart
strategies. Today's flat `programs:` array becomes Layer 0; the next
M2 deliverable is a `Tree` CRD whose nodes are ComputeUnit references
with strategy + restart-intensity bounds. Lunatic already does this in
Rust; we follow that wire.

### II.3. OTP behaviors (`gen_server`, `gen_statem`, `gen_event`)

Behaviors are *typed protocols* — a set of callbacks the runtime calls
in known order:

- `gen_server` — request/response server with state. Callbacks:
  `init/1`, `handle_call/3`, `handle_cast/2`, `handle_info/2`,
  `terminate/2`, `code_change/3`.
- `gen_statem` — state machine. Each state has its own callback.
- `gen_event` — event manager with multiple handlers.
- `supervisor` — one of the four restart strategies above.
- `application` — a top-level app with `start/2` + `stop/1`.

Authors implement the callbacks; the runtime owns the lifecycle.

> **Why this is best-in-class:** behaviors *are* the right abstraction
> over "things that hold state and respond to messages." Every actor
> framework reinvents them poorly. OTP's are decades-load-tested.

**Take them verbatim** in caixa form. `(defcaixa :kind Servico)` is
already a behavior in spirit — runtime owns init/terminate, author
fills callbacks. We should add typed slots for `:on-init`, `:on-call`,
`:on-cast`, `:on-info`, `:on-state-change` so caixa.lisp gets to the
same structural completeness.

### II.4. Hot code reload — `appup`, `relup`, `release_handler`

OTP's deployment model is **the release** — a tarball of BEAM apps,
the ERTS, and metadata. Releases hot-upgrade via:

- `.appup` files — per-app upgrade instructions, e.g.
  `[{"0.2.0", [{load_module, my_module}]}]`. Tells the runtime *exactly*
  what changed between versions.
- `.relup` files — per-release upgrade instructions, generated from
  the constituent appups via `systools:make_relup`.
- `release_handler:install_release/1` — applies the relup to a running
  node. Each node runs `release_handler` locally; `sync_nodes` is the
  cluster-wide synchronization primitive.

Two-phase code load:
1. `code:load_module/1` — load v2 alongside v1; new code is
   "current", old code is "old".
2. `code:soft_purge/1` — wait until no process is running v1, then
   discard. (`code:purge/1` kills v1 immediately if you don't care.)

State migration uses `gen_server:code_change/3`:

```erlang
code_change("0.1.0", State, _Extra) ->
    %% migrate state from v0.1.0 shape to current shape
    NewState = migrate_v01_to_current(State),
    {ok, NewState}.
```

> **Why this is best-in-class:** declarative state migration *as part
> of the deployment artifact*. The runtime drives the upgrade; the
> author declares the delta. No "stop service, run migration script,
> start service" sequence; it's atomic.

**Take it verbatim:** caixa.lisp grows a `:upgrade-from` slot —

```lisp
(defcaixa
  :nome  "hello-rio"
  :versao "0.2.0"
  :upgrade-from
    ((:from "0.1.0" :instructions ((:load-module hello-rio)
                                    (:state-change "lib/migrations/v01-to-v02.lisp")))))
```

caixa-flux renders a `Lacre` migration delta; the wasm-operator runs
it before swapping pods (or before a Lunatic-style live upgrade). This
is a direct port of `gen_server:code_change/3` to caixa.

### II.5. Distributed Erlang — `epmd`, cookies, transparent message passing

Multiple BEAM nodes form a *cluster* by:

1. each node registering with `epmd` (Erlang port mapper daemon) on a
   well-known port
2. setting a shared `cookie` (auth token)
3. `net_adm:ping/1` to establish a connection

After that, message passing is *fully transparent*:

```erlang
%% same syntax whether Pid is local or on another node
Pid ! {hello, world}.
```

Failure of a remote node delivers `{nodedown, Node}` to anyone who
linked to it. Code can be loaded onto a remote node via
`rpc:call(Node, code, load_binary, [Mod, File, Bin])`.

> **Why this is best-in-class:** the *call site doesn't know* it's
> remote. Pid is opaque; transport is hidden. Distribution becomes a
> deployment property, not a code property.

**Adapt:**
- The cookie/epmd model is too weak for modern security; replace with
  *capability tokens* (caixa-style). Pony nailed this — see §VII.
- The wire protocol (Erlang's distribution protocol) is fast but not
  capability-secure; replace with QUIC + WIT-typed messages.
- The transparency claim, however, is gold. caixa-resolver + wasm-engine
  should make local-vs-cluster module loading *invisible to the source*
  — the same `(send :hello-world (request "/hello"))` works whether
  hello-world is in the same pod or another cluster.

This is also where Unison's "Adaptive Service Graph Compression"
(§IV) supersedes Erlang: not just transparent distribution, but
*automatic re-co-location based on call patterns*.

### II.6. Releases as the deployment unit

An OTP release is self-contained: BEAM + ERTS + apps + start scripts +
release config. Deployable to a fresh machine without OS-level
dependencies. Hot-upgradable via the appup/relup pipeline. Rollbackable
via `release_handler:remove_release/1`.

> **Why this is best-in-class:** the deployment unit *is* the typed
> migration unit. There's no separation of "what runs" and "how to
> upgrade what runs."

**Already taken** — caixa is the OTP release. caixa.lisp is the .app
file; lacre.lisp is the .appup. We should formalize the analogy in
documentation so people who know OTP can map onto caixa instantly.

---

## III. Lunatic — Erlang's principles in WebAssembly

[Lunatic][lunatic] is the most *directly applicable* prior art: an
Erlang-inspired runtime where actors are wasm component instances.
Already in production, already hardening the patterns we want.

[lunatic]: https://lunatic.solutions/

### III.1. Per-process sandboxing — *more powerful than BEAM*

Lunatic's headline insight: a wasm process can be sandboxed with
**per-process resource limits** the BEAM cannot give you:

- max linear memory (e.g. 1MB cap; BEAM gives unbounded heap-per-process)
- max CPU fuel (e.g. 1M instructions; BEAM uses cooperative reductions)
- per-process WASI capability set (file/network/env capabilities the
  BEAM can't express)
- safe FFI to C (wasm's sandbox contains it; BEAM's NIFs can crash the
  whole node)

> **Why this is better than BEAM:** isolation is *structural*, not
> conventional. BEAM trusts NIFs and trusts that processes are
> well-behaved; Lunatic enforces by construction.

**Take it verbatim.** wasm-engine's per-process limits should mirror
Lunatic's primitives 1:1:

```lisp
(defcaixa
  :limits ((:memory   "1MiB")
           (:fuel     1000000)
           (:wall-clock "30s")))
```

caixa-core grows a `:limits` slot; wasm-engine enforces them via
wasmtime's `Store::limiter` + fuel mechanism + WASI cap set. *Every*
caixa runs sandboxed by default; no "trust the author."

### III.2. Supervisor strategies as Rust types

Lunatic encodes the supervisor tree at the Rust type level — a
supervisor's children are typed, the strategy is a type parameter,
mismatches are compile errors. Static guarantees that BEAM's runtime
checks; you can't accidentally restart the wrong subtree.

**Take it adapted.** caixa.lisp grows a `:supervisor` slot whose
children are typed lacre references:

```lisp
(defcaixa
  :nome      "my-app-root"
  :kind      Supervisor
  :estrategia OneForOne
  :children  ((:caixa "worker-1" :versao "^0.1" :restart Permanent)
              (:caixa "cache-sup" :versao "^0.1" :restart Permanent)))
```

caixa-core gets a fourth `CaixaKind`: `Supervisor`. wasm-operator
reconciles the tree as a hierarchy of ComputeUnits with restart
relationships — the runtime analog to OTP's supervisor tree.

### III.3. Preemption at fuel boundaries

Lunatic uses wasmtime's **fuel** mechanism — every wasm instruction
consumes one fuel unit; when fuel hits zero, the runtime preempts the
process. No process can hog the scheduler. BEAM does this with
reductions (~2000 reductions per scheduler slice); Lunatic ports the
exact same idea to wasm.

> **Why this is best-in-class for our world:** soft-real-time
> guarantees at the *runtime* level, not the application level.
> Authors don't have to think about cooperative yielding.

**Take it verbatim.** wasm-engine M2 wires fuel-based scheduling;
breathability + KEDA become a property of the engine, not the pod.

---

## IV. Unison — content-addressed code, distributed by default

Unison [1.0][unison-1.0] (Nov 2025) is *the* state-of-the-art in
content-addressed runtimes. Their core ideas map directly onto caixa
+ lacre + tameshi.

[unison-1.0]: https://www.unison-lang.org/docs/the-big-idea/

### IV.1. Hash-of-AST = code identity

Every term is identified by the BLAKE3-equivalent hash of its
fully-resolved AST. Names are just bindings to hashes. Two syntactically
different but equivalent definitions resolve to the same hash. Renaming
is free; the hash is the truth.

Practical consequences:

- **Builds are deterministic by definition** — same hash → same bytes,
  forever.
- **Dependencies pin themselves** — depending on a function pins its
  hash, which transitively pins every hash it uses. Closures are
  closed by construction.
- **Code can move freely** — the hash is the address; the code can
  live anywhere.

> **Why this is best-in-class:** the *content addressing isn't a build
> system add-on*; it's the language identity model. Bazel's hash-based
> caching is a poor man's version of this layered on top of a name-
> based language.

**Already taken in spirit.** caixa-lacre uses BLAKE3 closures over
caixa.lisp + dep tree; tameshi attests the closure root. We should
*deepen* this — make the wasm-engine's module loader content-addressed:
a module is loaded by hash, not by URL. The github: / oci:// URLs are
just *resolvers* that produce a hash; once produced, the hash is the
identity.

### IV.2. Adaptive Service Graph Compression

Unison's runtime profiles call patterns and **dynamically co-locates
frequently-calling services**. If service A calls service B 10× per
second, the runtime moves B's code to run *inside A's process*; what
looked like a network request becomes a function call. When call
patterns shift, Unison re-distributes.

> **Why this is best-in-class for our world:** the substrate optimizes
> the topology *itself*, not just within a topology. Operators stop
> doing "service mesh tuning"; the runtime does it.

**Take it adapted.** wasm-engine M3 grows a *call-pattern profiler*
that emits a typed signal: `(:hot-pair :caller foo :callee bar :rate
12.4/s)`. The wasm-operator subscribes; if two ComputeUnits exceed a
threshold, it co-schedules them on the same node (via K8s pod
affinity) or in the same wasm-engine process (if both are wasm). At
the limit, it inlines bar into foo (since both are content-addressed,
inlining is just a pre-link step). Network-as-type-system.

### IV.3. Distributed-by-default semantics

Unison's `Remote` ability lets a function delegate to another node
with the *same syntax* as a local call:

```haskell
remote.do node 'do
  result = compute-on-remote-node x y
```

Missing dependencies are deployed on the fly — the node downloads any
hashes it doesn't have. This is BEAM's distributed Erlang made typed
and content-addressed.

**Take it adapted.** tatara-lisp grows a `(remoto :node n :do …)` form
that wraps `(send …)` in the lacre-aware envelope. wasm-engine
resolves the node by capability token; on miss, fetches the hash from
peer-cluster cache or upstream registry; on success, evaluates locally
or remotely based on the Adaptive compression signal.

---

## V. Common Lisp — REPL + conditions/restarts + MOP

Common Lisp has been the testbed for "live introspectable typed
substrate" since 1984. Three patterns we should absorb.

### V.1. Conditions and restarts — error handling as a recoverable choice

CL doesn't catch exceptions; it *signals* conditions, and the *signaler*
binds restarts that the *handler* can choose:

```lisp
(handler-bind
    ((file-error (lambda (c)
                    (invoke-restart 'use-default-config))))
  (load-config))

;; inside load-config:
(restart-case (error 'file-error :pathname p)
  (use-default-config () (default-config))
  (retry           () (load-config))
  (abort           () (return-from load-config nil)))
```

> **Why this is best-in-class:** error handling becomes *interactive
> repair*. The signaler doesn't decide what to do; it offers options.
> The handler picks. At the REPL, the human picks. In production, the
> supervisor picks.

**Take it adapted.** tatara-lisp grows `(restart-case …)` and
`(handler-bind …)`. wasm-engine's WASI error trap becomes a condition
signal with restarts. A failing OCI fetch offers `(retry)`,
`(use-cached)`, `(use-peer-cluster)`, `(abort)`. The supervisor (or a
human via REPL-on-pod) picks. This *is* the Erlang "let it crash"
philosophy — with structured choice instead of binary outcome.

### V.2. CLOS MOP — `update-instance-for-redefined-class`

When you redefine a class in a running CL image, every existing
instance is *automatically* updated by calling the generic function
`update-instance-for-redefined-class` (or
`update-instance-for-different-class` for `change-class`). The MOP
(metaobject protocol) makes this hookable.

> **Why this is best-in-class:** *no manual data migration* on schema
> change. The runtime handles existing instances; the author hooks the
> conversion logic per slot.

**Take it adapted.** caixa.lisp `:state-schema` slot declares the
schema; redeploying with a new schema triggers
`update-instance-for-redefined-state` per persisted entity. caixa-flux
emits the migration job; shinka runs it. This is OTP's
`gen_server:code_change/3` *meets* CL's MOP — declarative + auto-
applied.

### V.3. `save-lisp-and-die` — image dump

CL implementations (SBCL, CCL) can dump the entire running world
(heap + bindings + classes + everything) to a single file. Restart
loads that file, becomes the running world *exactly* as it was. This
is what makes CL "image-based development."

**Take it.** tatara-script grows `(image-save "myapp.image")` and
`tatara-script --image=myapp.image`. wasm-engine's process can be
checkpointed (Lunatic does this; wasmtime supports `Store` snapshot)
and restored — same idea, wasm shape.

---

## VI. Smalltalk — image-based dev + `become:`

Smalltalk-80 (1980) had:

- **Image-based dev**: the running system is the file. Edit a method,
  it takes effect immediately, persists across restarts.
- `become:` — *swap object identities*. `obj1 become: obj2` rewrites
  every reference to obj1 to point at obj2 (and vice-versa). Used for
  schema migration without breaking existing references.
- `doesNotUnderstand:` — when a message isn't found, this method is
  called. Used for proxies, missing-method handlers, transparent
  forwarding.

> **Why this is best-in-class:** Smalltalk's image-based model is the
> *original* "live introspectable typed substrate". Every modern
> system below is an answer to "Smalltalk, but with X" (X = static
> types, distribution, sandboxing, etc.).

**Take adapted:**

- Image dump: covered by §V.3 (CL also has it). tatara-script does this.
- `become:` is *too powerful for caixa today* — it'd break content
  addressing. But the *use case* (rewriting references during state
  migration) maps onto caixa-flux's typed delta in §II.4.
- `doesNotUnderstand:` maps onto WASI capability *miss* — when a
  caixa requests a capability not in its token set, the runtime can
  call a typed `(:on-capability-miss …)` handler. Used for transparent
  proxying, lazy capability negotiation.

---

## VII. Pony — capability-secure typed actors

[Pony][pony] is the actor model with **compile-time data-race
freedom**. Six reference capabilities, statically checked:

[pony]: https://www.ponylang.io/

| capability | meaning | example |
|---|---|---|
| `iso` | unique reference; safe to send | mutable buffer crossing actors |
| `val` | immutable value; freely shareable | string constants |
| `ref` | actor-local mutable | per-actor cache |
| `box` | read-only alias | inspecting val/iso/ref locally |
| `trn` | transitionable to val or iso | mutable then frozen |
| `tag` | identity-only (for actor refs) | "send me a message" |

The compiler refuses to send mutable references across actors. No
locks, no atomics, no data races. Actor scheduling has lower overhead
than BEAM because the type system carries the safety guarantees.

> **Why this is best-in-class:** Pony achieves at compile time what
> Erlang achieves at runtime via copy-on-send. *And* what Rust achieves
> via single ownership. The 6-capability vocabulary is more nuanced
> than Rust's two (move/borrow); a static answer to "who can mutate
> what across actors."

**Take it adapted.** caixa.lisp `:capabilities` are runtime tokens
today (`http-in:0.0.0.0:8080`, `kube-secret-read@…`). Add a *static*
typed layer:

```lisp
(defcaixa
  :capabilities
    ((:wasi :network :iso)        ;; unique mutable
     (:wasi :env     :val)        ;; immutable shared
     (:kube :secret-read :ref     ;; actor-local
            :scope "photos/s3-credentials")))
```

caixa-core types them via `#[derive(TataraDomain)]`; wasm-operator
mints typed RBAC + capability tokens. Static rejection of "Servico A
sends a writable handle to Servico B" — the build fails before the
container ever runs.

---

## VIII. Akka + Orleans — modern distributed actors at scale

Akka (Scala/Java) and Orleans (.NET) are the production-hardened
actor frameworks for clusters with millions of entities.

### VIII.1. Cluster sharding (Akka)

A cluster of N nodes hosts **one entity per id**, distributed by
hash. Adding a node triggers automatic rebalance. Each entity is a
`PersistenceId` + state. `Cluster Sharding` ensures only one active
instance per id, even during failover.

> **Why this is best-in-class:** the substrate decides *where* an
> entity lives based on cluster topology; the application code says
> "send to entity 42", not "send to node 3".

**Take it adapted.** Maps onto caixa pattern G2 (hash-sharded request
routing) from RUNTIME-PATTERNS.md. wasm-operator + KEDA HTTPScaler
implement the sharding; entity = ComputeUnit; persistence = lacre +
PVC.

### VIII.2. Event sourcing as canonical persistence (Akka Persistence)

State is the fold of an event log. The log is the source of truth;
state is derived. Replay rebuilds state from genesis; snapshots are
optimization.

> **Why this is best-in-class:** time-travel debugging, audit trails,
> and recovery are *the same primitive*. Per Akka docs, "It's
> recommended to use state-store-mode=eventsourced as it's much faster
> and more scalable than ddata."

**Take it adapted.** Maps onto caixa pattern F1 (replay-based roll
forward via tameshi BLAKE3 chain). The event log = lacre delta chain;
each ComputeUnit reconciliation is an event; state = fold of the chain
up to a hash.

### VIII.3. Virtual actors (Orleans)

Grains in Orleans are *virtual* — they're never explicitly created or
destroyed. Calling a grain by ID activates it on demand; idle grains
deactivate after a timeout. The runtime owns lifecycle entirely.

> **Why this is best-in-class:** caller never thinks about activation.
> Single-threaded per-grain semantics give Erlang-shape isolation
> without explicit process management.

**Take it adapted.** caixa Servicos with KEDA `minReplicas: 0` are
already virtual actors — first request activates; idle deactivates.
Orleans's grain placement strategies (random, prefer-local, hash-
based) become wasm-operator pod-affinity overlays.

### VIII.4. Remembered entities

Akka's "remembered entities" auto-restart entities after a rebalance
or crash, even before any message arrives. Useful for entities that
need to be ready (timers, subscriptions).

**Take it adapted.** caixa.lisp's `:on-init` callback (when we add
behaviors per §II.3) declares "I need to be running even before any
request"; the wasm-operator pre-warms one instance.

---

## IX. OS-level adjacents — CRIU, eBPF, cgroups v2

The substrate runs on Linux; we should know what Linux gives us.

### IX.1. CRIU — checkpoint/restore in userspace

CRIU dumps a Linux process tree (memory, fds, sockets, namespaces) to
files; another invocation loads them and resumes execution. Used by
Kubernetes' `Forensic Container Checkpointing` (1.30+ feature gate).

> **Why this is best-in-class:** *language-agnostic* live migration.
> wasmtime in Pulley mode is CRIU-compatible; we can checkpoint the
> wasm-engine process itself, ship it, resume.

**Take it.** wasm-engine M3 supports CRIU checkpoint/restore as the
canonical live-migration primitive (RUNTIME-PATTERNS.md §B.3).

### IX.2. eBPF — programmable kernel observability

eBPF programs run in the kernel, attached to syscalls/tracepoints/
network hooks. Aya / libbpf give Rust APIs. BPF CO-RE makes them
portable across kernels.

> **Why this is best-in-class:** zero-overhead observability when not
> firing; near-zero overhead when firing. The *kernel* tells us what
> the wasm processes are doing.

**Take it adapted.** wasm-engine emits BPF programs that trace WASI
syscalls per process; aggregated to the typed `Caixa` event log; fed
into the same lacre BLAKE3 chain as ComputeUnit reconciliations. One
chain, kernel-level provenance.

### IX.3. cgroups v2 — hierarchical resource control

cgroups v2 manages memory, CPU, IO, PIDs in a unified hierarchy.
Per-controller delegation lets a parent cgroup constrain children
without seeing into them.

**Take it.** Lunatic-style limits (§III.1) can map onto cgroups v2 at
the pod level *and* wasmtime fuel/memory limits at the wasm-component
level. Two layers of resource control; both typed via the caixa
`:limits` slot.

---

## X. The synthesis — the caixa-BEAM-Unison frame

Putting all of the above together, the substrate's *load-bearing
identity* is:

> **A content-addressed, typed actor runtime where wasm components
> are the processes, caixa.lisp is the OTP application descriptor,
> lacre is the appup, supervisor trees are first-class, and the
> network is part of the type system.**

In one paragraph: caixa is OTP's typed-deployment shape × Lunatic's
sandboxed actor runtime × Unison's content-addressed code × Pony's
typed capabilities × Smalltalk's image-based liveness × CL's
condition/restart system × Akka's cluster sharding × CRIU's process
migration. Every layer is typed; every layer is hot-swappable.

### The wasm-engine as the BEAM

| BEAM | wasm-engine (target) | status |
|---|---|---|
| `beam.smp` OS process | wasm-engine pod | ★ |
| Erlang process | wasm component instance | ★ |
| Process heap | wasm linear memory | ★ |
| Per-process GC | per-instance memory release | ★ |
| Reductions/scheduler | wasmtime fuel/Pulley scheduler | ◐ M2 |
| Mailbox + selective receive | WIT-typed message channel | ○ M3 |
| `link/2`, `monitor/1` | typed lacre dependency edges | ○ M3 |
| Supervisor tree | ComputeUnit tree CRD | ○ M2 |
| `code:load_module/1` | wasm-engine module hot-swap (component model) | ○ M3 |
| `gen_server:code_change/3` | caixa.lisp `:upgrade-from` slot | ○ M2 |
| OTP release | caixa repo at v<versao> tag | ★ |
| `release_handler:install_release/1` | `feira deploy --apply` | ★ |
| Distributed Erlang | wasm-engine peer-cluster resolver | ○ M3 |
| `epmd` + cookies | tameshi BLAKE3 chain + capability tokens | ◐ |
| `rpc:call/4` | `(remoto :node … :do …)` form | ○ M3 |

### The caixa.lisp as the OTP `.app` + `.appup`

```lisp
(defcaixa
  :nome      "my-service"
  :versao    "0.2.0"
  :kind      Servico

  ;; OTP behavior — typed callback contract
  :behavior  (:on-init    "lib/init.lisp"
              :on-call    "lib/handlers.lisp"
              :on-cast    "lib/handlers.lisp"
              :on-info    "lib/handlers.lisp"
              :on-state-change "lib/migrations.lisp")

  ;; Lunatic-style sandboxing
  :limits    (:memory "64MiB" :fuel 1000000 :wall-clock "30s")

  ;; Pony-style typed capabilities
  :capabilities ((:wasi :network :iso)
                 (:kube :secret-read :ref :scope "photos/s3"))

  ;; OTP supervisor strategy
  :supervisor (:estrategia OneForOne :max-restarts 5 :within "60s")

  ;; OTP appup — typed migration from prior versions
  :upgrade-from
    ((:from "0.1.0"
      :instructions ((:load-module my-service)
                     (:state-change "lib/migrations/v01-to-v02.lisp"))))

  ;; runtime contract for the cluster operator
  :servicos  ("servicos/my-service.computeunit.yaml"))
```

This is the synthesis target. Every slot is a port of a load-bearing
prior-art primitive.

---

## XI. The absorption roadmap

Concretely, what to build, in priority order:

### M2 (next quarter — the runtime substrate becomes BEAM-shaped)

| absorb from | as | scope |
|---|---|---|
| Erlang OTP behaviors | caixa.lisp `:behavior` slot | caixa-core types |
| Erlang OTP appup | caixa.lisp `:upgrade-from` slot | caixa-core types |
| Lunatic per-process limits | caixa.lisp `:limits` slot | caixa-core + wasm-engine enforcement |
| Lunatic supervisor strategies | `CaixaKind::Supervisor` + tree CRD | caixa-core + wasm-operator reconciler |
| Common Lisp conditions/restarts | tatara-lisp `(restart-case …)` form | tatara-lisp + tatara-script |
| K8s in-place pod resize | wasm-operator reconciler M2 | kube-rs SSA |

### M3 (mid-year — distribution + introspection become typed)

| absorb from | as | scope |
|---|---|---|
| Erlang distribution + Unison Remote | `(remoto :node … :do …)` form | tatara-lisp + wasm-engine resolver |
| Unison Adaptive compression | wasm-operator co-location signal | wasm-operator + lacre profiler |
| BEAM mailbox + link/monitor | WIT-typed channels per ComputeUnit | wasm-engine M3 |
| CL save-lisp-and-die | `feira repl --restore <hash>` | caixa-feira + tatara-script |
| Pony typed capabilities | caixa.lisp `:capabilities` static layer | caixa-core types |
| CRIU live migration | wasm-engine checkpoint/restore | substrate primitive |
| eBPF observability | substrate's eBPF tracer crate | new caixa-bpf crate |

### M4 (late year — the operator surface becomes self-contained)

| absorb from | as | scope |
|---|---|---|
| OTP release_handler | `feira upgrade --to <versao>` | caixa-feira deploy variant |
| Smalltalk image-based dev | `feira repl --pod <name>` | live REPL into a running pod |
| Akka cluster sharding | hash-sharded ComputeUnit | wasm-operator G2 pattern |
| Orleans virtual actors | KEDA-driven activation/deactivation | already partial; formalize |
| Akka event sourcing | tameshi-driven log replay | F1 pattern formalized |

### M5 (next year — the substrate self-hosts)

The substrate runs *itself* under itself: caixa-operator, caixa-resolver,
wasm-engine, wasm-operator all become caixa Servicos hosted by
lareira-tatara-stack. The substrate is the program that runs the
program that runs the program. The recursive loop closes.

---

## XII. What to deliberately reject

Not everything in the prior art is worth taking. A few we should *not*:

- **Erlang's weak crypto in the distribution protocol.** Replaced by
  capability tokens + WIT-typed channels.
- **Smalltalk's `become:` at runtime.** Breaks content addressing;
  the typed delta in §II.4 covers the legitimate use case without the
  hazard.
- **CL's runtime *anything*-can-redefine-anything**. Defeats the
  capability model; tatara-lisp's `(eval …)` must be capability-gated.
- **BEAM's NIFs.** Crash isolation is by construction in wasm; we
  don't need an escape hatch that *removes* it.
- **Orleans's "fall back to caller" semantics on grain unavailability.**
  Surprising; prefer explicit `restart-case` / supervisor handling.
- **Akka's untyped "anything goes" actors.** The whole point of caixa
  is typed; preserve that, mirror Akka *Typed*.

---

## XIII. The bigger picture

What makes this synthesis worth doing — beyond "we get nice features"
— is that it's *the convergence of 40 years of liveness research*
into a single load-bearing substrate. Each tradition above optimized
for one concern:

- Erlang: fault tolerance via supervision
- Smalltalk: live introspection via image
- Common Lisp: structural error recovery via conditions
- Unison: content addressing via hash
- Pony: data-race freedom via capabilities
- Lunatic: sandboxing via wasm
- Akka/Orleans: cluster scaling via virtual entities
- CRIU/eBPF: process portability + observability via the kernel

Each traded off something. No single system has *all* the properties.
The caixa substrate, by virtue of being *typed, content-addressed,
and wasm-hosted*, can compose all of them without the historical
trade-offs:

- Sandboxing? wasm gives it for free.
- Content addressing? lacre BLAKE3 gives it for free.
- Type safety? caixa-core gives it for free.
- Liveness? tatara-script + lunatic-style hot-reload gives it.
- Distribution? Unison-style transparent + Adaptive compression gives it.
- Fault tolerance? OTP supervisor trees give it.
- Observability? eBPF + lacre event chain give it.

The substrate's job for the next year is to *pull these together*
into one operator-facing surface. RUNTIME-PATTERNS.md is the index of
"what becomes possible"; this doc is the index of "where the patterns
came from"; the M2-M5 roadmap is "how we get there."

---

## XIV. Sources

The patterns above draw on:

- [Lunatic — Erlang-inspired runtime for WebAssembly](https://lunatic.solutions/) ([GitHub](https://github.com/lunatic-solutions/lunatic))
- [Erlang OTP appup cookbook + release_handler](https://www.erlang.org/doc/system/appup_cookbook.html)
- [Unison 1.0 — content-addressed code + Adaptive Service Graph Compression](https://www.unison-lang.org/docs/the-big-idea/)
- [Pony reference capabilities](https://tutorial.ponylang.io/reference-capabilities/reference-capabilities.html) and [Deny Capabilities for Safe, Fast Actors](https://www.ponylang.io/media/papers/fast-cheap.pdf)
- [Akka Cluster Sharding + Event Sourcing](https://doc.akka.io/libraries/akka-core/current/typed/cluster-sharding.html)
- [The BEAM Book — Erlang runtime internals](https://blog.stenmans.org/theBeamBook/)
- [Smalltalk-80: The Language and its Implementation](https://en.wikipedia.org/wiki/Smalltalk) (Goldberg & Robson, 1983)
- [Common Lisp the Language, 2nd ed.](https://www.cs.cmu.edu/Groups/AI/html/cltl/cltl2.html) (Steele, 1990) — chapter on conditions & restarts; CLOS MOP
- [CRIU — Checkpoint/Restore In Userspace](https://criu.org/)
- [Kubernetes: in-place pod resize (1.27+)](https://kubernetes.io/docs/tasks/configure-pod-container/resize-container-resources/)

Each is worth reading at depth. The substrate's intellectual debt is
real, and the goal is to repay it by making the synthesis available
to anyone via `feira deploy`.
