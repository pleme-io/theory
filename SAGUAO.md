# Saguão — fleet-wide identity, authorization, and self-service portal

> **Frame.** [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) places one
> Servico per ComputeUnit; [`MESH-COMPOSITION.md`](./MESH-COMPOSITION.md)
> composes Servicos into typed Aplicacaos within a single cluster.
> **This doc is the next layer up:** how *people* — family members,
> operators, future invitees — sign in once and self-serve across
> *every cluster in every location* the fleet hosts. Identity,
> authorization, and the user-facing portal are typed primitives at
> the **fleet** level, not the cluster level.
>
> `saguão` (Brazilian-Portuguese: lobby / foyer; ASCII-folded:
> `saguao`) is the umbrella name for the four primitives that, taken
> together, are the canonical pleme-io answer to "let my family
> sign in with Google and manage their own access to my homelabs."
>
> The doc is normative. Every homelab pleme-io operates — `rio`
> (Bristol) today, `mar` (Parnamirim) and `rai` (Natal) tomorrow,
> any cluster anywhere — adopts this shape. Deviation requires an
> explicit `skip-saguao:` note at the top of the deviating
> cluster's `CLAUDE.md`.

---

## I. The fleet-portal problem

### I.1. What a homelab fleet actually needs

The substrate, today, runs one cluster (rio). Vault, photos,
Jellyfin, Home Assistant, Paperless, HedgeDoc, Cal.com, the
newsletter, and Gitea are deployed, gated by Cloudflare Zero Trust
Access at the edge and Authentik provisioned-but-suspended in
cluster. The operator (`drzln@protonmail.com`) and a handful of
placeholder family emails are listed in
`pangea-architectures/workspaces/cloudflare-pleme/domains/quero.cloud.yaml`
under `zero_trust_access.groups[0].include`. Adding a member is
"edit one line, `bundle exec pangea apply quero_cloud`."

That works for one cluster, ~10 apps, three users. It breaks the
moment the fleet looks like:

- **3 clusters in 3 locations** — `rio.bristol`, `mar.parnamirim`,
  `rai.natal`. Each cluster runs a vault, a photos, a media server.
  Three distinct Vaultwardens cannot all live at `vault.quero.cloud`.
- **20+ family members** with overlapping but non-identical access
  — wife sees Bristol's photos but not Natal's lab, cousin sees only
  the family chat, parents see only the photo album from their
  trips, etc. Cloudflare Access's group-membership model is
  one-dimensional (groups list emails); pleme-io needs *per-user,
  per-location, per-cluster, per-service, per-verb* authorization.
- **Source of truth in a database, not a repo.** Authentik's policy
  authoring lives in its admin UI as Python-expression policies
  inside its Postgres. Cloudflare Access groups live in
  Cloudflare's API. Neither is grep-able, version-controlled,
  reviewable in a PR, or readable by other systems (e.g., the
  family-facing portal that needs to render "what can I see").
- **The portal itself is missing.** A new family member receives
  an email OTP from Cloudflare Access for an app they may not even
  know exists. There is no front door at `quero.cloud` saying
  "welcome — here are the things you can use."

### I.2. The four breakages, named

Today's de-facto architecture has four distinct failure modes that
N=1 hides:

1. **Hostname collision.** `vault.quero.cloud` claims the apex zone
   for a single cluster's service. Cluster #2 has nowhere to put
   its vault.
2. **Authz model collision.** Cloudflare Access's
   `group → application → policy` model assumes one decision per
   subdomain. Per-user-per-location-per-verb cannot be expressed
   without N×M Cloudflare apps.
3. **Identity model collision.** Two independent identity stores
   (Cloudflare Access database + Authentik database) with no
   structural reason for both. The duplication compounds with each
   new family member.
4. **Portal absence.** No artifact answers "what does
   `cousin@example.com` get when they navigate to `quero.cloud`?"
   There is no place for cousin to see, configure, or reason about
   their own access.

### I.3. What we want

A typed fleet primitive — declared once, rendered everywhere — that
yields:

- **One sign-in surface.** A user authenticates once (Google OIDC,
  via the fleet-wide IdP). Cookie covers every gated service across
  every cluster, every location.
- **One portal.** Apex `quero.cloud` is the family-facing PWA. It
  shows the user every location they have access to, every cluster
  inside each location, every service inside each cluster — only
  the ones they're allowed to see.
- **Typed, version-controlled authz.** "Who can see what" is a
  typed declarative source-of-truth in a git repo, authored as
  `(defcrachá …)`, consumed by every component that needs to make
  an access decision (the portal, the per-cluster forward-auth, the
  admin UI, audit tooling).
- **Cluster-local enforcement.** Every cluster's Ingress controller
  gates every request through a forward-auth subrequest *inside*
  the cluster — no Cloudflare Access in the critical path. The
  cluster is authoritative for its own traffic, with the
  fleet-wide identity and policy as control-plane inputs.
- **Cloudflare reduced to transparent transport.** Cloudflare
  Tunnel terminates TLS and tunnels packets from the public
  internet to the cluster's ingress-nginx. Nothing else.
  Cloudflare Access (the Zero Trust SSO product) is removed from
  the architecture.
- **Single-declaration cluster onboarding.** Adding cluster N to
  the fleet is one typed entry; DNS, cluster registration,
  forward-auth deployment, and portal visibility propagate
  automatically.

---

## II. Topology — control plane vs data plane

### II.1. The split is the load-bearing decision

Saguão decomposes into a **control plane** (identity + policy)
that lives **once** at the fleet edge, and a **data plane**
(forward-auth enforcement) that lives **on every cluster**.

```
                       ┌──────────────────────────────────┐
                       │ CONTROL PLANE — at quero.cloud   │
                       │                                  │
                       │  passaporte ──► auth.quero.cloud │
                       │  (fleet IdP — Authentik wrapped, │
                       │   Google OIDC federated)         │
                       │                                  │
                       │  crachá    ──► cracha.quero.cloud│
                       │  (typed AccessPolicy CRD +       │
                       │   gRPC API + admin UI)           │
                       │                                  │
                       │  varanda   ──► quero.cloud,      │
                       │                <loc>.quero.cloud,│
                       │                <clu>.<loc>.…     │
                       │  (Yew PWA — reads passaporte JWT,│
                       │   queries crachá API, renders    │
                       │   "what can I see")              │
                       └──────────────┬───────────────────┘
                                      │ OIDC + gRPC
            ┌─────────────────────────┼─────────────────────────┐
            │                         │                         │
            ▼                         ▼                         ▼
  ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐
  │ DATA PLANE      │       │ DATA PLANE      │       │ DATA PLANE      │
  │ rio (bristol)   │       │ mar (parnamirim)│       │ rai (natal)     │
  │                 │       │                 │       │                 │
  │ vigia ◄─ ingress│       │ vigia ◄─ ingress│       │ vigia ◄─ ingress│
  │  (forward-auth) │       │  (forward-auth) │       │  (forward-auth) │
  │                 │       │                 │       │                 │
  │ vault, photos,  │       │ vault, photos,  │       │ media, dev,     │
  │ jellyfin, …     │       │ jellyfin, …     │       │ archive, …      │
  └─────────────────┘       └─────────────────┘       └─────────────────┘
```

### II.2. Why split

Three reasons:

1. **Single source of truth.** Identity and authorization are
   fleet primitives — the same human is the same human regardless
   of which cluster they're logging into. Storing identity twice
   guarantees drift; storing it once with cluster-local enforcement
   gets the property right.
2. **Independent failure domains.** Per-cluster forward-auth
   continues serving cached JWTs for the duration of the user's
   session if the control plane is briefly unreachable. A cluster
   does not lose authn the instant `auth.quero.cloud` blips.
3. **Cluster onboarding is mechanical.** New cluster N gets a
   `vigia` HelmRelease pointing at `auth.quero.cloud` and
   `cracha.quero.cloud`. No new IdP to provision, no new policy
   store to seed, no new admin UI to wire up.

### II.3. Cloudflare's role — exactly one thing

**Cloudflare is a transparent tunnel and DNS provider. Nothing else.**

Concretely, what Cloudflare does in the saguão architecture:

- Hosts DNS for `quero.cloud` and all subdomains.
- Runs **Cloudflare Tunnel** (`cloudflared`) on every home-edge
  cluster's Ingress — terminates the TCP connection from the public
  internet, encapsulates the traffic, and forwards it to the
  cluster's `ingress-nginx` Service. This is the only way services
  are reachable from outside the cluster's local network.
- Provides **TLS termination at the edge** (free wildcard certs per
  zone) and DDoS protection at the network layer.

What Cloudflare **does not** do:

- **Cloudflare Access (Zero Trust SSO).** Removed entirely. The
  `zero_trust_access:` block is dropped from
  `pangea-architectures/workspaces/cloudflare-pleme/domains/quero.cloud.yaml`.
  No more email OTP, no more Cloudflare-side identity store, no more
  Cloudflare-side policy authoring.
- **Cloudflare Workers as application logic.** No saguão component
  runs as a Worker. (Cloudflare Pages is permitted as a static-asset
  CDN for the varanda PWA bundle — the Worker layer is unused.)
- **Cloudflare Identity / WAF rules as authorization decisions.**
  All authz happens in-cluster, in `vigia`, against `crachá`.

The rule for any future "should this go on Cloudflare?" question
is: **if it's not packet routing, it does not go on Cloudflare**.

### II.4. Why control plane lives at one cluster (not federated)

Three configurations were considered:

| Shape | Identity store | Authz store | Failure mode |
|---|---|---|---|
| **Centralized (chosen)** | one IdP at `auth.quero.cloud` | one crachá API at `cracha.quero.cloud` | control-plane outage = no new sign-ins fleet-wide; cached sessions keep working |
| **Per-cluster federated** | one IdP per cluster, federated upstream | one crachá per cluster | complex; identity drift; bad UX (sign in N times) |
| **Per-location, central per location** | one IdP per location | one crachá per location | partial; still leaks the "across locations" use case |

The centralized shape wins because (a) the user-visible benefit of
"sign in once everywhere" requires it, (b) the failure mode is
softened by aggressive JWT caching in the data plane (24h–168h
sessions already in use), (c) the control-plane components are
small enough to host on a tiny dedicated VM if/when rio needs to be
unblocked from carrying it.

Where the control-plane physically *runs* is not load-bearing on
the architecture; the user-facing hostnames (`auth.quero.cloud`,
`cracha.quero.cloud`) abstract the location. See §VI for the
recommended trajectory: rio today, dedicated tiny cluster when
N≥2.

---

## III. The four typed primitives

Each primitive is a typed Rust crate with `#[derive(TataraDomain)]`,
a Helm chart for the deployable parts, and an entry in the relevant
substrate flake builder. The names are Brazilian-Portuguese in the
"enclosed-space / civic" semantic family; the ASCII fold drops
diacritics for repo, crate, and chart names.

### III.1. passaporte — the typed identity primitive

> "Passport." The credential that proves who you are.

| Aspect | Detail |
|---|---|
| **Repo** | `pleme-io/passaporte` |
| **Crate** | `passaporte` (Rust library) — typed config form, JWT verification helpers built on `kenshou` |
| **Chart** | `lareira-passaporte` (Helm umbrella, supersedes `lareira-authentik`) |
| **Deployable** | Authentik (upstream `goauthentik/authentik`), wrapped with pleme-io defaults; KEDA HTTP scale-to-zero on server, always-on outpost + worker |
| **Typed form** | `(defpassaporte :idp authentik :federated [google] :host auth.quero.cloud :scopes [openid email profile groups] :session-duration 24h)` |
| **What it owns** | The single fleet identity store. Issues OIDC ID + access tokens. Federates Google as a social provider. Owns user records, group records, MFA settings, session lifetimes. |
| **What it does NOT own** | Authorization decisions. passaporte answers "who is this," not "what can they do." |

**Why a typed Rust wrapper over Authentik (vs replacing Authentik):**
Authentik is a mature OIDC provider (~10 years, security-audited).
Replacing it with a from-scratch Rust IdP is a multi-month security-
critical project with no compounding payoff. Wrapping it with a
typed config form (a) gives pleme-io callers `(defpassaporte …)` —
the Lisp surface — instead of clicking through Authentik's admin
UI, (b) lets the Rust crate provide JWT verification helpers that
every other saguão component uses (varanda for session reads, vigia
for forward-auth checks), (c) preserves the option of swapping the
IdP backend later without breaking callers.

The Helm chart `lareira-passaporte` succeeds the existing
`lareira-authentik` chart in `helmworks/charts/` — same upstream
dependency, same defaults, renamed to match the typed primitive.

### III.2. crachá — the typed authorization primitive

> "Badge." The credential that says what you're allowed to do.

| Aspect | Detail |
|---|---|
| **Repo** | `pleme-io/cracha` (ASCII-folded; doc uses `crachá`) |
| **Crate** | `cracha-core` (typed AccessPolicy IR), `cracha-controller` (kube-rs reconciler), `cracha-api` (gRPC + REST API) |
| **Chart** | `lareira-cracha` (Helm chart deploying the controller + API) |
| **CRD** | `AccessPolicy` (cluster-scoped); `AccessGroup` (cluster-scoped); `ServiceCatalog` (cluster-scoped) |
| **Typed form** | `(defcrachá :name family :members [drzln cousin wife] :grants [(grant :user drzln :locations [* ] :clusters [* ] :services [* ] :verbs [* ]) (grant :user wife :locations [bristol] :clusters [rio] :services [photos jellyfin notes] :verbs [read]) (grant :user cousin :locations [bristol] :clusters [rio] :services [chat] :verbs [read write])])` |
| **What it owns** | The single source of truth for authorization across the fleet. Renders to `AccessPolicy` CRDs in the control-plane cluster; serves authz decisions over gRPC to vigia; serves "what can this user see" over REST to varanda. |
| **What it does NOT own** | Identity (that's passaporte). The actual enforcement (that's vigia). |

**Why a typed CRD (vs Authentik expression policies):**
Authentik's policy DSL is Python expressions evaluated inside its
Postgres-backed admin UI. That works for "is this user in this
group" — but the source of truth lives in a database row, not in
git. The crachá CRD makes the policy a typed value in the k8s
repo, version-controlled, reviewable in PRs, readable by both
varanda (for "what can I see") and vigia (for "should I let this
request through"). Authentik becomes a dumb token signer; crachá
owns the actual authz decision.

**The five verbs.** Authorization is intentionally coarse-grained
at this layer. The verb lattice is:

| Verb | Meaning | Example |
|---|---|---|
| `read` | view-only access (HTTP GET) | wife can view photos |
| `write` | create/update (HTTP POST/PUT/PATCH) | cousin can post in chat |
| `delete` | remove (HTTP DELETE) | operator can delete a vault entry |
| `admin` | configure the service itself | only operator |
| `*` | all of the above | only operator |

Per-resource ACL (e.g., "wife can edit *this specific* photo") is
explicitly out of scope — that lives inside the application
itself, not in saguão. Saguão's job is "should this user reach this
service at all, and at what verb tier?"

### III.3. vigia — the per-cluster forward-auth data plane

> "Sentinel." The guard at every door.

| Aspect | Detail |
|---|---|
| **Repo** | `pleme-io/vigia` |
| **Crate** | `vigia` (Rust binary, axum) |
| **Chart** | `lareira-vigia` (Helm chart, deployed to every cluster) |
| **Deployable** | Rust HTTP service running per-cluster; `nginx.ingress.kubernetes.io/auth-url` annotations point at it |
| **Typed form** | `(defvigia :cluster rio :location bristol :passaporte-url https://auth.quero.cloud :cracha-url grpc://cracha.quero.cloud :session-cache 1024 :jwt-cache-ttl 5m)` |
| **What it owns** | Per-cluster authn-and-authz enforcement. Receives every request bound for a gated Ingress as a subrequest from nginx, validates the passaporte-issued JWT, calls crachá over gRPC to make the authz decision, returns 200/401/403. Caches JWT verification keys + recent decisions. |
| **What it does NOT own** | Identity (passaporte) or policy authoring (crachá). vigia is pure enforcement. |

**Phase-1 implementation note.** Until vigia is built, the
forward-auth role is filled by the **Authentik embedded outpost**
(deployed by `lareira-passaporte`), wired via the existing
`pleme-lib.compliance.authn.oidc` template helper in
`helmworks/charts/pleme-lib/templates/_compliance_authn.tpl`.
crachá's gRPC API is consulted via Authentik's "external policy
webhook" hook. When vigia is built (Phase 2 — see §VII), the
helper gains a `provider: "vigia"` option and the migration is one
values-file change per chart, no consumer code edits.

The pleme-lib helper signature is the load-bearing abstraction —
charts call `pleme-lib.compliance.authn.oidc` with `provider:
"authentik"` (today) or `provider: "vigia"` (Phase 2); the helper
emits the right `auth-url` annotation. **Charts never reference
Authentik or vigia directly.**

### III.4. varanda — the family-facing PWA

> "Porch / front balcony." The first place a guest sets foot when
> arriving at the house.

| Aspect | Detail |
|---|---|
| **Repo** | `pleme-io/varanda` |
| **Crate** | `varanda` (Rust + Yew, WASM) |
| **Deployable** | Cloudflare Pages (Freescape) — single SPA bundle deployed to `quero.cloud` + `www.quero.cloud` + every `<location>.quero.cloud` + every `<cluster>.<location>.quero.cloud` (same bundle, hostname picks the slice it renders). Aliases (`www`, `app`, `home`) collapse to Fleet view. |
| **Typed form** | `(defvaranda :host quero.cloud :passaporte-host auth.quero.cloud :cracha-api https://cracha.quero.cloud :ishou-tokens default :modes [fleet location cluster])` |
| **Design system** | Consumes [`ishou`](https://github.com/pleme-io/ishou) at build time — the central pleme-io tokens (color / typography / spacing / shadow / motion / brand). The flake input renders `ishou-tokens.css` byte-identically across every consumer. The `industrial.css` override layer adds varanda's mechanical aesthetic on top of those tokens. **NEVER hand-author colors/fonts/spacing in varanda** — extend ishou first. Full design language: [`varanda/docs/design.md`](https://github.com/pleme-io/varanda/blob/main/docs/design.md). |
| **What it owns** | The user-facing front door. Reads the passaporte session cookie (set by Authentik when the user signs in); queries the crachá REST API for "what can I see"; renders the appropriate slice based on hostname (apex = fleet view, location subdomain = location view, cluster subdomain = cluster view); generates the deep links to the actual services. |
| **What it does NOT own** | Authentication (passaporte handles sign-in via redirect-to-Authentik), authorization (crachá), or any protected user data (the actual services hold their own data). The Cloudflare configuration (Pages custom-domains, DNS records, Tunnel ingress) is owned by `pangea-architectures/workspaces/cloudflare-pleme/` and reconciled by **pangea-operator on rio** (see §VI.4 below). |

**Why Cloudflare Pages.** varanda is a static SPA; it never holds
secrets and never makes server-side decisions. Cloudflare Pages is
free, hosts the asset bundle on the same anycast edge as Cloudflare
Tunnel (so latency is uniform with the gated services), and
handles wildcard custom-domain routing without per-zone setup. The
`cloudflare-headless-blog` skill establishes the precedent in the
fleet (zuihitsu blog at `blog.quero.cloud`); varanda is the same
shape with zero runtime backend.

**Why Yew.** Rust-end-to-end means varanda shares types with
crachá-core (same `AccessPolicy` struct deserializes cleanly from
the API response), shares the JWT verification helpers from the
passaporte crate, and benefits from `wasm-pack`'s aggressive
tree-shaking (~150KB gzipped target). Substrate already ships
`rust-wasm` builders (`substrate/lib/`) used by other Yew apps.

### III.5. saguão — the umbrella term

`saguão` is **not** a separate component or repo. It is the umbrella
name for the four primitives above when discussed as a system.

The lobby of a Brazilian house is the room every visitor passes
through, the room where guests are received before being shown
where they are allowed to go. The metaphor maps cleanly:
`varanda` is the porch (you're outside, then you knock);
`passaporte` proves who you are; `crachá` says which rooms you can
enter; `vigia` is the guard standing at each room's door.

When discussing the system as a whole — in CLAUDE.mds, in skill
docs, in `theory/` — the term is **saguão**. When discussing one
component, use its specific name.

---

## IV. Hostname pattern (locked, fleet-wide)

### IV.1. The four-part rule

**Every reachable thing in the fleet uses a hostname of the form:**

```
<role>.<scope>.quero.cloud
```

…where `(role, scope)` decomposes as follows:

| Pattern | Use |
|---|---|
| `<app>.<cluster>.<location>.quero.cloud` | A workload service on a specific cluster (e.g., `vault.rio.bristol.quero.cloud`, `photos.mar.parnamirim.quero.cloud`) |
| `<cluster>.<location>.quero.cloud` | The varanda cluster-view portal (e.g., `rio.bristol.quero.cloud`) |
| `<location>.quero.cloud` | The varanda location-view portal (e.g., `bristol.quero.cloud`) |
| `quero.cloud` | The varanda fleet-view portal — the front door |
| `<control-role>.quero.cloud` | Reserved for control-plane services (see IV.2) |

**This rule is non-negotiable.** Old 3-part hostnames
(`vault.quero.cloud`, `photos.quero.cloud`, etc.) are migrated to
4-part during Phase 1 (§VII.3) — there is no permanent exception.

### IV.2. Reserved control-plane hostnames

The fleet-edge control plane uses dedicated 2-part hostnames at
the apex zone. These are reserved fleet-wide and cannot be reused
by any cluster:

| Hostname | Component | Notes |
|---|---|---|
| `auth.quero.cloud` | passaporte (Authentik) | The fleet IdP. Cloudflare-tunneled to whichever cluster currently hosts passaporte (today: rio). |
| `cracha.quero.cloud` | crachá API | The fleet authz API. Same hosting as above. |
| `webhook.quero.cloud` | reserved | Future control-plane webhooks (e.g., Authentik notifications). |
| `blog.quero.cloud` | zuihitsu | Pre-saguão; grandfathered. Public; no auth. |
| `webhook.blog.quero.cloud` | zuihitsu webhook | Pre-saguão; grandfathered. HMAC-gated. |

**Why bare 2-part for control-plane.** These are *fleet*
primitives, not cluster primitives — they don't belong in any
single cluster's namespace. The hostname abstracts the physical
hosting cluster so it can be relocated without an API change.

### IV.3. Why four parts (not three)

The 4-part rule encodes the actual layering of concerns:

- `<app>` says *what* (vault, photos, jellyfin).
- `<cluster>` says *which instance* (rio's vault is not mar's vault).
- `<location>` says *where* (bristol vs parnamirim is a meaningful
  geographic distinction — different ISPs, different blast radius,
  different physical access).
- `quero.cloud` says *whose* (the family namespace).

When the fleet has 5 clusters across 3 locations, every subdomain
is unambiguous and every URL is a permalink. Bookmarking
`vault.rio.bristol.quero.cloud` means "vault on the rio cluster in
Bristol" forever, even after `mar.parnamirim` gets its own vault.

### IV.4. Apex is the fleet view

The user types `quero.cloud` into a browser. They are bounced
through `auth.quero.cloud` for sign-in (one time, then session
cookie covers everything). They land back at `quero.cloud` —
varanda's fleet-view mode. They see:

```
Welcome, [first name]

Bristol, TN, USA           Parnamirim, RN, Brazil      Natal, RN, Brazil
─ rio                      ─ mar                       ─ rai
   • Vault                    • Vault                     • Media archive
   • Photos                   • Family photos             • Dev sandbox
   • Jellyfin                 • Home Assistant
   • Home Assistant           • Notes
   …
```

Each tile is a deep link to the actual service. Cluster-view
(`rio.bristol.quero.cloud`) shows just rio's tiles; location-view
(`bristol.quero.cloud`) shows all clusters at Bristol; fleet-view
(`quero.cloud`) shows everything the user can see.

The fleet-view is the canonical "one interface" — every other URL
shape exists for direct access and bookmarking.

---

## V. Cluster onboarding flow

### V.1. The single typed declaration

When cluster N comes online, the entirety of the saguão wiring is
expressed as **one Lisp form** authored anywhere convenient
(canonically: `pleme-io/cracha/examples/<name>.lisp`):

```clojure
(defcluster
  :name     "mar"
  :location "parnamirim"
  :country  "RN, Brazil"
  :role     consumer        ; consumer | control-plane | hybrid
  :saguao   (:vigia      #t
             :varanda    #t
             :passaporte #f       ; #f = consume fleet's passaporte
             :cracha     #f))     ; #f = consume fleet's crachá
```

`:role consumer` means this cluster runs `vigia` (data plane) +
`varanda` (its cluster-view UI), and consumes the fleet's
passaporte + crachá. `:role control-plane` means this cluster
*hosts* passaporte + crachá in addition to its own vigia/varanda
(today: rio). `:role hybrid` means this cluster hosts a fallback
passaporte/crachá replica (future, for HA — see §VI.3).

**The renderer (`cracha render cluster <file.lisp> --out dir/`)
emits 4 artifacts from this single declaration:**

| File | Destination repo |
|---|---|
| `nix-fleet-domains-<name>.nix.fragment` | `pleme-io/nix/lib/pleme-fleet.nix` (locations map) |
| `vigia-<name>-helmrelease.yaml` | `pleme-io/k8s/clusters/<name>/infrastructure/vigia/release.yaml` |
| `pangea-cloudflare-pleme-<name>-additions.yaml` | `pleme-io/pangea-architectures/workspaces/cloudflare-pleme/domains/quero.cloud.yaml` |
| `cracha-cluster-registration-<name>.yaml` | ServiceCatalog snippet for the control-plane cluster |

The operator places each file (or — preferably — `feira cluster
apply` does the multi-repo write, planned). Adding cluster N is
**one Lisp edit + one render command + four `cp` operations**, no
hand-authored YAML across 4 repos.

### V.1.5. Fleet-level declaration

For "what's in the whole fleet?", the umbrella `(deffleet …)` form
composes every cluster + names the control-plane endpoints:

```clojure
(deffleet
  :name "pleme"
  :tld  "quero.cloud"
  :passaporte (:host "auth.quero.cloud"   :on-cluster "rio")
  :cracha     (:host "cracha.quero.cloud" :on-cluster "rio")
  :clusters
  ((:name "rio" :location "bristol" :role control-plane
    :saguao (:vigia #t :varanda #t :passaporte #t :cracha #t))
   (:name "mar" :location "parnamirim" :role consumer
    :saguao (:vigia #t :varanda #t :passaporte #f :cracha #f))))
```

`cracha render fleet examples/pleme.lisp --out out/` invokes the
per-cluster renderer for every cluster + emits a `fleet-summary.md`
naming the control-plane endpoints + cluster topology.

### V.1.6. ServiceCatalog auto-derivation

Per §III.2, ServiceCatalog used to require an explicit CRD entry
for every service. **As of cracha-controller v0.1**, the
controller watches `helm.toolkit.fluxcd.io/v2/HelmRelease` resources
labeled `app.kubernetes.io/part-of=saguao-service` and
auto-derives ServiceCatalog entries from them. The contract:

```yaml
# In every lareira-<service> HelmRelease metadata:
metadata:
  labels:
    app.kubernetes.io/part-of: saguao-service     # required
    app.kubernetes.io/name: vault                 # required → service slug
    pleme.io/cluster: rio                         # required → cluster name
  annotations:
    saguao.pleme.io/display-name: "Vaultwarden"   # required
    saguao.pleme.io/icon: "https://..."           # optional
    saguao.pleme.io/description: "Password manager" # optional
```

Adding a service is now **one HelmRelease**; no separate
ServiceCatalog edit. Explicit `ServiceCatalog` CRD entries still
work and win on `(slug, cluster)` collision (escape hatch for
services not deployed via lareira).

### V.2. What propagates from one declaration

Adding the form above propagates to:

| Artifact | Where it's rendered | What changes |
|---|---|---|
| DNS records | `pangea-architectures/workspaces/cloudflare-pleme/domains/quero.cloud.yaml` | New `parnamirim.quero.cloud`, `mar.parnamirim.quero.cloud`, and one tunnel CNAME per service |
| Cluster registration | `pleme-io/nix/lib/pleme-fleet.nix` `locations` map | `mar = "parnamirim";` |
| Vigia HelmRelease | `k8s/clusters/mar/infrastructure/vigia/release.yaml` | `lareira-vigia` chart, `passaporte_url=https://auth.quero.cloud`, `cracha_url=grpc://cracha.quero.cloud`, `cluster_name=mar`, `location=parnamirim` |
| Varanda Cloudflare Pages project | `pangea-architectures/workspaces/cloudflare-pleme/` | New Pages project for `mar.parnamirim.quero.cloud` (or wildcard custom-domain on the existing varanda project) |
| Crachá cluster registration | `cracha-cluster-registry` ConfigMap | `mar` added to known-cluster list so policies can grant access to it |

The declaration is rendered by tatara — none of the artifacts above
are hand-edited.

### V.3. Adding a new service to an existing cluster

After the cluster exists, adding a service is a one-entry change in
the cluster's `programs.yaml` (the `feira deploy` workflow per
[`CAIXA-SDLC.md`](./CAIXA-SDLC.md)) plus an `AccessPolicy`
amendment for who can see it. The lareira values pattern
(`compliance.authn.oidc.provider: vigia`) is set fleet-wide in
pleme-lib defaults — the application chart inherits it without
opting in per-app.

The DNS record for the new app's hostname
(`<app>.<cluster>.<location>.quero.cloud`) and the cloudflared
tunnel ingress are created by the same pangea apply that processes
the lareira chart's `cloudflared.expose: true`.

### V.4. Adding a new location

A new location (e.g., `tokyo` if a Japan cluster is ever added) is:

1. A new entry in `pangea-architectures/workspaces/<new-location>/`
   with location metadata (country, timezone, primary contact).
2. A new entry in the saguão-known-locations registry (a
   ConfigMap consumed by crachá so policies can grant access to
   the location).
3. The first cluster in that location is added per V.1.

Locations are passive: they don't run anything. They are a typed
container for the clusters within them, and a layer in the URL
hierarchy.

### V.5. Adding a new family member

The lifecycle is:

1. Operator (or a future admin family member) edits a
   `(defcrachá :grants […])` form to add the new member, citing
   their email and the locations/clusters/services they should see.
2. The crachá-controller observes the CRD change and updates its
   in-memory policy cache.
3. The new member visits `quero.cloud`, signs in with Google
   (passaporte → Authentik → Google OIDC). Authentik creates a
   user record on first sign-in (provisioning is automatic).
4. Varanda renders only the tiles the new member is allowed to
   see, per crachá.
5. When the new member clicks a tile, vigia checks the request
   against crachá and lets it through.

No emails, no OTP setup, no Cloudflare Access policy edit. The
operator's edit is to the typed Lisp form; everything else is
mechanical.

---

## VI. Identity placement and HA

### VI.1. Where does passaporte physically run

The control-plane components (passaporte + crachá) need a home
cluster. The hostname `auth.quero.cloud` is abstract — Cloudflare
Tunnel routes it to whichever cluster currently hosts the
deployment. The choice of cluster is operational, not
architectural.

Three options were considered:

#### Option 1 — On the primary home-edge cluster (rio today)

passaporte runs as a HelmRelease on rio. Cloudflare Tunnel from
rio's `cloudflared` advertises `auth.quero.cloud`.

- **+** Cheapest. Zero new infra.
- **+** Reuses rio's existing observability, backups, alerting.
- **−** rio is the IdP for the entire fleet. If rio is down or
  network-partitioned from the home network, every cluster
  everywhere loses *new* sign-ins. Existing sessions (24h–168h)
  keep working via vigia's JWT cache, but new users can't sign in
  and existing users can't refresh past their session limit.
- **−** Couples cluster lifecycle to fleet identity lifecycle:
  rio reboots → control plane blip.

**Use when:** fleet has 1–2 clusters and the operator runs both.
The blast radius of "rio is offline" is identical to "the operator
is dealing with a rio outage anyway."

#### Option 2 — Tiny dedicated control-plane cluster

A single small VM (e.g., Hetzner CX22 at €5/mo, or AWS t4g.micro
under free-tier-equivalent) runs only passaporte + crachá. No
workloads. Independent from any home cluster.

- **+** Independent failure domain. Home-edge clusters can come and
  go without affecting fleet identity.
- **+** Cheaper than HA — single small instance handles the entire
  fleet's authn load comfortably (Authentik with KEDA scale-to-zero
  uses ~50 MiB idle, ~512 MiB under burst).
- **−** Modest recurring cost (~€5/mo).
- **−** New cluster to operate (kindling lifecycle, Flux, secrets,
  backups).

**Use when:** fleet hits 3+ clusters or the operator wants the
"one cluster down ≠ fleet auth down" property.

#### Option 3 — Active-active across two clusters

Authentik with shared Postgres replicated across two clusters
(e.g., rio + mar). True HA.

- **+** No SPOF. One cluster offline = no impact.
- **−** Significantly more complex (Postgres replication, leader
  election, shared session cache).
- **−** Authentik upstream support for active-active is OK but not
  trivial.

**Use when:** the fleet is providing services to family members
who depend on it (i.e., it's not just the operator's hobby
anymore) AND the two clusters can both reliably reach the public
internet.

### VI.2. Recommended trajectory

```
Phase 1 (today):   Option 1   — passaporte on rio
                                ↓ (when fleet hits cluster #2 or #3)
Phase 2 (mid):     Option 2   — dedicated tiny control-plane cluster
                                ↓ (when family relies on it)
Phase 3 (future):  Option 3   — active-active across two clusters
```

The migration between phases is operational, not architectural —
the public hostnames (`auth.quero.cloud`, `cracha.quero.cloud`)
abstract the physical hosting. Phase 1→2 is a Helm release move +
DNS flip; Phase 2→3 is Postgres replication + a second instance.

### VI.3. Failure modes and mitigations

The "control-plane down → fleet auth down" failure mode of Phase 1
is mitigated by:

1. **Long session durations.** Defaults: 24h for sensitive apps
   (vault, paperless), 168h (1 week) for low-sensitivity (jellyfin,
   notes). Already in the existing config.
2. **JWT cache in vigia.** Forward-auth caches successful JWT
   verifications for the JWT's full lifetime (typically 1h on the
   token itself). A control-plane outage of <1h is invisible to
   active users.
3. **Decision cache in vigia.** Crachá responses are cached for a
   bounded TTL (default 5min) per `(user, service, verb)` tuple.
   A control-plane outage of <5min doesn't block active sessions.
4. **Stale-OK fallback.** vigia configurable to serve stale
   crachá decisions for up to 1h on control-plane unreachability,
   with a loud Prometheus alert. Default off; opt-in for clusters
   where local availability matters more than fresh policy.
5. **Documented manual override.** A break-glass `vigia` mode
   that bypasses crachá entirely (allow-all for the `family`
   group) — gated by an in-cluster Secret, used for
   "control-plane is down and I need vault" emergencies. Audit-
   logged via the heartbeat chain (cf. `tameshi`) on any use.

Cloudflare Tunnel itself has the same property: if the apex
provider is down, the home-edge cluster's services are
unreachable from the public internet. Mitigated by direct LAN
access (`*.rio.lan` patterns) for in-home users.

---

## VI.4. Cloudflare = transport, reconciled by pangea-operator on rio

The architecture's "Cloudflare is transport only" rule (§II.3) has a
control-plane corollary: **the Cloudflare configuration itself is
also reconciled from in-cluster, not from an operator workstation.**

Mechanism: the `pangea.pleme.io/v1alpha1.InfrastructureTemplate` CRD
shipped by [`pleme-io/pangea-operator`](https://github.com/pleme-io/pangea-operator)
runs on rio. One InfrastructureTemplate (named `cloudflare-pleme`)
points at the `cloudflare-pleme` workspace inside `pangea-architectures`
and the `quero_cloud` template within. The operator clones the repo,
synthesizes the workspace, and applies it via its embedded engine on
every git commit + every `refreshInterval`.

What this replaces: the workstation `bundle exec pangea apply
quero_cloud` workflow. After this lands on rio
(`k8s/clusters/rio/infrastructure/cloudflare-pleme/`), every
`*.quero.cloud` Cloudflare change is a git commit, a Flux sync, and
an operator reconcile — no operator-machine state required.

What materializes via this template:

- DNS records for every `pages_apps[*].custom_domains` entry (saguão front door + per-location + per-cluster portals)
- Cloudflare Pages projects + per-domain bindings (varanda + zuihitsu)
- Cloudflare Tunnel ingress for the cluster's gated services
- Zone settings (TLS 1.2+, always-HTTPS, browser check, DNSSEC)

The varanda Pages project's `custom_domains` list is THE source of
truth for "what hostnames load varanda" — the four-part hostname
rule (§IV.1) is enforced operationally by what's listed there. To
add the `mar.parnamirim.quero.cloud` cluster portal once mar comes
online: append `mar.parnamirim` to that list, commit, the operator
reconciles, the new portal hostname loads varanda automatically.

Bootstrap + operational guide:
[`pleme-io/k8s/clusters/rio/infrastructure/cloudflare-pleme/README.md`](https://github.com/pleme-io/k8s/blob/main/clusters/rio/infrastructure/cloudflare-pleme/README.md).

The `pages_apps:` field on the `CloudflareDomain` architecture is
the typed extension that supports this — every static SPA gets one
entry; the architecture stamps `cloudflare_pages_project` +
`cloudflare_pages_domain` + `cloudflare_dns_record` per
custom domain. Apex (`@`) is bound via Pages's apex-attachment
(no explicit CNAME, avoids NS-record collision).

---

## VII. Migration from current rio state

The current rio state is described in `clusters/rio/SECURITY.md`:
Authentik HelmRelease provisioned and *suspended*, Cloudflare
Access live with email-OTP, ~10 services at 3-part hostnames,
operator + 3 family-email placeholders in the access group.

The migration runs in seven phases. **Each phase is independently
shippable and reversible.** Phases 1–4 land the user-visible
behavior; phases 5–7 land the typed pleme-io-native primitives.

### VII.0. Pre-flight

- Audit the current consumer set. Are family members actively
  using vault/photos today? If yes, Phase 3 (rename) needs a
  redirect Worker so existing bookmarks keep working. If no
  (operator-only), the rename can be a hard cut.
- Confirm Authentik admin Secret is provisioned (SOPS-encrypted in
  `nix/secrets.yaml` at `authentik.secret_key`,
  `authentik.postgresql.password`, `authentik.bootstrap_password`,
  `authentik.bootstrap_token`).

### VII.1. Phase 1 — unsuspend Authentik, federate Google

1. Patch `lareira-authentik` HelmRelease: `suspend: false`.
2. Wait for KEDA HTTP scale-to-zero scaffold + outpost to come up
   at `auth.quero.cloud`.
3. In Authentik admin UI: add Google as a federated social
   identity provider (one-time, ~5 min — OAuth client ID/secret
   from Google Cloud Console).
4. Verify: navigate to `auth.quero.cloud`, click "Sign in with
   Google," confirm a user record is created on first sign-in.

This phase changes nothing for users; it brings the in-cluster
IdP online with Google federation.

### VII.2. Phase 2 — flip charts to Authentik forward-auth

For each chart with `cloudflared.expose: true`:

1. Set `compliance.authn.oidc.provider: authentik` (already the
   default in pleme-lib for fedramp-moderate+; explicit in chart
   values for the few apps that opt out today).
2. The chart's Ingress now stamps `auth-url` /
   `auth-signin` annotations pointing at the Authentik outpost.
3. Visiting any gated app now first redirects to
   `auth.quero.cloud` for sign-in, then back to the app.

At this point, **two SSO layers are live in parallel** —
Cloudflare Access at the edge AND Authentik forward-auth in the
cluster. Users will see Cloudflare Access's email-OTP first, then
Authentik's redirect. Tolerable for the duration of one phase;
fixed in Phase 3.

### VII.3. Phase 3 — drop Cloudflare Access entirely

1. Edit `pangea-architectures/workspaces/cloudflare-pleme/domains/quero.cloud.yaml`:
   - Delete the entire `zero_trust_access:` block.
2. `cd ~/code/github/pleme-io/pangea-architectures/workspaces/cloudflare-pleme && bundle exec pangea apply quero_cloud`.
3. Verify: cold-cache visit to a gated app now sees only the
   Authentik sign-in, not the Cloudflare Access challenge.

Cloudflare's role is now exactly transport: DNS + Tunnel + edge
TLS. **Cloudflare Access is gone from the architecture.**

### VII.4. Phase 4 — rename to 4-part hostnames

For each chart with `cloudflared.expose: true`:

1. Update `cloudflared.hostname` from `<app>.quero.cloud` to
   `<app>.rio.bristol.quero.cloud`.
2. The pangea apply that processes the lareira chart values
   creates the new tunnel ingress and DNS record.
3. **Bookmark redirect Worker** (optional, but recommended):
   deploy a tiny Cloudflare Worker at the old `<app>.quero.cloud`
   host that 301-redirects to the new 4-part hostname. Burns no
   compute past the redirect. Retire after ~30 days.

After this phase, the URL shape matches the long-term canonical
pattern. New clusters can claim their own `<app>.<cluster>.<loc>.quero.cloud`
without collision.

### VII.5. Phase 5 — crachá CRD + controller

1. Scaffold `pleme-io/cracha` (Cargo workspace: `cracha-core`,
   `cracha-controller`, `cracha-api`).
2. `cracha-core`: `AccessPolicy` types with
   `#[derive(TataraDomain)]`. `(defcrachá …)` form authoring.
3. `cracha-controller`: kube-rs reconciler watching
   `AccessPolicy` CRDs, building an in-memory authz index.
4. `cracha-api`: gRPC server (`POST /authorize` returns
   allow/deny) + REST server (`GET /accessible-services?user=…`
   returns the user's portal manifest).
5. Helm chart `lareira-cracha` deploys to the control-plane
   cluster (today: rio).
6. Hostname: `cracha.quero.cloud` via cloudflared tunnel,
   protected by the same pleme-lib OIDC helper (so even crachá's
   own API requires Authentik sign-in for admin access).
7. Author the initial `family` AccessPolicy as a Lisp form;
   render to YAML; commit to `k8s/clusters/rio/access-policies/`.

After this phase, authz is typed and version-controlled. Authentik's
internal policies are reduced to "is this user in the
`saguao-users` group, yes/no" — the *real* decision is delegated
to crachá via Authentik's webhook policy hook.

### VII.6. Phase 6 — varanda PWA

1. Scaffold `pleme-io/varanda` (Yew + wasm-pack via
   `substrate/lib/wasm-build` helpers).
2. Three view modes (fleet, location, cluster) keyed off the
   request hostname.
3. Reads passaporte JWT from cookie; calls
   `cracha.quero.cloud/accessible-services?user=<sub>` for the
   tile manifest; renders.
4. Deploy to Cloudflare Pages with wildcard custom domains for
   `quero.cloud`, `<location>.quero.cloud`, `<cluster>.<location>.quero.cloud`.
5. Visiting any of those hostnames now lands on varanda; clicking
   a tile deep-links to the gated service.

After this phase, the user-facing front door is live.
`quero.cloud` is "the one interface" the user wanted.

### VII.7. Phase 7 — vigia (optional, when justified)

1. Scaffold `pleme-io/vigia` (Rust axum binary).
2. Implements OIDC token verification (against passaporte's JWKS)
   + crachá gRPC client + decision cache.
3. Helm chart `lareira-vigia` deploys to every cluster.
4. `pleme-lib.compliance.authn.oidc` template grows a
   `provider: vigia` branch that emits the right `auth-url`
   pointing at vigia instead of Authentik outpost.
5. Per-cluster opt-in via values file; flip clusters one at a
   time with full rollback capability.

When vigia is opt-in cluster-by-cluster, Phase 7 can be skipped
indefinitely — the Authentik outpost is a perfectly good
forward-auth implementation. Phase 7 exists for the
"customizable, in Rust" property when that matters more than
"already shipped."

---

## VIII. Anti-patterns

These are the shapes saguão **forbids**. If a PR contains one,
revert and rework.

| Anti-pattern | Why forbidden | Right shape |
|---|---|---|
| **Per-cluster IdP** | Identity drift; users sign in N times across the fleet; group memberships diverge | One passaporte at `auth.quero.cloud`, fleet-wide |
| **Per-app authz hardcoded in chart values** | Authz lives in 30 places, not version-controlled together, no cross-app view | One `AccessPolicy` CRD per group, consumed fleet-wide |
| **Cloudflare Access used as authz boundary** | Authz lives in Cloudflare's database, not in git, not visible to varanda | Cloudflare Access removed; vigia is the authz boundary |
| **Hand-maintained service catalog in varanda** | Drifts from reality; new services don't show up; old services linger | varanda reads `crachá.api/accessible-services`; the catalog *is* the authz state |
| **Auth bypass for "internal" services** | The "internal" service eventually needs external access; bypass = security gap | Every Ingress goes through vigia; "internal" means LAN-only routing, not auth-skipped |
| **Bookmark URLs that hardcode the IdP location** | Couples consumers to physical cluster placement; breaks Phase 1→2 migration | Always use `auth.quero.cloud`; the hostname abstracts the physical host |
| **3-part hostnames for new services** | Hostname collision when a second cluster wants the same app | 4-part `<app>.<cluster>.<location>.quero.cloud` is the only allowed shape |
| **Cloudflare Workers for saguão logic** | Splits the authz model across two trust boundaries; complicates audit | All saguão logic runs in-cluster (vigia, crachá controller); Cloudflare is transport |
| **Per-resource ACL in crachá** | Conflates fleet-portal authz with application-internal authz | crachá owns `(user, service, verb)`; per-resource ACL lives inside the service |
| **Skipping vigia for "trusted" namespaces** | Trust is a property of a request, not a namespace | Every Ingress with `cloudflared.expose: true` must have `compliance.authn.oidc.provider` set |

---

## IX. FAQ

### Why not just keep using Cloudflare Access?

Cloudflare Access works for "1 cluster, ~10 apps, single-dimension
groups." Saguão is for "N clusters in M locations, per-user
per-cluster per-service per-verb authz, source-of-truth in git,
typed Rust on both ends." The Cloudflare Access policy authoring
surface (web UI + API, JSON-shaped) cannot express the second
shape; even if it could, the policy would live in Cloudflare's
database, not in git, not readable by varanda.

Cloudflare Access also assumes a particular trust model — the
edge is the policy decision point, the cluster is the policy
enforcement point. Saguão places both in the cluster, with the
edge as transport only. This makes the cluster authoritative for
its own traffic, simplifies audit (one decision log per cluster),
and removes a third-party dependency from the critical path of
authz.

### Why not just use Authentik's admin UI for policy?

Same reason: Authentik's policy DSL is Python expressions
evaluated inside its Postgres-backed admin UI. The source of
truth is a database row. crachá makes the source of truth a typed
Lisp form in git, version-controlled, reviewable in PRs,
consumable by varanda and audit tools and any future pleme-io
component that needs to know "what can this user do."

Authentik stays — but as a dumb token signer. The interesting
authz decision moves to crachá.

### What about service mesh authz (Cilium / Istio)?

Different layer. Service mesh authz gates *pod-to-pod* traffic
inside the cluster (workload identity → workload identity).
Saguão gates *user-to-service* traffic at the cluster ingress
(human identity → application). Both can compose:

- **Today:** saguão alone (ingress-level user authz). Pod-to-pod
  is open within a namespace.
- **Future:** saguão at ingress + Cilium AuthorizationPolicy at
  pod-to-pod. The two trust models are orthogonal — adding mesh
  authz later doesn't conflict with saguão.

`caixa-mesh` (per `MESH-COMPOSITION.md`) is the typed substrate
for the mesh layer; its `:contratos` slot can later carry
identity-based pod-to-pod policies. Saguão owns the human-facing
edge; caixa-mesh owns the workload-facing fabric.

### How does saguão interact with caixa Aplicacao mesh?

Cleanly. An `Aplicacao` is a typed graph of Servicos with WIT
contracts and Cilium NetworkPolicies between them. Saguão is the
external authz boundary in front of the Aplicacao's `:entrada`
(typed gateway). The Aplicacao's `:entrada` declares its hostname
(`<app>.<cluster>.<loc>.quero.cloud`); the lareira chart that
renders the Ingress for that hostname automatically gets vigia
forward-auth via the pleme-lib helper.

### What if I want a public app (no auth)?

Two ways:

1. **Per-application opt-out.** Set
   `compliance.authn.allowList.unauthenticatedIngress: true` in
   the chart values. The pleme-lib validation allows this for
   genuine public-facing pages (marketing site, public booking
   page). Audit-flagged so you can't do this accidentally.
2. **Per-path opt-out.** vigia (Phase 7+) supports a
   `:public-paths` config — e.g., the Cal.com public booking page
   at `/booking/*` is unauthenticated, but `/admin/*` is gated.
   Authentik outpost in Phase 1 supports the same via
   `nginx.ingress.kubernetes.io/auth-snippet` configuration.

### What about mobile app SSO?

Mobile apps that consume saguão services follow the standard
OIDC mobile flow — PKCE-enabled OAuth2 against
`auth.quero.cloud/application/o/<client-id>/`. Authentik (and
later vigia) issues short-lived access tokens; the mobile app
caches them. Family members use the same Google sign-in.

### What about `*.rio.lan` (LAN-side, no internet)?

The LAN-side hostname pattern (`*.rio.lan` for direct in-home
access without going through Cloudflare Tunnel) continues to
work. The same `pleme-lib.compliance.authn.oidc` helper stamps
the same forward-auth annotations on the same Ingress —
auth-url points at `auth.quero.cloud` regardless of which
hostname the request came in on. LAN users still sign in via
passaporte; vigia still consults crachá. The transport is
different; the authz is identical.

(Edge case: if `auth.quero.cloud` is unreachable from the LAN
during a WAN outage, in-home users can't sign in. Mitigated by
running a passaporte stub in-home, or — more pragmatically — by
the JWT cache in vigia keeping existing sessions alive.)

### Can crachá grant access to a *future* cluster (one not yet running)?

Yes. The `:locations` and `:clusters` slots are typed lists; the
controller validates that referenced names exist in the
cluster registry, but warns rather than rejects on unknowns. This
allows declaring "wife will have access to mar's photos" before
mar exists; the grant becomes effective the moment mar
registers.

### What if I want to revoke a family member?

Edit the `(defcrachá …)` form, remove their email from the
`:members` list, commit + apply. The crachá-controller observes
the change and updates the in-memory policy index immediately.
vigia's decision cache (default 5min TTL) flushes the user's
existing entries on a control-plane signal; existing JWT-cached
sessions expire at JWT TTL (typically 1h).

For immediate revocation: the `cracha-cli revoke <email>` tool
invalidates the user's Authentik sessions and broadcasts a flush
to all vigia instances.

---

## X. Where this fits in the larger theory

Saguão is a **Tier-1 fleet primitive** — alongside `pangea-jit-builders`
(JIT capacity), `caixa-mesh` (per-cluster Aplicacao mesh), and
`tameshi` (attestation chain). Each occupies one corner of the
fleet-operations space:

| Primitive | Concern | Lives at |
|---|---|---|
| `pangea-jit-builders` | on-demand cloud capacity | per-client (operator nodes, CI runners) |
| `caixa-mesh` | per-cluster Aplicacao composition | per-cluster |
| `tameshi` | deterministic integrity attestation | per-artifact |
| **`saguão`** | **fleet-wide identity, authz, portal** | **fleet edge + per-cluster data plane** |

Theory cross-references:

- [`META-FRAMEWORK.md`](./META-FRAMEWORK.md) §"layered compute" —
  saguão sits at the *operator surface* layer (above ComputeUnit,
  above Aplicacao, below the AI agent layer).
- [`MESH-COMPOSITION.md`](./MESH-COMPOSITION.md) §"per-Aplicacao
  entrada" — saguão's vigia is what gates the Aplicacao's typed
  gateway from external (human) traffic.
- [`THEORY.md`](./THEORY.md) §VII (Operation) — saguão is the
  fleet-portal counterpart to FluxCD's per-cluster reconciler;
  one is "git → cluster," the other is "human → service."
- [`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md)
  §"compounding" — saguão is the single typed primitive that turns
  every future cluster's authz into a one-line declaration. It
  *is* the compounding directive applied to the fleet-portal
  problem.
- [`VOCABULARY.md`](./VOCABULARY.md) — the four primitive names
  and supporting terms are added under "Operation" and "Naming"
  sections.

---

## XI. Implementation roadmap snapshot

Status as of the doc's writing (2026-04-28). The phases match §VII.

| Phase | Component | Repo | Status |
|---|---|---|---|
| 1 | passaporte (Authentik unsuspend + Google federation) | `pleme-io/k8s/clusters/rio/infrastructure/authentik/` | HelmRelease provisioned, suspended; `lareira-authentik` chart in `helmworks/charts/` ready |
| 2 | Forward-auth on every chart | `pleme-io/helmworks/charts/pleme-lib/` | Helper template `pleme-lib.compliance.authn.oidc` exists; needs per-chart `compliance.authn.oidc.provider: authentik` adoption |
| 3 | Drop Cloudflare Access | `pleme-io/pangea-architectures/workspaces/cloudflare-pleme/` | YAML edit + pangea apply; one-line change |
| 4 | Rename to 4-part hostnames | rio's HelmReleases + `quero.cloud.yaml` | Mechanical; bookmark-redirect Worker recommended if family members are active |
| 5 | crachá | `pleme-io/cracha` (does not exist yet) | New repo via repo-forge `rust-workspace-release` archetype |
| 6 | varanda | `pleme-io/varanda` (does not exist yet) | New repo via repo-forge `rust-wasm` archetype |
| 7 | vigia (replace Authentik outpost) | `pleme-io/vigia` (does not exist yet) | New repo via repo-forge `rust-tool-release` archetype; opt-in per cluster |

Phases 1–4 are operational changes against existing primitives;
they ship the user-visible behavior without writing new Rust.
Phases 5–7 build the typed pleme-io-native primitives that make
saguão a compounding fleet asset.

The substrate flake builders for each new repo:

- crachá: `substrate/lib/rust-workspace-release-flake.nix` for the
  CLI; `substrate/lib/rust-service-flake.nix` for the controller;
  Helm chart in `helmworks/charts/lareira-cracha/`.
- varanda: substrate `wasm-build` helpers; Cloudflare Pages deploy
  via `cloudflare-headless-blog` skill (modified for SPA bundle
  instead of static site).
- vigia: `substrate/lib/rust-service-flake.nix`; Helm chart in
  `helmworks/charts/lareira-vigia/`.

---

## XII. Glossary (saguão-local terms)

These terms are also added to [`VOCABULARY.md`](./VOCABULARY.md)
for fleet-wide use.

| Term | One-line definition |
|---|---|
| **saguão** | The umbrella name for the fleet-wide identity + authz + portal system. ASCII fold: `saguao`. |
| **passaporte** | The typed identity primitive — typed Rust wrapper over Authentik, federated to Google. Lives at `auth.quero.cloud`. |
| **crachá** | The typed authorization primitive — `AccessPolicy` CRD authored as `(defcrachá …)`. Lives at `cracha.quero.cloud`. ASCII fold: `cracha`. |
| **vigia** | The per-cluster forward-auth data plane. Rust binary; one instance per cluster. Phase 1 role filled by Authentik embedded outpost. |
| **varanda** | The family-facing PWA. Yew + Cloudflare Pages. Renders the user's portal at `quero.cloud` (fleet view) and per-location/per-cluster subdomains. |
| **fleet edge** | The set of services hosted at apex `quero.cloud` and at `<control-role>.quero.cloud` — the control plane of saguão. |
| **cluster data plane (saguão sense)** | The vigia instance running on a cluster; the in-cluster enforcement point for human-to-service authz. |
| **(defcrachá …)** | Authoring form for an AccessPolicy. Renders to a typed `AccessPolicy` CRD. |
| **(defpassaporte …)** | Authoring form for the fleet IdP config. Renders to `lareira-passaporte` Helm values. |
| **(defvigia …)** | Authoring form for a per-cluster vigia HelmRelease. Renders to `lareira-vigia` values. |
| **(defvaranda …)** | Authoring form for the varanda PWA deploy spec. Renders to a Cloudflare Pages project + wildcard custom domain. |
| **fleet IdP** | The single OIDC provider serving all of saguão. Today: Authentik wrapped by passaporte. |
| **fleet authz API** | The single source of truth for "who can see what." Today: crachá's gRPC + REST API at `cracha.quero.cloud`. |
| **bookmark redirect Worker** | Optional Cloudflare Worker deployed during Phase 3 migration to 301-redirect old 3-part hostnames to new 4-part hostnames. Retired after ~30 days. |
| **break-glass mode** | A vigia configuration that bypasses crachá entirely for the `family` group, used for control-plane outages. Audit-logged via tameshi heartbeat chain. |
