# MADO ⇄ TEAR — M5 De-Overlap

> Canonical roadmap for cutting feature overlap between **mado** (the
> pleme-io GPU terminal emulator) and **tear** (the pleme-io
> tmux-compatible multiplexer). Replaces the older
> `tear/CLAUDE.md` § "tear ↔ mado pairing" table as the source of
> truth for the M5 transition.

## I. Destination

| Layer | Owns | Does NOT do |
|---|---|---|
| **mado** | GPU rendering, VT byte → cells, IME, scrollback, hyperlinks, kitty graphics, OSC, MCP introspection of the *single visible terminal* | Multiplexing of any kind. Zero `Pane`/`Tab`/`Window` state. Zero "multi-pane render path." |
| **tear** | Sessions, windows, panes, layouts, splits, focus, keybind tables, status bars, status across client disconnects | GPU pixels. Font shaping. Direct GPU buffer ownership. |
| **shared** | Typed primitives (`tear-types`), shikumi config style, ishou tokens, parking_lot, mimalloc | — |

**Operator experience:** identical to ghostty + tmux today. `tear` runs inside mado as a normal child process (just like `tmux` inside Alacritty / `ghostty`). The difference is structural: one process draws pixels, one process owns the multiplexer state. No primitive lives in two places.

**Why this matters:** the org-level Compounding Directive forbids "solving the same problem in two repos." Today mado has `pane.rs` (418 LOC), `tab.rs` (331 LOC), `window.rs` (468 LOC) duplicating what tear already does correctly. Every fix has to land in both places; every divergence is silent drift. M5 collapses this.

## II. Real four-phase plan

Phase numbering corresponds to commits in the tear + mado repos.

### Phase 1 — tear-daemon RPC functional ✓ SHIPPED

Commit `tear@0d0a240`. tear-daemon binds a UDS, tear-client connects, every `MultiplexerControl` op round-trips. 21 new tests green. The wire is CBOR over length-prefixed framing (CBOR was chosen because `LayoutNode` uses `#[serde(tag = "kind")]` — bincode rejects internally-tagged enums). `tear daemon` and `tear attach` subcommands work end-to-end.

This unblocks every consumer that wants to drive a running tear from outside the process tree (the `tear` CLI, future scriptable drivers, fleet operators over SSH). It does NOT unblock mado, which goes through Phase 2 instead.

### Phase 2 — tear-core gains per-pane vte parsing + cell grids

**Status:** ★ MVP SHIPPED at `tear@0dbfd9c`. The full surface (mado's terminal.rs moves here) is Phase 2.5, deferred.

**What landed in MVP:** `tear-core::pane_grid::PaneGrid` wraps a `vte::Parser` + a `[rows][cols]` cell buffer with cursor tracking + auto-wrap + scroll-on-overflow. `tear-types::pane_snapshot::{Cell, PaneSnapshot}` are the wire-serializable types. `InProcess` installs one grid per pane in `spawn_pty_for`; bytes from the PTY reader thread feed straight into the parser. `MultiplexerControl::pane_snapshot(pane_id) -> ControlResult<PaneSnapshot>` is the new trait method (default impl returns `Rejected` for backends like tear-tmux that can't observe rendered state). The wire ferries `Request::PaneSnapshot` / `Response::PaneSnapshot`. A `tear snapshot <pane-id>` CLI prints the rendered grid against a running daemon. End-to-end test (`end_to_end_send_keys_then_pane_snapshot_shows_output`) verifies the full vertical: `send_keys → kernel PTY → child shell → stdout → reader thread → PaneGrid::feed → vte parser → snapshot → CBOR → UDS → client`.

**Out of MVP scope (deferred to Phase 2.5):** SGR colors / attrs, alt-screen, scrollback, tab stops, scrolling regions, IRM, DECSCUSR, hyperlinks, OSC, DEC mode 2026 (synchronized output), Kitty graphics, sixel. These all already live in mado's `terminal.rs` (5,485 LOC); Phase 2.5 ports that code into `tear-core::pane_grid` so both apps share one state machine.

### Phase 2.5 — port mado's terminal.rs to tear-core (heavy lift)

**Status:** not started. **Depends on:** Phase 2 MVP (done).

The 5,485-LOC `mado::terminal::Terminal` MOVES into `tear-core::pane_grid` as the full SGR + alt-screen + scrollback + hyperlinks + Kitty + OSC + sync-output surface. The receiver (`tear-core`) inherits the entire terminal-state-machine; the donor (`mado`) becomes a pure renderer over a typed grid handle. All 575 mado terminal-state-machine tests run against the moved code unchanged.

### Phase 3 MVP — mado linked against tear-client ✓ SHIPPED

**Status:** shipped at `mado@3c193f4` (binary) + `nix@9eef0d2` (deployed). Phase 2.5 parts 1-3 shipped first (`tear@c5b1475` + `f370902` + `faa1587`) so tear-core covers SGR + alt-screen + scrollback + IL/DL/ICH/DCH/ECH/REP/IRM + OSC titles + DEC 25 + push subscriptions.

**What landed:** `mado tear-attach <pane> --socket <path>` subcommand. Mado links against `tear-types` + `tear-client`, opens a fresh UDS connection to the daemon, subscribes via `Request::Subscribe`, and writes received `Response::PaneBytes` straight to stdout. The host terminal renders SGR / cursor moves / alt-screen correctly because the bytes are the literal VT stream from the child shell.

End-to-end integration test (`mado/tests/tear_attach_integration.rs`) spawns `tear_daemon::start` in-process, creates a `/bin/sh` session via `tear-client`, subprocess-spawns the just-built mado binary with `tear-attach`, sends `printf MARKER\n`, asserts the marker text appears on mado's piped stdout. Proves the full vertical:

```
tear-client RPC → daemon → InProcess.send_keys → kernel PTY →
/bin/sh → child stdout → reader thread → on_bytes →
subscribers fan-out → daemon's serve_subscription thread →
CBOR Response::PaneBytes → UDS → mado-as-subscriber's reader
thread → callback → mado stdout
```

### Phase 3.1 — mado renders panes in its own GPU window

**Status:** not started. **Depends on:** Phase 3 MVP (done) + Phase 2.5 hyperlinks/sync output (partial today).

Currently `mado tear-attach` writes bytes to stdout — the host terminal renders. Phase 3.1 opens a real mado window and feeds the bytes into mado's own `Terminal` instance so the existing GPU render pipeline renders. The wire stays identical; only the consumer-side changes. Adds keyboard forwarding (mado window → `client.send_keys(pane, bytes)`) and resize forwarding (mado window resize → tear-daemon size update, when that RPC exists).

**Scope:** replace `mado::render::SharedTerminal` (currently `Arc<RwLock<Terminal>>`) with `Arc<InProcess>` + the focused `PaneId`. Each render frame:

1. Read focused `PaneId` from mado's "view" state (one tiny struct).
2. Call `inproc.pane_snapshot(focused)` to get the typed cell grid.
3. Render exactly as today (rect pipeline + text pipeline + post-process).

Mado's keybind handlers for split / new-tab / focus-pane call `inproc.split_pane(...)` etc. directly. No tear-client / no RPC. The InProcess is held in-process, just like a normal Rust struct.

**Acceptance:**
- All 575 mado tests still pass (they now run against InProcess-backed grids).
- Existing mado UX is preserved bit-for-bit (Ctrl-B prefix, all keybinds, all MCP introspection).
- mado's `WindowState::all_panes` / `focused_pane` / etc. become trivial inproc wrappers (or are removed entirely).

### Phase 3.1 MVP ✓ SHIPPED

**Status:** shipped at `mado@2297aff` + `nix@da5334f` (deployed). Adds `mado tear-attach --gpu <pane>` mode: opens a real madori GPU window backed by a single `Terminal` (no WindowState), subscribes to the pane's PTY byte stream from tear-daemon, feeds bytes into the local Terminal's vte parser, renders via mado's existing `TerminalRenderer` in its single-pane fallback path. Forwards text-producing keystrokes via `client.send_keys`, forwards window resize via `client.pane_resize_absolute`. New module `mado/src/gui_tear_attach.rs` (158 LOC) lives parallel to mado's legacy WindowState-backed GUI so Phase 4 can delete the legacy later.

MVP non-goals (Phase 3.1.x slices): special-key translation (arrows / function / chords), clipboard / selection / search / Rhai scripts. Only `KeyEvent.text` (UTF-8 char input) forwards today.

### Phase 4 — delete legacy

**Status:** 4.1 + 4.2 SHIPPED at `mado@bdb8721` + `nix@5217496`. 4.3 → 4.5 deferred.

**Phase 4.1 (shipped):** MCP `split_pane` + `new_tab` tool handlers + `SplitPaneInput` struct + their unit tests removed. mado's MCP no longer advertises multi-pane operations.

**Phase 4.2 (shipped):** `Action::NewTab` / `CloseTab` / `NextTab` / `PrevTab` / `SplitHorizontal` / `SplitVertical` / `FocusNext` / `FocusPrev` / `ClosePane` enum variants deleted. `parse_action` returns `None` for the corresponding strings + `goto_split:*` + `close_surface`. Default binding count: 26 → 17. main.rs keybind handler arms for these actions deleted. 5 tests removed that exercised the deleted actions. Net `-153` LOC across mado.

**Phase 4.3 — 4.5 (deferred, not started):** Strip `Arc<Mutex<WindowState>>` from main.rs (~113 lines reference WindowState/pane/tab; 34 call `window_for_events.lock()` directly). Refactor `render.rs` to drop the multi-pane branch (`render_multi_pane` + `PaneRect` iteration). Delete `pane.rs` / `tab.rs` / `window.rs`. Verify all 570+ mado tests + every scenario YAML still pass.

Net delete at full completion: 1,217 LOC (pane.rs 418 + tab.rs 331 + window.rs 468). main.rs single-Terminal refactor adds tighter Terminal-direct call sites in place of the WindowState indirection.

**Cut-list precondition:** mado main.rs (2,208 LOC) holds `window_for_events.lock().unwrap()` calls throughout the keybind handler. Phase 4.3 demands replacing those with single-Terminal lookups — substantial refactor risk-bounded by mado's 570-test bin-test suite + the existing scenarios. Each callsite needs `WindowState::focused_pane()` → `terminal.read()` style rewriting + dropping the multi-pane methods (`split`, `new_tab`, `focus_next`, etc.).

## VII. Cumulative session ledger (Phases 1 → 3.0)

| Phase | Commit | What |
|---|---|---|
| 1 | `tear@0d0a240` | daemon UDS RPC + client (CBOR over length-prefix), 4 daemon + 2 client tests |
| 2 | `tear@0dbfd9c` | `MultiplexerControl::pane_snapshot` wire, e2e poll test |
| 2.5.1 | `tear@c5b1475` | SGR colors (8/16/256/truecolor) + alt-screen (47/1047/1049) + scrollback + DECSC/DECRC + xterm-deferred wrap |
| 2.5.2 | `tear@f370902` | IL/DL/ICH/DCH/ECH/REP/IRM + OSC 0/1/2 (titles) + DEC mode 25 (cursor visibility) |
| 2.5.3 | `tear@faa1587` | Push subscription wire (`Request::Subscribe` → streamed `Response::PaneBytes`), e2e push test |
| 3 MVP | `mado@3c193f4` + `nix@9eef0d2` | `mado tear-attach <pane>` subcommand (stdout streaming), cross-process integration test |
| 3.0+ | `tear@f7f64b1` + `mado@5266193` + `nix@b10f419` | `pane_resize_absolute` RPC (PTY + grid in lockstep) + 4 proptests on PaneGrid (random bytes never panic, cursor in bounds, dimensions consistent, resize keeps cursor in bounds) + 2 fanout/resize integration tests |
| tests | `tear@8ea3fd9` | many-sessions stress + subscriber-cleanup-after-kill |
| roadmap | `theory@55bfaa7` | this document, kept current |
| classification | `mado@a0813df` | mado/CLAUDE.md flags pane/tab/window LEGACY |
| 3.1 MVP | `mado@2297aff` + `nix@da5334f` | `mado tear-attach --gpu` opens real GPU window backed by Terminal + tear subscription + key/resize forwarding (158 LOC new module) |
| 4.1+4.2 | `mado@bdb8721` + `nix@5217496` | MCP split_pane/new_tab gone, 9 multiplexing keybind Actions gone, default bindings 26→17, net -153 LOC |

**Workspace test total**: 70+ green across tear (35 core w/ 4 proptests + 8 client w/ 4 e2e + 4 daemon + 15 types + others) + 570 mado bin tests + 1 mado tear-attach integration test (full cross-process vertical: tear-client → daemon → InProcess → /bin/sh PTY → reader thread → subscribers → CBOR/UDS → mado-as-subscriber → stdout). Zero flakes across multiple sequential runs.

## III. What `tear daemon` is for, given mado embeds InProcess

When mado embeds `Arc<InProcess>` directly, the tear-daemon isn't on mado's hot path. It still ships value:

- **CLI access to a running mado**: `tear list` / `tear attach` lets the operator inspect / drive panes from outside the GPU process. mado at boot spins up its own embedded InProcess AND opens a tear-daemon UDS exposing it.
- **Persistence**: tear-daemon's lifecycle is decoupled from mado's; sessions survive a mado crash. Phase 6 — a future tear-daemon mode that mado can hand off its InProcess to before exit + reconnect from on next launch.
- **Fleet operators over SSH**: a remote `tear-client` can drive a remote mado's panes for centralized fleet ops.
- **Non-mado consumers**: any TUI / status-bar / scripted driver that wants typed session info.

So Phase 1's RPC is foundational for tear-as-a-platform, even though mado itself bypasses it.

## IV. Cut-list (what disappears at Phase 4)

From `pleme-io/mado/src/`:

| File | LOC | Goes where |
|---|---|---|
| `pane.rs` | 418 | tear-core (`PaneGrid`, layout) |
| `tab.rs` | 331 | tear-core (window/tab management) |
| `window.rs` | 468 | tear-core (`Registry`) |
| Multi-pane branches in `render.rs` | ~200 | Deleted — mado renders one pane at a time |
| `MCP split_pane` / `new_tab` stubs | ~30 | Deleted — tear-daemon's MCP owns these |

Mado's own `terminal.rs` (5,485 LOC) MOVES to `tear-core::pane_grid` at Phase 2 rather than being deleted — it's the gravitational center of the shared state machine.

## V. Interim posture (today, between Phase 1 and Phase 2)

- mado retains `pane.rs` / `tab.rs` / `window.rs` unchanged.
- mado's MCP `split_pane` / `new_tab` stay stubs returning a not-implemented response (today's behavior).
- tear-daemon is functional; `tear attach` proves the RPC works.
- New tools / scripts / agents that want multiplexer ops use `tear-client`.
- New mado features should NOT add to `pane.rs` / `tab.rs` / `window.rs` — those modules are slated for deletion.

## VI. Acceptance for "M5 done"

1. `pleme-io/mado/src/{pane,tab,window}.rs` do not exist.
2. mado's `nix build .#default` produces a binary that depends on `tear-core`.
3. All 575 mado terminal tests pass + all tear-core tests pass.
4. Running `mado` on a fresh shell, splitting via Ctrl-B %, and quitting work bit-for-bit identically to today's mado.
5. `theory/MADO-TEAR-M5.md` (this document) is marked "complete" at the top.
6. `tear/CLAUDE.md` § "tear ↔ mado pairing" is updated to past-tense.

## VII. Open questions

- **Per-pane vte parser lifecycle**: tear-core owns the parser; what happens if a client subscribes to byte stream during parse? Likely answer: tear-core's parser is always authoritative, byte-stream subscribers receive a *copy* of bytes (Phase 6 — not on M5 critical path).
- **Kitty graphics / image state**: per-pane in mado today; needs to follow the grid to tear-core. Phase 2 scope.
- **Search / selection / URL detection**: which side owns them? Probably mado (they're render-time concerns over a snapshot, not state-machine concerns). To be confirmed during Phase 3.
- **MCP surface split**: mado-MCP keeps `snapshot_grid` / `get_output` / `send_keys` (against the current focused pane). tear-MCP (future) gets `list_sessions` / `split_pane` / `new_window` / etc. Mado's existing `split_pane` / `new_tab` MCP tools either delete or forward to tear's MCP.

---

**Tracking:** this doc is updated at the end of each phase. After Phase 4 lands, mark all phases ✓ at the top and link the final commits.
