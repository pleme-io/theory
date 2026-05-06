# SELINUX-NIXOS — pleme-io's plan for SELinux on NixOS

> **Status:** canonical (2026-05-06). Implementation tracked in
> [`pleme-io/blackmatter-selinux`](https://github.com/pleme-io/blackmatter-selinux).
>
> This document specifies how pleme-io adds SELinux Mandatory Access
> Control to NixOS as a redistributable substrate component. It complements
> [`THEORY.md`](./THEORY.md) §V (three-pillar attestation), §VI.2 (typescape
> rendering), and BLACKMATTER pillar 7 (Kubernetes control rendered from
> typescape) and pillar 11 (mandatory alert layer on every compute unit).

---

## The frame

NixOS ships zero SELinux integration. Cause:

1. **Kernel.** Default `pkgs.linux` has `CONFIG_SECURITY_SELINUX*` off; the
   LSM framework loads AppArmor and Landlock but never SELinux.
2. **Userspace.** `nixpkgs.selinuxPackages` exists (libselinux, libsepol,
   policycoreutils, checkpolicy at 3.10) but no NixOS module wires them
   into systemd, `/etc/selinux/config`, or boot.
3. **Policy.** No NixOS-aware reference policy exists anywhere. The Fedora
   `selinux-policy-targeted` reference policy assumes FHS paths
   (`/usr/bin/sshd`, `/var/lib/postgres`, etc.). On NixOS every binary
   lives under `/nix/store/<32-char-hash>-<name>-<version>/bin/<binary>`,
   so the path-based `file_contexts` table from the reference policy
   labels nothing.

The third item is the load-bearing one. **The hard problem isn't enabling
SELinux on NixOS — it's authoring a policy that knows about `/nix/store`.**

This document scopes the work, names the substrate hooks, and lays out the
multi-month roadmap honestly.

---

## Threat model — what the gap actually is

The motivation for this work is concrete. Compare two equally-hardened
rootless-podman hosts under identical baseline configuration: non-root
user (UID 5000), private user-namespace remapping, `no-new-privs` on every
container, distinct UIDs per workload, no podman socket access, volume
allow-list per container. **Without SELinux, that hardening is a single
layer of defense.** With SELinux, it becomes one of multiple independent
layers — and that's the entire value proposition.

The threats SELinux materially defends against, and the tier of this plan
that closes each one:

| # | Threat | Without SELinux | With our T3 enforcing policy | Closed at |
|---|---|---|---|---|
| 1 | Container escape via kernel vuln | Attacker has full UID 5000 host access | Attacker confined to `container_t` domain — can't touch host files labeled `admin_home_t`, `nix_store_t`, etc. | M3 |
| 2 | Compromised app inside container | Full DAC access within container's namespace | Process confined to `container_t` — can't read sensitive files, inject into other processes, or reach kernel surfaces it doesn't need | M3 |
| 3 | Volume-mount misconfiguration | Cross-container access succeeds — no second barrier | Per-container MCS labels (`s0:c123,c456`) deny cross-container reads even when volume paths overlap | M3 |
| 4 | Symlink attack on volumes | Successful — race between `chcon` and `open()` is unguarded | File-label check fires in the kernel on every `open()`; symlink to a file labeled with a different MCS category gets denied | M3 |
| 5 | Process injection within host (same UID) | Possible — DAC permits same-UID `ptrace`, `kill`, `/proc/pid/mem` | SELinux domain separation denies cross-domain `ptrace`/`process_vm_*` regardless of UID | M3 |
| 6 | Single-layer failure (namespace escape, capability bug, seccomp gap) | No fallback — that one failure is total | Other independent layers (LSM domain, MCS, file labels) still enforce | M3 |

**Crucially for M0:** Tiers 1 + 2 alone (kernel + userspace in permissive
mode) close **none** of these threats. Permissive mode logs every access
decision but blocks none. The defense-in-depth promise lives entirely at
**M3** when an enforcing-mode policy lands. M0 + M1 + M2 are *necessary
groundwork* for M3 to ship; they are not themselves the deliverable.

This honest framing matters for milestone-driven decisions: a customer
asking "is the host secure under this threat model?" gets "no" until M3,
regardless of how much M0/M1/M2 work has shipped.

### Multi-Category Security (MCS) — the cross-container isolation primitive

Threats #3 and #5 in the table above turn on a feature most operators
encounter for the first time when they ship to production: **MCS labels**.

Every running container under enforcing-mode podman gets a unique MCS
label of the form `system_u:system_r:container_t:s0:c123,c456` — `s0` is
the sensitivity level (single-level), `c123,c456` are two of 1024
randomly-chosen categories. Files written by that container inherit the
same label. The SELinux policy includes a *constraint* that says:

> A subject in `container_t` may only access an object in
> `container_file_t` if the subject's category set is a superset of the
> object's category set.

Translation: container A (categories `c123,c456`) can read its own
files (same categories) but **cannot read files written by container B**
(categories `c789,c012`) — even if they share a volume mount, even if
DAC permissions allow it, even if the symlink looks valid. The kernel
denies the open() at the LSM hook before it ever returns a file
descriptor.

This is the mechanism the other-agent's analysis names as "cross-container
isolation by SELinux domain labels." It's not domain — domain is the
same `container_t` for both — it's MCS categories. We call it out
explicitly because:

1. **MCS is invisible until you turn it on.** Stock podman + container-selinux
   on Fedora ships MCS by default. Our T4 module must explicitly enable it
   in `containers.conf`; otherwise both containers run with the same
   `s0` label and #3 / #5 silently revert to "no protection."
2. **MCS only delivers value if file labels are applied.** The activation
   hook in T2 (`setfiles -F`) and T3's per-policy-load relabel are the
   load-bearing infrastructure. Without them, files end up `unlabeled_t`
   and the constraint silently passes (no `container_file_t` to compare).
3. **MCS is the cheapest defense-in-depth win in the entire system.** Once
   T3's `container_t` / `container_file_t` types exist and T4 enables
   labeling, MCS isolation comes free per container — podman generates
   categories per-container automatically.

Concretely: `blackmatter.components.selinux.containers.podman.mcsLabeling
= true` (default once T4 is enabled) is what makes threats #3 and #5
disappear. Operator never thinks about category numbers.

### FCOS vs NixOS-with-blackmatter-selinux comparison

Once M3 ships, the security comparison flips from "FCOS-with-SELinux vs
NixOS-without" (today's ecosystem gap) to "FCOS-with-SELinux vs
NixOS-with-blackmatter-selinux." For each row in the threat table above,
both systems hit the same column — same `container_t`, same MCS labels,
same kernel hooks. **The remaining difference is the maturity of the
policy itself**, not the presence/absence of SELinux.

NixOS retains its native advantages on top of that:

- **Reproducible policy builds.** The policy is a Nix derivation; same
  inputs always produce the same `.pp`. Fedora rebuilds policy on
  package install and the result depends on dnf transaction order.
- **Atomic policy rollback.** `nixos-rebuild switch --rollback` reverts
  the policy with the same atomicity as the rest of the system — no
  fiddling with `semodule -r` against a half-broken state.
- **Typed cluster-wide consistency.** The `SecurityPosture` typescape
  primitive (M6) makes "every node in this cluster runs the same
  policy revision" a compile-time invariant.

**Until M3, NixOS remains the weaker option** for any threat model that
includes container escape, multi-tenant workloads, or defense-in-depth
requirements. The honest line for customers: "we're closing this gap on
the M3 timeline; until then, run sensitive workloads on FCOS or wait."

---

## Why pleme-io builds this (vs. waiting for upstream)

Three durable reasons, each compounds:

1. **Compliance promises become theorems, not assertions.** A NixOS host
   running our SELinux policy in enforcing mode has a derivable,
   attestable answer to NIST 800-53 AC-3, AC-6, SC-3, SI-7. Pillar 11 and
   the compliant-artifact-provability stack (cartorio + lacre + provas +
   tabeliao — see [`THEORY.md` §V](./THEORY.md)) need this for the
   FedRAMP-High deliverables openclaw is targeting.
2. **The substrate gains a typed `SecurityPosture` slot.** Once the
   `(defsecurityposture …)` primitive exists in arch-synthesizer
   (§ "Typescape integration" below), every cluster declaration carries
   its LSM choice in the type system. The same mechanical convergence
   that picks a Helm chart or a Cilium NetworkPolicy now picks a
   security-module posture — no per-node drift, no audit surprises.
3. **Container volume labels work end-to-end.** `:Z` / `:z` on podman
   volumes is a one-line UX win that today silently no-ops on NixOS.
   Tier 4 closes that gap and makes it a typed slot on every
   `oci-containers.containers.<name>.volumes` consumer.

---

## The four-tier plan

The work decomposes into four tiers. Each tier ships independently and is
useful on its own.

| Tier | Scope | Output | Effort | Useful w/o upper tiers |
|---|---|---|---|---|
| **T1 — Kernel** | Custom kernel build with SELinux LSM compiled in | `pkgs.linux_selinux` overlay + `boot.kernelPackages` wiring | 2–3 weeks | Yes — enables SELinux for any user willing to write their own policy |
| **T2 — Userspace + init** | `policycoreutils`/`libselinux` installed; `/etc/selinux/config` rendered; systemd autorelabel; boot cmdline | NixOS module that boots in **permissive, no-policy** mode | 3–4 weeks | Yes — the host now logs every SELinux access decision; equivalent to Fedora's `setenforce 0` baseline |
| **T3 — NixOS-aware policy** | Reference policy that understands `/nix/store` paths, content-addressed binaries, NixOS service layout (systemd unit naming, log paths, runtime dirs) | `pkgs.selinux-policy-nixos` package + per-service policy modules | **4–14 months** (multi-engineer) | This tier is the value tier — without it, T1+T2 just collect denial logs |
| **T4 — Container labels + MCS** | Typed wiring for podman/docker `:Z`/`:z` volume labels; `container_t` / `container_file_t` types in the base policy; **MCS labeling** turned on so per-container categories deliver cross-container isolation | NixOS module flag + arch-synthesizer typed slot | 1–2 weeks (after T3) | No — requires T3 for `container_*` types to exist |

### What "useful" means at each tier

- **T1 alone** — security researcher / kernel hacker can boot a NixOS box
  with SELinux compiled in, then write whatever policy they want from
  scratch. Validates the kernel path. Lets us run kernel-level SELinux
  testsuites against NixOS.
- **T1 + T2** — production-quality "audit mode" deployment. Every system
  call still works (permissive mode), but the kernel logs every access
  decision to the audit log. Useful for: profiling what a service
  actually touches, generating starting policies via `audit2allow`, and
  satisfying compliance regimes that require audit logging without
  enforcement.
- **T1 + T2 + T3** — enforcing-mode SELinux on NixOS. The promise tier.
  Compliance theorems become derivable.
- **T1 + T2 + T3 + T4** — container hosts can use `:Z`/`:z` volume
  labels meaningfully. Closes the original use case.

---

## Tier 3 — the load-bearing problem and how we tractably scope it

The Fedora reference policy is ~150,000 lines of `.te` / `.fc` / `.if`
files. Porting it wholesale to NixOS is impractical (and most of it
labels paths that don't exist on NixOS anyway). Two viable scoping
strategies:

### Strategy A — Closure-scoped policy (pleme-io's choice)

Author a policy for **one workload class at a time** rather than a
general-purpose desktop NixOS. First target: **rootless podman container
hosts** (because that's what unblocks Tier 4 and what every pleme-io
homelab cluster runs).

- **Base module** — covers init (systemd), kernel, syslog, sshd. ~1,500
  lines of `.te`. Constant cost regardless of workload.
- **Per-service modules** — one `.te` per service that runs on the host.
  Authored by running the service in permissive mode, collecting the
  denials, running `audit2allow`, then hand-pruning the result.
  ~50–500 lines each. Marginal cost per service.

Estimate: 4 months for the base + podman + 5 representative services.
Each new service after that: 1–3 days of audit-and-prune work.

### Strategy B — Auto-generated policy from typescape

Longer-term win, much further out. The arch-synthesizer typescape
already knows what binaries each service ships, what files it reads,
what sockets it binds. In principle a typed renderer can emit a
service's `.te` module from its declaration.

This is research-grade work — there's no existing tool that does it for
SELinux. It's the right long-term direction (it makes policy authoring
mechanical instead of artisanal, in line with pillar 12 — generation
over composition), but it's strictly downstream of Strategy A. We need
hand-authored policies first to discover what the renderer must emit.

### The /nix/store path problem, concretely

SELinux file labels are stored in extended attributes on the filesystem,
populated from a `file_contexts` table that maps **path regexes** to
**type labels**:

```
/usr/bin/sshd       --      system_u:object_r:sshd_exec_t:s0
```

NixOS's equivalent path is something like
`/nix/store/qx81m2k7…-openssh-9.7p1/bin/sshd` — and the hash changes on
every rebuild. Three options:

1. **Path-pattern labels.** `/nix/store/[^/]*-openssh-.*?/bin/sshd ` →
   `sshd_exec_t`. Works but brittle: any package rename breaks it, and
   the regex engine in libselinux isn't designed for this volume of
   wildcards.
2. **Activation-time relabeling.** A NixOS activation hook that walks
   the current system closure and applies labels via `chcon`. Concrete
   and reliable, but adds noticeable time to every `nixos-rebuild
   switch`. **This is the strategy we go with.**
3. **Content-based labels.** Theoretical — would need a custom labeling
   backend that hashes binaries and looks up labels by content hash.
   Doesn't exist. Out of scope.

Tier 3 ships option 2: a `selinux-relabel-current-system` activation
hook that runs after every `switch-to-configuration` and labels the new
generation's binaries before the next service start.

---

## Substrate hooks (what pleme-io reuses regardless)

Even if the SELinux work stalls at Tier 1, four substrate primitives
get built that benefit other work:

1. **`substrate/lib/build/nixos/kernel-lsm-flake.nix`** — generic
   parameterized kernel-with-LSM builder. `(baseKernel, lsms,
   extraConfig) → overlay`. Reusable for Landlock policy work, eBPF-LSM
   experiments, future LSM stacking.
2. **NixOS VM-test harness for security modules** — Pattern in
   `blackmatter-selinux/checks/vm-*.nix` that boots a VM with the
   module enabled, asserts kernel features and audit-log output. Lifted
   to substrate as the canonical pattern for any `blackmatter-*`
   security component.
3. **Typed `SecurityPosture` slot in arch-synthesizer.** New primitive:

   ```ruby
   defsecurityposture :name "container-host" do
     lsm    :selinux             # | :apparmor | :landlock | :none
     mode   :enforcing           # | :permissive | :disabled
     policy "selinux-policy-nixos-podman"  # name in pkgs
   end
   ```

   Every `defcluster` / `defcaixa` form gains an optional
   `:security-posture <ref>` slot; the renderer mechanically adds the
   right `boot.kernelPackages`, `security.selinux.*` config, audit
   forwarders, and AlertmanagerConfig rules.
4. **Pillar-11 audit alert layer.** Every `selinux=enforcing` host
   automatically gets a `VMAlertmanagerConfig` watching for
   `audit_log_messages_total{type=AVC,result=denied}` spikes — a
   denial spike on a previously-quiet service is almost always a real
   incident or a missed policy update. Pattern reusable for any LSM.

---

## Repo layout

`pleme-io/blackmatter-selinux` follows the canonical
`blackmatter-component-flake` shape:

```
blackmatter-selinux/
├── flake.nix                            # uses substrate/lib/blackmatter-component-flake.nix
├── module/                              # HM module (researcher tools)
│   ├── default.nix
│   └── nixos/                           # NixOS module — the meat
│       ├── default.nix                  # composes T1-T4, gates each behind enable flags
│       ├── kernel.nix                   # T1: substantive
│       ├── userspace.nix                # T2: substantive (init)
│       ├── policy.nix                   # T3: framework + per-module loader
│       └── containers.nix               # T4: typed wiring
├── overlays/
│   └── kernel-selinux.nix               # T1: pkgs.linux_selinux variant
├── policy/                              # T3: deliverable lives here
│   ├── README.md                        # authoring guide
│   ├── modules/                         # one .te / .fc / .if per service
│   │   ├── base.te                      # systemd, kernel, syslog
│   │   ├── nix-store.te                 # /nix/store labeling
│   │   ├── nixos-init.te                # NixOS-specific init quirks
│   │   └── …                            # accretes over months
│   └── build.nix                        # checkmodule + semodule_package pipeline
├── checks/
│   ├── eval-nixos-module.nix            # auto-generated by builder
│   └── vm-selinux-permissive.nix        # boot test
├── README.md
├── CLAUDE.md
└── .envrc
```

Same shape as `blackmatter-security`, `blackmatter-kubernetes`,
`blackmatter-claude` — operators read one repo and know all of them.

---

## Configuration surface

```nix
blackmatter.components.selinux = {
  enable = true;                          # master toggle — composes the tiers below

  kernel = {
    enable = true;                        # Tier 1
    base = pkgs.linuxKernel.kernels.linux_6_12;  # default: nixpkgs latest LTS
    extraConfig = {};                     # operator overrides on the LSM-augmented config
  };

  userspace = {
    enable = true;                        # Tier 2 — implies kernel.enable
    bootMode = "permissive";              # | "enforcing" | "disabled"
    autorelabelOnSwitch = true;
  };

  policy = {
    enable = true;                        # Tier 3 — implies userspace.enable
    base = pkgs.selinux-policy-nixos;     # built from policy/ in this repo
    modules = [];                         # extra per-workload modules (paths or pkgs)
  };

  containers = {
    enable = true;                        # Tier 4 — implies policy.enable
    podman.relabelVolumes = true;         # turns :Z / :z handling on
    podman.mcsLabeling = true;            # per-container categories — closes threats #3, #5
    docker.relabelVolumes = true;
    docker.mcsLabeling = true;
  };

  audit = {
    forwardToVector = true;               # pillar 11 — auto-wires AVC → Vector → VictoriaLogs
    alertOnDenialSpike = true;            # auto-emits VMAlertmanagerConfig
  };
};
```

Tier-flag implication chain enforced in `module/nixos/default.nix` via
`assertions`. Operator can disable upper tiers and keep lower tiers (e.g.
"give me T1 + T2 only, I'll write my own policy").

---

## SDLC + CI

Standard `caixa-validate` + `eval-nixos-module` checks run on every PR.
Tier-specific gates:

| Tier | CI gate |
|---|---|
| T1 | `nix build .#checks.x86_64-linux.kernel-builds` — confirms `pkgs.linux_selinux` builds |
| T1 | `nix build .#checks.x86_64-linux.vm-selinux-permissive` — boots a VM, greps `/sys/kernel/security/lsm` for "selinux" |
| T2 | VM test boots with `enforcing=0`, confirms `getenforce` returns `Permissive` |
| T3 | `policy/build.nix` runs `checkmodule` + `semodule_package` on every module; failure = red CI |
| T3 | Permissive-mode VM test boots a stock NixOS service (sshd, nginx) and confirms zero AVC denials in the audit log (regression: a policy update that breaks a stock service) |
| T4 | VM test runs a podman container with `-v /tmp:/data:Z`, confirms `getfattr` shows `container_file_t` on `/tmp` after the run |
| T4 | **MCS isolation test** — boots VM in enforcing mode, runs two podman containers (same image, same volume mount), asserts container A cannot read files written by container B. The proof that threat #3 / #5 are closed |

---

## Promotion path into the fleet

1. **M0 — repo lands** (this PR). T1 substantive, T2-T4 stubbed. CI green.
2. **M1 — rio adopts T1+T2 in permissive mode.** rio is the homelab
   playground; perfect target for "logs everything, blocks nothing." Sets
   up the audit-log-to-VictoriaLogs pipeline.
3. **M2 — Tier 3 base policy.** systemd + sshd + nix-daemon + journald
   covered. rio still in permissive mode but with policy loaded;
   `audit2allow`-driven iteration.
4. **M3 — Tier 3 podman policy + Tier 4.** Container labeling works
   end-to-end. rio flips one container to enforcing mode as the canary.
5. **M4 — rio in full enforcing mode.** Every service has a policy
   module. AlertmanagerConfig rules block fleet-wide on AVC spikes.
6. **M5 — Promote to akeyless-dev / openclaw clusters.** Compliance
   theorems become derivable. Cartorio receipts include `selinux:enforcing`
   posture as an attestable dimension.
7. **M6 — `SecurityPosture` typescape primitive lands.** Cluster
   declarations carry their LSM choice in the type system. Mechanical
   propagation across the fleet.

---

## Anti-patterns this plan forbids

- **Hand-rolling a kernel build outside the substrate.** Every kernel
  permutation goes through `kernel-lsm-flake.nix`. If the builder is
  missing a knob, fix the builder.
- **Hand-rolling a policy `.te` outside `policy/modules/`.** Every
  policy module ships in this repo so `audit2allow`-driven iteration is
  reproducible across operators.
- **Putting SELinux options under `services.blackmatter.selinux`.**
  Use `blackmatter.components.selinux` to match the rest of the
  blackmatter component namespace.
- **Enabling enforcing mode anywhere before M4.** Every operator who
  flips a node to enforcing mode without a tested policy generates a
  3-AM page. Permissive mode is the default and stays the default until
  the per-cluster policy graduates the test gate.
- **Calling `chcon` / `semanage` ad-hoc from a NixOS module.** Every
  label decision flows through the activation hook. Drift = unowned
  labels = silent denials in production.

---

## Inspirations + acknowledgements

- **Fedora's `selinux-policy-targeted`** — the reference for what a
  full-coverage policy looks like; not directly portable but the
  policy-module decomposition pattern is canonical.
- **`containers/container-selinux`** ([github.com/containers/container-selinux](https://github.com/containers/container-selinux))
  — the canonical podman/cri-o SELinux policy. Defines `container_t`,
  `container_file_t`, `container_runtime_t`, the MCS constraint, and
  the runc/crun/conmon transition rules. **Our T3 podman-host module
  ports this directly with NixOS-aware file contexts.** No re-invention
  — container_t in our policy is bit-compatible with Fedora's so any
  container image labeled by Fedora's container-selinux runs unchanged
  on a NixOS host with our policy.
- **abbradar's `nixos-selinux` branch (~2017–2019)** — earliest known
  attempt at NixOS SELinux. Not maintained; informative for the kernel
  config and `/etc/selinux/config` rendering.
- **NixOS's existing AppArmor module** (`security.apparmor.*`) — the
  shape we mirror for `security.selinux.*` (which we contribute back
  upstream once Tier 2 is stable).
- **`virtualisation.podman.enableSelinuxLabeling`** — option doesn't
  exist yet in nixpkgs; tracked as an upstream contribution.
