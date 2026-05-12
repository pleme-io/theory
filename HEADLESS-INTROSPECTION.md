# Headless Introspection — agent-driven self-diagnosis for GPU apps

> **★★★ CSE / Knowable Construction.** Canonical CSE specification at
> `pleme-io/theory/CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`. This
> document declares a specific application of CSE to the pleme-io GPU
> app fleet (mado, fumi, kagi, hibiki, namimado, nami, tanken, …):
> every GPU app exposes a **typed, headless, MCP-driven
> introspection surface** so any operator-visible bug becomes a
> reproducible artifact in one command — and every fix lands with its
> permanent regression test.

## Destination

**Every pleme-io GPU app provably self-diagnoses.** Every visual,
input, state, or behavioural bug is reproducible by an agent or a CI
job inside a sandbox; every fix lands with a permanent regression
test; the corpus of provable behaviour grows monotonically commit
after commit. Three promises follow:

| Promise | Mechanism |
|---|---|
| "Visual rendering is correct" | Scenario PNG diff against golden (M1+) |
| "Terminal / state core is correct" | Scenario grid diff against golden (M0, today) |
| "Bug fix → permanent regression test" | `mado record` + scenario harness — one command |

The substrate gets strictly stronger with every commit. There is no
fix-and-forget path.

## The four primitives

Every GPU app implementing this pattern ships four typed primitives.
mado is the first consumer — every primitive is concrete code today.

### 1. Headless mode

A subcommand that runs the app's state core (terminal, music player,
chat model, browser DOM) **without winit / wgpu / windowing**. The
state core's reader pumps still run, the event loop still steps —
but no surface is created and no GPU adapter is acquired. For mado
this lives behind `mado mcp` (also serves as the MCP transport).

### 2. Typed snapshot

A serde-serializable struct that captures **every byte of state
the renderer reads**. For mado this is `GridSnapshot` — `cols × rows
× CellSnapshot { ch, fg, bg, attrs, width }` plus cursor row/col/
visible. The snapshot is the source of truth for "what does the app
think is on screen"; a renderer can only produce pixels that
faithfully translate this state.

### 3. MCP tool surface

A `kaname`-backed MCP server exposing the canonical seven tools:

| Tool | Purpose |
|---|---|
| `spawn_term` (or `spawn_<app>`) | Create a headless session from a typed spec |
| `send_keys` (or `send_input`) | Write raw bytes / structured events |
| `get_output` | Read state as plain text (compact) |
| `snapshot_grid` (or `snapshot_<state>`) | Full structured snapshot for diffing |
| `resize_session` | Change geometry mid-session |
| `close_session` | Drop the session, kill its child |
| `list_sessions` | Enumerate live sessions in the registry |

The MCP subcommand **must** route tracing to stderr (e.g.
`shidou::init_tracing_to_stderr`) — stdout is the JSON-RPC channel.

### 4. Scenario harness

A `tests/scenarios/` directory of `*.scenario.yaml` files + a
`tests/scenarios.rs` integration test that walks them, dispatches
each through the app's `scenario-run <path>` subcommand, and asserts.
Each YAML declares: dimensions, shell/state-core, input steps
(`send`, `wait_for_text`, `wait_ms`, `resize`, `reset`), and `expect`
assertions on text / cursor / specific cells.

## The compounding loop

```
operator sees bug X
    │
    ▼
mado record --output X.scenario.yaml -- sh -c 'reproducer'
    │
    ▼
operator edits the captured YAML: adds `expect:` assertions for
the cells / text that *should* hold
    │
    ▼
file lands in tests/scenarios/  →  cargo test --test scenarios
    │
    ├── (a) FAILS at landing → diagnose, fix the bug, re-run
    └── (b) PASSES → commit; the bug is locked closed forever
```

There is no scenario without an underlying real bug or a real
promise. Speculative scenarios are dead weight. The corpus is a
substrate of provable behaviour, not a wishlist.

## Cell-state vs pixel-state — the two-tier diagnosis tree

Visual bugs always belong to exactly one tier:

```
                 visual rendering bug
                          │
                ┌─────────┴─────────┐
                ▼                   ▼
        snapshot_grid           snapshot_grid
        says the same           says different
        thing as before         from before
                │                   │
                ▼                   ▼
      pixel-state bug         cell-state bug
      (renderer)              (state core)
                │                   │
                ▼                   ▼
        fix wgpu /              fix vte parser /
        glyphon /               state machine /
        box_drawing             input handling
        path; capture           path; capture
        snapshot_png            snapshot_grid
        golden (M1)             golden (M0)
```

The cell-state tier is **load-bearing today** — `snapshot_grid` is
implemented and CI-gated. The pixel-state tier is M1 (wgpu offscreen
target → PNG → golden diff); see "Roadmap" below.

## Anti-patterns

The PRIME DIRECTIVE forbids:

- **Diagnosing visual bugs from screenshots.** Screenshots are read
  by humans; cell snapshots are read by both humans and CI. A bug
  reproduced only on the operator's screen is one bug; a bug
  reproduced in a scenario is a class of bugs we provably catch.
- **Per-app duplication of the four primitives.** Headless mode +
  snapshot + MCP + scenario harness compose as one shape; when the
  second GPU app lands, the shape lifts into `garasu-introspect`.
- **Stdout pollution in MCP mode.** Any log line on stdout breaks
  the JSON-RPC framing. The shared
  `shidou::init_tracing_to_stderr` exists to enforce this — never
  call `init_tracing()` in an MCP subcommand.
- **Scenarios without ties to bugs / promises.** The corpus is
  audit-gated by the `scenario-corpus-present` cse-lint invariant;
  it doesn't validate that every scenario traces to a real
  reproducer, but the principle stands: don't add scenarios that
  don't earn their place.
- **Hand-written test bytes for complex apps.** For atuin / fzf /
  vim, the byte stream isn't worth hand-authoring — use
  `mado record`. Hand-written sequences are appropriate only for
  scenarios that test specific escape sequences (cursor positioning,
  SGR, etc.) where the bytes ARE the test.

## What lifts (substrate compounding)

The mado-first implementation contains primitives that other GPU
apps will demand verbatim:

- **`shidou::init_tracing_to_stderr`** — already lifted (commit
  `bd1f0b5`). Every MCP/LSP-shaped pleme-io binary inherits.
- **`Session` / `Registry` / `Snapshot` shape** — mado-local today.
  Lifts to `garasu-introspect` (or `kaname-app-introspect`) the day
  the second consumer (fumi or kagi) implements the pattern. The
  trait surface that emerges:
  ```rust
  pub trait Introspectable {
      type Snapshot: serde::Serialize;
      fn snapshot(&self) -> Self::Snapshot;
  }
  pub trait Driveable {
      fn send_input(&self, bytes: &[u8]);
      fn resize(&self, w: u32, h: u32);
  }
  ```
- **Scenario YAML format + runner** — mado-local today. Same shape
  works for any introspectable state. Lifts to
  `garasu::scenario` once the second consumer needs it.
- **`mado record`** subcommand — mado-local today. The
  capture-from-PTY shape is mado-specific; the
  capture-from-state-core shape generalises and lifts to
  `garasu::record`.
- **cse-lint invariants** — `gpu-app-headless-mode`,
  `mcp-stdout-clean`, `scenario-corpus-present`. Fleet-wide gates
  the day they land.

## Roadmap

| Milestone | Deliverable | Status |
|---|---|---|
| **M0** | mado headless + MCP + grid snapshot + scenario harness + record | **shipped** |
| M1 | `snapshot_png` via wgpu offscreen target + PNG golden diff | ~1h next |
| M2 | Scenario corpus growth: every real-app bug captured + commit | continuous |
| M3 | Property tests for input fuzz + Unicode width invariant + resize consistency | **shipped** |
| M4 | `garasu-introspect` lift (when second consumer arrives) | demand-driven |
| M5 | xterm vttest replay → "vttest pass rate" gauge | week 2 |
| M6 | cse-lint invariants land — `gpu-app-headless-mode`, `mcp-stdout-clean`, `scenario-corpus-present` | this week |
| M7 | `rust-gpu-app` repo-forge archetype ships the scenario directory + headless mode by default | week 3 |

## Theory frame

This document is downstream of:

- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md) — operator+agents+substrate model
- [`THEORY.md`](./THEORY.md) — Pillar 10 (proof discipline), Pillar 12 (generation over composition)
- The pleme-io PRIME DIRECTIVE — every recurring pattern lifts to a macro / library

The headless-introspection pattern is **Pillar 10 operationalised
for GPU apps**: every visible behaviour becomes a typed proof
obligation; every fix is a theorem; the substrate's promises about
GPU app correctness move from self-report to compiler-and-CI-checked
fact.

## Canonical example — mado

- Headless mode: [`pleme-io/mado/src/session.rs`](../mado/src/session.rs)
- MCP tools: [`pleme-io/mado/src/mcp.rs`](../mado/src/mcp.rs)
- Scenario harness: [`pleme-io/mado/src/scenario.rs`](../mado/src/scenario.rs) + [`tests/scenarios/`](../mado/tests/scenarios/)
- Property tests: [`pleme-io/mado/src/terminal.rs#proptests`](../mado/src/terminal.rs)
- Record subcommand: `mado record --output X.scenario.yaml -- <cmd...>`

When fumi / kagi / hibiki / namimado / nami next need MCP-driven
debugging, they implement the same four primitives. The day the
second consumer lands, mado's `Session` shape lifts into
`garasu-introspect` and both apps inherit the canonical surface.
