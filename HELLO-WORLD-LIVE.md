# Hello-world live — what we proved 2026-04-26

> **Status: ACHIEVED.** A tatara-lisp program is **live, curl-able, and
> returns valid JSON** today. The full stack — source → resolver →
> runtime → HTTP — works end-to-end from a single `.tlisp` file.
>
> ```sh
> $ tatara-script main.tlisp &
> $ curl http://localhost:8080/hello
> {"message":"Hello, world!","served-by":"tatara-script"}
> ```
>
> This document captures **what we learned**, what was good about the
> path, and the canonical patterns to replicate everywhere.

---

## I. The full path that worked

```
1. Author      pleme-io/programs/hello-world/main.tlisp     20 lines of Lisp
2. Compose     (json-stringify (list (list "message" "Hello, world!")))
3. Serve       (http-serve-static 8080 (list (list "/hello" 200 body)))
4. Run         tatara-script main.tlisp
5. Curl        curl http://localhost:8080/hello
6. Receive     {"message":"Hello, world!","served-by":"tatara-script"}
```

Every step composed from existing pleme-io primitives. **Zero per-program
Rust, zero per-program containers, zero glue scripts.** The cluster
deployment (when wasm-operator publishes per Phase B) will run the
SAME `.tlisp` source — only the runtime changes.

## II. What was good about the path

### II.1 Pure-Lisp authoring with serialization at boundaries

Original (bad):
```lisp
(http-serve-static 8080
  (list (list "/hello" 200 "{\"message\":\"Hello, world!\"}")))
```

Refactored (good):
```lisp
(define hello-body
  (json-stringify
    (list (list "message" "Hello, world!")
          (list "served-by" "tatara-script"))))

(http-serve-static 8080
  (list (list "/hello" 200 hello-body)))
```

**Lesson**: response shape is Lisp data. JSON serialization is a
boundary concern handled by `json-stringify`. **No embedded
escape-string soup ever.**

This generalizes to every other format: `yaml-stringify`, `toml-stringify`,
`sexp-stringify` — all produce strings from Lisp data; never the
other direction inside source code.

### II.2 Stdlib design — tiny + composable + Go/Python-flavored

Inspired by Go and Python stdlib organization
([per the user's note](https://github.com/pleme-io/theory/blob/main/SCRIPTING.md)):

```
tatara-lisp-script/src/stdlib/
├── http.rs           HTTP client (existing)
├── http_server.rs    HTTP server (NEW — added 2026-04-26)
├── json.rs           JSON parse + stringify
├── yaml.rs           YAML parse + stringify
├── toml.rs           TOML parse + stringify
├── encoding.rs       base64 + hex
├── crypto_extra.rs   hmac, sha2
├── env.rs            env vars + arg-parsing helpers
├── fs.rs             file I/O
├── time.rs           clocks + parse + format
├── string.rs         string ops (Python-like)
├── string_ext.rs     string extensions (Go-style helpers)
├── regex.rs          regex match / replace
├── log.rs            structured logging
├── os.rs             OS bindings
├── process.rs        subprocess invocation
├── sops.rs           sops decrypt / encrypt
└── module.rs         (require ...) + load
```

Each module:
- Owns one namespace of `register_fn` calls.
- Lives in <300 LoC.
- Has zero cross-module imports beyond `Value` + `EvalError`.
- Adds one bullet to install_stdlib's call list.

**Adding a new primitive**: write `<name>.rs`, register it in
`mod.rs`, ship. ~30 minutes from "I want this primitive" to "every
.tlisp file has it."

### II.3 Reuse the type system: Value::List as a universal data carrier

We don't need a separate Map type — `Value::List` of `Value::List`
pairs is canonical Lisp alist shape. JSON objects, request maps,
response maps, route tables — all encoded the same way:

```lisp
;; A "map" is a list of (key value) pairs.
'(("name" "value") ("name2" "value2"))

;; Constructed with (list ...) for evaluation:
(list (list "name" "value") (list "name2" "value2"))
```

Every stdlib primitive that takes "structured data" walks the same
list-of-pairs shape. **One serialization model, infinite reuse.**

### II.4 The quote-vs-list lesson

`'(...)` produces `Value::Sexp` (literal source form). `(list ...)`
produces `Value::List` (evaluated). When passing data to native
functions that pattern-match on `Value::List`, **always use
`(list ...)`** — never `'(...)`.

The stdlib's primitives could in principle accept both, but explicit
`(list ...)` is faster (no Sexp→List coercion at runtime) and makes
the data shape obvious to readers.

### II.5 The hello-world layered identity

Same `.tlisp` source has two run paths:

```
Year 1 (today): tatara-script + http-serve-static
   ↓ exact same source ↓
Year 2 (Phase B): wasm-operator + ComputeUnit service shape
                  + KEDA HTTP scale-to-zero
```

The contract layer (paths, status codes, response bodies) is
identical. **The runtime is the only thing that changes.** Cluster-
side YAML, ingress, observability, alerts — all unchanged. This is
[`META-FRAMEWORK.md` §V](META-FRAMEWORK.md) "Year 1 → Year 2
evolution" demonstrated concretely.

## III. The canonical pattern — replicate this everywhere

### III.1 Authoring shape

```lisp
;; <program-name>/main.tlisp

;; ── responses authored as Lisp data, serialized at boundaries ─────
(define <name>-body
  (json-stringify
    (list (list "key1" "value1")
          (list "key2" "value2"))))

;; ── routing table — list of (path status body) triples ─────────────
(http-serve-static <port>
  (list
    (list "/path1" 200 <name>-body)
    (list "/path2" 200 <other-body>)))
```

### III.2 Repo placement

```
pleme-io/programs/<program-name>/
├── main.tlisp        the program (≤100 lines target)
├── computeunit.yaml  reference CR (kubectl-friendly)
└── README.md         cross-links to theory + cookbook patterns

helmworks/charts/lareira-<program-name>/
├── Chart.yaml        depends on pleme-computeunit
├── values.yaml       cluster-wide defaults (~30 lines)
└── README.md         install + customize

clusters/<cluster>/programs/release.yaml
└── programs:
      - name: <program-name>
        module: { source: github:pleme-io/programs/<program-name>/main.tlisp?ref=v0.1.0 }
        trigger: { service: { ... } }
        capabilities: [ ... ]
```

### III.3 Test contract

Every program supports two run modes:

```sh
# 1. LOCAL (tatara-script today):
nix run github:pleme-io/tatara-lisp#script -- ./main.tlisp
curl http://localhost:8080/<route>

# 2. CLUSTER (wasm-operator, Phase B):
helm install <name> ./helmworks/charts/lareira-<name>
curl https://<name>.<cluster>.<domain>/<route>
```

Both runs return the **identical** response shape. Diverging behavior
between them is a bug in the runtime, not the program.

## IV. What this enables — abstract away

**Any future program is a 30-line .tlisp file + a 10-line consumer
chart values block.** Add a route → `(list "/new-route" 200 body)`.
Change the response → edit the `(define ...)` form. Bump version →
push a new git tag.

The architectural layers from
[`META-FRAMEWORK.md`](META-FRAMEWORK.md) — source, artifact, typed
CR, library chart, consumer chart, deployment — are all
mechanically derivable from the .tlisp source. The user only ever
edits Layer 0 (source) or Layer 4 (cluster overlay values).

## V. Standardization — what's now mandatory

For any new pleme-io program:

1. **Stdlib-only authoring**: every primitive comes from `tatara-lisp-script`'s
   stdlib. No per-program Rust crates. New primitives go in stdlib
   following [§II.2's pattern](#ii2-stdlib-design--tiny--composable--gopython-flavored).

2. **Lisp data first**: structured data is `(list (list k v) ...)`.
   Serialization happens at the boundary via `json-stringify` /
   `yaml-stringify` / `toml-stringify`. **No embedded escape strings
   in source code.**

3. **Test live before merging**: every program runs end-to-end via
   `tatara-script` before its consumer chart commits. The local
   curl path is the contract; the cluster path is mechanical.

4. **Same source, two runtimes**: every program runs both via
   `tatara-script` (local) and via `wasm-operator` (cluster). The
   diff between them is the runtime, never the source.

5. **Helm-first deployment**: per
   [`BREATHABILITY.md` §VII.7](BREATHABILITY.md), the consumer
   `lareira-<name>` chart depending on `pleme-computeunit` is the
   canonical install path. Loose `kubectl apply` is reference only.

## VI. What gets documented + where

This document is the canonical "we got there" artifact. References
land in:

- [`THEORY.md`](THEORY.md) — Pillar 1 reaffirmed (Lisp + WASM/WASI).
- [`SCRIPTING.md` §V](SCRIPTING.md) — example: hello-world is now
  the reference implementation of Tier V.4 (pure-Lisp authoring).
- [`META-FRAMEWORK.md` §VII](META-FRAMEWORK.md) — promotion-ladder
  step "test locally" now points at the live-curl shape.
- [`WASM-STACK.md` §VI](WASM-STACK.md) — the Year-1 (host) /
  Year-2 (cluster) layered identity is exemplified.
- [`blackmatter-pleme/skills/tatara-lisp-program/`](https://github.com/pleme-io/blackmatter-pleme/tree/main/skills/tatara-lisp-program) — NEW skill (TODO):
  step-by-step author-a-program walkthrough using hello-world as the
  template.

## VII. The next iteration's surface

Stdlib primitives we'll add when a future program needs them
(documented as future work, not commitments):

| Domain | Primitive | When |
|---|---|---|
| http_server | `http-serve` (closure-aware) | when a program needs dynamic responses |
| http_server | `http-serve-files` | static-file server |
| http_server | path params (`/hello/:name`) | trivial extension to current shape |
| nats | `nats-subscribe` / `nats-publish` | first function-shape program |
| kube | `kube-list` / `kube-patch` / `kube-watch` | first controller-shape program |
| sql | `sql-query` / `sql-exec` | first DB-using program |

Each future primitive lands in its own `<name>.rs`, follows the
[§II.2 pattern](#ii2-stdlib-design--tiny--composable--gopython-flavored),
and is registered in `mod.rs`. Adding a domain is mechanical;
authoring a program in that domain is one .tlisp file.

## VIII. See also

- [`THEORY.md` Pillar 1](THEORY.md) — language constraint
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — layer hierarchy
- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49-pattern cookbook
- [`pleme-io/programs/hello-world`](https://github.com/pleme-io/programs/tree/main/hello-world) — the source
- [`pleme-io/tatara-lisp/tatara-lisp-script/src/stdlib/http_server.rs`](https://github.com/pleme-io/tatara-lisp/blob/main/tatara-lisp-script/src/stdlib/http_server.rs) — the new primitive
