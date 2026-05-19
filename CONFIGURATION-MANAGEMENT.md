# ★★ Configuration Management — pleme-io Prime Directive

> **Status:** canonical. Referenced from `pleme-io/CLAUDE.md` § "Prime Directives". Every operator-facing tool in the fleet either complies, has a tracked migration in flight, or carries an explicit `skip-shikumi:` waiver at the top of its `CLAUDE.md` justifying the deviation.

The pleme-io fleet has converged on **one** way to do configuration. This document is the contract.

---

## I. The shape

Every operator-facing tool in pleme-io exposes its configuration through **four mandatory primitives** and **three Nix surfaces**.

### Primitive 1 — Typed schema

A Rust struct (or, for non-Rust tools, an equivalent typed declaration) describing every operator-tunable knob.

* Lives in a dedicated `<tool>-config` crate (or the tool's main `src/config.rs` if it has no library boundary).
* Derives `serde::Deserialize + Deserialize` (or Serialize for round-trip tests).
* Carries `#[serde(deny_unknown_fields)]` so typos in operator YAML surface at parse time, not runtime.
* Every newly-added field has `#[serde(default = "...")]` for backwards-compat — a pre-existing operator's YAML keeps parsing across upgrades.

### Primitive 2 — Discovery + load via shikumi

```rust
let store = shikumi::ConfigStore::<MyConfig>::load_and_watch(
    &path,
    "<TOOL>_",   // env-prefix; nested keys via FOO_BAR_BAZ
    |cfg| { /* react to live edits */ },
)?;
```

* `ConfigStore::load(path, env_prefix)` for one-shot tools (`tool -c '…'`).
* `ConfigStore::load_and_watch(path, env_prefix, on_reload)` for long-running tools.
* `ConfigStore::replace(value)` for runtime-driven config push (RPC, MCP tool, programmatic theme toggle).
* `ConfigStore::reload()` for manual operator-triggered re-read.
* `store.get() -> Guard<Arc<T>>` for lock-free reads.

Shikumi gives you, in one call: XDG discovery + env-prefix override layer + `Arc<ArcSwap<T>>` lock-free reads + notify file watcher + generation counter + last-publish + last-reload-error bookkeeping. No tool re-implements any of this.

### Primitive 3 — Hot-reload

Every long-running tool watches its config and reacts within the operator's edit-to-effect window (≤ 1 second). Implemented automatically via `ConfigStore::load_and_watch`.

* **No restarts** for theme changes, keybind tweaks, prompt edits, palette swaps.
* **Crash-safe**: malformed YAML keeps the prior config in place + logs the error + surfaces it via `store.last_reload_error()`.

### Primitive 4 — Multi-subscriber broadcast (when needed)

For tools whose runtime state is consumed by multiple processes (a daemon broadcasting to N attached clients), wrap the `ConfigStore` with a tool-local subscribe / fan-out layer.

* Pattern: `LiveConfig` in `tear-config` — `Vec<mpsc::Sender<Arc<T>>>` + swap-remove on dead-sender prune.
* Why tool-local: shikumi has a single `on_reload` callback; broadcast topology is consumer-shaped and doesn't belong upstream.

---

## II. The Nix surfaces

Every tool that ships an operator-facing config publishes **three** Nix modules in `module/`:

### Surface A — `module/home-manager/default.nix`

* Mirrors the Rust schema **one-to-one**: every typed field reachable as a typed Nix option.
* Generates `~/.config/<tool>/<tool>.yaml` from `pkgs.formats.yaml` so operators get `darwin-rebuild` / `home-manager switch` type-checking instead of editor save.
* `manageConfig = false` opt-out for operators who hand-author the YAML.
* `configPath = "..."` override for non-standard XDG setups.
* `extraConfig = { ... }` escape hatch for fields added to the Rust struct after the module's last update.
* `setAsInteractiveShell` / `setAsDefaultEditor` / equivalent opt-in where applicable.

### Surface B — `module/nixos/default.nix`

* System-wide install via `environment.systemPackages`.
* Opt-in `/etc/shells` / `/etc/services` / equivalent registration where applicable.
* Per-user typed config still flows through HM — the NixOS module is "binary on PATH + optional system registration."

### Surface C — `module/darwin/default.nix`

* nix-darwin sibling of Surface B. Same shape; macOS-specific extras (launchd agents, NSApp wiring) live here.

### Wiring

The tool's `flake.nix` exposes:

```nix
homeManagerModules.default = import ./module/home-manager;
nixosModules.default       = import ./module/nixos;
darwinModules.default      = import ./module/darwin;
```

Consumers (the `nix` private repo, blackmatter-pleme, etc.) `imports = [ <tool>.homeManagerModules.default ];` and configure via the typed Nix surface. The YAML emission is mechanical; the operator never hand-edits unless they explicitly opt out.

---

## III. Cross-tool composition

Once **two** tools share a config concept (palette, font, keybind), the concept lifts to a shared crate (working name: `pleme-shared`) and both tools declare a typed reference:

```yaml
# ~/.config/mado/mado.yaml
theme:
  palette_ref: nord-frost

# ~/.config/frost/frost.yaml
prompt:
  palette_ref: nord-frost
```

The shared reference resolves through one source of truth. Theme changes propagate fleet-wide via one edit.

Same pattern applies to font refs, keybind tables, and any other concept ≥ 2 tools touch.

---

## IV. Verification ladder

Every shikumi consumer ships **at minimum**:

1. **Default round-trip test** — `cfg = T::default(); yaml = to_string(cfg); back = from_str(yaml); assert_eq!(cfg, back);`
2. **Empty-YAML → defaults test** — `cfg = from_str("{}"); assert_eq!(cfg, T::default());` — proves backwards-compat for pre-existing operator files.
3. **Unknown-field rejection** — `from_str("typo_field: 42")` must fail.
4. **Missing-file → defaults** — `ConfigStore::load("/does/not/exist")` returns defaults, not an error.
5. **Malformed-YAML → typed error, no panic** — `ConfigStore::load("invalid: : :")` returns `Err(ShikumiError)`.

The first three are pure unit tests. The last two are integration tests against a real `ConfigStore`. Mado + tear + frost each have ≥ 6 of these.

---

## V. Compliance ladder

A tool that does **not** comply must carry one of:

* `skip-shikumi: <one-line reason>` at the top of its `CLAUDE.md` — for tools that genuinely don't have operator-edited config (a pure subprocess wrapper, a one-shot CLI utility).
* `pending-shikumi: M<n> in <repo>/docs/SHIKUMI-ECOSYSTEM.md` — for tools mid-migration.

Otherwise the tool is **in violation** of this directive and the next operator-facing change to it should pull in the migration. See `~/code/github/pleme-io/frost/docs/SHIKUMI-ECOSYSTEM.md` for the canonical migration plan template.

---

## VI. Non-Rust tools

Where Rust isn't the implementation language, the **schema-and-discovery contract** still applies; only the storage layer differs:

* **Ruby (Pangea Ruby DSL)** — typed `Dry::Struct` resources mirror what shikumi would produce; the operator-facing format is still YAML when there's an edit-this-file surface.
* **Tatara-lisp authoring** — the lisp compiles to a shikumi YAML, which the runtime then loads. Operators get the lisp ergonomics; runtime + tooling get the typed YAML. See `frostmourne` migration M2.
* **TypeScript / Zig / Go** — same contract: typed schema, XDG discovery, env-prefix overrides, hot-reload if long-running. Today there's no shikumi binding outside Rust; when one of these tools needs adoption, the highest-leverage move is to publish a thin shikumi binding in the target language rather than re-implement.

---

## VII. The why

Three reasons this is a prime directive, not a suggestion:

1. **Operator UX surface area.** One config grammar across N tools means muscle memory transfers. Without this directive, every tool's YAML conventions drift slightly and the cost is real and recurring across every operator's day.

2. **Test infrastructure compounds.** `ConfigStore::load_and_watch` is tested once in shikumi; every consumer inherits the verification. The L1/L2/L3 mado verification ladder (CPU rect invariants → headless GPU readback → scenario goldens) is the same pattern: pin the contract once in the primitive, every consumer gets the regression coverage for free.

3. **Cross-tool composition becomes possible.** When palettes/fonts/keybinds live as typed refs in a shared schema, one declaration flows everywhere. Today operators re-author the same color palette in three configs. With this directive's section III in force, one declaration propagates fleet-wide.

---

## VIII. Status snapshot (2026-05-19)

| Tool | shikumi | HM module | NixOS module | Darwin module |
|---|---|---|---|---|
| mado | ✓ | ✓ | n/a | ✓ |
| tear | ✓ (M1 today) | partial | n/a | n/a |
| frost | ✓ | ✓ | ✓ | ✓ |
| frostmourne | partial (M2) | ✓ | n/a | n/a |

Other operator-facing tools (ayatsuri, kaname, kontena, cofre, tend, kikai, seibi, kurage, umbra, codesearch, zoekt-mcp, kindling, namimado, hibikine) are subject to fleet-wide audit results — see `pleme-io/CLAUDE.md` § "Compliance audit" for the next-session table.

The directive applies fleet-wide from this commit forward. Every new tool starts compliant by default — substrate's `rust-tool-release-flake.nix` + the per-tool config crate template + the HM/NixOS/Darwin module trio is the canonical scaffold.
