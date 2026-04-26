# Lisp controllers in WASM with YAML on top

> **Frame.** [`WASM-STACK.md`](WASM-STACK.md) packages WASM/WASI as the
> K8s runtime; [`WASM-PATTERNS.md`](WASM-PATTERNS.md) catalogues 49
> recurring shapes. This document names *the* canonical authoring
> pattern: **the controller's logic lives in tatara-lisp (compiled to
> WASM); the operator's policy lives in YAML on top.** Lisp's
> homoiconicity makes this trivial — YAML is just an S-expression
> dialect with different syntax — and the result is the same shape
> as Kyverno + Rego, except both halves stay inside the pleme-io
> typed-domain world.

---

## I. The two layers

```
┌────────────────────────────────────────────────────────────────┐
│ AUTHOR LAYER (tatara-lisp → WASM module)                        │
│                                                                  │
│   defines the typed CONTRACT a YAML config must satisfy         │
│   defines the IMPERATIVE fallback (custom transforms,           │
│     external HTTP calls, whatever can't go in YAML)             │
│   one-time effort by a developer                                 │
│   versioned + content-addressed (BLAKE3 → OCI registry)          │
└────────────────────────────────────────────────────────────────┘
                             ▼ instantiated by ▼
┌────────────────────────────────────────────────────────────────┐
│ POLICY LAYER (YAML — usually a CRD spec field)                  │
│                                                                  │
│   describes WHAT to reconcile (selectors, targets, thresholds)  │
│   describes WHICH transforms apply, in what order                │
│   describes WHEN to dispatch the imperative escape hatch         │
│   ongoing operator surface — operators edit this, not Lisp       │
└────────────────────────────────────────────────────────────────┘
```

The pattern works because:

1. Lisp reads YAML natively — `(parse-yaml input)` returns a Sexp
   tree the controller pattern-matches on.
2. Tatara-domains give the YAML schema a typed validation layer for
   free — `#[derive(TataraDomain)]` over Rust structs becomes both
   the JSON schema for K8s admission AND the deserializer the Lisp
   controller uses internally.
3. Macros turn YAML rule blocks into Lisp pattern-matches at
   *expansion time*, so the runtime cost is the same as if the rule
   were hand-written in Lisp.

## II. Concrete: a DNS reconciler

Pattern #2 from the cookbook — sync `Service` and `Ingress` resources
with annotation `dns.pleme.io/host: <fqdn>` to a Cloudflare zone.

### II.1 The author layer (tatara-lisp → WASM)

```lisp
;; pleme-io/programs/dns-reconciler/main.tatara

(defcontroller dns-reconciler
  :version "v0.1.0"

  ;; Typed config schema — Tatara-domain auto-generates the CRD spec.
  :config-schema
    (defschema DnsReconcilerConfig
      :provider             (enum :cloudflare :route53 :hetzner :gcp)
      :zone-id              :string
      :credential-secret    :secret-ref
      :rules                (list-of DnsRule))

  :rule-schema
    (defschema DnsRule
      :match                (selector :services :ingresses)
      :record-type          (enum :A :AAAA :CNAME)
      :ttl                  (default :integer 300)
      :proxied              (default :bool false)
      :source               (xpath ".metadata.annotations[\"dns.pleme.io/host\"]")
      :transform            (default :sexp nil))    ; optional Lisp escape hatch

  ;; Watch targets — the operator computes informers from these.
  :watches [(:v1 :services) (:networking.k8s.io/v1 :ingresses)]

  ;; Reconcile loop — runs per matched resource per rule.
  (defreconcile [resource rule]
    (let* [host  (extract resource (:source rule))
           value (resource-value resource (:record-type rule))
           value (if (:transform rule)
                   (eval-sexp (:transform rule)
                              :bindings { :resource resource, :host host, :value value })
                   value)]
      (when (and host value)
        (dns/upsert :provider     (:provider config)
                    :zone-id      (:zone-id config)
                    :credentials  (read-secret (:credential-secret config))
                    :record-type  (:record-type rule)
                    :name         host
                    :value        value
                    :ttl          (:ttl rule)
                    :proxied      (:proxied rule))))))
```

Roughly 30 lines. Compiled to a 2 MiB WASM module + an auto-generated
`DnsReconciler` CRD with a typed schema.

### II.2 The policy layer (YAML in the cluster)

```yaml
apiVersion: dns.pleme.io/v1alpha1
kind: DnsReconciler
metadata:
  name: rio-cluster-dns
  namespace: dns-system
spec:
  provider: cloudflare
  zoneId:   abc123…
  credentialSecret:
    name: cloudflare-pangea
    key:  api-token

  rules:
    # Pure-declarative rule: every Service annotated dns.pleme.io/host gets
    # an A record pointing at the LB IP.
    - match:    { kind: Service, annotations: { dns.pleme.io/host: "*" } }
      recordType: A
      ttl: 60

    # Pure-declarative rule: every Ingress annotated similarly gets a
    # CNAME to the cluster's external endpoint.
    - match:    { kind: Ingress, annotations: { dns.pleme.io/host: "*" } }
      recordType: CNAME
      ttl: 300
      proxied:   true        # Cloudflare orange cloud

    # Mixed rule — the YAML sets the bulk; the transform: escape hatch
    # is a tatara-lisp expression for the 5% case (here, lower-case the
    # hostname before publishing — the imperative branch).
    - match:        { kind: Service, namespaces: [public-api] }
      recordType:   A
      ttl:          60
      transform: |
        (let [h (:host bindings)]
          (str/lowercase (str/strip-prefix h "internal-")))
```

The operator authors only YAML for the common 95% — selectors, types,
TTLs, proxied flags. The 5% that needs custom logic uses a single
`transform:` escape hatch with a Lisp expression evaluated against
typed bindings (`resource`, `host`, `value`).

### II.3 The wired ComputeUnit

```yaml
apiVersion: wasm.pleme.io/v1alpha1
kind: ComputeUnit
metadata:
  name: dns-reconciler
  namespace: dns-system
spec:
  module:
    source: oci://ghcr.io/pleme-io/programs:dns-reconciler-v0.1.0
  trigger:
    watch:
      crd: dns.pleme.io/DnsReconciler   # the controller's own CRD
      and:
        - { group: "v1",                kind: "Service" }
        - { group: "networking.k8s.io", kind: "Ingress" }
  capabilities:
    - kube-cr-watch@dns.pleme.io/dnsreconcilers
    - kube-cr-watch@v1/services
    - kube-cr-watch@networking.k8s.io/ingresses
    - kube-secret-read@dns-system/cloudflare-pangea
    - http-out:api.cloudflare.com
```

When the wasm-operator instantiates this, it:

1. Sets up watches on `DnsReconciler` CR + the typed targets.
2. On any change, dispatches to the WASM module's `reconcile` hook.
3. The module reads the cluster's `DnsReconciler` CRs, runs the
   `defreconcile` body for each rule × resource pair.
4. Logs / events / failures bubble back to the CR's `.status`.

## III. Why this dominates pure-imperative controllers

Three properties no plain Rust+kube-rs operator gets:

1. **Day-2 velocity.** Adding a new rule is a YAML edit by an
   operator. No WASM rebuild, no image push, no controller restart.

2. **Provable correctness for the declarative half.** The
   YAML rule schema is generated from typed Lisp domains (themselves
   `#[derive(TataraDomain)]` Rust structs). Admission validation
   rejects bad rules before they reach the controller. The
   imperative half (the `transform:` Lisp expression) is evaluated
   in a sandboxed Lisp interpreter with only the bindings the
   schema declares — no arbitrary I/O.

3. **Separable evolution.** The author publishes
   `dns-reconciler-v0.2.0` with new rule fields; the WASM-operator
   does a hot-replacement (Pattern #48) — old YAML keeps working,
   new YAML keeps working, no operator-side disruption.

## IV. Why this dominates Kyverno / Rego / CEL

Same shape (YAML rules + escape hatch), but everything is in the
pleme-io typed domain:

| Aspect | Kyverno (Rego) | Lisp + YAML on WASM |
|---|---|---|
| Schema validation | manual JSON schema | auto-generated from `defschema` |
| Escape hatch | Rego (different language) | tatara-lisp (same language) |
| Custom domains | foreign function interface | `#[derive(TataraDomain)]` for free |
| Composition | per-policy independent | macros + `defreconcile` reusable |
| Hot-replacement | Kyverno-internal | wasm-operator standard (Pattern #48) |
| Resource cost | always-on Kyverno admission controller | scale-to-zero between events |

Kyverno is great for K8s admission policy. For *anything else*
(reconciliation, integration, orchestration), Lisp + YAML on WASM is
the canonical pleme-io shape.

## V. The four authoring tiers

Different tiers of this pattern, ordered by Lisp-vs-YAML mix:

### V.1 100% YAML

Trivial reconcilers. The `defreconcile` body is fixed; rules are
purely match-and-do. **Most of the 49 cookbook patterns fall here.**
Example: backup scheduler — match by label, take a snapshot, push
to S3. No imperative logic; the YAML rule literally says
"label=backup, schedule=daily, target=s3://bucket".

### V.2 YAML + tiny escape hatches

90% YAML, 10% Lisp expressions inside `transform:` / `predicate:`
fields. The DNS reconciler above is this tier. Operators rarely
touch the Lisp; when they do, it's a one-liner.

### V.3 YAML config + Lisp logic

50/50. The CRD's spec is the configuration surface (datasources,
thresholds, schedule), but the reconciliation body itself is in Lisp.
Custom alert evaluators, RCA assistants, agent-as-controller fall
here. Operators can read the Lisp but rarely edit it.

### V.4 100% Lisp (no CRD)

Pure Lisp module, `ComputeUnit` is the entire interface. Used for
one-off programs (the `oneShot` shape) or small controllers that
don't need an operator-facing config surface. Trade-off: any change
needs a republish.

The pattern's expressiveness comes from being able to slide between
tiers as a project matures. Start at V.4 to prototype; lift the
config surface to V.3 once the shape stabilizes; collapse to V.2 or
V.1 as the rules become uniform across operators.

## VI. Macros that compose

The shapes above repeat. Library tatara-lisp macros lift them once:

```lisp
;; library: pleme-io/tatara-lisp-controllers/lib.tatara

(defmacro defrule-driven-controller
  [name & {:keys [config-schema rule-schema watches reconcile]}]
  ;; Auto-generates:
  ;;   - the CRD spec from config-schema + rule-schema
  ;;   - the K8s informer setup from watches
  ;;   - the per-rule × resource dispatch loop
  ;;   - error / status propagation back to the CR
  `(defcontroller ~name
     :config-schema ~config-schema
     :rule-schema   ~rule-schema
     :watches       ~watches
     (defreconcile [resource rule]
       (with-error-tracking [resource rule]
         ~reconcile))))

;; Then the DNS reconciler from §II.1 collapses to:
(defrule-driven-controller dns-reconciler
  :config-schema  DnsReconcilerConfig
  :rule-schema    DnsRule
  :watches        [(:v1 :services) (:networking.k8s.io/v1 :ingresses)]
  :reconcile      (dns/upsert ...))
```

Every controller in the cookbook that fits the rule-driven shape
becomes ~10 lines. The macro lives once; the patterns multiply.

## VII. CRD generation

Each `defcontroller` emits:

1. **A typed CRD** for the controller's config (from `:config-schema`)
   + the rule list (from `:rule-schema`).
2. **An OpenAPI v3 schema** Helm renders into the chart's `crds/` dir.
3. **A `kubectl explain`-friendly description** sourced from the Lisp
   docstrings — `(defschema DnsReconcilerConfig
   :provider "Which DNS provider to push records to"  ...)` becomes the
   field's `description:` in the OpenAPI schema.

So a new controller pattern *automatically*:

- Has a CRD ready to deploy.
- Has admission validation.
- Has `kubectl explain` documentation.
- Has YAML-authored policy on top.
- Has a Lisp escape hatch where needed.
- Compiles to a 2 MiB WASM module the operator runs scale-to-zero.

## VIII. The library backlog

These macros / typed domains land first to make Lisp+YAML controllers
ergonomic across the cookbook:

| Macro / domain | Provides | Used by patterns |
|---|---|---|
| `defrule-driven-controller` | rule-list reconciler shape | 2, 7, 27, 39, 40 |
| `defwebhook-handler` | HTTP signature + dispatch | 9, 28, 44 |
| `defcron-job` | schedule + retention | 1, 3, 4, 19, 20, 21, 41 |
| `defetl-pipeline` | source → transform → sink | 19, 20, 21, 22 |
| `defslo-evaluator` | burn-rate + alerting | 24 |
| `defrole-binding-controller` | typed RBAC | 39, 40 |
| `defproxy-service` | request routing + transform | 42, 43, 44, 45 |
| `defsaga-controller` | step list + compensation | 31, 32, 33, 34 |

Eight macros cover ~35 of the 49 cookbook patterns. Each macro
becomes a short tatara-lisp file in `pleme-io/tatara-lisp-controllers/`.

## IX. Phasing

- **Phase A (now):** this design doc; the cookbook
  ([`WASM-PATTERNS.md`](WASM-PATTERNS.md)) referencing this pattern.
- **Phase B:** image flakes for `wasm-platform` + `tatara-lisp-script`
  + `openclaw` so the runtime can deploy.
- **Phase C:** `pleme-io/tatara-lisp-controllers` repo with the 8
  library macros + 1 worked example (DNS reconciler).
- **Phase D:** convert the first 5 cookbook patterns end-to-end:
  PVC autoresizer (1), DNS reconciler (2), Slack notifier (10),
  custom alert evaluator (23), webhook signature verifier (44).
- **Phase E:** publish the typed CRDs from those controllers as a
  shared schema package the rest of the fleet reuses.

## X. See also

- [`WASM-STACK.md`](WASM-STACK.md) — the runtime
- [`WASM-PATTERNS.md`](WASM-PATTERNS.md) — 49 patterns
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`THEORY.md` Pillar 1](THEORY.md#pillar-1-language) — language constraint
- [tatara-lisp](https://github.com/pleme-io/tatara-lisp) — `#[derive(TataraDomain)]`
- The Kyverno comparison stems from the same insight: declarative
  rules + tiny escape hatches scale better than pure imperative code.
  The difference is that pleme-io keeps both halves in the same
  typed-domain ecosystem, so escape hatches can call any Rust crate
  the fleet has bound to a `TataraDomain`.
