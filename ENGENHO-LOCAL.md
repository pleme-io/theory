# engenho-local — the canonical operator local-dev cluster

> **Frame.** Companion to [`ENGENHO.md`](./ENGENHO.md) — that doc owns
> the *runtime* (apiserver, datastore, kubelet, controllers, scheduler).
> This doc owns the *operator local-dev path*: the canonical way an
> operator runs a working Kubernetes-compatible cluster on a Mac via
> `nix run .#rebuild`. Pairs with [`kasou`](https://github.com/pleme-io/kasou)
> (Apple Virtualization.framework Rust API — the VM substrate) and
> [`kikai`](https://github.com/pleme-io/kikai) (k3s/engenho cluster
> lifecycle on kasou).
>
> The substrate this doc describes is **already 95% implemented**. The
> destination is to wire the existing components onto the operator's
> node such that `nix run .#rebuild` produces a `kubectl`-reachable
> Kubernetes cluster — k3s today as a 1:1 wire-compatible bridge, then
> engenho once M0.4 lands without any operator-facing config change.
>
> **Status.** Draft v1. The kikai HM module, kasou VM API, and
> `modules/kasou-vm` NixOS base module exist today. The gap is two
> small surgeries: (a) add `"engenho"` to kikai's `vmMode` enum, (b)
> wire `blackmatter.components.kubernetes.clusters.engenho-local` into the operator's
> node config. Both are operator-approval-gated; this doc is the
> design they're approving.
>
> **Name.** `engenho-local` is not a new pleme-io primitive — it's the
> declarative configuration that composes the existing substrate
> (kasou + kikai + the kasou-vm NixOS module + engenho) into a working
> local cluster. The composition pattern itself is the artifact.

---

## I. The repeating pattern

Every pleme-io operator needs a working Kubernetes-compatible cluster
on their dev machine to test caixa workloads, develop controllers,
exercise compliance overlays, and validate engenho's M0.1+ work as it
lands. Today the options are:

| Path | Problem |
|---|---|
| Docker Desktop + kind | Replaces operator's container runtime; closed-source; resource-hungry; license restrictions for non-personal use |
| minikube via vfkit | vfkit ignores `mac=` (the bug kasou exists to fix); DHCP leases drift; cluster IP not stable |
| k3d | Requires Docker Desktop; same pitfalls |
| Real cluster (rio, home-edge) | Round-trip latency, slow iteration, shared blast radius |
| Hand-rolled VM + manual k3s install | Snowflake setup, breaks on reboot, no Nix declarativity |

None of these are pleme-io-shaped. None are typed. None survive `nix
run .#rebuild` round-trip cleanly. The substrate exists for a better
answer; this doc names that answer.

---

## II. The destination

One Nix module declaration in the operator's node config produces a
working `kubectl`-reachable cluster:

```nix
# nodes/cid/engenho-local.nix
{ ... }: {
  home-manager.users.drzzln = { ... }: {
    blackmatter.components.kubernetes.clusters.engenho-local = {
      enable = true;
      vmMode = "engenho";    # k3s today as 1:1 bridge; engenho M0.4+
      cpus = 4;
      memory = 8192;          # MiB
      diskSize = "50G";
      apiPort = 6443;
      autoStart = true;
      timeouts.boot = 300;
    };
  };
}
```

After `nix run .#rebuild`:

1. The kikai HM module materializes `~/.config/kikai/clusters.yaml`
   with the typed `engenho-local` entry.
2. A launchd agent (`org.pleme.kikai.engenho-local`) is registered.
3. The launchd agent starts `kikai daemon engenho-local`, which:
   - Builds the cluster's NixOS image (`nix build` on a flake that imports
     `pleme-io/nix/modules/kasou-vm` + the k3s/engenho server payload).
   - Creates the 50 GiB sparse data disk + a 1 MiB seed disk.
   - Spawns the VM via `kasou::VmHandle::create(VmConfig { ... })` with
     the deterministic MAC address (this is exactly what kasou exists
     to provide — see [`kasou/README.md`](https://github.com/pleme-io/kasou)).
   - Watches the DHCP lease file for the VM's IP.
   - Bootstraps secrets via SOPS: k3s server token, admin password,
     age keypair.
   - Waits for the API server on `:6443` to respond healthy.
   - Materializes `~/.kube/configs/engenho-local` with the cluster CA,
     server URL, and admin token.
   - Health-loops every 60s: `kubectl --context=engenho-local get nodes`.
   - Restarts the VM if health fails for `maxFailures` consecutive checks.

The operator's experience:

```bash
$ nix run .#rebuild          # waits for VM to come up + reach health
$ kubectl --context engenho-local get nodes
NAME            STATUS   ROLES                  AGE   VERSION
engenho-local   Ready    control-plane,master   2m    v1.32.0+k3s1
```

That's the destination. Everything below describes how the existing
substrate composes to reach it.

---

## III. The substrate stack

```
macOS (operator's Mac, e.g. cid)
  ├── nix-darwin: imports kikai's HM module (already a flake input)
  ├── home-manager: blackmatter.components.kubernetes.clusters.engenho-local = { ... }
  ├── launchd agent: org.pleme.kikai.engenho-local → kikai daemon
  │
  ├── kikai daemon (Rust binary, kasou-driven)
  │     ├── kasou::VmConfigBuilder — typed VM spec with deterministic MAC
  │     ├── kasou::VmHandle        — Apple Virtualization.framework lifecycle
  │     ├── kasou::dhcp            — DHCP lease watcher (kasou ships this)
  │     ├── kikai::state           — FSM (Down → Spawning → Booting → Healthy)
  │     ├── kikai::health          — kubectl + API probe; restart on failure
  │     ├── kikai::sops            — server token + admin password decryption
  │     └── kikai::init / up / down / destroy / snapshot / park / pause / resume
  │
  └── kasou-managed Linux VM (NixOS aarch64-linux)
        ├── modules/kasou-vm/default.nix — virtio drivers, ext4 root, console=hvc0
        ├── networking: NAT 192.168.64.x with deterministic MAC (the kasou fix)
        ├── secrets: SOPS-decrypted at boot via systemd activation
        ├── runtime selector — driven by kikai's vmMode:
        │     • vmMode="k3s"     → k3s server v1.32+ (today's wire-compat bridge)
        │     • vmMode="engenho" → engenho binary    (M0.4+; same wire surface)
        │     • vmMode="builder" → no k8s; just an aarch64-linux nix-builder
        └── workload runtime: containerd + youki (per ENGENHO.md §I)
```

Every box above already exists except the kikai vmMode="engenho"
variant (a one-line enum extension; see §VII).

---

## IV. Wire compatibility — k3s today, engenho tomorrow, same kubectl

The 1:1 compatibility the operator wants is provided by **the Kubernetes
API surface itself**. Both k3s and engenho speak it (engenho per the
ENGENHO.md §III.1 conformance contract; k3s by being upstream-vendored
Go). The Mac-side wire surface is identical:

- TLS on `:6443` (configurable via `apiPort`)
- Standard kubectl/Helm/Flux/Crossplane all work unchanged
- Same `~/.kube/configs/engenho-local` shape (it's just a kubeconfig)
- Same RBAC / SSA / CRD machinery the upstream API defines

The migration from k3s to engenho is one line in the Nix declaration:

```diff
 blackmatter.components.kubernetes.clusters.engenho-local = {
   enable = true;
-  vmMode = "k3s";
+  vmMode = "engenho";
 };
```

`nix run .#rebuild` triggers a new NixOS image build for the VM with
engenho instead of k3s. kikai's lifecycle daemon stops the old VM,
starts the new one, and re-validates kubectl connectivity. The
operator's kubectl commands never change.

This is the "substrate carries the swap" pattern named in
[`CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md`](./CONSTRUCTIVE-SUBSTRATE-ENGINEERING.md) —
the operator declares what they want; the substrate (kikai + kasou +
engenho/k3s) renders the implementation.

---

## V. What exists today (95%)

### V.1. kasou — Apple Virtualization.framework Rust API

- [`pleme-io/kasou`](https://github.com/pleme-io/kasou) — the library
  this whole stack rides on.
- `VmConfigBuilder` → `VmConfig` → `VmHandle` — pure data types + a
  thin runtime handle around `VZVirtualMachine`.
- `deterministic_mac(hostname)` — fixes the vfkit `mac=` bug at the
  hypervisor level. The reason this library exists.
- `dhcp` module — watches `/var/db/dhcpd_leases` for the assigned IP.
- Events broadcast via `VmEventBus`; consumers observe lifecycle.

### V.2. kikai — cluster lifecycle on kasou

- [`pleme-io/kikai`](https://github.com/pleme-io/kikai) — Rust binary
  with CLI subcommands `init / up / down / status / destroy / daemon /
  park / pause / resume / snapshot / health / state-verify / dump-config`.
- `kikai/module/default.nix` — home-manager module factory exposing
  `services.kikai.{package, runtimeDeps, clusters.<name>.*}` with
  typed options (cpus, memory, diskSize, apiPort, sshPort, secrets,
  timeouts, health intervals, vmMode).
- Cross-platform: Darwin emits launchd agents; Linux emits systemd
  user services. Via `substrate/lib/hm-service-helpers.nix`.
- SOPS-encrypted secrets via the operator's `secrets.yaml`.

### V.3. NixOS guest base — `modules/kasou-vm`

- `pleme-io/nix/modules/kasou-vm/default.nix` — base NixOS module
  every kasou-bootable VM imports.
- Sets up: virtio drivers, ext4 root on `/dev/vda`, DHCP on virtio-net,
  `console=hvc0`, `aarch64-linux` host platform default.
- Boot path: direct kernel + initrd (no grub/systemd-boot). kasou's
  `BootConfig` carries the paths.

### V.4. The operator's existing infrastructure

- `pleme-io/nix/flake.nix` already declares `inputs.kikai = { url = github:pleme-io/kikai; ... }`.
- `pleme-io/nix/darwinConfigurations/default.nix` already has the
  shared `darwinModules` list pattern; adding `inputs.kikai.homeManagerModules.default`
  is one line.
- `nodes/cid/` (this operator's Mac) has the precedent for `tatara-os-vm.nix`
  (currently disabled pending aarch64-linux builder fix); engenho-local
  follows the same node-local file pattern.

### V.5. What's missing (5%)

| Gap | Surgery |
|---|---|
| kikai's `vmMode` enum lacks `"engenho"` variant | Add to `kikai/module/default.nix` enum AND `blackmatter-kubernetes/module/home-manager/kikai/default.nix` wrapper; for M0–M3 it routes to k3s under the hood with a note printed |
| The operator's `nodes/cid/engenho-local.nix` doesn't exist | Create it with `blackmatter.components.kubernetes.clusters.engenho-local = { ... }` |
| Shared `darwinModules` import | Already wired — `blackmatter-kubernetes` lands on every node via `inputs.blackmatter.homeManagerModules.blackmatter` (the aggregator). No change needed. |
| SOPS secrets `clusters/engenho-local/{server-token,age-key,admin-password}` | Operator runs `sops edit secrets.yaml` once (or `kikai init engenho-local` after first rebuild to bootstrap them in-place) |
| kikai's vmMode-to-image-payload mapping is k3s-only | M0.0–M0.4: trivially keep k3s under all aliases. M0.4+: route `vmMode="engenho"` to engenho image |
| **vmnet DHCP gate** | ryn's clusters are flagged off due to "vmnet DHCP blocker". `kasou::dhcp` watches `/var/db/dhcpd_leases`; if macOS vmnet isn't allocating leases the cluster never reaches Healthy. Verify on cid before enabling. |

That's the entire delta.

---

## VI. Operator wiring on `cid` (this Mac)

Two files change (blackmatter aggregator already imports the kikai
wrapper on every node — no shared `darwinModules` edit needed).

### VI.1. Declare the cluster on the cid node

Create `pleme-io/nix/nodes/cid/engenho-local.nix`:

```nix
# nodes/cid/engenho-local.nix
#
# The canonical operator local-dev cluster — kasou-managed Linux VM
# running k3s today (1:1 wire-compat) → engenho (M0.4+). Spec:
# pleme-io/theory/ENGENHO-LOCAL.md.
{ inputs, pkgs, ... }: {
  home-manager.users."drzzln" = { ... }: {
    # SOPS secrets — see §VI.3 for the secrets.yaml additions.
    sops.secrets = {
      "clusters/engenho-local/server-token"   = { };
      "clusters/engenho-local/age-key"        = { };
      "clusters/engenho-local/admin-password" = { };
    };

    blackmatter.components.kubernetes = {
      kikaiPackage     = inputs.kikai.packages.${pkgs.stdenv.hostPlatform.system}.default;
      kikaiRuntimeDeps = with pkgs; [ nix sops mtools dosfstools openssh age ];

      clusters.engenho-local = {
        enable       = true;
        vmMode       = "k3s";          # → "engenho" once M0.4 ships
        autoStart    = true;
        cpus         = 4;
        memory       = 8192;
        diskSize     = "50G";
        apiPort      = 6443;
        sshPort      = 2222;
        nixFlake     = "/Users/drzzln/code/github/pleme-io/nix";
        secretsFile  = "/Users/drzzln/code/github/pleme-io/nix/secrets.yaml";
        sopsYaml     = "/Users/drzzln/code/github/pleme-io/nix/.sops.yaml";
        macAddress   = "5a:94:ef:cd:00:01";  # deterministic; pinned DHCP lease

        decryptedSecrets = let d = "/Users/drzzln/.config/sops-nix/secrets"; in {
          serverToken   = "${d}/clusters/engenho-local/server-token";
          ageKey        = "${d}/clusters/engenho-local/age-key";
          adminPassword = "${d}/clusters/engenho-local/admin-password";
        };

        timeouts = { boot = 300; shutdown = 120; };
        health   = { interval = 60; maxFailures = 3; };
      };
    };
  };
}
```

### VI.2. Import the new file from cid's default.nix

In `pleme-io/nix/nodes/cid/default.nix`'s `imports = [ … ]`:

```nix
imports = [
  ../../modules/darwin/blackmatter
  ../../profiles/darwin-developer
  ./identity.nix
  ./pangea-builder.nix
  ./tatara-os-vm.nix
  ./tatara-builders.nix
  ./engenho-local.nix         # ← new
];
```

### VI.3. Bootstrap SOPS secrets (one-time)

```bash
cd ~/code/github/pleme-io/nix
sops edit secrets.yaml      # add three keys under clusters.engenho-local
```

Add the following (pleme-io's existing age recipients apply):

```yaml
clusters:
  engenho-local:
    server-token:   "<head -c 24 /dev/urandom | base64 — k3s server token>"
    age-key:        "<age-keygen output — private half>"
    admin-password: "<random — k3s admin password>"
```

Alternatively run `kikai init engenho-local` once after the first
rebuild — kikai's init flow generates and SOPS-encrypts the three
secrets in-place. The first rebuild then re-reads them.

That's the entire wiring.

---

## VII. The kikai engenho variant (small kikai PR)

`kikai/module/default.nix` today:

```nix
vmMode = lib.mkOption {
  type = lib.types.enum [ "k3s" "builder" ];
  default = "k3s";
  description = "Workload runtime in the VM.";
};
```

Becomes:

```nix
vmMode = lib.mkOption {
  type = lib.types.enum [ "k3s" "engenho" "builder" ];
  default = "k3s";
  description = ''
    Workload runtime inside the kasou VM.

    * `k3s` — upstream k3s server. Today's stable bridge.
    * `engenho` — pleme-io's typed, attested, Rust-native K8s runtime
      (theory/ENGENHO.md). Currently routes to `k3s` under the hood at
      M0.0–M0.3; switches to native engenho at M0.4 without changing
      this option. 1:1 wire-compat with k3s throughout the transition.
    * `builder` — no k8s; a generic aarch64-linux nix-builder VM.
  '';
};
```

Plus a print-once status line in `kikai daemon` when vmMode="engenho"
is configured but engenho's M0.4 image isn't available yet:

```
note: vmMode=engenho currently routes to k3s under the hood (engenho M0.0–M0.3).
      kubectl wire-compat is identical; M0.4+ switches transparently.
      Track: theory/ENGENHO.md §X.
```

That's the entire kikai patch. ~10 LoC + one log line.

---

## VIII. Determinism contract

- **VM disk identity**: kasou uses a deterministic path
  `~/.local/share/kikai/engenho-local/data.img`. Snapshots via
  `kikai snapshot create` are content-addressed by BLAKE3 of the disk image.
- **MAC address**: `kasou::deterministic_mac("engenho-local")` —
  same hostname → same MAC across reboots → stable DHCP lease.
- **NixOS image**: `nix build` produces a content-addressed image
  derivation; identical image config → identical image hash → no
  unnecessary rebuilds.
- **Kubeconfig**: kikai writes the kubeconfig deterministically (same
  inputs → same file bytes), enabling reproducible operator setups
  across `nix run .#rebuild` runs.
- **SOPS secrets**: server token + admin password are stored once
  per cluster name in `secrets.yaml`, encrypted via age. Same name →
  same secrets on rebuild.

---

## IX. Verifiability surface

L0 — type-axis. Module options are typed (`enum`, `int`, `port`,
`strMatching`). Invalid configs fail evaluation at `nix run .#rebuild`.

L1 — kikai unit tests. `cargo test --workspace` exercises the state
machine, health checker, SOPS reader, etc.

L4 — integration smoke test (operator-side, runs once after rebuild):

```bash
# After `nix run .#rebuild`:
launchctl print gui/$UID/org.pleme.kikai.engenho-local | head
kubectl --context engenho-local get nodes
kubectl --context engenho-local get pods --all-namespaces
kubectl --context engenho-local cluster-info
```

L5 — e2e. Apply a Deployment, watch it materialize, scale it, delete it:

```bash
kubectl --context engenho-local create deployment hello --image=nginx
kubectl --context engenho-local scale deployment hello --replicas=3
kubectl --context engenho-local rollout status deployment hello
kubectl --context engenho-local delete deployment hello
```

L6 — Sonobuoy (CNCF Certified Kubernetes Software Conformance) once
engenho M0.4 ships; the cluster is then a valid conformance target.

---

## X. Phase plan

| Phase | When | What runs in the VM | Operator UX |
|---|---|---|---|
| **A** | Today (engenho M0.0–M0.3) | k3s | `vmMode = "k3s"` or `vmMode = "engenho"` (alias). kubectl works. |
| **B** | engenho M0.4 ships | engenho binary | `vmMode = "engenho"` routes to native. Nix declaration unchanged. |
| **C** | engenho M2+ (typescape-native) | engenho with native Caixa CRDs | `blackmatter.components.kubernetes.clusters.engenho-local.caixaNative = true` enables direct caixa→pod path |
| **D** | engenho M4 (CNCF Certified) | engenho passing conformance | Operator's local cluster is conformance-tested; Sonobuoy in CI gates regressions |

---

## XI. Anti-patterns

- **Running k3s/engenho directly on macOS** — no Linux kernel. Always
  in a kasou VM.
- **Docker Desktop / Docker for Mac as the substrate** — pleme-io
  doesn't depend on Docker; kasou owns the VM, containerd inside owns
  the container runtime.
- **Hand-editing `~/.kube/config`** — kikai writes
  `~/.kube/configs/<name>`; operators merge via context, never via
  hand-edits.
- **Manual VM management** (`vfkit -c …` invocations from the shell) —
  kikai owns the VM lifecycle; the operator declares it in Nix.
- **Stateful workarounds in the VM disk** — the VM disk is
  reproducible from `nix build` + SOPS; never `ssh` in and `apt
  install` something.
- **Forgetting `aarch64-linux` cross-build dependency** — kikai's
  image build depends on a reachable aarch64-linux builder (this is
  why cid's `tatara-os-vm.nix` is currently gated on
  `wantAarch64LinuxBuilder`). When the builder is unavailable,
  fallback is the libkrun-builder or a vendored pre-built k3s image.

---

## XII. Open questions

1. **Builder dependency**: kikai's NixOS image build needs aarch64-linux.
   cid's current state is `wantAarch64LinuxBuilder = false`. Decision:
   prebuild a stable image and reference it directly, OR fix the builder
   and rebuild from source. Recommendation: prebuild a pinned image,
   bump on schedule.

2. **Kubeconfig merge vs. separate files**: today kikai writes
   `~/.kube/configs/<name>`. Should `KUBECONFIG` env var be auto-managed
   so the cluster is visible without explicit `--context`? Recommendation:
   yes — kikai HM module exports `home.sessionVariables.KUBECONFIG`
   chain.

3. **Storage class**: for caixa workloads that need persistent volumes,
   k3s's local-path provisioner works. engenho M0 includes `engenho-localpath`
   per `theory/ENGENHO.md §II.1`.

4. **Multi-cluster on one Mac**: `services.kikai.clusters.<name>` is a
   map, so the operator can declare engenho-local + engenho-staging +
   builder-only-vm if needed. The 192.168.64.x DHCP pool supports many.

---

## XIII. Relationship to other primitives

| Primitive | Relationship |
|---|---|
| `kasou` | The VM substrate. engenho-local always runs inside a kasou VM. |
| `kikai` | The cluster lifecycle daemon. engenho-local is one `kikai.clusters.<name>` entry. |
| `engenho` | The runtime that will eventually replace k3s inside the VM (M0.4+). engenho-local is engenho's primary dev environment. |
| `tatara` | engenho's components run as `defguest`-supervised daemons inside the VM (per ENGENHO.md §IX). engenho-local exercises this. |
| `cofre` | k3s/engenho secrets flow through cofre per `ENGENHO.md §II.1`. engenho-local validates the cofre→k8s-secret path. |
| `caixa` / `feira` | M2+: caixa workloads deploy directly via `feira deploy --context engenho-local`. engenho-local is the dev target for caixa-native reconciliation. |
| `tatara-os-vm` (cid's existing module) | Sibling pattern. tatara-os-vm runs *tatara programs* in a kasou VM; engenho-local runs *Kubernetes* in a kasou VM. Both compose kasou. |
| `libkrun-builder` (cid's existing module) | Sibling pattern. libkrun-builder runs an aarch64-linux *nix-builder* VM; engenho-local runs a *Kubernetes* VM. Different workloads, same VM-on-Mac substrate. |
| `pleme-io/nix` | Owns the operator's node config. engenho-local lives in `nodes/cid/engenho-local.nix` (opt-in per node). |

---

*Substrate is 95% built. Three files change. After rebuild, kubectl works.*
