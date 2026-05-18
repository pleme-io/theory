# NixAST — the comprehensive typed Nix expression AST

> **Frame.** [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> says every typed structure renders to its target language *via a typed
> AST*, never via string concatenation. The Crossplane providers entry in
> [`pleme-io/CLAUDE.md`](../CLAUDE.md) ★★ section enforces this rule for
> Go/YAML emission ("`format!()` of Go/YAML syntax is forbidden"). **This
> doc is the same rule applied to Nix emission:** every renderer that
> currently emits Nix via `out.push_str("...")` or `format!("{...}")`
> migrates to building a `NixValue` AST and rendering through one
> canonical pretty-printer.
>
> The doc is normative. Every pleme-io tool that emits Nix code — flake.nix,
> home-manager modules, NixOS configurations, overlays, anything else —
> constructs a `NixValue` from typed source and renders through
> `repo_forge_render::nix_value::render`. String concatenation of Nix
> syntax is an antipattern; `skip-nix-ast:` in the deviating file's first
> line to opt out (only legitimate for plain-text non-Nix files).
>
> **Status.** Draft v1. Refactor target tracked as M0.57 in the current
> shigoto/tend arc. `NixValue` already exists in
> `repo-forge-ast/src/nix_value.rs` with a minimal 7-variant surface;
> this doc expands it to comprehensive.

---

## I. Why the existing surface is too small

`repo-forge-ast::nix_value::NixValue` today has 7 variants:

```rust
pub enum NixValue {
    Null, Bool(bool), Int(i64), Str(String), Path(String),
    List(Vec<NixValue>), AttrSet(Vec<(String, NixValue)>),
    Raw(String),
}
```

That covers values, not expressions. Real flake.nix files use:

| Construct | Example | Today's escape |
|---|---|---|
| Function literal | `inputs @ { self, nixpkgs, ... }: ...` | `NixValue::Raw` |
| Function application | `import "${substrate}/x.nix" { args }` | `Raw` |
| Attrset spread | `}) // { homeManagerModules.default = ... }` | `Raw` |
| `with` | `with pkgs; [ openssl ]` | `Raw` |
| `let-in` | `let lib = ...; in { packages.default = lib; }` | `Raw` |
| `inherit` | `inherit nixpkgs crate2nix;` | `Raw` |
| Interpolated string | `"${substrate}/lib/rust-tool.nix"` | `Raw` (loses the typed reference) |
| Indented multi-line string | `''description...''` | `Raw` |
| `if-then-else` | `if cond then a else b` | `Raw` |
| Operators | `==`, `!=`, `+`, `//`, etc. | `Raw` |
| Attr-path access | `pkgs.lib.eachDefaultSystem` | `Raw` (split into idents would help) |

Every renderer ends up either pushing strings directly (the antipattern this
doc forbids) or wrapping huge swaths of nix in `NixValue::Raw` (the
escape hatch becoming load-bearing).

The survey: `repo-forge-render/src/flake.rs` contains **127** `push_str` /
`format!` sites in 462 LOC; `hm_module.rs` has 49; `workflow.rs` has 46
(YAML, sibling concern); `readme.rs` has 30 (markdown, sibling concern).
Without a comprehensive Nix AST, every consumer chooses one of two
losing paths.

---

## II. The destination

### II.1. One comprehensive AST

```rust
pub enum NixValue {
    // ── Atoms ───────────────────────────────────────────────
    Null,
    Bool(bool),
    Int(i64),
    Float(f64),
    Str(String),
    /// Multi-line indented string (`''...''`). Each entry is one line
    /// without trailing newline. Renderer applies the canonical
    /// 2-space dedent.
    IndentedStr(Vec<String>),
    /// Path literal (no quotes): `./foo`, `/abs/path`, `<nixpkgs>`.
    Path(String),
    /// String with `${...}` interpolation: `"${pkgs.openssl}/bin/openssl"`.
    InterpolatedStr(Vec<StrPart>),

    // ── Identifiers + access ───────────────────────────────
    /// Bare identifier: `pkgs`, `nixpkgs`, `self`.
    Ident(String),
    /// Dotted path: `pkgs.lib.eachDefaultSystem`. Rendered with `.`.
    AttrPath(Vec<String>),

    // ── Collections ────────────────────────────────────────
    List(Vec<NixValue>),
    /// `recursive`: emits `rec { … }` when true.
    AttrSet {
        recursive: bool,
        entries: Vec<AttrSetEntry>,
    },

    // ── Functions ──────────────────────────────────────────
    /// Lambda literal: `params: body`.
    Lambda {
        params: LambdaParams,
        body: Box<NixValue>,
    },
    /// Function application: `func arg1 arg2 ...`. For curried calls
    /// (which is most Nix), pass one arg at a time per application
    /// level; renderer parenthesizes as needed.
    Apply {
        func: Box<NixValue>,
        args: Vec<NixValue>,
    },

    // ── Control / scoping ──────────────────────────────────
    /// `let bindings; in body`.
    Let {
        bindings: Vec<LetBinding>,
        body: Box<NixValue>,
    },
    /// `with scope; body`.
    With {
        scope: Box<NixValue>,
        body: Box<NixValue>,
    },
    /// `if cond then then_ else else_`.
    If {
        cond: Box<NixValue>,
        then_branch: Box<NixValue>,
        else_branch: Box<NixValue>,
    },

    // ── Operators ──────────────────────────────────────────
    /// Binary operator: `left op right`.
    BinOp {
        op: NixBinOp,
        left: Box<NixValue>,
        right: Box<NixValue>,
    },
    /// Unary operator: `op operand`.
    UnaryOp {
        op: NixUnaryOp,
        operand: Box<NixValue>,
    },
    /// `attrset.attr or default`.
    AttrOr {
        attrset: Box<NixValue>,
        attr: Vec<String>, // dotted path
        default: Box<NixValue>,
    },
    /// `attrset ? attr.path`.
    HasAttr {
        attrset: Box<NixValue>,
        attr: Vec<String>,
    },

    // ── Escape hatch ───────────────────────────────────────
    /// Verbatim Nix. Auditable. Use ONLY for syntax not yet representable
    /// — every Raw call site is a debt against this doc.
    Raw(String),
}

pub enum AttrSetEntry {
    /// `key.path = value;`
    KeyValue {
        key: AttrPath,
        value: NixValue,
    },
    /// `inherit name1 name2 ...;` or `inherit (from) name1 name2 ...;`
    Inherit {
        from: Option<NixValue>,
        names: Vec<String>,
    },
}

pub struct AttrPath(pub Vec<AttrKey>);

pub enum AttrKey {
    /// Plain identifier: `a` in `a.b`.
    Ident(String),
    /// String key: `"weird-key"` in `pkgs."weird-key"`.
    Str(String),
    /// Interpolated: `${expr}` in `a.${b}.c`.
    Interp(NixValue),
}

pub enum LambdaParams {
    /// `x: ...`
    Single(String),
    /// `{ a, b ? default, ... } @ args: ...`
    Destructured {
        fields: Vec<ParamField>,
        ellipsis: bool,
        /// `args @ { ... }` binding name (the @ form).
        binding: Option<String>,
    },
}

pub struct ParamField {
    pub name: String,
    pub default: Option<NixValue>,
}

pub enum LetBinding {
    /// `name = value;`
    Bind { name: String, value: NixValue },
    /// `inherit name1 name2;` or `inherit (from) name1 name2;`
    Inherit {
        from: Option<NixValue>,
        names: Vec<String>,
    },
}

pub enum StrPart {
    Literal(String),
    /// `${expr}` interpolation.
    Interp(NixValue),
}

pub enum NixBinOp {
    /// `//` — attrset update
    Update,
    /// `++` — list concat
    Concat,
    /// `+` `-` `*` `/`
    Add, Sub, Mul, Div,
    /// `==` `!=` `<` `>` `<=` `>=`
    Eq, Neq, Lt, Gt, Le, Ge,
    /// `&&` `||` `->`
    And, Or, Impl,
}

pub enum NixUnaryOp {
    /// `-x`
    Neg,
    /// `!x`
    Not,
}
```

That's the comprehensive set. Every Nix construct we've actually used
in pleme-io renderers maps to one of these variants. `Raw` remains for
the unknown — and every `Raw` site is now tracked debt against this doc.

### II.2. The canonical renderer

A single `render(value: &NixValue, indent: usize) -> String` function
lives in `repo-forge-render::nix_value`. Every consumer that needs to
emit Nix builds a `NixValue` and calls this one function.

The renderer is:
- **Deterministic** — same AST → byte-identical output.
- **Idempotent** — `parse(render(ast)) == ast` modulo formatting (when
  a parser exists; not v1).
- **Pretty** — nixpkgs-style: 2-space indent, attrset entries on their
  own lines, lists with all-atom items inline.
- **Operator-precedence-aware** — `BinOp` parenthesizes only when the
  parent operator's precedence would re-bind the child.
- **Test-asserted** — `NixValue → String → NixValue` (via parser if
  available) is golden-tested. Without a parser, byte-equality against
  hand-crafted expected output for each variant is the floor.

### II.3. Where consumers live

```
repo-forge-ast/src/nix_value.rs           — the NixValue enum itself
repo-forge-render/src/nix_value.rs        — the canonical renderer
repo-forge-render/src/flake.rs            — FlakeNixAst::to_nix_value()
repo-forge-render/src/hm_module.rs        — HmModuleAst::to_nix_value()
                                            (any future Nix emitter)
```

Each renderer module has the same shape:

```rust
pub fn render(ast: &SomeAst) -> String {
    let nix = ast.to_nix_value();
    crate::nix_value::render(&nix, 0)
}

// On the AST struct:
impl SomeAst {
    pub fn to_nix_value(&self) -> NixValue { … }
}
```

No `out.push_str`, no `format!("nix syntax {var}")`. The `to_nix_value`
method is the typed bridge.

---

## III. Migration plan (M0.57 sub-milestones)

Each sub-milestone is one commit landing on main, leaving the workspace
buildable + tests passing.

**M0.57a — Extend `NixValue`.** Add every variant from §II.1 to
`repo-forge-ast/src/nix_value.rs`. Add `repo-forge-render/src/nix_value.rs`
support for each new variant. Comprehensive unit tests asserting
expected rendering for every variant (~300 LOC of tests). No consumer
changes yet.

**M0.57b — `flake.rs` refactor.** Replace every `push_str` /
`format!`-of-nix-syntax in `repo-forge-render/src/flake.rs` with a
`FlakeNixAst::to_nix_value()` impl plus a one-line `render` that
delegates. The existing roundtrip property tests stay (deterministic +
idempotent); new tests assert specific AST shape for each
`FlakeOutputs` variant.

**M0.57c — `hm_module.rs` refactor.** Same pattern for HM module
emission. `HmModuleAst::to_nix_value()`.

**M0.57d — YAML emitters.** `repo-forge-render/src/workflow.rs` (GH
Actions) migrates to `serde_yaml::Value`-built AST. YAML is well-served
by `serde_yaml`; no need to invent. Same pattern: `WorkflowAst::to_yaml()`.

**M0.57e — Markdown emitters.** `repo-forge-render/src/readme.rs`
migrates to a typed `Markdown` AST. The Markdown AST is small (heading
/ paragraph / list / code-block / link / inline-code / emphasis); ships
in this commit. Same pattern: `ReadmeAst::to_markdown()`.

**M0.57f — Plain-text emitters (deferred).** `direnv.rs` / `license.rs`
/ `gitignore.rs` emit line-list files. These don't have a syntax tree;
their "AST" is `Vec<Line>` joined with `\n`. Marginal benefit; M0.57f
optional. Decision deferred until a real divergence appears.

After M0.57e lands, M0.56 (the SubstrateLibraryWorkspace variant) and
M0.6 (the shigoto scaffold) ride on the clean architecture.

---

## IV. Invariants the AST enforces

1. **No string concatenation of Nix syntax.** A consumer that needs an
   attrset spread builds `NixValue::BinOp { op: Update, … }`, not `format!("{} // {}", ...)`.

2. **Every operator precedence is explicit.** The renderer
   parenthesizes per Nix's precedence table; consumers don't think
   about parens.

3. **Interpolation is typed.** A reference to `${substrate}` is
   `StrPart::Interp(NixValue::Ident("substrate".into()))`, not a raw
   `"${substrate}"` string. The substrate reference is a typed
   identifier the AST tracks.

4. **Inherit is its own thing.** Both attrsets and let-bindings have
   `Inherit` variants because Nix lets them appear in both contexts;
   the AST mirrors that exactly.

5. **Function destructuring is fully typed.** `args @ { a, b ? d, ... }`
   has all three parts (binding name, fields with optional defaults,
   ellipsis) modeled in `LambdaParams::Destructured`.

6. **`Raw` use is debt.** Every `Raw` call site is a TODO against this
   doc — either the AST is missing a variant the consumer needs (add
   it), or the consumer is being lazy (refactor).

---

## V. Decision log

- **Comprehensive over minimal.** Per operator directive ("most
  comprehensive nix AST rendering possible"). Add every construct used
  in real pleme-io flakes; defer obscure constructs (`tryEval`,
  `builtins.toString`, etc.) to first-consumer-need.
- **One canonical renderer.** No per-consumer renderer. Every Nix
  output flows through `repo-forge-render::nix_value::render`.
- **`Raw` retained as escape hatch.** Banning it entirely would force
  premature AST expansion. Tracking it as debt is healthier.
- **Pretty-printing is opinionated.** nixpkgs-style. No knobs. Same
  AST always renders the same way.
- **Operator precedence baked into the renderer.** Consumers never
  think about parens.
- **YAML lifts to `serde_yaml::Value`, not a custom AST.** Existing
  crate; well-tested; we'd duplicate it for no gain.
- **Markdown gets a small custom AST.** No standard typed Markdown AST
  in our dep tree; `pulldown-cmark` is parse-only. We define our own
  (heading/paragraph/list/code/link/inline) — small surface, easy to
  test.
- **Plain-text emitters not migrated (M0.57f).** Marginal benefit. Line
  lists are already typed enough.

---

## VI. References

- [`SHIGOTO.md`](./SHIGOTO.md) — sibling typed-primitive doc, same
  shape.
- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md) §
  "renders to concrete systems mechanically" — the methodological
  source.
- `pleme-io/CLAUDE.md` ★★ Crossplane providers — the precedent for
  banning `format!()` of target syntax (applied to Go/YAML there; now
  to Nix here).
- `repo-forge-ast/src/nix_value.rs` — the current minimal `NixValue`.
- `repo-forge-render/src/nix_value.rs` — the current minimal renderer.
- `repo-forge-render/src/flake.rs` — the worst-offender consumer to
  refactor (M0.57b).

---

*v1. Open for review before M0.57a code lands.*
