# Theme Architecture

> Companion to [`THEORY.md`](./THEORY.md) §III (the typescape) and §V
> (provability). This document is the canonical destination for how
> theming flows across the pleme-io fleet — from one typed source of
> truth to every render target, with sRGB/linear correctness enforced
> by the type system rather than the operator's vigilance.

## The promise

**Theming is a typescape primitive, owned by `ishou`, consumed by every
pleme-io surface, gamma-correct by construction.**

A bug in Nord lands in one place. A new scheme lands in one place. A
new render target lands in one place. The five hardcoded copies of
`#2e3440` we currently carry across the fleet become a single typed
binding; the sRGB-vs-linear confusion that paints the mado window
washed-out becomes a compile error.

## What's broken today (the symptom)

Five proximate observations, one structural cause:

1. **mado** hardcodes its own Nord palette in `src/theme.rs` + parses
   hex strings into raw `(f64, f64, f64)` tuples in `main.rs` → feeds
   those values directly into `wgpu::Color` clear ops on a
   `Bgra8UnormSrgb` surface. The GPU interprets the values as linear,
   gamma-corrects on store → output sRGB ≈ 0.45 where 0.18 was wanted.
   The "washed-out gray instead of Nord dark" the operator sees in
   `mado` post-rebuild is this bug.
2. **blackmatter-mado** carries its own `themes/nord/colors.nix` —
   a third copy of the same palette.
3. **Most blackmatter-* GUI app HM modules** don't consume
   `config.lib.stylix.colors.*`; only `blackmatter-ayatsuri` does
   (correctly). Every other GUI repo would carry its own copy as it
   matures.
4. **stylix is wired** into the nix repo (`darwin-developer` profile
   pins `base16Scheme = nord.yaml` from the `base16-schemes` package)
   but **the canonical Nord values live somewhere different from
   ishou's `NORD` from irodori** — so the fleet has at least two
   slightly-different Nord sources.
5. **ishou's color tokens are untyped** — `Rgb { r, g, b: u8 }` carries
   no marker for sRGB vs linear. Every consumer is on the honor system
   to convert, and `mado` proves what that earns us.

The structural cause is the same in all five: **theme values are
untyped strings or untyped numeric tuples that flow through nine
different code paths, each of which is on the honor system for gamma,
palette source, and role-to-color mapping**. The compounding directive
is explicit about this: a problem fixed locally re-emerges in every
sibling consumer.

## The destination

```
┌─────────────────────────────────────────────────────────────────────┐
│  ishou-tokens — the typed source of truth                           │
│                                                                     │
│   Scheme :: NordDark | NordLight | Dracula | …                      │
│      ├─ palette : ColorPalette                                      │
│      │    └─ entries : Srgb (u8) ────────┐                          │
│      ├─ roles   : SemanticRoles          │                          │
│      └─ ansi    : Ansi16Palette          │                          │
│                                          ▼                          │
│   Srgb ───────[ srgb_to_linear ]────► Linear (f32, gamma-correct)   │
│      │                                   │                          │
│      │                                   ▼                          │
│      │                                Linear → wgpu::Color          │
│      │                                Linear → glsl `vec3`          │
│      └────────────────► Srgb → "#2e3440"  (hex string for css/etc.) │
│                                                                     │
│   ★ Type-level invariant: the only constructor for a wgpu::Color    │
│     consumed by a pleme-io render path is `From<Linear>`. You can   │
│     not write an Srgb to a wgpu surface — the type system refuses.  │
└─────────────────────────────────────────────────────────────────────┘
                              │  ishou-render targets
            ┌─────────┬───────┼───────┬─────────┬─────────┬────────┐
            ▼         ▼       ▼       ▼         ▼         ▼        ▼
          rust    stylix-  ghostty  tmux       glsl      tui      css/
        (consts)  base16  (config) (.conf)   (#define) (ratatui) tailwind
            │       │       │       │         │         │        scss/svg
            │       │       │       │         │         │
            ▼       ▼       ▼       ▼         ▼         ▼
       mado /   nix's     stock    tear's     mado's    tear,
       hibiki / stylix    Ghostty  rendered   shaders   tobirato,
       kagi  /  base16Scheme       tmux.conf            ayatsuri
       kekkai           (when mado isn't installed)
       /…
```

Three nested layers:

1. **Source** — `ishou-tokens`. Every theme value lives once, typed,
   with gamma-correct conversion operations the type system can
   verify. There is one Nord; it is the ishou Nord; every other
   "Nord" file in the fleet is a derived artifact.

2. **Translation** — `ishou-render`. The renderer crate emits whatever
   shape each consumer needs: typed Rust consts, a stylix-compatible
   base16 YAML, a Ghostty config block, glsl `#define`s, ratatui
   `Color` tables, css custom properties. Adding a new shape (e.g. an
   `alacritty.toml` block, or a new typed binding shape for hibiki) is
   one render-target module — every theme already in ishou flows
   through it automatically.

3. **Distribution** — two channels, per consumer kind:
   - **pleme-io Rust apps** (mado, hibiki, kagi, kekkai, fumi,
     namimado, tobirato, ayatsuri, hikyaku, future …) consume
     `ishou-tokens` as a Cargo dependency. They get **typed colors at
     compile time**, **gamma-correct constructors at runtime**, and
     **zero ability to round-trip through a stringly-typed config**.
   - **Foreign apps** (GTK, GNOME, alacritty, kitty, btop, K9s, …)
     consume stylix's system-wide base16 distribution. `ishou`'s
     `stylix-base16` render target *is the source* the fleet's
     `stylix.base16Scheme` reads — so every foreign app the operator
     ever runs aligns with whatever Nord-or-other-scheme ishou
     declares.

## Type-level invariants (the load-bearing pieces)

The architecture's promises are only as strong as the type signatures
that enforce them. The non-negotiables:

```rust
//─ ishou_tokens::color (proposed) ────────────────────────────────────

/// Color in sRGB space. The colour you see in a hex picker, the
/// colour stylix ships in base16 YAML, the colour a designer writes
/// into a config file. Cannot be written to a GPU surface without
/// going through `to_linear`.
#[derive(Debug, Clone, Copy, Serialize, Deserialize, PartialEq, Eq)]
pub struct Srgb { pub r: u8, pub g: u8, pub b: u8 }

/// Gamma-corrected colour in linear space, ready for any GPU pipeline
/// that writes to an sRGB-storage surface (the default everywhere in
/// the pleme-io GPU stack). Only constructor that costs nothing is
/// `Srgb::to_linear()`; there is no `Linear::from_hex` or
/// `Linear::from_floats` — by design, so an operator who wrote a hex
/// string can never accidentally pass it as linear.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct Linear { pub r: f32, pub g: f32, pub b: f32 }

impl Srgb {
    pub const fn to_linear(self) -> Linear { /* gamma 2.4 per IEC 61966-2-1 */ }
    pub fn hex(&self) -> String { /* "#RRGGBB" */ }
}

impl Linear {
    pub fn with_alpha(self, a: f32) -> LinearRgba { /* … */ }
}

#[cfg(feature = "wgpu")]
impl From<LinearRgba> for wgpu::Color {
    fn from(c: LinearRgba) -> Self { wgpu::Color { r: c.r as f64, g: c.g as f64, b: c.b as f64, a: c.a as f64 } }
}

// ★ Critically: no `From<Srgb> for wgpu::Color`. By absence.

/// Optional: OkLab for perceptual operations (contrast checks,
/// luminance-preserving lightening / darkening). Round-trips
/// `Srgb → OkLab → Srgb` cleanly.
#[derive(Debug, Clone, Copy)]
pub struct OkLab { pub l: f32, pub a: f32, pub b: f32 }
```

The shape is straightforward, but the *absence* of `From<Srgb> for
wgpu::Color` is what fixes the bug the operator just saw. mado's
`main.rs` calls `wgpu::Color { r: hex_r, … }` directly today and the
GPU treats those numbers as linear → washed out. After the migration,
the corresponding call site is `wgpu::Color::from(linear_rgba)` where
`linear_rgba` is produced via `Srgb::to_linear`. There is no other
constructor; the bug becomes uncompilable.

A second invariant — **the ANSI 16-color palette is derived
deterministically from the scheme** (base16 → ansi standard mapping)
and exposed as a typed `Ansi16Palette` whose fields are role-named, so
mado's terminal can't accidentally swap `ansi_red` and `ansi_yellow`
when applying a theme.

A third invariant — **consumers declare their theme requirements as
trait associated types**. The compiler refuses to compile an app
against a scheme that doesn't supply every role the app reads.

## Stylix's role, precisely

stylix is not the source. stylix is one of ishou's render targets, and
it is the system-wide distribution mechanism for *foreign* apps.

```
ishou::Scheme::NordDark
        │
        │ (rendered at flake-build time)
        ▼
  /nix/store/…-ishou-stylix-nord-dark.yaml
        │
        │ (consumed at HM-build time)
        ▼
  stylix.base16Scheme  ←  one knob
        │
        ▼
  every foreign app stylix knows about
  (GTK, GNOME, alacritty, kitty, btop, k9s, vim, …)
```

The fleet's `darwin-developer` profile gets one substitution:

```nix
# before
stylix.base16Scheme = "${pkgs.base16-schemes}/share/themes/nord.yaml";

# after
stylix.base16Scheme = inputs.ishou.packages.${system}.stylix-base16-nord-dark;
```

Now `pkgs.base16-schemes`'s Nord and ishou's Nord can't drift — the
former is gone from the fleet's source chain. Switching scheme: edit
one line in `ishou-tokens`, all of stylix's foreign-app derivations
re-derive on next rebuild.

The HM modules for pleme-io's *own* Rust apps don't need to read from
stylix at all (they consume ishou directly via Cargo), but they
**also** honour `config.lib.stylix.colors.baseXX` as a default
override so an operator can run a non-ishou stylix scheme system-wide
(say, light mode for a screen-share) and our apps follow.

## CSE invariants the architecture earns

Per [`THEORY.md` §V](./THEORY.md) — "verification by construction" —
these become *theorems* (compile-time provable) rather than *claims*
(written down somewhere and hoped for):

| Promise | Type-level enforcement |
|---|---|
| "mado's window background is sRGB-correct" | `wgpu::Color` constructible only from `LinearRgba` |
| "mado's Nord is the same Nord as the fleet" | `mado::theme` is `ishou::scheme::nord_dark` re-exported; deleting the re-export breaks `cargo build` |
| "Every pleme-io GUI consumes ishou" | `cse-lint` invariant `gui-app-consumes-ishou` |
| "Switching the fleet scheme is one change" | The set of files referencing concrete Nord values reduces to `ishou-tokens/src/color.rs` |
| "A scheme always supplies every role" | Trait-bound on `Scheme<R: Roles>` |
| "ANSI palette mapping is canonical" | `Ansi16Palette::from_base16(scheme)` is the only constructor |

## Render-target taxonomy

`ishou-render` already covers nine targets. The destination state adds
**stylix-base16** (the missing one that closes the loop) and refines
**rust** to ship typed `Linear` / `Srgb` rather than raw `[u8; 3]`.

| Target | Today | Destination |
|---|---|---|
| `rust` | `pub const RGB_*` arrays | `pub const SRGB_*: Srgb` + `pub const LINEAR_*: Linear` |
| `stylix-base16` | *(missing)* | Renders to base16 YAML stylix can ingest |
| `ghostty` | full config block | unchanged (already typed) |
| `glsl` | `#define` floats | switch to `vec3` of linear values + a `linear_to_srgb` GLSL fn |
| `tui` | ratatui `Color::Rgb` | unchanged (terminals are sRGB-native) |
| `css` / `tailwind` / `scss` | hex strings | unchanged |
| `json` | W3C tokens | unchanged |
| `svg` | brand mark | unchanged |

The interesting new target is **`stylix-base16`**: 16 hex strings in
YAML, formatted to stylix's contract. About 100 LOC; one structural
type for `Base16Scheme`; round-trip tested against
`pkgs.base16-schemes/nord.yaml` so the migration is byte-identical
unless we choose to change values.

## Phases

1. **M0 — Typed colour spaces in ishou-tokens.** `Srgb`,
   `Linear`, `LinearRgba`, `OkLab`. Gamma conversion. `wgpu`-gated
   feature for `From<LinearRgba> for wgpu::Color`. Const-correct
   where the math allows. Comprehensive tests against canonical
   sRGB test vectors. **No consumer migration yet.**
2. **M1 — Stylix-base16 render target.** `ishou-render` gains
   `stylix.rs` that emits a base16 YAML compatible with
   `nix-community/stylix`. Cross-pinned against
   `pkgs.base16-schemes/nord.yaml` so we know our Nord matches
   theirs byte-for-byte.
3. **M2 — Fleet stylix swap.** `darwin-developer` profile's
   `base16Scheme` line moves from `pkgs.base16-schemes/nord.yaml`
   to `inputs.ishou.packages.${system}.stylix-base16-nord-dark`.
   Foreign apps still get Nord; the source is now ishou.
4. **M3 — mado migration.** `mado` depends on `ishou-tokens` with
   the `wgpu` feature. `parse_hex_color` and the hardcoded Nord
   table in `theme.rs` are deleted. Render code consumes typed
   `Linear` values. The gamma bug becomes uncompilable. The
   fleet's most-visible GUI app is the prove-out.
5. **M4 — Sibling GUI migration.** ayatsuri, hibikine, kagibako,
   kekkai, fumi, namimado, tobirato, hikyaku. Each PR is 5–50 LOC
   (mostly imports + replacing local palette references).
6. **M5 — `cse-lint` invariants.** Add `gui-app-consumes-ishou`
   (any caixa marked `:kind GuiApp` MUST depend on `ishou-tokens`).
   Add `no-foreign-nord-source` (the only `nord.yaml` in the
   pleme-io source tree is ishou's renderer test fixture). Make
   the directive structurally unforgeable.
7. **M6 — Document + propagate.** Update repo-forge's gui-app
   archetype scaffolding (typed-groups gain a `theme` group whose
   `nullOr str` fields auto-source from
   `config.lib.stylix.colors.baseXX`). Update CLAUDE.md's typescape
   anchor section. Future GUI apps inherit the pattern.

## What the operator does after each phase

| Phase | Operator action | Visible effect |
|---|---|---|
| M0 | none — ishou-tokens has new types but no consumer changes | nothing visible; foundation laid |
| M1 | none | nothing visible; stylix renderer exists |
| M2 | `nix run .#darwin-rebuild` | foreign apps still Nord, sourced from ishou |
| M3 | `nix run .#darwin-rebuild` | mado's window renders crisp Nord, gamma-correct, full-window |
| M4 | rebuild per app | every GUI surface aligned |
| M5 | no action | future GUI apps cannot ship un-themed |
| M6 | no action | future GUI archetype scaffolds with the right shape |

## Anti-patterns this kills

- Local Nord palette files in repos other than ishou. ✕
- `parse_hex_color` style stringly-typed colour parsing in render code. ✕
- `wgpu::Color { r: hex_r as f64, … }` — direct sRGB → wgpu construction. ✕
- App-specific `themes/` directories shipping their own palette JSON / YAML. ✕
- Hardcoded ANSI palettes in terminal renderers. ✕
- `pkgs.base16-schemes/<scheme>.yaml` referenced anywhere downstream
  of ishou. ✕

## Why this is the destination, not a stop along the way

A theme system is the kind of thing every project's lead engineer
"plans to clean up someday." pleme-io has the unusual property that
**the type system can structurally prevent the bugs**, **the renderer
graph can structurally prevent the drift**, and **`cse-lint` can
structurally prevent the regression**. None of these are afterthoughts
glued onto an existing design — they're properties of the design
described above. After M6 there is no realistic refactor that earns
back what M3–M5 ship: every imaginable scheme change, every imaginable
sibling-app addition, every imaginable foreign-app stylix bridge lands
in the same place that ishou-tokens already names. The work has a
clear end state.

## Pointers

- `ishou` repo: <https://github.com/pleme-io/ishou>
- Existing token surface: `crates/ishou-tokens/src/{color,typography,spacing,radius,shadow,motion,shader,brand}.rs`
- Existing render targets: `crates/ishou-render/src/{css,tailwind,scss,rust,json,glsl,ghostty,tui,svg}.rs`
- stylix wiring point: `pleme-io/nix/profiles/darwin-developer/default.nix:182`
- The bug this displaces: `pleme-io/mado/src/main.rs:230` (`parse_hex_color` → `wgpu::Color` direct)
- The reference for what "right" looks like already in the fleet: `blackmatter-ayatsuri/module/default.nix`
- Related theory: [`THEORY.md` §III](./THEORY.md) (the typescape), [§V](./THEORY.md) (provability), [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
