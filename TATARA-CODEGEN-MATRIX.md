# Tatara-lisp codegen matrix — every structured spec → Lisp domain

> **Frame.** [`TATARA-CODEGEN.md`](TATARA-CODEGEN.md) shows OpenAPI →
> tatara-lisp domain. **The same pipeline applies to every other
> structured-spec domain in the pleme-io fleet:** gRPC protobuf,
> GraphQL SDL, Terraform/OpenTofu providers, SQL DDL, Kubernetes CRDs,
> MCP tool catalogs, CLI completion specs. This matrix catalogs all
> of them. The generator is always tatara-lisp; the output is always
> tatara-lisp; what varies is the input parser.

---

## I. The unifying claim

```
                  ┌─────────────────────────────────────────────────┐
                  │  any structured input format                     │
                  │  ──── → S-expression tree                       │
                  │  └ OpenAPI 3 / Swagger                            │
                  │  └ gRPC .proto                                    │
                  │  └ GraphQL SDL (.graphql)                         │
                  │  └ Terraform provider.json                        │
                  │  └ SQL DDL / migration files                      │
                  │  └ K8s CRD OpenAPI v3                             │
                  │  └ JSON Schema / Smithy                           │
                  │  └ MCP tool catalog                               │
                  │  └ Bash completion spec / fish_complete           │
                  └────────────────────────┬────────────────────────┘
                                           │
                                           ▼
                  ┌─────────────────────────────────────────────────┐
                  │  tatara-lisp transformation pipeline             │
                  │  Sexp tree → flatten/resolve → group → emit      │
                  └────────────────────────┬────────────────────────┘
                                           │
                                           ▼
                  ┌─────────────────────────────────────────────────┐
                  │  generated tatara-lisp domain                   │
                  │  (defdomain :name :provider/service              │
                  │              :endpoints (...)                    │
                  │              :schemas (...))                     │
                  └─────────────────────────────────────────────────┘
```

**Every codegen target follows this shape.** Different parser, same
emitter, same consumer ergonomics. The maintainer of one parser per
input format is the entire fleet's API/CRD/SQL/RPC client surface.

## II. The matrix

| Input format | Source spec | Generated `defdomain` shape | Consumer use |
|---|---|---|---|
| **OpenAPI v3** (REST) | provider's openapi.yaml | `:endpoints (defn create-record …)` + `:schemas` | `(cloudflare/dns/create-record …)` |
| **gRPC protobuf** | service.proto + buf | `:rpcs (defrpc list-things …)` + `:messages (defmessage Thing …)` | `(grpc-call my-service :ListThings req)` |
| **GraphQL SDL** | schema.graphql | `:queries / :mutations / :subscriptions` + `:types` | `(graphql/query :find-user {:id "..."})` |
| **Terraform provider** | provider's `provider.json` schema | `:resources (defresource aws_s3_bucket …)` + `:data-sources` | `(aws/s3-bucket "main" {:bucket "…"})` |
| **SQL DDL** (Postgres / MySQL / SQLite) | schema.sql / migrations | `:tables (defmodel User …)` + `:relations` | `(db/find :user :where {:email "..."})` |
| **Kubernetes CRD** | crd.yaml openapi v3 | `:resources (defresource DnsReconciler …)` + admission validators | `(k8s/apply :dns-reconciler {…})` |
| **JSON Schema / Smithy** | spec.json | typed value validators + accessors | `(validate :my-schema value)` |
| **MCP tool catalog** | mcp.json | `:tools (deftool list-pods …)` exposed to Claude/agents | served via `mcp-server.tlisp` |
| **CLI completion** | bash/fish completion files | `:commands (defcommand kubectl …)` + arg parsers | shell completion + scripted invocation |

Each row is **one parser + one shared emitter**. The parsers are
bounded: ~200-500 lines of tatara-lisp each.

## III. IaC — Terraform/OpenTofu provider codegen

The Rust side already has [`forge-gen`](https://github.com/pleme-io/forge-gen) +
[`pangea-forge`](https://github.com/pleme-io/pangea-forge) — codegen
from OpenAPI to Terraform/Pulumi/Crossplane/Ansible/Pangea Ruby.
The **Lisp side completes the surface**:

```lisp
;; Generated from terraform-provider-cloudflare's provider.json schema:

(defdomain cloudflare/iac
  :version "v4-cf-provider-4.x"

  :resources
  (list
    (defresource cloudflare-dns-record
      :type "cloudflare_dns_record"
      :provider :cloudflare
      :spec   (defschema CloudflareDnsRecordSpec
                :zone-id  :string
                :name     :string
                :type     (enum :A :AAAA :CNAME :TXT :MX)
                :value    :string
                :ttl      (default :integer 1)
                :proxied  (default :bool false))
      :outputs (defschema CloudflareDnsRecordOutputs
                :id       :string
                :hostname :string))

    ;; ... one defresource per cloudflare_* type ...
  )

  :data-sources
  (list
    (defdata-source cloudflare-zone
      :type "cloudflare_zone"
      :spec   (defschema CloudflareZoneFilter :name :string)
      :outputs (defschema CloudflareZoneData :id :string :status :string))
  ))
```

Consumer:

```lisp
(use-domain cloudflare/iac)

(define zone (cloudflare/iac/data-source :cloudflare-zone {:name "quero.cloud"}))

(cloudflare/iac/cloudflare-dns-record "drive"
  {:zone-id (:id zone)
   :name "drive"
   :type :A
   :value "10.0.0.5"})
```

Synthesizes to Terraform JSON; pangea applies it. Same flow as the
existing pangea-architectures Ruby DSL, but in Lisp + content-pinned
to the provider's openapi spec version.

## IV. gRPC — protobuf to defdomain

```lisp
;; Generated from auth/v1/auth.proto:

(defdomain auth/v1
  :version "v1.2.0"
  :transport :grpc
  :service "auth.v1.AuthService"

  :messages
  (list
    (defmessage AuthenticateRequest
      :token  :string
      :scopes (list-of :string))

    (defmessage AuthenticateResponse
      :user-id :string
      :session :string
      :expires-at :timestamp))

  :rpcs
  (list
    (defrpc authenticate
      :request  AuthenticateRequest
      :response AuthenticateResponse
      :doc      "Verify a bearer token and return a session reference")

    (defrpc revoke-session
      :request  RevokeSessionRequest
      :response Empty)))
```

Consumer:

```lisp
(use-domain auth/v1)

(define resp
  (auth/v1/authenticate
    {:token "abc123" :scopes ["read" "write"]}
    :endpoint "auth.svc.cluster.local:443"))

(println (:user-id resp))
```

The runtime uses `tonic` under the hood; `tatara-lisp-script` exposes
a `(grpc-call domain rpc request opts)` primitive that parses the
generated domain's transport metadata and dispatches.

## V. GraphQL — schema → defdomain

```lisp
;; Generated from github.graphql:

(defdomain github/graphql
  :version "2024-04-26"
  :transport :graphql
  :endpoint "https://api.github.com/graphql"

  :types
  (list
    (defschema Repository
      :id     :string
      :name   :string
      :owner  Actor
      :issues (connection-of Issue))

    (defschema Issue
      :id      :string
      :title   :string
      :state   (enum :OPEN :CLOSED)))

  :queries
  (list
    (defquery viewer-repositories
      :doc "All repos accessible to the authenticated user"
      :args (defschema { :first (default :integer 10) })
      :returns (connection-of Repository)))

  :mutations
  (list
    (defmutation close-issue
      :args (defschema { :issue-id :string :reason (default :string "completed") })
      :returns Issue))

  :subscriptions [])
```

Consumer:

```lisp
(use-domain github/graphql)

(define repos (github/graphql/viewer-repositories {:first 5}))
(each repos
  (lambda (r) (println (:name r))))
```

## VI. SQL ORM — DDL → defmodel

The user's hint about SeaORM + Prisma: same shape — DDL is structured
data; codegen emits typed access patterns.

```lisp
;; Generated from migrations/0001_create_users.sql:

(defdomain pleme/db/users
  :backend :postgres
  :schema "public"

  :models
  (list
    (defmodel User
      :table "users"
      :primary-key :id
      :fields
      (list
        (deffield :id        :uuid :default :gen-uuid)
        (deffield :email     :string :unique true :nullable false)
        (deffield :created-at :timestamp :default :now)
        (deffield :archived  :bool :default false))
      :relations
      (list
        (defhas-many :sessions :table "sessions" :foreign-key :user-id)
        (defbelongs-to :tenant :table "tenants" :foreign-key :tenant-id)))

    (defmodel Session
      :table "sessions"
      :primary-key :token
      :fields ...))

  :queries
  (list
    (defquery find-user-by-email
      :model User
      :where {:email :param/email}
      :returns :one-or-nil)

    (defquery list-active-users
      :model User
      :where {:archived false}
      :order-by [[:created-at :desc]]
      :returns :many)))
```

Consumer:

```lisp
(use-domain pleme/db/users)

(define u (pleme/db/users/find-user-by-email {:email "drzzln@protonmail.com"}))
(when u
  (println (:id u))
  (define sessions (relation u :sessions))
  (each sessions (lambda (s) (println (:token s)))))
```

The runtime uses an existing connection pool (`pleme/db` stdlib
domain provides connection mgmt); the generator owns the typed-query
surface.

Migrations are codegen too — `defmigration` blocks emit DDL for a
shinka migration runner. **The DDL is the source of truth; both the
ORM and the migrations are derived.**

## VII. Kubernetes CRDs — same OpenAPI v3 shape

Already covered by [`LISP-YAML-CONTROLLERS.md`](LISP-YAML-CONTROLLERS.md):
`(defschema ...)` blocks auto-generate the CRD's OpenAPI spec.
**Reverse direction** is also valuable: an existing CRD's openapi
schema → generated `defdomain` so a controller can consume any
third-party CRD with typed fields.

```lisp
;; Generated from cert-manager.io_certificates.yaml CRD:

(defdomain cert-manager.io/v1
  :resources
  (list
    (defresource Certificate
      :group   "cert-manager.io"
      :version "v1"
      :kind    "Certificate"
      :spec    (defschema CertificateSpec
                :secret-name :string
                :issuer-ref  IssuerRef
                :common-name :string
                :dns-names   (list-of :string))
      :status  (defschema CertificateStatus
                :conditions  (list-of Condition)
                :renewal-time :timestamp))))
```

Now any tatara-lisp controller can `(k8s/list :certificates)` and get
typed `Certificate` values back, validated against the CRD's schema.

## VIII. MCP tools — agent-callable surface

Pleme-io already has [`mcp-forge`](https://github.com/pleme-io/mcp-forge)
generating Rust MCP servers from OpenAPI. The **Lisp side** generates
agent-callable tool catalogs:

```lisp
;; Generated from cloudflare/openapi.json + custom tool annotations:

(defdomain cloudflare/mcp
  :tools
  (list
    (deftool list-zones
      :description "List Cloudflare zones for the authenticated account"
      :input-schema (defschema { :name (default :string nil) })
      :handler (lambda (args) (cloudflare/dns/list-zones args)))

    (deftool create-record
      :description "Create a DNS record in a Cloudflare zone"
      :input-schema (defschema { :zone-id :string :name :string :type :A :value :string })
      :handler (lambda (args) (cloudflare/dns/create-record (:zone-id args) args)))))
```

Run `tatara-script mcp-server.tlisp` to expose every `deftool` over
stdio MCP — the agent (Claude, etc.) lists + invokes them with full
typing.

**One generator output, two consumers:** scripts call the generated
domain directly; the MCP server wraps the same tools for agents.

## IX. CLI completion specs — bash + fish

```lisp
;; Generated from kubectl bash_completion:

(defdomain shell/kubectl
  :commands
  (list
    (defcommand get
      :description "Display one or many resources"
      :positional
      (list (defarg :resource :type :string :completion :resource-types))
      :flags
      (list (defflag :namespace :short "-n" :type :string :completion :namespaces)
            (defflag :output :short "-o" :type (enum :json :yaml :wide))))

    (defcommand apply
      :description "Apply a configuration"
      :flags
      (list (defflag :file :short "-f" :type :path :completion :yaml-files)))))
```

Consumer: `skim-tab`, fish-shell wrappers, mcp servers — all consume
the same Sexp domain.

## X. The generator catalog repo

`pleme-io/tatara-codegen` (planned) holds:

```
tatara-codegen/
├── lib/
│   ├── openapi.tlisp       OpenAPI 3.x parser
│   ├── proto.tlisp         protobuf parser (uses buf for bootstrapping)
│   ├── graphql.tlisp       GraphQL SDL parser
│   ├── tf-provider.tlisp   provider.json parser
│   ├── sql-ddl.tlisp       SQL DDL parser (sqlparser-rs lib via FFI? or pure tlisp)
│   ├── crd.tlisp           K8s CRD OpenAPI parser
│   ├── jsonschema.tlisp    JSON Schema parser
│   ├── smithy.tlisp        Smithy parser
│   ├── mcp.tlisp           MCP catalog parser
│   ├── completion.tlisp    bash/fish completion parser
│   └── emit.tlisp          shared (defdomain ...) emitter
├── bin/
│   ├── tatara-codegen.tlisp     CLI dispatch
│   └── auto-discover.tlisp      walk a repo, detect spec files, emit domains
└── domains/
    ├── cloudflare/...
    ├── github/...
    ├── google/...
    └── ...
```

`auto-discover.tlisp` walks any pleme-io repo, detects the spec files
present (openapi.yaml, schema.graphql, .proto, etc.), and runs the
right parser + emits the matching domain. **Add a new external
service: drop its spec file in your repo, run
`tatara-codegen auto-discover .`, get a generated domain.**

## XI. Why this is uniquely strong

Three things compound:

1. **One emitter, N parsers.** Every input format converges on the
   same `(defdomain ...)` Sexp emitter. Adding a new format is a
   new parser; the consumer ergonomics are identical.

2. **The generator is Lisp.** No template language, no Jinja, no
   FFI between codegen languages. The whole codegen pipeline is
   tatara-lisp transformations on Sexp trees. Authors of new
   parsers write idiomatic Lisp.

3. **The output is Lisp.** Consumers can macroexpand, inspect,
   modify, or compose generated domains at runtime — homoiconicity.
   `(extend-domain :cloudflare/dns ...)` adds a custom retry policy
   without touching the generated file.

The fleet's complete external-API + database + CRD + IaC surface
becomes **one repo of generated `.tlisp` domains**, content-pinned
to spec versions, regenerable on demand. **Day-2 maintenance shrinks
to bumping a `?ref=` qualifier.**

## XII. Relationship with the Rust-side codegens

Pleme-io has Rust-side codegen tools that generate Rust:

- `forge-gen` — OpenAPI → Rust SDKs + IaC + MCP servers + schemas + docs
- `mcp-forge` — OpenAPI → Rust MCP servers
- `completion-forge` — OpenAPI → skim-tab YAML + fish completions
- `pangea-forge` (Rust generator emitting Ruby DSL) — OpenAPI → Pangea resources
- `forge-gen` umbrella — orchestrates all the above

The Lisp-side `tatara-codegen` is the **complementary surface**:

| Output format | Generator | When you want this |
|---|---|---|
| Rust crate | `forge-gen` / `mcp-forge` | static binary, max performance, type-safe at compile time |
| Pangea Ruby DSL | `pangea-forge` | infrastructure synthesis through the existing Pangea pipeline |
| Tatara-lisp domain | `tatara-codegen` (this) | scripted access, hot-reload, agent-callable, runtime composition |

All three consume the same OpenAPI (or .proto, etc.) input. **The
input is the source of truth; the generators emit different runtimes
of the same logical surface.**

## XIII. Phasing

- **Phase A (this doc):** the catalog + the unifying claim.
- **Phase B:** `tatara-codegen` repo with `lib/openapi.tlisp` +
  `bin/tatara-codegen.tlisp` (per [`TATARA-CODEGEN.md` §X](TATARA-CODEGEN.md)).
- **Phase C:** add `lib/{proto,graphql,tf-provider,sql-ddl,crd}.tlisp`
  parsers — one per session, each unblocking a category of programs.
- **Phase D:** `auto-discover.tlisp` — walk repos, emit domains.
- **Phase E:** integrate with `forge-gen` so one CI pipeline updates
  the Rust + Pangea + Lisp surfaces from the same spec.

## XIV. See also

- [`TATARA-CODEGEN.md`](TATARA-CODEGEN.md) — OpenAPI deep dive
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — 4-layer compute hierarchy
- [`TATARA-PACKAGING.md`](TATARA-PACKAGING.md) — git+nix native packaging
- [`forge-gen`](https://github.com/pleme-io/forge-gen) — Rust-side OpenAPI codegen
- [`mcp-forge`](https://github.com/pleme-io/mcp-forge) — Rust-side MCP server codegen
- [`pangea-forge`](https://github.com/pleme-io/pangea-forge) — Pangea Ruby DSL codegen
