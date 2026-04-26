# Tatara packaging — git+nix native, content-addressed, no package manager

> **Frame.** Bundler has a Gemfile, Cargo has a Cargo.toml, npm has
> package.json. Each maintains a registry, a resolver, a lock file, and
> a separate index of what packages exist. **The tatara ecosystem skips
> all of that.** Git is the registry. Nix flakes are the resolver. BLAKE3
> is the lock. URLs are the names. There is no package manager because
> there is no separate registry to manage.
>
> This document codifies the philosophy the user named on 2026-04-26:
> *"packaging must be fully content-addressable and like Gemfile and
> bundler and the best of cargo as well. and be very nix native. git +
> nix native, no package management — purely git-based."*

---

## I. The principle

Every tatara-lisp package is reached by a **typed URL** that names a git
revision (or content hash) directly. Resolution is git fetch +
optional BLAKE3 verification. Caching is filesystem-keyed on BLAKE3.
Reproducibility is a hash, not a manifest.

```
github:pleme-io/programs/hello-world/main.tlisp?ref=v0.1.0
   │           │         │           │         │
   │           │         │           │         └─ revision pin (tag, commit, or branch)
   │           │         │           └─────────── path within the repo
   │           │         └─────────────────────── repo
   │           └───────────────────────────────── owner / org
   └───────────────────────────────────────────── forge (github | gitlab | codeberg | https)
```

That URL **is** the package identity. Pin it (`?ref=v0.1.0`),
content-pin it (`#blake3=abc123…`), or float it (no qualifier — HEAD).
The runtime fetches once, BLAKE3-keys it in `~/.cache/tatara/sources`,
and returns the same bytes on every subsequent resolve.

## II. What we adopted from each ecosystem

### II.1 From Bundler / Gemfile

- **Source diversity by URL prefix**: just like `gem 'foo', git: '...'`
  or `gem 'foo', path: '...'`, tatara accepts `github:`, `gitlab:`,
  `codeberg:`, `https://`, and local paths uniformly. The forge isn't
  privileged — git is.
- **Lock files as content-pin records**: `tatara.lock` (planned)
  records the resolved `?ref=` → BLAKE3 mapping for every transitive
  import. Same shape as `Gemfile.lock`, content-addressed instead of
  version-pinned.
- **`(require ...)` is `require` from Ruby**: load a transitive Lisp
  file relative to current. Path-relative for local sources, URL-
  relative for remote.

### II.2 From Cargo

- **Workspace cohesion**: `tatara-lisp` is a Cargo workspace. Every
  crate (`tatara-lisp`, `tatara-lisp-eval`, `tatara-lisp-script`,
  `tatara-lisp-source`, `tatara-lisp-derive`) shares lints, version,
  edition, dependency policies. Same shape Cargo workspaces.
- **`features` for optional capability sets**: future tatara-script
  builds will gate stdlib domains behind features (`http-server`,
  `nats`, `kube`) so a minimal scripting binary doesn't pull every
  domain.
- **Build output reproducibility**: `crate2nix` materializes
  Cargo.lock into per-crate Nix derivations; `tool-image-flake`
  compiles the same hash on any machine.

### II.3 From Nix flakes

- **The URL grammar is Nix's URL grammar**: `github:owner/repo[?ref=tag]`
  reads identically. Operators learn one schema for both nix flake
  inputs and tatara-lisp imports.
- **Content-addressed caching**: `~/.cache/tatara/sources/<blake3>/data`
  parallels `/nix/store/<hash>-<name>`. Two clusters fetching the same
  URL hit the same BLAKE3 and skip duplicate work.
- **The flake is the source of truth**: every `pleme-io/<repo>/flake.nix`
  exposes the canonical entrypoints — `packages.<n>` for build outputs,
  `apps.<n>` for `nix run` targets. Tatara-script is just one such
  app; it doesn't define its own package format.

## III. What we explicitly DON'T have

| Concept other ecosystems carry | Why we don't |
|---|---|
| Package registry (npm, crates.io, RubyGems) | Git is the registry. Forge URL is the package name. |
| Package manifest (`package.json`, `gem foo`) | The `?ref=tag` qualifier on the URL is the manifest. |
| Version-resolution algorithm (npm semver, RubyGems pessimistic) | URLs pin to git refs. No resolution; the hash IS the answer. |
| Version range syntax | A consumer pins a specific tag. Want a range? Update your URL. |
| Lockfile generator | The cache layout `~/.cache/tatara/sources/<blake3>/` already records resolutions. |
| Publishing tools (`gem push`, `cargo publish`) | `git push` + `git tag`. |
| Authentication tokens for a registry | `GH_TOKEN` for private GitHub. Same auth as any git operation. |
| Nag screens about deprecated packages | The repo's README is the surface for that. |

The collective absence of this scaffolding is not a feature gap; it's
the design. **One less thing to maintain, one less thing to attack,
one less thing to learn.**

## IV. The resolver invariants

`tatara-lisp-source::Resolver` enforces these at the boundary:

1. **One URL → one resolution**: the same URL string always yields the
   same bytes (modulo `?ref=` floating refs). The cache key is the
   URL's canonical form.
2. **BLAKE3 verification on pinned URLs**: if the URL declares
   `#blake3=<hex>`, fetched bytes must match. Mismatch is a typed
   `HashMismatch` error — no silent resolution against changed content.
3. **Content-addressed cache**: bytes are stored under
   `<root>/sources/<blake3>/data`. Two URLs that resolve to the same
   bytes share the cache entry transparently.
4. **No network on local paths**: `./foo.tlisp` is read directly; the
   resolver never reaches out for local sources.
5. **Auth via env var only**: `GH_TOKEN` / `GITHUB_TOKEN` is sent as
   `Authorization: Bearer <token>` for GitHub URLs. No config files.

## V. The promotion ladder for a tatara-lisp package

Same shape as a Cargo crate or a Ruby gem, mapped to git operations:

```
1. Author     pleme-io/programs/<name>/main.tlisp
2. Test       tatara-script ./main.tlisp                       (local)
3. Commit     git add main.tlisp && git commit
4. Tag        git tag v0.1.0
5. Push       git push --tags
6. Reference  github:pleme-io/programs/<name>/main.tlisp?ref=v0.1.0
7. Run        tatara-script <URL-from-step-6>                  (remote)
```

Step 4 is the equivalent of `gem build` + `gem push`. **`git tag`
is the publish.** Anyone consuming `?ref=v0.1.0` gets the exact
content the author tagged, content-verified.

## VI. The Helm side — same content addressing

Helm consumes WASM modules via `ComputeUnit.spec.module.source`:

```yaml
spec:
  module:
    source: github:pleme-io/programs/hello-world/main.tlisp?ref=v0.1.0
    blake3: abc123…   # optional pin — operator refuses to compile
                       # different bytes than this hash
```

The wasm-operator runs the same resolver. **Same URL on the host
(via `tatara-script`) and in the cluster (via wasm-operator)
yields the same bytes.** No duplicate fetches across the fleet
when the cluster's module store is BLAKE3-keyed.

## VII. The overall content-addressed surface

```
                        AUTHOR'S LAPTOP                            CLUSTER
            ┌──────────────────────────────────┐    ┌──────────────────────────────────┐
            │  ~/.cache/tatara/sources/<blake3>/data          /var/lib/wasm-store/sources/<blake3>/data │
            └──────────────────────────────────┘    └──────────────────────────────────┘
                              ▲                                      ▲
                              │ tatara-script resolves               │ wasm-operator resolves
                              │                                      │
                                 same URL, same Source struct, same BLAKE3
                              │                                      │
            ┌─────────────────────────────────────────────────────────────────┐
            │           github:owner/repo/path?ref=tag                          │
            │           (or gitlab:, codeberg:, https://#blake3=...)            │
            └─────────────────────────────────────────────────────────────────┘
                              ▲
                              │
                  ┌──────────────────────┐
                  │     git tag v0.1.0   │
                  └──────────────────────┘
```

Every box has the same key. The cache is content-addressed all the
way down. `git tag` is the publish; BLAKE3 is the integrity check.
There is no separate package management because git + nix already
provide every guarantee a package manager would offer.

## VIII. Substrate + forge standardization

Per the user's mandate: **standardize with substrate/forge as much
as possible across the entire tatara ecosystem.** The pattern:

- **Every Rust workspace in pleme-io** uses
  `substrate/lib/rust-workspace-release-flake.nix` for `nix build`,
  `nix run .#release`, etc. `tatara-lisp` already does
  ([`flake.nix:21`](https://github.com/pleme-io/tatara-lisp/blob/main/flake.nix#L21)).
- **Every container image** built via
  `substrate/lib/build/rust/tool-image-flake.nix`. Distroless OCI
  layered images, content-addressed, multi-arch. Added to
  `tatara-lisp` 2026-04-26 in [`flake.nix`](https://github.com/pleme-io/tatara-lisp/blob/main/flake.nix);
  `wasm-platform`, `openclaw-artifact-registry` next.
- **Every publish step** routes through `forge` (the pleme-io CI
  platform). One `nix run .#release` from any workspace flake
  publishes images, charts, and attestations. The user-facing
  surface is identical across every repo.
- **Every flake input** uses `inputs.<name>.inputs.nixpkgs.follows
  = "nixpkgs"` to keep closures small. The
  [pleme-io flake conventions](https://github.com/pleme-io/blackmatter-pleme/blob/main/docs/pleme-io-CLAUDE.md)
  spell this out.
- **Every consumer chart** in `helmworks` follows the
  `pleme-lib`-or-`pleme-computeunit`-dependency pattern. The user
  wires values; the library chart emits the right resources.

## IX. What this enables

### IX.1 "Nix run from anywhere"

```sh
nix run github:pleme-io/tatara-lisp#script -- \
  github:pleme-io/programs/hello-world/main.tlisp?ref=v0.1.0
```

One command — fetches `tatara-script` via Nix, fetches the program
via `tatara-lisp-source`, runs it. Both content-addressed all the
way through. Works from any machine with Nix installed; no extra
setup.

### IX.2 Reproducible cluster restoration

```sh
git clone github:pleme-io/k8s
flux bootstrap github --owner=pleme-io --repository=k8s --path=clusters/rio
```

The cluster reconciles itself from the git tree. Every program is a
URL. Every chart is a `pleme-charts` ref. Every secret is SOPS-encrypted
in-tree. **The cluster's full state is reachable from one git URL.**

### IX.3 Air-gapped reproduction

The BLAKE3 cache layout means:

```sh
# On a connected laptop:
tatara-script github:pleme-io/programs/hello-world/main.tlisp?ref=v0.1.0
# (cache populated)
tar czf /media/usb/cache.tgz ~/.cache/tatara/sources/

# On the air-gapped target:
tar xzf /media/usb/cache.tgz -C ~/.cache/tatara/sources/
tatara-script github:pleme-io/programs/hello-world/main.tlisp?ref=v0.1.0
# (cache hit — no network needed)
```

The cache is a self-contained reproduction unit. Move `~/.cache/tatara`
across machines and every URL still resolves.

## X. Future affordances (deferred until needed)

- **`tatara.lock`** at the consumer-program level recording transitive
  `(require ...)` resolutions. Mirrors `Gemfile.lock`. Adds CI-time
  drift detection.
- **OCI-published WASM** as a registry shape (`oci://ghcr.io/pleme-io/programs:hello-world-v0.1.0`).
  Already present as a `Source::Oci` variant in
  `tatara-lisp-source`; fetcher impl is the next step.
- **Nix flake-equivalent inputs** in tatara-lisp programs themselves —
  a `(use-flake "github:foo/bar")` form that pulls multiple files as
  a unit, with a manifest. Would close the analog with Nix flakes
  fully.

## XI. See also

- [`THEORY.md` Pillar 1](THEORY.md) — language constraint
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp as scripting standard
- [`WASM-STACK.md`](WASM-STACK.md) — runtime
- [`WASM-PACKAGING.md`](WASM-PACKAGING.md) — URL grammar (formalized)
- [`META-FRAMEWORK.md`](META-FRAMEWORK.md) — 4-layer hierarchy
- [`FLEET-DECLARATION.md`](FLEET-DECLARATION.md) — every cluster's
  programs in one Helm release
- [`HELLO-WORLD-LIVE.md`](HELLO-WORLD-LIVE.md) — proof of pattern
- [`tatara-lisp-source`](https://github.com/pleme-io/tatara-lisp/tree/main/tatara-lisp-source) —
  the resolver implementation
- [`substrate/lib/rust-workspace-release-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/rust-workspace-release-flake.nix) —
  the workspace pattern
- [`substrate/lib/build/rust/tool-image-flake.nix`](https://github.com/pleme-io/substrate/blob/main/lib/build/rust/tool-image-flake.nix) —
  the image pattern
