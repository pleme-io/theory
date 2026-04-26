# Rust ↔ tatara-lisp embedding — bidirectional hosting

> **The user, 2026-04-26**: *"eventually I wanna get to being able to
> do advanced things like host rust and rust host tatara-lisp."*
>
> The two directions compose into a complete embedding story:
>
> 1. **Rust hosts tatara-lisp** — already partially shipped. Any
>    Rust binary embeds the `tatara-lisp-eval::Interpreter`, registers
>    Rust functions as Lisp primitives, evaluates `.tlisp` source.
>    `tatara-lisp-script` is the canonical example.
>
> 2. **Tatara-lisp hosts Rust** — the new direction. A Lisp program
>    instantiates Rust libraries (via WASM components or `dlopen`),
>    invokes their typed functions, returns the results into the Lisp
>    runtime. The user wants this; this document specifies how.
>
> When both directions are live, **the language boundary is dissolved**.
> Every typed Rust crate becomes a tatara-lisp domain (free-of-charge
> codegen via `#[derive(TataraDomain)]`). Every tatara-lisp program is
> embeddable in any Rust binary. The two halves of the pleme-io
> language stack interleave at function-call granularity.

---

## I. Direction 1 — Rust hosts tatara-lisp (today)

```
       tatara-lisp-eval::Interpreter<Ctx>     <─── owns the parser + evaluator
                          │
                          │ register_fn("name", arity, |args, ctx, sp| { ... })
                          ▼
              your Rust crate
                                ┌────────────────────────┐
                                │  ScriptCtx (host data) │   ← per-process state
                                │  http_agent, sops keys │
                                │  registered fns → ...  │
                                └────────────────────────┘
                          │
                          │ .eval_program(source, ctx)
                          ▼
                    Result<Value, EvalError>
```

This is shipped. Examples in pleme-io:

- `tatara-lisp-script` — wraps the interpreter for command-line use.
- `cordel` — embeds the interpreter for Builder-Wake DSL evaluation.
- `seibi` — embeds for sops + spotlight-sync rule evaluation.
- `tend` — could embed for declarative workspace config (planned).

The pattern is the same in every embedder:

```rust
let mut interp: Interpreter<MyCtx> = Interpreter::new();
install_primitives(&mut interp);          // arithmetic, list ops, etc.
my_domain::install(&mut interp);          // your project's primitives
let mut ctx = MyCtx::new();
let forms = read_spanned(source)?;
for form in &forms {
    interp.eval_spanned(form, &mut ctx)?;
}
```

### I.1 Domain extension via #[derive(TataraDomain)]

Rust crates that already use `#[derive(TataraDomain)]` get
auto-generated Lisp surfaces:

```rust
#[derive(TataraDomain, Serialize, Deserialize)]
#[tatara(keyword = "defworkspace")]
pub struct WorkspaceSpec {
    pub name: String,
    pub clusters: Vec<String>,
    pub interval: Duration,
}

pub fn register() {
    tatara_lisp::domain::register::<WorkspaceSpec>();
}
```

Authors write `(defworkspace :name "rio" :clusters ["rio"] :interval 5m)`,
the macro expands to a typed `WorkspaceSpec` available in the host
ScriptCtx. **Six lines of Rust = one new Lisp authoring surface.**

## II. Direction 2 — Tatara-lisp hosts Rust (the new direction)

The reverse direction has three implementation paths, each with
different tradeoffs. Pleme-io will implement all three because they
solve different problems:

### II.1 Path A — Component-model WASM (the canonical path)

Compile a Rust crate to a `wasm32-wasi` component with a `wit-bindgen`
interface; tatara-lisp loads it via wasmtime + invokes typed functions.

```lisp
;; tatara-lisp side:
(define image-resize
  (load-wasm-component "github:pleme-io/rust-libs/image-resize.wasm?ref=v0.1.0"
                       :exports '(resize encode-jpeg encode-webp)))

(define resized-bytes
  (image-resize/resize image-bytes
                       {:width 200 :height 200 :fit :cover}))

(define jpeg-bytes
  (image-resize/encode-jpeg resized-bytes :quality 85))
```

```rust
// Rust side (compiled to wasm32-wasi):
wit_bindgen::generate!({
    world: "image-resize",
    exports: { world: ImageResize },
});

struct ImageResize;
impl exports::pleme::image::ImageResize for ImageResize {
    fn resize(input: Vec<u8>, opts: ResizeOpts) -> Vec<u8> { ... }
    fn encode_jpeg(rgba: Vec<u8>, quality: u8) -> Vec<u8> { ... }
    fn encode_webp(rgba: Vec<u8>, quality: u8) -> Vec<u8> { ... }
}
```

**Properties:**
- Capability-bounded by WASI (the Rust crate has no ambient authority).
- Deterministic; sandboxed; cross-platform.
- Cold start ~10ms per `load-wasm-component` call (cached after first).
- Crate's Cargo.toml stays standalone — no Lisp-aware build hacks.
- Same content-addressed packaging as everything else
  ([`TATARA-PACKAGING.md`](TATARA-PACKAGING.md)).

This is the **default** path for Lisp-hosts-Rust. Most Rust crates
the fleet uses (image processing, crypto, format encoders, serializers,
database clients) compile to wasm32-wasi cleanly and benefit from
the sandboxing.

### II.2 Path B — Native FFI via `dlopen`

For Rust crates that can't or shouldn't compile to wasm32-wasi
(crates with C bindings, dynamic linking, host-only syscalls):

```lisp
;; Loads a native shared library; calls extern "C" functions.
(define libtatara-image
  (load-native-library "github:pleme-io/native-libs/image-x86_64-linux.so?ref=v0.1.0"
                        :exports '((resize "ti_resize" :args (:bytes :int :int) :ret :bytes))))

(define resized (libtatara-image/resize image-bytes 200 200))
```

The shared library exposes `extern "C"` ABI; tatara-lisp threads
arguments through `libloading`-style bindings. **Trades sandboxing
for native performance.** Used sparingly — primarily for FFI to
external systems (bindings to `libnvidia-ml`, `libsystemd`, etc.)
that can't be sandboxed.

### II.3 Path C — Embedded Rust subprocess (the safe-default path)

The Rust crate stays a separate binary; tatara-lisp invokes it via
`process` stdlib + structured IPC (JSON over stdin/stdout, or
`tatara-lisp-eval`-format Sexp):

```lisp
(define result
  (process/invoke "myrust-tool"
                  :args ["resize" "--width=200"]
                  :input image-bytes
                  :input-format :binary
                  :output-format :binary))
```

**Already shipped** — `tatara-lisp-script`'s `process` stdlib supports
this. Used when:
- The Rust binary is pre-built and shipped as an OCI image / Nix package.
- Sandboxing is desired but WASM compilation is cumbersome (e.g., for
  GPU work, kernel-bypass networking, etc.).
- The interaction is short (one invocation, one result).

Process invocation has higher latency (~5-50ms per call) than WASM
(~0.05ms after warm-up) but is the simplest interop surface.

## III. Decision matrix

| Need | Path |
|---|---|
| Pure-Rust library with no external system deps | A — WASM component |
| Image / crypto / format processing | A — WASM component |
| Rust crate using libc / FFI to native systems | B — dlopen |
| GPU / kernel / specialized hardware | B or C — depends on whether sandboxed-WASI works |
| One-off invocation of a Rust CLI (e.g., `cargo`, `rustc`) | C — subprocess |
| Long-running server (e.g., a DB) | not embedded — make it a separate K8s service consumed via HTTP / gRPC |

## IV. The compose-in-both-directions advantage

Two-way hosting enables programs of either flavor to compose freely:

```
┌─────────────────────────────────────────────────────────────────┐
│ tatara-lisp program (controller-shape)                          │
│                                                                  │
│   - watches K8s CR (via kube domain)                            │
│   - on event, computes a hash via load-wasm-component:           │
│       (define hash (blake3-hasher/hash bytes))                   │
│   - logs via log domain                                          │
│   - emits result back to CR (via kube domain)                   │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ embeds + calls
                              │
            ┌─────────────────────────────────────────────┐
            │ Rust libraries called by the Lisp:           │
            │                                              │
            │   - blake3-hasher (WASM component)           │
            │   - jose-jwt-verify (WASM component)          │
            │   - some-fast-parsing-thing (WASM component) │
            └─────────────────────────────────────────────┘

────────────────────────────────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────┐
│ Rust binary (long-running daemon)                                │
│                                                                  │
│   - hot-reloads tatara-lisp config / policy at runtime           │
│   - runs user-defined Lisp logic per-request:                    │
│       interp.eval_program(user_lisp_program, &mut ctx)?          │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ embeds + evaluates
                              │
            ┌─────────────────────────────────────────────┐
            │ Lisp programs hosted by the Rust:           │
            │                                              │
            │   - per-customer routing rules                │
            │   - dynamic alert evaluators                  │
            │   - LLM-prompted controllers                  │
            └─────────────────────────────────────────────┘
```

**Either side can host the other.** A Rust service that needs
runtime-flexible policy embeds Lisp; a Lisp script that needs
performant inner loops calls into Rust. Same content-addressed git
URLs reference both; same BLAKE3 cache holds both.

## V. The shipping order

The fleet doesn't need all three paths immediately. Phased rollout:

### Phase A (now) — Rust hosts Lisp, paths C (subprocess) for reverse

Already shipped. Most Rust binaries embed Lisp via the standard
`tatara-lisp-eval::Interpreter` integration. Reverse direction goes
through `process/invoke` when needed.

### Phase B — Lisp hosts Rust via Path A (WASM component)

Add a `wasm` stdlib domain to `tatara-lisp-script`:

```lisp
;; (load-wasm-component URL :exports '(...))
;; (load-wasm-component LOCAL-PATH :exports '(...))
```

Rust libraries land in `pleme-io/rust-libs/` (new repo); each crate
exposes `lib.rs` + a `wit/world.wit` interface; CI compiles to
wasm32-wasi via `cargo component`. Distribution via the same
content-addressed model
([`TATARA-PACKAGING.md`](TATARA-PACKAGING.md)).

Implementation: ~500 lines of Rust in `tatara-lisp-script/src/stdlib/wasm.rs`
using `wasmtime::component`. Each `(wasm/X)` call is a typed function
invocation against a cached `wasmtime::component::Instance`.

### Phase C — Lisp hosts Rust via Path B (dlopen)

For the small set of crates that need native FFI. Lower priority.

### Phase D — bidirectional embedding maturity

Once Phase B is healthy:
- All `pleme-io/rust-libs/<crate>/` ship `wit/world.wit`.
- The codegen
  ([`TATARA-CODEGEN-MATRIX.md`](TATARA-CODEGEN-MATRIX.md)) gains a
  WIT parser → emits a `(defdomain ...)` for any WIT interface.
- Lisp programs `(use-domain github:pleme-io/rust-libs/<crate>?ref=...)`
  and the runtime fetches + caches + JIT-instantiates.

End state: **a Lisp program that uses 10 Rust crates feels exactly
like a Lisp program that uses 10 Lisp domains.** The boundary is
invisible.

## VI. Hot-replacement extends to embedded Rust

When a Lisp-hosted Rust component is upgraded
([`LIVE-DEPLOYMENT.md`](LIVE-DEPLOYMENT.md)), the wasm-engine pod's
runtime re-instantiates the new component. Component state is *not*
preserved by default (WASM components are stateless across
instantiations); state-bearing components must implement the same
`(snapshot)` / `(receive-state)` hooks the host program does.

This is identical to the host-side hot-replace algorithm — the
component IS a child program from the operator's perspective, just
nested.

## VII. Fleet-wide patterns enabled

Once embedding is bidirectional:

- **Plugin systems** — A Rust service exposes `(extension/X)` hooks;
  Lisp programs register handlers; the service calls into Lisp at
  hook points. Pattern usable for nginx-like extension models, IDE
  plugins, browser extensions.
- **Dynamic dispatch** — A Lisp controller chooses a Rust component
  per-request based on payload (e.g., "image format A → libwebp WASM
  / image format B → libavif WASM"). No re-deploy to add a new format.
- **Polyglot domains** — A `defdomain` is partially auto-generated
  from OpenAPI (codegen-matrix), partially hand-written tatara-lisp,
  partially an embedded Rust component. Authors mix freely.
- **Zero-copy interop** — WASM component model supports shared
  memory; large buffers pass by reference between Lisp and Rust
  with no marshalling. Image, audio, video pipelines benefit.

## VIII. The closing principle

The architecture's language boundary becomes a **selection**, not a
**barrier**:

| Choose Rust when... | Choose tatara-lisp when... |
|---|---|
| Inner loop performance dominates | Iteration speed dominates |
| Memory layout needs precise control | Logic shape changes per cluster / customer |
| Compile-time type safety is the proof | Runtime composition + macros are the value |
| External binding (FFI, C lib, kernel) | All deps are Lisp / Rust / WASM components |
| Long-running, never-changes process | Hot-replace + state migration matter |

The fleet picks per-task. The two languages embed each other. Same
git URLs. Same BLAKE3 cache. Same Helm packaging. Same operator
hot-replace. The user's stated long-term goal — *"host rust and
rust host tatara-lisp"* — is the steady state.

## IX. See also

- [`THEORY.md` Pillar 1 + Pillar 12](THEORY.md) — Rust + tatara-lisp
  + WASM/WASI; generation over composition
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp authoring standard
- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`TATARA-CODEGEN-MATRIX.md`](TATARA-CODEGEN-MATRIX.md) — codegen
  surface (the WIT parser fits here)
- [`LIVE-DEPLOYMENT.md`](LIVE-DEPLOYMENT.md) — hot-replace algorithm
- [`tatara/docs/rust-lisp.md`](https://github.com/pleme-io/tatara/blob/main/docs/rust-lisp.md)
  — the existing Rust+Lisp cookbook (Direction 1 deep-dive)
- [`wit-bindgen`](https://github.com/bytecodealliance/wit-bindgen)
  — the component-model interface compiler
- [`wasmtime` component model docs](https://docs.wasmtime.dev/) — the
  embedding API that powers Path A
