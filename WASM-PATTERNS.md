# WASM/WASI patterns on Kubernetes — the pleme-io cookbook

> **Frame.** [`WASM-STACK.md`](WASM-STACK.md) packages the runtime
> (programs / jobs / services / controllers — same `ComputeUnit` CR
> with different `spec.trigger`). This document catalogs **what to
> build on top.** Each pattern is a recurring shape across pleme-io
> + the broader K8s world that becomes a 2 MiB WASM module instead of
> a 200 MiB container, a multi-day kube-rs project, or a bash-and-prayer
> CronJob.
>
> **How to use.** Find a pattern that matches your need, copy the
> `ComputeUnit` skeleton, ship the WASM module. Most patterns are 50–200
> lines of source code; many are single-file tatara-lisp scripts.

---

## Pattern index

```
┌──────────────────────────────────────────────────────────────────┐
│ I.  Operational / Infrastructure                                  │
│     1. PVC autoresizer       2. DNS reconciler                    │
│     3. Backup scheduler      4. Drift detector                    │
│     5. Resource janitor      6. Image cache pre-puller            │
│     7. Cert renewal observer 8. Lease cleaner                     │
│                                                                    │
│ II. Glue / Integration                                            │
│     9. Webhook → FluxCD reconcile  10. Slack/ntfy notifier        │
│    11. Incident creator           12. CR → ticket bridge          │
│    13. State backend observer     14. Cross-cluster sync          │
│                                                                    │
│ III. Authoring / Code Generation                                  │
│    15. Repo scaffolder            16. Migration orchestrator      │
│    17. Chart-version bumper       18. Image promotion gate        │
│                                                                    │
│ IV.  Data / ETL                                                   │
│    19. Log → S3 archiver          20. Metrics downsampler         │
│    21. Database backup uploader   22. Event-source replayer       │
│                                                                    │
│ V.   Custom observability                                         │
│    23. Custom alert evaluator     24. SLO calculator              │
│    25. RCA assistant (LLM-driven) 26. Synthetic prober            │
│                                                                    │
│ VI.  Security / Compliance                                        │
│    27. Secrets rotator            28. Image-vuln admission gate   │
│    29. SOC2 evidence collector    30. Audit-log shipper           │
│                                                                    │
│ VII. Multi-step orchestration                                     │
│    31. Saga reconciler            32. Multi-step deploy           │
│    33. Workflow DAG runner        34. Conditional rollback        │
│                                                                    │
│ VIII. AI / LLM integration                                        │
│    35. LLM reconciler             36. Document indexer / RAG      │
│    37. Semantic search refresh    38. Agent-as-controller         │
│                                                                    │
│ IX.  Tenancy                                                      │
│    39. Tenant provisioner         40. Tenant cleanup              │
│    41. Tenant billing rollup                                      │
│                                                                    │
│ X.   Domain primitives                                            │
│    42. HTTP rate-limit service    43. Image thumbnailer            │
│    44. Webhook signature verifier 45. Form processor               │
│                                                                    │
│ XI.  Meta-patterns (compose the above)                            │
│    46. Pattern composition        47. Module bundle (multi-shape)  │
│    48. Hot-replacement            49. Rolling controller upgrade   │
└──────────────────────────────────────────────────────────────────┘
```

49 patterns. Each entry below: **name → shape → capabilities → 5-15
line ComputeUnit skeleton → "replaces" → notes.**

---

## I. Operational / Infrastructure

### 1. PVC autoresizer

Watch every PVC labelled `breathable=true`; expand to `1.25×` on >80%
utilization.

- **Shape:** job (cron `*/5 * * * *`)
- **Capabilities:** `kube-pvc-list`, `kube-pvc-patch`, `prom-query@vmsingle-vm`
- **Replaces:** the Rust binary in `helmworks/charts/pleme-storage-elastic/image/`.

```yaml
spec:
  module:    { source: oci://ghcr.io/pleme-io/programs:pvc-autoresizer-v0.1.0 }
  trigger:   { cron: "*/5 * * * *" }
  capabilities:
    - kube-pvc-list
    - kube-pvc-patch
    - prom-query@vmsingle-vm.monitoring.svc:8429
```

### 2. DNS reconciler

Sync `Service` and `Ingress` resources with annotation
`dns.pleme.io/host: <fqdn>` to a Cloudflare zone (or any DNS
provider with a typed-domain plugin).

- **Shape:** controller (watch Services + Ingresses)
- **Capabilities:** `kube-cr-watch@v1/services`, `kube-cr-watch@networking.k8s.io/ingresses`, `http-out:api.cloudflare.com`, `kube-secret-read@cloudflare/api-token`
- **Replaces:** ExternalDNS (heavyweight Go; pleme version is ~80 lines of tatara-lisp).

### 3. Backup scheduler

Take a restic snapshot of every PVC labelled `backup.pleme.io/policy: <name>`,
push to S3, prune by retention.

- **Shape:** job (cron, often `@daily`)
- **Capabilities:** `kube-pvc-list`, `kube-pod-exec`, `s3-write@<bucket>`, `kube-secret-read@backup/restic-passphrase`

### 4. Drift detector

Compare actual K8s state vs declared FluxCD state; emit
`DriftDetected` events on any divergence.

- **Shape:** job (cron `*/15 * * * *`)
- **Capabilities:** `kube-resource-list@*`, `git-fetch@github.com/pleme-io/k8s`, `event-emit`
- **Replaces:** ad-hoc `kubectl diff` shell scripts.

### 5. Resource janitor

Delete `Job`/`Pod`/`PVC` resources older than N days unless labelled
`pleme.io/keep=true`. Per-namespace policies via a `JanitorPolicy` CRD.

- **Shape:** job (cron `@hourly`)
- **Capabilities:** `kube-cr-watch@pleme.io/janitorpolicies`, `kube-pod-list`, `kube-job-list`, `kube-pvc-list`, `kube-resource-delete@<filtered>`

### 6. Image cache pre-puller

For every Pod spec that hasn't been pulled on a node, run a
`hostNetwork` DaemonSet that prefetches via `crictl pull`.

- **Shape:** controller (watch Pods + Nodes)
- **Capabilities:** `kube-cr-watch@v1/pods`, `kube-cr-watch@v1/nodes`, `node-runtime-pull`

### 7. Cert renewal observer

Watch `Certificate` CRDs; alert when a renewal fails twice in a row.

- **Shape:** controller (watch cert-manager.io/Certificate)
- **Capabilities:** `kube-cr-watch@cert-manager.io/certificates`, `event-emit`

### 8. Lease cleaner

Delete stale `coordination.k8s.io/Lease` records where the holder
hasn't renewed in N×TTL. Catches operator pods that crashed without
releasing their lease.

- **Shape:** job (cron `@hourly`)
- **Capabilities:** `kube-resource-list@coordination.k8s.io/leases`, `kube-resource-delete@coordination.k8s.io/leases`

---

## II. Glue / Integration

### 9. Webhook → FluxCD reconcile

Receive GitHub push webhooks at `/git-webhook`, verify signature, kick
the FluxCD `GitRepository` it maps to. Lets pushes propagate in <10s
instead of the default 5min poll.

- **Shape:** service (HTTP listener on `/git-webhook`)
- **Capabilities:** `http-in:0.0.0.0:8080`, `kube-resource-patch@source.toolkit.fluxcd.io/gitrepositories`, `kube-secret-read@flux-system/webhook-secret`

```yaml
spec:
  module:    { source: oci://ghcr.io/pleme-io/programs:flux-webhook-v0.2.0 }
  trigger:
    service:
      port: 8080
      paths: ["/git-webhook"]
      hosts: ["git-webhook.flux-system.svc.cluster.local"]
  capabilities:
    - http-in:0.0.0.0:8080
    - kube-resource-patch@source.toolkit.fluxcd.io/gitrepositories
    - kube-secret-read@flux-system/webhook-secret
  breathability: { enabled: true, minReplicas: 0, cooldownPeriod: 600 }
```

### 10. Slack / ntfy / Discord notifier

Subscribe to `Event` resources matching a label selector; format and
post to a chat webhook. One module, three transports.

- **Shape:** controller (watch v1/Events)
- **Capabilities:** `kube-cr-watch@v1/events`, `http-out:hooks.slack.com`, `http-out:ntfy.<...>`, `http-out:discord.com`

### 11. Incident creator

When a critical-severity Alertmanager alert fires, open a
PagerDuty / OpsGenie incident with deep-link references to the
relevant Grafana dashboard.

- **Shape:** service (Alertmanager webhook receiver)
- **Capabilities:** `http-in:0.0.0.0:9093`, `http-out:events.pagerduty.com`

### 12. CR → ticket bridge

Watch any CRD; for each spec change, create or update a JIRA / Linear
ticket. Useful for governance — "every infrastructure change is
tracked".

- **Shape:** controller
- **Capabilities:** `kube-cr-watch@<group>/<kind>`, `http-out:<atlassian/linear>`, `kube-secret-read@governance/jira-token`

### 13. State backend observer

Poll the Terraform state bucket for new state file revisions; emit a
`PangeaStateApplied` event the operator dashboards subscribe to.

- **Shape:** job (cron `*/2 * * * *`)
- **Capabilities:** `s3-list@<state-bucket>`, `event-emit`

### 14. Cross-cluster sync

Replicate selected resources (Secrets, ConfigMaps, CRs) from a hub
cluster to N edge clusters. One module, multiple `--target=<kubeconfig-secret>`
invocations.

- **Shape:** controller (hub-side) + job (edge-side scheduled pull)
- **Capabilities:** `kube-cr-watch@<filtered>`, `kube-secret-read@hub-system/edge-kubeconfigs`, `http-out:edge-cluster.api.example.com`

---

## III. Authoring / Code Generation

### 15. Repo scaffolder

When a `RepoSpec` CR is applied, call `repo-forge` to scaffold the
matching archetype, push to GitHub, return the URL.

- **Shape:** controller (watch repo.pleme.io/RepoSpec)
- **Capabilities:** `kube-cr-watch@repo.pleme.io/repospecs`, `http-out:api.github.com`, `kube-secret-read@repo-forge/github-token`

### 16. Migration orchestrator

Run shinka-style DB migrations on a schedule, gated by a
`Migration` CR's `spec.approvedAt` timestamp.

- **Shape:** controller + job
- **Capabilities:** `kube-cr-watch@shinka.pleme.io/migrations`, `db-connect:postgres@<url>`, `kube-secret-read@<ns>/db-creds`

### 17. Chart-version bumper

Poll upstream Helm repo indexes; for any chart used in `helmworks/charts/`
where a newer version exists, open a PR that bumps the dependency.

- **Shape:** job (cron `@daily`)
- **Capabilities:** `http-out:*.github.io`, `git-push@github.com/pleme-io/helmworks`, `kube-secret-read@bumper/github-token`

### 18. Image promotion gate

Listen for `ImagePromoted` events on the build pipeline; verify
attestation against tameshi; if green, retag in target environment's
registry; if red, refuse and emit a `PromotionRejected` event.

- **Shape:** controller (watch tameshi events)
- **Capabilities:** `kube-cr-watch@tameshi.pleme.io/attestations`, `oci-pull@<src>`, `oci-push@<dst>`, `kube-secret-read@build/registry-creds`

---

## IV. Data / ETL

### 19. Log → S3 archiver

Every hour, query VictoriaLogs for the last hour's logs from labels
matching a `LogArchive` CR; write a daily-rolled NDJSON to S3 with
a tameshi attestation chain.

- **Shape:** job (cron `@hourly`)
- **Capabilities:** `http-out:victoria-logs.monitoring.svc`, `s3-write@<archive-bucket>`, `tameshi-sign`

### 20. Metrics downsampler

Read 1-second resolution series older than 7 days from VMSingle;
write 5-minute averages back; delete the high-res originals.

- **Shape:** job (cron `0 4 * * *`)
- **Capabilities:** `http-out:vmsingle-vm`, `prom-write@vmsingle-vm`

### 21. Database backup uploader

`pg_dump | restic backup -` against an external Postgres instance,
upload to S3 with restic, prune by retention. Multi-tenant via a
`BackupTarget` CR per database.

- **Shape:** job (cron per spec)
- **Capabilities:** `kube-cr-watch@pleme.io/backuptargets`, `db-connect:postgres@<url>`, `s3-write@<bucket>`

### 22. Event-source replayer

Read events from a NATS / Kafka topic; replay them as K8s Events
during incident reconstruction. `--start-time` + `--end-time` flags.

- **Shape:** program (one-shot, manual invocation)
- **Capabilities:** `nats-subscribe@<subject>`, `event-emit`

---

## V. Custom observability

### 23. Custom alert evaluator

Express alert logic that PromQL can't (e.g., "alert if more than
3 distinct error codes appeared in the last hour"). Reads
VictoriaLogs / VMSingle, fires `Alert` CRs the alertmanager
picks up.

- **Shape:** job (cron `*/30 * * * *`)
- **Capabilities:** `http-out:victoria-logs.monitoring.svc`, `http-out:vmsingle-vm`, `kube-cr-write@monitoring.coreos.com/alerts`

### 24. SLO calculator

Compute error-budget burn rates for a `ServiceLevelObjective` CR;
fire `SloBudgetExhausting` when burn rate > 14× sustained 1h.

- **Shape:** controller (watch slo.pleme.io/serviceleveobjectives) + cron job
- **Capabilities:** `kube-cr-watch@slo.pleme.io/slos`, `prom-query@vmsingle-vm`

### 25. RCA assistant (LLM-driven)

When a critical alert fires, gather logs / metrics / dashboards into
a context bundle, send to Claude / Anthropic API, return suggested
root cause analysis as an annotation on the alert.

- **Shape:** controller (watch Alertmanager webhook events)
- **Capabilities:** `kube-cr-watch@monitoring.coreos.com/alerts`, `http-out:victoria-logs.monitoring.svc`, `http-out:api.anthropic.com`, `kube-secret-read@rca/anthropic-key`

### 26. Synthetic prober

Hit a list of HTTP endpoints from a `SyntheticProbe` CR; emit
latency / status metrics; fire `ProbeFailed` events.

- **Shape:** job (cron `*/5 * * * *`) or service (continuous)
- **Capabilities:** `kube-cr-watch@probes.pleme.io/syntheticprobes`, `http-out:*`, `prom-write@vmsingle-vm`

---

## VI. Security / Compliance

### 27. Secrets rotator

For each `Secret` labelled `rotation.pleme.io/policy: <name>`, generate
a fresh value, update upstream (Akeyless / AWS SM / Vault), patch the
in-cluster Secret, restart Deployments referencing it.

- **Shape:** job (cron per policy) + controller
- **Capabilities:** `kube-cr-watch@pleme.io/rotationpolicies`, `kube-secret-read+write@*`, `http-out:<vault-provider>`, `kube-deployment-restart@*`

### 28. Image-vulnerability admission gate

Validating webhook: on `Pod` create, look up the image's
VulnerabilityReport CR; reject if any CRITICAL CVEs are unfixed.

- **Shape:** service (admission webhook)
- **Capabilities:** `http-in:0.0.0.0:8443`, `kube-cr-watch@aquasecurity.github.io/vulnerabilityreports`

### 29. SOC2 evidence collector

Daily snapshot of cluster state — RBAC, NetworkPolicies, image
sources, audit log samples — to a content-addressed S3 path. tameshi
attestation chain proves nothing was modified after capture.

- **Shape:** job (cron `@daily`)
- **Capabilities:** `kube-resource-list@*`, `s3-write@<evidence-bucket>`, `tameshi-sign`

### 30. Audit-log shipper

Tail the cluster's audit log; redact known-secret patterns; ship to
Splunk / Datadog / S3 with delivery acknowledgement.

- **Shape:** service (always-on ingest)
- **Capabilities:** `audit-log-tail`, `http-out:<sink>`, `regex-redactor`

---

## VII. Multi-step orchestration

### 31. Saga reconciler

For every `Saga` CR, walk the declared step list, invoking each
step's WASM module in sequence. On step N failure, run compensation
steps from N back to 1.

- **Shape:** controller (watch saga.pleme.io/Saga)
- **Capabilities:** `kube-cr-watch@saga.pleme.io/sagas`, `kube-cr-write@wasm.pleme.io/computeunits` (recursively dispatches sub-modules)

### 32. Multi-step deploy

Provision in dependency order: cert → DNS → ingress → app. Each step
is a sub-ComputeUnit that the parent waits on.

- **Shape:** controller
- **Capabilities:** `kube-cr-watch@deploy.pleme.io/multidots`, `kube-cr-write@wasm.pleme.io/computeunits`

### 33. Workflow DAG runner

Read a `Workflow` CR with a typed-DAG of steps + dependencies;
schedule each step as a `ComputeUnit`; track progress.

- **Shape:** controller
- **Capabilities:** `kube-cr-watch@workflow.pleme.io/workflows`, `kube-cr-write@wasm.pleme.io/computeunits`

### 34. Conditional rollback

When a deploy emits a `RollbackOnError` annotation and a step fails,
unwind to the last known-good state. Pairs with the Saga reconciler.

- **Shape:** controller
- **Capabilities:** `kube-cr-watch@deploy.pleme.io/*`, `kube-cr-write@deploy.pleme.io/*`

---

## VIII. AI / LLM integration

### 35. LLM reconciler

Watch a `LlmTask` CR with a natural-language goal; an LLM agent
proposes a plan, the operator validates against typed schemas,
executes the plan as a chain of sub-ComputeUnits.

- **Shape:** controller
- **Capabilities:** `kube-cr-watch@llm.pleme.io/tasks`, `http-out:api.anthropic.com`, `kube-cr-write@wasm.pleme.io/computeunits`

### 36. Document indexer / RAG builder

For a `RagSource` CR pointing at a git repo or S3 prefix, periodically
re-index documents into a vector DB; expose `/query` endpoint for
agent retrieval.

- **Shape:** job (cron) + service (query endpoint)
- **Capabilities:** `git-fetch@<source>` OR `s3-list@<bucket>`, `http-out:<vector-db>`, `http-in:0.0.0.0:8080`

### 37. Semantic search refresh

Re-embed a corpus when the embedding model version changes. One-shot,
parameterized by `--model=<id>`.

- **Shape:** program
- **Capabilities:** `http-out:<embedding-api>`, `s3-rw@<corpus-bucket>`

### 38. Agent-as-controller

A long-running LLM agent that watches a domain CR, reasons about
state, emits typed proposals on a `Proposal` CR. Operator reviews
and approves; on approval, the proposal becomes a saga.

- **Shape:** controller (long-lived; LLM session keeps state)
- **Capabilities:** `kube-cr-watch@*`, `http-out:api.anthropic.com`, `kube-cr-write@governance/proposals`

---

## IX. Tenancy

### 39. Tenant provisioner

Apply a `Tenant` CR; the controller creates a Namespace, RBAC
bindings, ResourceQuota, NetworkPolicy, default SOPS secrets, and
GitOps source pointers.

- **Shape:** controller
- **Capabilities:** `kube-cr-watch@platform.pleme.io/tenants`, `kube-namespace-create`, `kube-rbac-write@*`, `kube-resourcequota-write@*`, `kube-networkpolicy-write@*`

### 40. Tenant cleanup

When a Tenant CR is deleted, scrub everything that references it —
Namespace, persistent storage, S3 prefix, observability series — with
a 30-day grace period via finalizers.

- **Shape:** controller (with finalizer)
- **Capabilities:** `kube-cr-watch@platform.pleme.io/tenants`, `kube-namespace-delete`, `s3-delete@<tenant-prefix>`

### 41. Tenant billing rollup

Daily aggregation of resource usage per tenant (CPU-seconds,
memory-byte-hours, GiB-PVC-hours, GiB-egress) → a `TenantBill` CR.

- **Shape:** job (cron `@daily`)
- **Capabilities:** `prom-query@vmsingle-vm`, `kube-cr-write@billing.pleme.io/tenantbills`

---

## X. Domain primitives

### 42. HTTP rate limit service

Sidecar / standalone service implementing token bucket per IP / per
header. WASM+WASI Preview 2 gives precise CPU accounting; lighter
than nginx's lua-resty-limit-traffic by 5×.

- **Shape:** service
- **Capabilities:** `http-in:0.0.0.0:8080`, `kv-rw@redis.<ns>.svc:6379`

### 43. Image thumbnailer

POST `/v1/resize?w=<width>&fit=<contain|cover>` with image bytes →
JPEG/WebP/AVIF thumbnail. Handles pleme-io's photo-archive use case.

- **Shape:** service
- **Capabilities:** `http-in:0.0.0.0:8080`, `s3-read@<originals>`, `s3-write@<thumbnails-cache>`

### 44. Webhook signature verifier

Generic webhook receiver: validate HMAC / OAuth signature, forward
to the underlying handler if green, reject with detailed reason if
red. One module per provider (GitHub, Stripe, Linear, etc.).

- **Shape:** service
- **Capabilities:** `http-in:0.0.0.0:8080`, `http-out:<downstream>`, `kube-secret-read@webhooks/<provider>-secret`

### 45. Form processor

Receive multipart form POSTs, validate against a JSON Schema, route
to a typed downstream (database insert, email send, ticket create).

- **Shape:** service
- **Capabilities:** `http-in:0.0.0.0:8080`, `kube-cr-watch@forms.pleme.io/formspecs`, `db-connect:postgres@<url>`

---

## XI. Meta-patterns

### 46. Pattern composition

A `ComputeUnit` whose module imports two or more other modules via
component-model `wit-bindgen`. Patterns become Lego bricks.

```yaml
spec:
  module:
    components:
      - source: oci://.../webhook-verifier:v1
        instance_name: verifier
      - source: oci://.../slack-notifier:v1
        instance_name: notifier
      - source: oci://.../my-glue-logic:v1
        instance_name: glue
        wires:
          - { from: verifier.verified, to: glue.input }
          - { from: glue.output,       to: notifier.send }
```

The wasm-engine resolves the wires at instantiation. Composition
lives in the CR, not in source code.

### 47. Module bundle

Multiple shapes from one source repo, built into one OCI artifact
with multiple manifest entries. `program`, `controller`, and `service`
all derived from the same domain types.

### 48. Hot replacement

Push a new version of the module to the registry; the operator
notices the new BLAKE3 hash; instantiates the new module side-by-side
with the old; promotes once health checks pass; reaps the old.

- **Shape:** controller behavior, automatic per `module.spec.upgradePolicy`
- **Capabilities:** the operator's, not the module's

### 49. Rolling controller upgrade

For controller-shape ComputeUnits, the wasm-operator can replace one
running watch loop with another without dropping events — the new
instance reuses the existing informer cache via shared-state IPC.

- **Built into the operator;** users opt in via `spec.controller.zeroDowntime: true`.

---

## XII. Mapping patterns to existing pleme-io repos

Many patterns above replace or absorb existing pleme-io tools. As the
WASM stack matures, the conversion order is:

| Pattern # | Replaces | Migration trigger |
|---|---|---|
| 1 | `helmworks/charts/pleme-storage-elastic/image/` (Rust binary) | First WASM dogfood — once `wasm-platform` images publish |
| 9 | `seibi`'s git-webhook handler (if present) | When seibi's bash → tatara-lisp conversion lands |
| 10 | `falcosidekick` webhook → ntfy | When custom Falco filtering is needed |
| 17 | bash `chart-bumper.sh` (per `SCRIPTING.md` migration backlog) | Class 1 priority |
| 18 | `forge`'s image-promotion bash (if any) | Class 1 priority |
| 27 | `akeyless-nix`'s rotation hooks | When akeyless secrets need K8s-side rotation visibility |
| 35–38 | Various LLM-driven pleme-io tools (eventually) | After basic patterns proven |

---

## XIII. Authoring strategy

For each pattern above:

1. **Author once** in tatara-lisp (preferred, per [`SCRIPTING.md`](SCRIPTING.md))
   or Rust (when typed Rust libraries pull) or Go / Python.
2. **Compile to `wasm32-wasi`.**
3. **Publish** to ghcr.io/pleme-io/programs (or any OCI registry) by
   content hash.
4. **Apply** a `ComputeUnit` CR with the right `spec.trigger`.

Every pattern has a typed contract via `wit-bindgen` interfaces. The
`wasm-types::ComputeUnit::module.contract` field declares what
interfaces the module implements; the operator refuses to schedule a
module that doesn't satisfy the trigger's required contract.

## XIV. See also

- [`WASM-STACK.md`](WASM-STACK.md) — the runtime
- [`SCRIPTING.md`](SCRIPTING.md) — tatara-lisp authoring
- [`BREATHABILITY.md`](BREATHABILITY.md) — fleet-wide use-causes-spin-up
- [`THEORY.md` Pillar 1](THEORY.md#pillar-1-language) — language constraint
- [`tatara`](https://github.com/pleme-io/tatara) — the convergence computer
  that orchestrates patterns at the inter-program level (saga / multi-step
  patterns above all benefit from tatara DAG support).

---

## XV. Patterns NOT yet covered (next pass)

- Network-mesh patterns (sidecar mesh injection, service-to-service
  mTLS rotation, traffic shadow / canary).
- Storage patterns (replication, snapshot consistency, encryption
  key rotation per PVC).
- Multi-cluster federation patterns (cluster-bootstrap reconciler,
  cross-cluster CRD sync, federated CRD admission).
- Cost-control patterns (auto-rightsizing recommender, idle workload
  suspender, per-tenant cost projection).
- Edge / IoT patterns (MQTT bridges, Modbus polling, edge-aggregator
  with tail-only sync to hub).

These deserve their own document once the core 49 are dogfooded.
