# Constructive Crossplane Providers

> Companion to [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
> and [`CONSTRUCTIVE-ACTIONS.md`](./CONSTRUCTIVE-ACTIONS.md).
> Operationalises **Pillar 12** (generation over composition) for the
> Crossplane-provider class of artifact: every per-vendor Crossplane
> provider pleme-io ships is **regenerated from typed substrate
> primitives**, not hand-written, not externally tool-generated.

## Thesis

A Crossplane provider for vendor `V` is a *function* of three typed
inputs:

1. The vendor's API surface, expressed as a TOML corpus
   (`pleme-io/V-terraform-resources/resources/*.toml`) — one file per
   managed resource, declaring CRUD endpoints + identity + attributes.
2. The vendor's OpenAPI 3.0 spec (`pleme-io/V-go/api/openapi.yaml` or
   upstream) — the authoritative SDK type surface.
3. A `provider.toml` declaring auth + provider metadata.

That function is `iac-forge generate --backend crossplane`. Output:
the entire provider tree (`apis/`, `internal/controller/`,
`cmd/provider/`, `helm/`, `package/crds/`, `go.mod`) — every artifact
typed-AST-emitted, never `format!()`-string-emitted.

The provider is therefore a **proof object**: given the same three
inputs, two operators get byte-equal output. Promises about behaviour
collapse to promises about the substrate that emits it.

## The substrate stack

```
                      ┌─────────────────────────────────────────┐
                      │  pleme-io/crossplane-V                   │ ← generated
                      │  ──────────────────────                  │
                      │  apis/<r>/v1alpha1/{types,gvi,deepcopy,  │
                      │                     managed}.go          │
                      │  apis/V/v1alpha1/{providerconfig,...}.go │
                      │  apis/apis.go (AddToScheme)              │
                      │  internal/controller/<r>/controller.go   │
                      │  internal/controller/setup.go            │
                      │  cmd/provider/main.go                    │
                      │  go.mod, helm/, package/crds/            │
                      └────────────────────▲────────────────────┘
                                           │
                                  iac-forge generate --backend crossplane
                                           │
                      ┌────────────────────┴────────────────────┐
                      │  pleme-io/crossplane-forge               │
                      │  ──────────────────────                  │
                      │  types_gen.rs           ← managed-resource Go types
                      │  controller_gen.rs      ← ExternalClient + Setup + Connect
                      │  provider_gen.rs        ← ProviderConfig + main.go + Helm
                      │  deepcopy_gen.rs        ← DeepCopyObject/DeepCopy/DeepCopyInto
                      │  managed_methods_gen.rs ← 12 resource.Managed accessors
                      │  backend.rs             ← composes them all per Backend trait
                      └────────────────────▲────────────────────┘
                                           │
                                  builds typed GoFile values via:
                                           │
                      ┌────────────────────┴────────────────────┐
                      │  pleme-io/iac-forge::goast                │
                      │  ──────────────────────                  │
                      │  GoFile / GoDecl / GoType / GoStmt / GoExpr
                      │  KubeMarker (typed kubebuilder annotations)
                      │  GoStructTag::{Json,Yaml}                │
                      │  GoPrinter — gofmt-stable rendering      │
                      └─────────────────────────────────────────┘
```

## The five emitters and what each typed primitive proves

| Emitter | Output | Typed invariant |
|---|---|---|
| `types_gen` | Per-resource `<Kind>Parameters/Observation/Spec/Status/Kind/List` | Spec embeds `xpv1.ResourceSpec`; `Status` embeds `xpv1.ResourceStatus`; required→`KubeMarker::Required` on the field; immutable→`KubeMarker::XValidationCEL{rule:"self == oldSelf"}`. |
| `controller_gen` | Per-resource `<resource>/controller.go` (Setup + Connect + Observe + Create + Update + Delete + Disconnect) | Each method's signature exactly matches `crossplane-runtime/pkg/reconciler/managed.ExternalClient`; SDK chain is `e.client.V2API.<method>(ctx).<body>(body).Execute()` derived from `iac_forge::sdk_naming`. |
| `provider_gen` | `apis/<provider>/v1alpha1/providerconfig*.go`, `cmd/provider/main.go`, `internal/controller/setup.go`, `go.mod`, `helm/*` | ProviderConfig embeds `xpv1.CommonCredentialSelectors`; `main.go` registers everything via `apis.AddToScheme`; Helm chart serialises from typed serde structs. |
| `deepcopy_gen` | Per-package `zz_generated_deepcopy.go` | Every Kind + KindList implements `runtime.Object` via DeepCopyObject + DeepCopy + DeepCopyInto. |
| `managed_methods_gen` | Per-Kind `zz_generated_managed.go` | Every Kind implements `crossplane-runtime/pkg/resource.Managed` via 12 accessor methods delegating to embedded ResourceSpec/ResourceStatus. |

## ResourceShape — per-resource heterogeneity, structurally typed

Vendor SDK body types are heterogeneous (some `Name string`, some
`Name *string`, some `EsmName`, some composite-keyed). The emitter
expresses this via [`controller_gen::ResourceShape`](https://github.com/pleme-io/crossplane-forge/blob/main/src/controller_gen.rs):

```rust
pub struct ResourceShape {
    pub identifier_field: String,    // "Name" / "EsmName" / "UscName" / ...
    pub identifier_pointer: bool,    // Name *string vs Name string
    pub stub: bool,                  // composite-key / mixed-per-CRUD: emit no-op
}
```

Helpers: `::default()`, `::name_pointer()`, `::alt_field("...")`,
`::stub()`. The akeyless override map is hardcoded today; future
iteration replaces it with OpenAPI introspection at generation time.

When `shape.stub` is true the emitter produces a controller that
satisfies `ExternalClient` but no-ops every method — compile-correct
+ marked for M3.2 graduation. The substrate ships 100% buildable
even for heterogeneous resources whose body-shape doesn't fit the
single-identifier pattern.

## Substrate-hygiene invariants

These invariants are **load-bearing** — every emitter and every test
in `crossplane-forge` enforces them:

1. **No `format!()` of Go syntax.** All Go output flows through
   `iac_forge::goast::GoFile` → `print_file`. `format!()` is allowed
   only for *data construction* (paths, identifier names, doc-comment
   text, string-literal contents).
2. **No `format!()` of YAML syntax.** Helm chart metadata flows
   through typed `serde`-derive structs serialised via
   `serde_yaml_ng`; templated YAML flows through
   `serde_yaml_ng::Value` trees with template directives as opaque
   string scalars.
3. **Tests assert on AST shape.** `matches!(field.ty, GoType::Pointer(_))`,
   `markers.iter().any(|m| matches!(m, KubeMarker::ObjectRoot))`. AST
   tests survive printer changes; substring tests don't.
4. **Round-trip determinism.** Two structurally identical
   `GoFile` values render to byte-equal output. Locked by tests.
5. **No external code-generation tools at build time.** No `controller-gen`,
   no `angryjet`, no `deepcopy-gen`. The substrate emits everything
   the upstream tools would.

## Bugs the AST refactor caught (that `format!()`-strings would have
hidden)

These two bugs surfaced as test failures the moment they were
introduced. Both would have rendered "successfully" as
`format!()`-string output and only broken at `go build` time on a
real cluster:

- **`make([]T{}, n)` instead of `make([]T, n)`** — slice-literal
  emission used `GoExpr::SliceLit` (renders `[]T{...}`) where a TYPE
  was needed. Fix: introduce `GoExpr::TypeExpr(GoType)` so types
  appear in expression position. Test
  `type_expr_renders_as_bare_type` locks the invariant.
- **`if !ok.` (stray `.`)** — the `Connect` function used
  `Selector { recv: Ident("!ok"), sel: "" }` instead of
  `Ident("!ok")`. Renders correctly only because the AST shape was
  wrong. Fix: emit `!ok` as a plain `GoExpr::Ident`.

The `format!()`-string approach renders both bugs as syntactically
plausible Go, defers detection until `go build`, and surfaces them
without an obvious connection back to the emission code. The AST
approach makes both *structurally impossible* once an explicit AST
node type exists.

## Compounding validation

Tested 2026-05-06: the same generator pipeline driven against three
different TOML corpora produced 12 cross-cutting scaffold artifacts
each, all with correct per-vendor naming/module-path/chart-name:

| Corpus | Resources | Cross-cutting scaffold artifacts | Per-resource artifacts |
|---|---|---|---|
| `akeyless-terraform-resources` | 119 | 12 | 595 (5 per resource × 119) |
| `datadog-terraform-resources` | 10 | 12 | 0 — needs upstream Datadog OpenAPI spec |
| `splunk-terraform-resources` | 4 | 12 | 0 — needs upstream Splunk OpenAPI spec |

Zero new emitter code between vendors. The substrate took two
unfamiliar TOML corpora and produced correct provider scaffolding for
both. **Per-resource controller emission is gated only on upstream
OpenAPI spec acquisition** — a per-vendor onboarding step, not a
substrate gap.

## Adding a new vendor (operator playbook)

```bash
# 1. Land the vendor's TOML resource corpus (manual authoring; one .toml per
#    managed resource declaring [crud], [identity], [fields]).
cd ~/code/github/pleme-io
gh repo create pleme-io/<V>-terraform-resources --public
cd <V>-terraform-resources
# author resources/*.toml + provider.toml

# 2. Land the vendor's OpenAPI spec snapshot (vendored — typically as
#    api/openapi.yaml; matches the akeyless-go pattern).
gh repo create pleme-io/<V>-go --public
# vendor the OpenAPI spec

# 3. Generate the Crossplane provider.
gh repo create pleme-io/crossplane-<V> --public
cd crossplane-<V>
make regenerate    # invokes iac-forge --backend crossplane

# 4. (If upstream Go SDK isn't already pleme-managed) generate the SDK too.
cd ../<V>-go
make regenerate    # invokes forge-gen --sdks go

# 5. Verify.
cd ../crossplane-<V>
go build ./...     # should be clean if the vendor's body shape is uniform
                   # ResourceShape overrides patch heterogeneity
```

Substrate work is bounded: one `ResourceShape` table entry per
heterogeneous resource. Everything else is upstream-vendor onboarding.

## Status (2026-05-06)

- **`pleme-io/crossplane-akeyless`** — 100% `go build ./...` clean,
  **119/119 controllers with full body-construction (zero stubs)**.
  The substrate's `ResourceShape` + `BodyTemplate` cover all six
  heterogeneity classes the akeyless body types exhibit (default
  string identifier, alt-field, per-method pointer/value, per-method
  field-name, singleton NoIdentifier, int64-from-external-name,
  composite SpecFields).
- **`pleme-io/crossplane-datadog`** — substrate scaffold validated;
  per-resource controllers blocked on Datadog OpenAPI spec acquisition.
- **`pleme-io/crossplane-splunk`** — substrate scaffold validated;
  per-resource controllers blocked on Splunk OpenAPI spec acquisition.
- **Test coverage:** 192 tests in `crossplane-forge`, 579 in
  `iac-forge`, every emission path covered with AST-shape assertions.

## Outstanding substrate work

- **OpenAPI-driven `ResourceShape` derivation** — replaces the
  hardcoded akeyless override map with body-shape introspection at
  generation time. Reads each `[crud].schema` body from the OpenAPI
  spec, derives the appropriate `BodyTemplate` automatically, and
  surfaces the derived shape in a typed report so operators can
  inspect what was inferred.
- **OpenAPI introspection at generation time** — extends `iac-forge`
  to read the SDK body schema and derive `ResourceShape` automatically
  per `(resource, CRUD-method)` tuple.
- **Per-method `ResourceShape`** — slice 2 of `ResourceShape` so a
  single resource can have `Create.Name string` and `Get.Name *string`
  without falling through to a stub.
- **Helm-chart enrichment** — NetworkPolicy, ServiceMonitor,
  PodDisruptionBudget, leader-election RBAC tightening; integrate with
  helmworks for fleet-shared chart conventions.

## Related substrate work

- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
  — the substrate-engineering thesis this lives inside.
- [`CONSTRUCTIVE-ACTIONS.md`](./CONSTRUCTIVE-ACTIONS.md)
  — same shape applied to GitHub Actions; `pleme-io/pleme-actions` is
  to GHA what `pleme-io/crossplane-akeyless` is to Crossplane.
- `pleme-io/iac-forge::goast` — the typed Go AST primitive every
  pleme-managed Go-emitting backend must build on (terraform-forge,
  ansible-forge, future rust-forge etc. graduate to it as their
  refactor lands).
