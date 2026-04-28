# Saguão migration — staged diffs (NOT applied)

> **Companion to** [`SAGUAO.md`](./SAGUAO.md). This doc holds the
> concrete file-level diffs for the seven-phase migration of the
> rio cluster from "Cloudflare Access + suspended Authentik" to
> the saguão architecture. **Nothing in this doc has been applied.**
> Operator (or a follow-up agent) executes phases one at a time,
> verifying each before proceeding.

The phases match [`SAGUAO.md` §VII](./SAGUAO.md). Each section
gives the exact files to touch, the diff to apply, and the
verification command.

---

## Phase 0 — Pre-flight

**Goal:** confirm prerequisites before touching anything live.

### Checks

```bash
# Authentik admin Secret is provisioned in SOPS
sops -d ~/code/github/pleme-io/nix/secrets.yaml \
  | grep -E "secret_key|postgresql_password|bootstrap_password|bootstrap_token"
# Should return 4 lines.

# Cloudflare API token + account ID accessible
sops -d ~/code/github/pleme-io/nix/secrets.yaml \
  | grep "cloudflare.api-token"

# Family member usage audit
echo "Are family members ACTIVELY using vault.quero.cloud / photos.quero.cloud / etc.?"
echo "If yes: the Phase 4 rename needs the bookmark redirect Worker."
echo "If no (operator-only): Phase 4 can hard-cut without the Worker."
```

---

## Phase 1 — Unsuspend Authentik + federate Google

**Goal:** bring the in-cluster IdP online; add Google sign-in.

### Diff: `k8s/clusters/rio/infrastructure/authentik/release.yaml`

```diff
 spec:
   interval: 30m
-  suspend: true
+  suspend: false
   chart:
```

### Apply

```bash
cd ~/code/github/pleme-io/k8s
git add clusters/rio/infrastructure/authentik/release.yaml
git commit -m "saguao Phase 1: unsuspend authentik (lareira-passaporte)"
git push
# Flux reconciles within 30m, or force:
flux reconcile kustomization clusters-rio --with-source
```

### One-time admin UI step (Google federation)

1. Wait for `auth.quero.cloud` to be reachable (`curl -I https://auth.quero.cloud/` → 302).
2. Sign in with the bootstrap admin (credentials in SOPS at
   `authentik.bootstrap_password`).
3. Directory → Federation & Social login → Create → Google. Paste
   the OAuth client ID/secret from Google Cloud Console.
4. Flows & Stages → default-source-enrollment-flow → bind the new
   Google source so first-time sign-ins auto-create user records.

### Verify

```bash
# Authentik server pods up
kubectl --context=rio -n authentik get pods

# auth.quero.cloud responds
curl -sI https://auth.quero.cloud/ | head -1

# Sign in with Google in incognito; user record auto-created
```

---

## Phase 2 — Flip charts to Authentik forward-auth

**Goal:** every gated app's Ingress goes through the Authentik
outpost via `pleme-lib.compliance.authn.oidc`.

### Diff per chart (example: vault)

`k8s/clusters/rio/apps/vault/release.yaml`:

```diff
 spec:
   values:
     cloudflared:
       expose: true
       hostname: vault.quero.cloud
+    compliance:
+      authn:
+        oidc:
+          provider: authentik
+          group: family
```

Repeat for: `vault, photos, jellyfin, ha, paperless, notes, book,
newsletter, git, drive` and any other gated lareira HelmRelease.

### Apply

```bash
cd ~/code/github/pleme-io/k8s
git add clusters/rio/apps/*/release.yaml
git commit -m "saguao Phase 2: enable Authentik forward-auth on every gated chart"
git push
```

### Verify

```bash
# Every Ingress now stamps auth-url
for app in vault photos jellyfin ha paperless notes book newsletter git drive; do
  kubectl --context=rio -n $app get ingress -o jsonpath='{.items[*].metadata.annotations.nginx\.ingress\.kubernetes\.io/auth-url}{"\n"}'
done
# Each should print a non-empty auth-url ending in /outpost.goauthentik.io/auth/nginx
```

**Note:** Cloudflare Access is still live at this point.
Visiting any gated app shows BOTH challenges (Cloudflare Access
first, then Authentik). Tolerable for the duration of one phase;
fixed in Phase 3.

---

## Phase 3 — Drop Cloudflare Access

**Goal:** Cloudflare reduced to transparent transport. SSO is
exclusively in-cluster.

### Diff: `pangea-architectures/workspaces/cloudflare-pleme/domains/quero.cloud.yaml`

```diff
-# Cloudflare Zero Trust Access — edge SSO for every quero.cloud
-# subdomain that goes through Cloudflare Tunnel. Default IdP is the
-# built-in email OTP (no IdP setup needed for day one). Authentik on
-# rio gets federated in once the in-cluster instance is running by
-# uncommenting the identity_providers block + adding `allowed_idps:
-# [authentik]` to applications that should require it.
-zero_trust_access:
-  # identity_providers:
-  #   - name: authentik
-  #     ...
-  groups:
-    - name: family
-      include:
-        - email_domain: quero.cloud
-        - email: cousin@example.com
-        - email: wife@example.com
-        - email: drzln@protonmail.com
-  applications:
-    - slug: vault
-      ...
-    - slug: photos
-      ...
-    [...all other applications...]
```

(Delete the entire `zero_trust_access:` block — ~120 lines.)

### Apply

```bash
cd ~/code/github/pleme-io/pangea-architectures/workspaces/cloudflare-pleme
export CLOUDFLARE_API_TOKEN=$(sops -d --extract '["cloudflare"]["api-token"]' ../../../nix/secrets.yaml)
export CLOUDFLARE_ACCOUNT_ID=97d01f39d2967f21320f41bf71249ed1

bundle exec pangea synth quero_cloud
# Review the diff: should DELETE every cloudflare_zero_trust_access_application
# and the cloudflare_zero_trust_access_group resources. NO additions.
bundle exec pangea apply quero_cloud
```

### Verify

```bash
# Visit gated app in incognito — only Authentik challenge appears, no Cloudflare Access
curl -sI https://vault.quero.cloud/ | head -5
# Should redirect to https://auth.quero.cloud/outpost.goauthentik.io/start?... (Authentik)
# NOT to https://<account>.cloudflareaccess.com/cdn-cgi/access/login/... (Cloudflare Access)
```

**Reversible:** restore the deleted `zero_trust_access:` block from
git history and `bundle exec pangea apply quero_cloud` again.

---

## Phase 4 — Rename to 4-part hostnames

**Goal:** every workload service uses `<app>.<cluster>.<location>.quero.cloud`.

### Diff per chart (example: vault)

`k8s/clusters/rio/apps/vault/release.yaml`:

```diff
 spec:
   values:
     cloudflared:
       expose: true
-      hostname: vault.quero.cloud
+      hostname: vault.rio.bristol.quero.cloud
```

Repeat for every `cloudflared.expose: true` chart.

### Optional: bookmark redirect Worker

For the transition window (~30 days), deploy a small Cloudflare
Worker that 301-redirects old 3-part hostnames to new 4-part
hostnames. Author via the `pangea-cloudflare` Workers resource;
deploy alongside the next pangea apply.

```javascript
// bookmark-redirect.js — to be wrapped by pangea-cloudflare Workers resource
export default {
  async fetch(request) {
    const url = new URL(request.url);
    const map = {
      'vault.quero.cloud':      'vault.rio.bristol.quero.cloud',
      'photos.quero.cloud':     'photos.rio.bristol.quero.cloud',
      'jellyfin.quero.cloud':   'jellyfin.rio.bristol.quero.cloud',
      'ha.quero.cloud':         'ha.rio.bristol.quero.cloud',
      'paperless.quero.cloud':  'paperless.rio.bristol.quero.cloud',
      'notes.quero.cloud':      'notes.rio.bristol.quero.cloud',
      'book.quero.cloud':       'book.rio.bristol.quero.cloud',
      'newsletter.quero.cloud': 'newsletter.rio.bristol.quero.cloud',
      'git.quero.cloud':        'git.rio.bristol.quero.cloud',
      'drive.bristol.quero.cloud': 'drive.rio.bristol.quero.cloud',
    };
    const newHost = map[url.hostname];
    if (newHost) {
      url.hostname = newHost;
      return Response.redirect(url.toString(), 301);
    }
    return new Response('Not found', { status: 404 });
  },
};
```

Retire the Worker after ~30 days of metrics show <1 hit/day on the
old hostnames.

### Apply

```bash
cd ~/code/github/pleme-io/k8s
git add clusters/rio/apps/*/release.yaml
git commit -m "saguao Phase 4: rename rio apps to 4-part hostnames"
git push
# Flux reconciles; Cloudflare Tunnel ingress updates via pangea apply
```

```bash
cd ~/code/github/pleme-io/pangea-architectures/workspaces/cloudflare-pleme
bundle exec pangea apply quero_cloud   # picks up new tunnel CNAMEs
```

### Verify

```bash
# Each new hostname resolves and serves the app
for app in vault photos jellyfin; do
  curl -sI "https://${app}.rio.bristol.quero.cloud/" | head -1
done
```

---

## Phase 5 — Build crachá

**Goal:** typed AccessPolicy CRD + controller + API.

Repo scaffold lives at `pleme-io/cracha/` (created locally; not
yet pushed to GitHub). Implementation work tracked in
`pleme-io/cracha/CLAUDE.md` and the saguao skill's TODO sections.

### Subphases

1. Implement `cracha-core::AccessPolicy` evaluator (scaffold has the type + 5 unit tests).
2. Add `tatara-lisp-derive` integration so `(defcrachá …)` round-trips.
3. Implement `cracha-controller` reconciler (kube-rs Controller<AccessPolicy>).
4. Implement `cracha-api` gRPC server (tonic) + REST server (axum).
5. Wire Authentik external policy webhook → cracha-api gRPC for forward-auth integration.
6. Author the canonical `family` AccessPolicy as `(defcrachá …)`; commit to `k8s/clusters/rio/access-policies/family.yaml`.
7. Deploy `lareira-cracha` HelmRelease to rio.

### Apply (when ready)

```bash
# After implementation:
cd ~/code/github/pleme-io/cracha
nix run .#release   # OCI image to ghcr.io/pleme-io/cracha:0.1.0

# Author HelmRelease in k8s repo
cat > ~/code/github/pleme-io/k8s/clusters/rio/infrastructure/cracha/release.yaml <<EOF
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: lareira-cracha
  namespace: cracha
spec:
  chart:
    spec:
      chart: lareira-cracha
      version: "0.1.x"
      sourceRef:
        kind: HelmRepository
        name: pleme-charts
        namespace: flux-system
  values:
    image:
      tag: "0.1.0"
EOF
```

---

## Phase 6 — Build varanda

**Goal:** family-facing PWA at `quero.cloud` (and per-location, per-cluster subdomains).

Repo scaffold lives at `pleme-io/varanda/`.

### Subphases

1. Implement `view::fleet`, `view::location`, `view::cluster` Yew components.
2. Implement `api::cracha_client` (gloo-net wrapper around `GET /accessible-services`).
3. Implement `session::passaporte_jwt` (cookie reader + decode).
4. Consume ishou tokens (CSS custom properties).
5. Wire Cloudflare Pages deploy via Wrangler (or via the `pangea-cloudflare` Pages resource).
6. Add wildcard custom domains: `*.quero.cloud` AND `quero.cloud` itself.

### Apply (when ready)

```bash
cd ~/code/github/pleme-io/varanda
nix build .#default      # produces dist/
wrangler pages deploy dist --project-name=varanda
```

---

## Phase 7 — vigia replaces Authentik outpost (optional)

**Goal:** typed Rust forward-auth on every cluster, replacing the
Authentik embedded outpost.

Repo scaffold lives at `pleme-io/vigia/`.

### Subphases

1. Implement vigia HTTP handlers (axum: `/auth`, `/healthz`, `/metrics`).
2. Wire OIDC JWT validation against passaporte's JWKS (kenshou).
3. Wire crachá gRPC client (tonic) with moka-cached decisions.
4. Audit-log emission to pleme-vector pipeline.
5. Helm chart `lareira-vigia`.

### Per-cluster opt-in

Once vigia is built, flip clusters one at a time:

`helmworks/charts/pleme-lib/values.yaml` gets a
`compliance.authn.oidc.providers.vigia` entry pointing at the
in-cluster vigia Service. Then per-chart:

```diff
 compliance:
   authn:
     oidc:
-      provider: authentik
+      provider: vigia
```

Roll back by flipping back. The Authentik outpost stays available
as long as `lareira-passaporte`'s `outposts.discover: true` is set.

---

## Reversibility table

| Phase | Reversible? | How |
|---|---|---|
| 1 | yes | re-suspend the HelmRelease |
| 2 | yes | remove `compliance.authn.oidc` from chart values; flux re-renders |
| 3 | yes | restore `zero_trust_access:` block from git history; pangea apply |
| 4 | yes | rename hostnames back; the bookmark Worker (if deployed) bridges either direction |
| 5 | yes | suspend `lareira-cracha`; consumers fall back to Authentik outpost's static policies |
| 6 | yes | deactivate the Cloudflare Pages project |
| 7 | yes | flip `provider: vigia` back to `provider: authentik` per chart |

No phase is destructive of state. The Authentik database survives
all phases (it's the source-of-truth for user records). Cloudflare
state is restorable from the Pangea-rendered YAML.

---

## Cross-references

- [`SAGUAO.md`](./SAGUAO.md) — canonical architecture (read first)
- [`blackmatter-pleme/skills/saguao/SKILL.md`](https://github.com/pleme-io/blackmatter-pleme/blob/main/skills/saguao/SKILL.md) — operational guide
- [`k8s/clusters/rio/SECURITY.md`](https://github.com/pleme-io/k8s/blob/main/clusters/rio/SECURITY.md) — rio's saguão integration
- [`pangea-architectures/workspaces/cloudflare-pleme/CLAUDE.md`](https://github.com/pleme-io/pangea-architectures/blob/main/workspaces/cloudflare-pleme/CLAUDE.md) — Cloudflare-as-tunnel-only rule
