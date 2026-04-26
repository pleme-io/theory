# Tatara-lisp code generation — OpenAPI to Lisp domains

> **Frame.** The user (2026-04-26):
> *"in the cases of APIs that perhaps we should have the LISP
> tatara-lisp native ideas of code generation maximally flexed and be
> able to do it off OpenAPI / Swagger documentation or other
> structured predictable API of which Google probably definitely
> follows."*
>
> Tatara-lisp is **homoiconic** — code IS data — so generating Lisp
> from a structured spec is *not* a code-generation pipeline; it's
> just a transformation between two S-expression trees. This document
> codifies that pattern for OpenAPI v3 (and equivalents: gRPC
> protobuf, Smithy, Google Discovery, JSON Schema) so any structured
> API becomes a typed tatara-lisp domain in one `nix run`.

---

## I. The principle

Most "API client libraries" are 80% mechanical — given an OpenAPI spec,
you can derive:

- Typed argument schemas (the request body / query string).
- Typed response shapes.
- Per-endpoint named functions following a naming convention.
- Auth policy + retry semantics + request-validation logic.

The remaining 20% is glue (auth bootstrap, idempotency keys, custom
header injection). In tatara-lisp, **the 80% is pure transformation
from OpenAPI Sexp to tatara-lisp Sexp**, and the 20% is a small
hand-written wrapper macro per provider.

```
                 OpenAPI v3 / Smithy / proto / JSON Schema
                                │
                                ▼
                       ┌─────────────────┐
                       │  tatara-codegen  │   read spec → emit (defdomain ...) Sexp
                       │  (tatara-lisp    │
                       │   transform)     │
                       └────────┬────────┘
                                │
                                ▼
                       Generated `.tlisp` file:
                       (defdomain :name :cloudflare/dns
                                  :version "v4"
                                  :endpoints
                                  (list-zone (...)
                                   create-record (...)
                                   ...))
                                │
                                ▼
                       (use-domain :cloudflare/dns)
                       in any consumer .tlisp
```

This is **substantially better** than ordinary codegen because:

1. The generator output is *Lisp*, which the consumer's Lisp can
   inspect, modify, or extend at runtime (homoiconicity).
2. The generator itself is *Lisp* — no separate template language;
   no Jinja / Handlebars / mustache; the codegen *is* a tatara-lisp
   program.
3. Consumers can `(extend-domain :cloudflare/dns ...)` to add
   per-org wrappers (auth defaults, custom retries) without touching
   the generated file.

## II. The shape of the generated domain

For an OpenAPI endpoint:

```yaml
paths:
  /zones/{zone_id}/dns_records:
    post:
      operationId: createRecord
      parameters:
        - name: zone_id
          in: path
          schema: { type: string }
      requestBody:
        content:
          application/json:
            schema: { $ref: '#/components/schemas/DnsRecord' }
      responses:
        '200': { content: { application/json: { schema: { $ref: '#/components/schemas/DnsRecordResponse' } } } }
```

Generate:

```lisp
;; cloudflare/dns/dns.tlisp — auto-generated. Do not hand-edit.

(defdomain cloudflare/dns
  :version "v4"
  :base-url "https://api.cloudflare.com/client/v4"
  :auth :bearer-token

  :endpoints
  (list

    (defn create-record
      [zone_id record]
      :doc "POST /zones/{zone_id}/dns_records — create a DNS record"
      :method :POST
      :path (string-append "/zones/" zone_id "/dns_records")
      :body record
      :request-schema  DnsRecord
      :response-schema DnsRecordResponse)

    (defn list-zones
      []
      :doc "GET /zones — list zones"
      :method :GET
      :path "/zones"
      :response-schema (list-of Zone))

    ;; ... one defn per operationId ...
  )

  :schemas
  (list
    (defschema DnsRecord
      :type     (enum :A :AAAA :CNAME :TXT :MX :NS)
      :name     :string
      :content  :string
      :ttl      (default :integer 1)
      :proxied  (default :bool false))

    (defschema DnsRecordResponse
      :result   DnsRecord
      :success  :bool
      :errors   (list-of ErrorDetail)
      :messages (list-of :string))

    ;; ... one defschema per components.schemas entry ...
  ))
```

The consumer `.tlisp`:

```lisp
(use-domain cloudflare/dns)

(define rec
  (cloudflare/dns/create-record
    "abc123"
    {:type :A :name "drive" :content "10.0.0.5"}))
```

## III. The generator pipeline

```
1. Load        (read-yaml openapi-spec.yaml) → Sexp tree
2. Resolve     resolve $ref → flat schema set
3. Group       collect operations by tag → domain partitions
4. Emit        build (defdomain ...) Sexp for each partition
5. Format      pretty-print + add :doc strings from "description" fields
6. Write       to <provider>/<service>/<service>.tlisp
```

Each step is a tatara-lisp transformation:

```lisp
(defn generate-domain [openapi-spec-path]
  (-> (read-yaml openapi-spec-path)
      resolve-refs
      group-by-tag
      (map emit-domain)
      (each format-and-write)))
```

Where each helper is itself a small Lisp function operating on Sexp
trees. **No template language**, no separate tooling, no FFI to a
codegen runtime.

## IV. Standard provider catalog

When the codegen lands, the canonical pleme-io set:

| Provider | Spec source | Output |
|---|---|---|
| Cloudflare | https://github.com/cloudflare/api-schemas | `cloudflare/{dns,zones,workers,r2,...}/` |
| GitHub | https://github.com/github/rest-api-description | `github/{repos,issues,actions,...}/` |
| GitLab | https://docs.gitlab.com/openapi.json | `gitlab/{projects,issues,merge-requests,...}/` |
| AWS | https://github.com/aws/aws-sdk-rust models | `aws/{s3,sqs,dynamodb,...}/` |
| Google APIs | https://discovery.googleapis.com/discovery/v1/apis | `google/{drive,sheets,calendar,...}/` |
| Datadog | https://docs.datadoghq.com/api/latest/ openapi.yaml | `datadog/{metrics,monitors,...}/` |
| Stripe | https://github.com/stripe/openapi | `stripe/{customers,subscriptions,charges,...}/` |
| Atlassian | https://developer.atlassian.com/cloud/jira/openapi.json | `atlassian/{jira,confluence,...}/` |

Each is one `nix run` away from a generated `.tlisp` domain. **No
hand-written API client code in pleme-io.**

## V. Why Google APIs are particularly easy

Google publishes a *Discovery Document* at
`https://discovery.googleapis.com/discovery/v1/apis` that catalogs
every Google service's OpenAPI-equivalent spec. Each follows the
exact same shape (gapi style). One generator handles all of them —
Drive, Sheets, Calendar, Cloud Storage, Pub/Sub, Spanner, BigQuery
— from one transformation pipeline.

```lisp
(define google-apis
  (-> (http-get-json "https://discovery.googleapis.com/discovery/v1/apis")
      (alist-get "items")
      (filter (lambda (api) (= (alist-get api "preferred") true)))))

(each google-apis
  (lambda (api)
    (-> (http-get-json (alist-get api "discoveryRestUrl"))
        google-discovery-to-openapi
        (write-domain (string-append "google/" (alist-get api "name") "/")))))
```

~10 lines of tatara-lisp generates the full pleme-io Google API surface.

## VI. The generator's own shape

`pleme-io/tatara-codegen` (repo, planned):

```
tatara-codegen/
├── flake.nix
├── lib/
│   ├── openapi.tlisp        OpenAPI 3.x → flat schema + endpoint set
│   ├── google-discovery.tlisp   Google Discovery Document → OpenAPI
│   ├── proto.tlisp          .proto → flat schema set (use buf)
│   ├── smithy.tlisp         Smithy → flat schema set
│   └── emit.tlisp           flat → (defdomain ...) Sexp
├── bin/
│   └── tatara-codegen.tlisp the CLI: input spec → output .tlisp
└── domains/
    ├── cloudflare/...       generated
    ├── github/...           generated
    └── google/...           generated
```

The generator IS itself a tatara-lisp program. It runs via
`tatara-script` and produces other tatara-lisp programs. The whole
chain is git+nix native:

```sh
nix run github:pleme-io/tatara-codegen -- \
  --spec https://api.cloudflare.com/api-schemas/openapi.json \
  --out  ./cloudflare-domains/
```

Outputs are committed back to `pleme-io/tatara-codegen/domains/`,
tagged at the spec version, and consumed via:

```lisp
(use-domain cloudflare/dns
  :version "v4-2026-04-26")
```

The `:version` qualifier picks the right git tag of the codegen
repo — content-addressed, just like every other tatara package
([`TATARA-PACKAGING.md`](TATARA-PACKAGING.md)).

## VII. Composition with the manual stdlib

Generated domains coexist with hand-written stdlib primitives:

```lisp
(use-domain cloudflare/dns)        ; generated
(use-domain pleme/tameshi)         ; hand-written
(use-domain :stdlib/http)          ; built-in

;; Compose freely:
(define attestation
  (tameshi/sign (cloudflare/dns/create-record ...)))
```

The boundary is per-domain `:auth` declarations. Generated domains
ship with `:auth :bearer-token` (or whatever the spec says); the
consumer wires the actual token via `(set-domain-auth!
:cloudflare/dns (env "CLOUDFLARE_TOKEN"))`. **One auth call, every
endpoint authenticated.**

## VIII. Fully-typed responses

Because the spec carries types, the generated domain's responses
arrive *typed*:

```lisp
(define recs (cloudflare/dns/list-records "abc123"))
;; recs is now :: (list-of DnsRecord)
;; The runtime can type-check downstream operations.

(map (lambda (rec)
       ;; rec is DnsRecord, so :type / :name / :content are typed
       (when (= (:type rec) :A)
         (println (:content rec))))
     recs)
```

If the API returns an unexpected shape, the runtime raises a typed
`SchemaMismatch` error pointing at the diff between declared and
actual. **No `KeyError: 'foo'` an hour later in production.**

## IX. Why this matters fleet-wide

Operating across 8+ external APIs (Cloudflare, GitHub, AWS, Google,
Datadog, Stripe, Atlassian, Akeyless) without code generation means:

- 8 hand-written client libraries to maintain.
- 8 separate auth bootstraps.
- 8 different error-handling conventions.
- 8 places where API drift introduces silent breakage.

With OpenAPI → tatara-lisp codegen:

- 1 generator (`tatara-codegen`).
- 1 auth pattern (`:auth :bearer-token | :basic | :oauth2`).
- 1 error type (`SchemaMismatch | RateLimited | Unauthorized`).
- 1 drift detection (regen → diff → PR with the actual changes).

A new API integration is one PR adding the spec URL to the catalog.
The codegen runs in CI, the domain lands in
`pleme-io/tatara-codegen/domains/`, and consumer programs `(use-domain
...)` it. **Four hours from "this provider exists" to "we can
script against it."**

## X. Phasing

- **Phase A (designed — this doc):** the architecture + provider catalog.
- **Phase B:** `pleme-io/tatara-codegen` repo with `lib/openapi.tlisp`
  + `bin/tatara-codegen.tlisp`. ~1-2 weeks.
- **Phase C:** seed the canonical domains: cloudflare, github, datadog
  (the three with the most traffic in pleme-io today).
- **Phase D:** wire `kindling` / `repo-forge` to auto-register a new
  provider when a `provider-spec.yaml` lands in a repo.
- **Phase E:** explore deeper integrations:
  - tameshi attestation per generated endpoint call (proof of audit).
  - `forge-gen` mode that generates IaC types alongside the API client.
  - Cross-domain saga shapes (Pattern #31 from
    [`WASM-PATTERNS.md`](WASM-PATTERNS.md)).

## XI. See also

- [`THEORY.md` Pillar 12](THEORY.md) — generation over composition
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as canonical scripting
- [`TATARA-PACKAGING.md`](TATARA-PACKAGING.md) — git+nix native packaging
- [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md) — typed CRD generation (sister pattern)
- [`forge-gen`](https://github.com/pleme-io/forge-gen) — Rust-side OpenAPI codegen for IaC
- [`mcp-forge`](https://github.com/pleme-io/mcp-forge) — Rust-side OpenAPI codegen for MCP servers
- The Rust side already has these for IaC + MCP; this document is the
  Lisp side, completing the codegen surface across both languages.
