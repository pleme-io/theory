# Breathability — the use-causes-spin-up architecture

> **Frame:** This document expands [Pillar 11 (JIT Infrastructure)](THEORY.md#vii3-jit-infrastructure-pillar-11)
> from a per-compute-unit invariant into a **fleet-wide architectural pattern**.
> The unifying claim: *every layer of the platform must scale to zero (or near-zero)
> when idle, and the act of using something must be the trigger that wakes only the
> infrastructure required to serve that use.*

---

## I. The principle

**Use causes spin-up. Disuse causes spin-down. Storage elastically tracks demand.**

In an idle homelab cluster, the resident footprint is a small set of
*lightweight always-on listeners* (ingress controllers, DaemonSet
collectors, controllers/operators) plus persistent state. Heavy work
(query planners, build fleets, reconcilers, dashboard renderers,
analytics planes) is *not running*. When a real request arrives, the
listener wakes the chain of infrastructure it actually needs, work
gets done, and the chain decays back to zero after a cooldown.

Storage is the asymmetric layer: it cannot scale to zero (data must
persist), and it cannot easily shrink (filesystems hate it). Instead,
storage is *elastically expanded* to keep utilization in a tight band
(default 80% used / 20% headroom), and the expansion is itself an
on-demand act — driven by a watcher that observes utilization and
calls the storage class's `allowVolumeExpansion`.

This is **strictly stronger** than Pillar 11. Pillar 11 says *every
compute unit is breathable*. This document says *every layer of the
fleet that can be breathable, is — and the trigger to wake any layer
is the demand signal from the layer above it*.

## II. Layers and their breathability profile

```
┌─────────────────────────────────────────────────────────────────────┐
│ user / agent                                                         │  ← origin of demand
└────────────────────────────┬────────────────────────────────────────┘
                             │ (HTTP request, git push, MCP call)
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│ EDGE: Cloudflare Tunnel + ingress-nginx                              │  ← always on (tiny)
│   resident: ~50 MiB. Routes the demand signal in.                    │
└────────────────────────────┬────────────────────────────────────────┘
                             │
            ┌────────────────┼────────────────────┐
            ▼                ▼                    ▼
┌───────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ KEDA HTTP add-on  │ │ FluxCD          │ │ pangea-operator │
│ (HTTP demand-gate)│ │ (git→CR)        │ │ (CR→WorkUnit)   │
│ resident: tiny    │ │ resident: small │ │ resident: small │
└─────────┬─────────┘ └────────┬────────┘ └────────┬────────┘
          │ HTTP request       │ git change       │ pending WorkUnit
          ▼                    ▼                  ▼
┌──────────────────────────────────────────────────────────────┐
│ ON-DEMAND WORK TIER (0..N pods, scale 0→N→0 by KEDA)          │
│   - Grafana   (HTTP-driven; cold ~10s)                        │
│   - workers    (CR-driven; cold ~30s)                          │
│   - vector-aggregator  (event-rate-driven)                     │
│   - vmalert / sift-investigation (alert-driven)                │
│   - shinryu queries / tatara-lisp REPLs (analytical)           │
└─────────────────────────────────┬────────────────────────────┘
                                  │ writes
                                  ▼
┌──────────────────────────────────────────────────────────────┐
│ ELASTIC STATE TIER (always present, expands on demand)        │
│   - VMSingle (metrics) → PVC, retention drops oldest          │
│   - VictoriaLogs       → PVC, similar                         │
│   - shinryu DuckDB     → PVC                                  │
│   - PostgreSQL         → PVC                                  │
│   - object stores      → S3 / R2 (infinitely elastic)         │
│ Auto-expand when utilization > 80%; never delete data.        │
└──────────────────────────────────────────────────────────────┘
```

### Layer policy table

| Layer | Policy | Idle cost | Wake trigger | Cooldown |
|---|---|---|---|---|
| Edge (CF Tunnel + ingress-nginx) | always-on, lightweight | ~50 MiB | n/a | n/a |
| Listeners (FluxCD, operators, KEDA, ESO) | always-on, lightweight | ~300 MiB | n/a | n/a |
| Vector collector (DaemonSet — node-level scrape) | always-on, lightweight | ~50 MiB/node | n/a | n/a |
| Vector aggregator (Deployment) | scale 0→N→0 | 0 | collector queue depth > 0 | 5 min |
| Grafana | scale 0→1→0 (HTTP-driven) | 0 | inbound request | 10 min |
| Pangea workers | scale 0→N→0 (CR-driven) | 0 | pending WorkUnits | 5 min |
| Sift investigations | scale 0→1→0 | 0 | alert match | 30 min |
| shinryu / tatara REPLs | scale 0→1→0 | 0 | MCP / SQL query | 5 min |
| State tier (VMSingle, VLogs, PG, shinryu) | always-on, elastic | ~PVC base | n/a (always running) | n/a |
| Builders (ASG, x86_64 / aarch64) | scale 0→N→0 (cordel-wake) | 0 | SSH ProxyCommand | 90 s |

The vertical stack is intentional: a request from the user enters the
edge (always on), reaches a listener (always on, cheap), the listener
wakes one or more on-demand workers, the workers write to elastic
state, and within minutes everything except the always-on tier is
back at zero. **An idle cluster's resident footprint is the always-on
column plus persistent state — nothing else.**

## III. Workspace shape determines breathability class

The pleme-io fleet runs ~50 Pangea workspaces. They are not fungible.
Naïvely classifying them as `small/medium/large` by CPU/memory misses
the structural differences that matter for scaling decisions:

### III.1 Provider plugin set

Tofu provider plugins are heavy (~50–200 MiB each, downloaded per
fresh worker pod). A workspace's provider set determines:

- **Cold-start cost.** AWS-only → ~80 MiB pull. Multi-cloud (AWS +
  Cloudflare + Akeyless + Hetzner) → ~400 MiB pull. The latter doubles
  the cold-start latency.
- **Cache reuse.** Workers that just ran an AWS workspace can serve
  the next AWS workspace with zero re-download. KEDA's stateless
  scaling assumption breaks the cache.
- **Lock contention.** Multi-provider workspaces hold more state
  locks (S3 + DynamoDB + provider-side rate limits).

**Implication:** WorkClasses must be parameterized by provider set,
not just resources. A `multi-cloud-medium` class is meaningfully
different from `aws-medium`.

Cataloged across the actual workspaces:

| Provider profile | Workspaces | Class |
|---|---|---|
| AWS-only | k3s-permissions, drill-iam, cluster-autoscaler-iam, platform-vpc, platform-iam, eks-scale-test, akeyless-platform, akeyless-dev-cluster | `aws-{small,medium,large}` |
| AWS + Packer | akeyless-dev-packer, platform-packer, platform-builder-fleet, platform-cache, platform-ami-seed | `aws-packer-large` |
| AWS + Cloudflare | platform-dns, pleme-dns, example-com-dns | `aws-cloudflare-small` |
| Cloudflare-only | cloudflare-pleme | `cloudflare-small` |
| Akeyless API | akeyless-community-public, akeylesslabs-public | `akeyless-small` |
| Multi-cloud (AWS + CF + AK) | inception, pleme-io-opensource | `multi-cloud-medium` |
| Grafana HTTP | rio-observability | `http-tiny` |

### III.2 Time profile (expected duration)

Apply duration varies by 100×:

- **Tiny (<30 s):** DNS records, IAM updates, Grafana dashboards.
- **Small (30 s – 3 min):** Single-stack VPC, IAM bundles, single
  S3 bucket + KMS.
- **Medium (3 – 15 min):** Cluster bootstraps, EKS managed node
  groups, multi-resource architectures.
- **Long (15 – 60 min):** AMI/Packer builds, full cluster bringup,
  multi-region replication.

KEDA's default cooldown of 5 min is wrong for both ends:

- A long Packer build that started 4 min ago shouldn't be
  preempted because the queue drained. The worker holds an active
  lease.
- A tiny DNS apply that took 20 s shouldn't keep its worker around
  for 5 min after.

**Implication:** Cooldown is a per-class, lease-aware decision.
Short classes have short cooldown (60 s); long classes hold the
worker as long as the lease is live + 30 s grace.

### III.3 Lock contention pattern

State locks (S3 + DynamoDB) serialize concurrent applies on the same
workspace. The pending WorkUnit count overstates *dispatchable* work:
a queue of 10 pending units for the same parent CR can only progress
one-at-a-time.

**Implication:** The KEDA trigger reads
`pangea_workunit_dispatchable_total` (controller-emitted gauge that
excludes lock-blocked units) — not the raw `pending` count.

### III.4 Cross-workspace dependencies

`platform-eks` reads outputs from `platform-vpc`. If both reconcile
in the same fleet push, `platform-eks` is in `Pending` until
`platform-vpc` finishes — but the queue depth shows 2.

**Implication:** Controller marks WorkUnits with `blockedOn` refs.
KEDA sees only the unblocked subset.

### III.5 Per-account isolation

`akeyless-prod` and `akeyless-dev` use distinct AWS accounts and
distinct sets of credentials. Mixing them on the same worker pod
reduces secrets blast radius.

**Implication:** WorkClass admits an optional `account` axis.
Workers for `akeyless-prod` never see `akeyless-dev` creds. Security-
conscious clusters opt in; cost-conscious clusters share.

## IV. Vector pipeline as a breathability case study

The naive pattern is: Vector DaemonSet (always on per node) →
VMSingle (always on) → Grafana (always on). Resident at idle on
rio: ~600 MiB.

The breathable pattern splits the pipeline:

```
Always-on tier (~50 MiB / node):
   Vector collector — reads /var/log/pods/, kubernetes_events,
   statsd UDP, http_server. Buffers to disk.
   Sinks: ONE single sink — the aggregator's Vector ingest
   endpoint, with an http buffer that grows on backpressure.

On-demand tier (0..N pods, KEDA-scaled):
   Vector aggregator — heavy transforms (enrichment, sampling,
   format conversion), routing decisions, tail to VictoriaLogs +
   write to VMSingle remote_write.
   Wake when collector buffer depth > threshold OR aggregator
   service has pending TCP connections.
   Drain when collector buffer < threshold AND aggregator has
   no inflight requests.
   Cooldown: 5 min.
```

Idle resident: ~150 MiB across 3-node rio (50 MiB × 3 collectors).
Active burst: scales aggregator 0→N, drains the buffer, scales
back.

The collector's disk buffer is the **decoupling capacitor**: if the
aggregator is briefly down (cold start, scaling hiccup), the
collector keeps accepting events to disk. No data loss.

## V. Storage elasticity at 80%

Storage layers can't scale to zero (data persists), so the
breathability pattern is *elastic expansion*:

```
target_utilization: 80%
headroom: 20%
expand_step: 25%      ← so post-expansion utilization is 64%
poll_interval: 60s
```

### V.1 PVC-backed services (VMSingle, VictoriaLogs, shinryu, PG)

Watcher pod (`pleme-storage-elastic`, scale 0→1 on schedule, runs
once / 60s):

```
for each PVC bound to a watched StatefulSet:
  capacity   = pvc.spec.resources.requests.storage
  used       = kubelet metric: kubelet_volume_stats_used_bytes
  if used / capacity > 0.80:
    new_capacity = capacity × 1.25
    kubectl patch pvc <name> --type merge \
      -p '{"spec":{"resources":{"requests":{"storage":"<new_capacity>"}}}}'
    record_event PVCExpanded
```

Requirements:
- StorageClass with `allowVolumeExpansion: true` (CSI-supported).
- The watcher requires `patch` on PVCs (RBAC).
- The first time a PVC expands triggers a filesystem resize — most
  CSI drivers handle this online (no pod restart).

### V.2 Local ZFS pools (rio-class single-node clusters)

ZFS doesn't auto-expand by adding disks. Instead:

- Watch utilization per pool (`zpool list`).
- At 80%: emit a `PVCExpanded` event to the operator's audit log
  AND fire `RioStorageExpansionRequired` warning to ntfy
  (rio-warning topic).
- Operator physically adds a disk → `zpool add tank mirror disk1 disk2`.
  No Pangea-side automation possible (the disk doesn't exist).

### V.3 Object stores (S3, R2)

Effectively infinite. Track spend, not space. A separate
`storage-spend` watcher emits warnings when monthly spend exceeds a
budget. Budget itself is breathable: alarm thresholds rise with
business-justified usage.

### V.4 Retention as last-resort eviction

When elastic expansion cannot keep up (storage class out of capacity
on the underlying nodes), retention kicks in:

- VMSingle: drops oldest data first. We accept this; it's the
  documented behavior.
- VictoriaLogs: same.
- shinryu DuckDB: archives old partitions to S3 via a CronJob,
  drops local copies. Older queries are slower but still possible.

This is the breathability invariant's escape hatch: when expansion
fails, the system degrades gracefully (older data accessed slower)
rather than refusing writes.

## VI. The activation chain catalog

A taxonomy of "use → spin-up → work → drain" patterns, each tuned
to the layer's characteristic latency:

### VI.1 User-driven HTTP request

```
User opens https://grafana.rio.quero.cloud
  → Cloudflare Tunnel (always on)
  → ingress-nginx (always on)
  → Grafana Service (selects 0 pods)
  → KEDA HTTP add-on intercepts (the http-add-on's interceptor)
  → KEDA scales Grafana 0→1
  → Grafana ready ~10 s
  → request served
  → after 10 min idle: scales 0
```

Cold start latency (~10 s) is the user-facing SLO concern. For
operator dashboards consulted ad-hoc, this is acceptable. For
always-watched dashboards (PagerDuty/ntfy follow-up links), pin
`minReplicas: 1` on those specific clusters.

### VI.2 Git-push-driven workspace reconcile

```
git push to pangea-architectures main
  → FluxCD reconciles (5 min poll, or webhook)
  → applies changed CR (PangeaArchitecture / InfrastructureTemplate)
  → pangea-operator observes
  → creates PangeaWorkUnit
  → KEDA scales matching worker class 0→1
  → worker claims unit, runs synth + tofu plan/apply
  → reports status
  → after 5 min cooldown: scales 0
```

Cold-start (~30 s) is amortized over multi-minute reconcile work.
Reduce via shared provider-plugin PVC (~5 s saved per worker boot).

### VI.3 Event-rate-driven log surge

```
A misbehaving service spikes logs 100× normal
  → Vector collector buffer grows
  → buffer-depth metric crosses threshold
  → KEDA scales Vector aggregator 0→N
  → aggregators drain buffer, write to VictoriaLogs
  → buffer recovers to normal
  → after cooldown: aggregators scale 0
```

The collector's disk buffer is what allows the aggregator to be
zero-resident: surges are buffered locally, never lost, and the
aggregator wakes to drain.

### VI.4 Alert-driven investigation

```
vmalert fires `PangeaWorkUnitsBacklog`
  → Alertmanager → ntfy webhook (rio-warning)
  → operator's phone vibrates
  → operator opens runbook URL → linked Grafana dashboard +
    sift investigation deeplink
  → KEDA scales Sift investigation pod 0→1
  → investigation runs, posts findings
  → after 30 min: scales 0
```

The chain composes: alert wakes investigation; investigation wakes
Grafana; Grafana wakes the underlying datasource queries (which are
themselves queries against an always-on VMSingle).

### VI.5 MCP / agent-driven query

```
Claude Code agent calls mcp__shinryu__query
  → MCP server (always on, scale 0→1 if not running)
  → shinryu DuckDB (always on, light)
  → result returned to agent
  → MCP server scales 0 after 5 min idle
```

Same shape as HTTP: cold start ~5 s, then served live, drains to
zero shortly after.

## VII. Renderer invariants for breathability

These are *rendering invariants* — pangea / helm / forge-gen
renderers should refuse to emit a workload that violates them:

1. **Every Deployment/StatefulSet declares a breathability tier**:
   `always-on-tiny` | `always-on-medium` | `scale-to-zero` |
   `scale-to-zero-stateful` | `elastic-storage`.

2. **Every `scale-to-zero` workload declares a wake trigger** (KEDA
   ScaledObject) and a cooldown.

3. **Every `elastic-storage` workload declares an expansion policy**
   (target utilization, expand_step, max_size).

4. **Every always-on workload declares a resource budget** (CPU,
   memory). The fleet's total always-on budget is bounded by a
   cluster-wide limit (typed in the cluster's PangeaCluster CR);
   exceeding it is a render-time error.

5. **Every wake chain has an `activationDescription:`** field — a
   human-readable string that an operator reading runbooks can
   follow. ("User opens dashboard → Grafana wakes → reads VMSingle →
   in 30 s the operator sees data.")

6. **Cooldowns are tested.** A breathable workload's chart includes
   a `tameshi`-attested test asserting that idle = 0 pods after
   `cooldownPeriod + slack`. Failing this test fails the gate.

7. **Helm-first authoring.** *Every in-cluster workload is expressed
   as a Helm chart consuming `pleme-lib` (and its sibling library
   charts: `pleme-operator`, `pleme-worker`, `pleme-statefulset`,
   `pleme-cronjob`, `pleme-microservice`, `pleme-web`).* Raw
   `HelmRelease` blocks with values inline in the FluxCD tree are
   drift — they bypass the breathability + observability defaults
   the library charts encode. When wrapping an upstream chart
   (Authentik, KEDA, FluxCD, cert-manager…), do it via an umbrella
   chart that:
     - declares the upstream chart as a `dependencies:` entry,
     - adds the pleme-lib-flavored sidecar resources on top
       (ScaledObject for breathability, ServiceMonitor + PrometheusRule
       for observability, NetworkPolicy when `networkPolicy.enabled`),
     - exposes a *minimal opinionated values surface* — any defaults
       the cluster wouldn't routinely override stay inside the umbrella.
   Pangea-rendered Terraform JSON is the only acceptable alternative
   to Helm, and only for resources outside the cluster (cloud-side IaC,
   edge SSO config like Cloudflare Zero Trust, DNS records). Anything
   running *inside* the cluster goes through Helm.

   Renderer enforcement: a `PangeaArchitecture` whose target is
   `kubernetes` and whose closure includes a raw `HelmRelease` with
   inline `values:` should fail validation with a hint to the matching
   library chart. Exception list lives in
   [`pangea-architectures/CLAUDE.md`](../pangea-architectures/CLAUDE.md)
   under "approved raw-Helm exceptions" (currently empty).

## VIII. Open frontiers

- **Predictive pre-warming.** A scheduled fleet push at 09:00 UTC
  could pre-warm the AWS-medium worker class at 08:55 to avoid
  cold-start tax on the first 10 reconciliations. The schedule
  itself is a CRD; pre-warm is opt-in per workspace.

- **Cross-cluster breathability.** A request to `seph` (DR cluster)
  could wake `lilitu` (prod) replication for a 1-hour window, then
  drain. Today the link is always on.

- **Tameshi for breathability invariants.** Add a layer to the
  attestation chain that proves "this workload renders correctly
  to a 0-replica idle state". Today renderers can violate VII.1–6
  silently.

- **Storage tier promotion.** When a PVC expands beyond a threshold
  (e.g., 1 TiB), the watcher could trigger a promotion to a
  different storage class (NVMe → SSD-backed → S3-backed cold).
  Cold tier is implicitly breathable (S3 = infinitely elastic).

- **Per-account workers without operational toil.** Currently per-
  account requires manual class declaration. A typed
  `PangeaAccountClass` could derive both the worker pool *and* the
  IAM role/policy bindings the worker assumes.

## IX. See also

- [`THEORY.md` §VII.3 (Pillar 11)](THEORY.md#vii3-jit-infrastructure-pillar-11) — the
  per-compute-unit invariant this doc generalizes.
- [`pangea-operator/docs/design/0003-helmworks-chart-and-breathable-workers.md`](../pangea-operator/docs/design/0003-helmworks-chart-and-breathable-workers.md) — the worker-pool design.
- [`pangea-operator/docs/design/0004-workspace-aware-breathability.md`](../pangea-operator/docs/design/0004-workspace-aware-breathability.md) — refines 0003 with the workspace-shape insights from §III.
- [`blackmatter-pleme/skills/pangea-jit-builders/SKILL.md`](../blackmatter-pleme/skills/pangea-jit-builders/SKILL.md) — the JIT fleet pattern this generalizes.
- [`helmworks/charts/pleme-vector/`](../helmworks/charts/pleme-vector/) — the Vector
  collector + aggregator split (TODO: implement aggregator tier).
- [`helmworks/charts/pleme-storage-elastic/`](../helmworks/charts/pleme-storage-elastic/) —
  the PVC autosize watcher (TODO: build).
