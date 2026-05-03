# Chart-adoption playbook: migrating raw manifests to a Helm chart safely

When a Flux-Kustomization-managed set of raw manifests transitions to
a `HelmRelease` pointing at a chart that emits the same resources,
**do it as a multi-commit sequence — never a single atomic commit.**
The 2026-05-03 pangea-database incident proved this is a real
footgun, not theoretical.

## The race

Flux runs two reconciliation loops in parallel that don't share
ownership semantics:

* `kustomize-controller` — applies Kustomization resources, default
  `prune: true` deletes resources that disappear from the source.
* `helm-controller` — applies HelmRelease, runs `helm install/upgrade`
  with Helm 3's 3-way merge.

If a single commit (a) removes a raw manifest from the Kustomization
source AND (b) adds a HelmRelease that renders the same resource,
the two controllers see contradictory desires:

* Kustomize: resource is *gone* from source → prune it.
* Helm: resource is *new* in chart render → install it.

Whichever controller acts first wins the race. In the pangea-database
incident, Kustomize pruned a `Cluster` CR (CNPG cascade-deleted PVCs)
before Helm fully owned it; Helm then installed a fresh Cluster CR
that triggered `bootstrap.initdb` against new empty PVCs.
**Result: data loss prevented only by the WAL archive.**

## The four-commit sequence

Each commit is reconciled independently before the next is pushed.
Run `flux reconcile kustomization <ks>` between commits to force a
known-good state.

### Commit A — disable Kustomization prune for the affected directory

```yaml
# k8s/clusters/<cluster>/flux-kustomizations/<ks>.yaml
spec:
  prune: false   # was: true
```

Push, reconcile. Verify `kubectl get kustomization <ks>` shows the
new generation as Applied.

**Alternatively:** keep `prune: true` cluster-wide and add the
annotation `kustomize.toolkit.fluxcd.io/prune: disabled` on each
specific resource that's about to migrate. This is finer-grained —
preferable when only some resources in the Kustomization are
migrating.

### Commit B — pre-apply Helm adoption metadata on the live raw resources

The `kubectl annotate` + `kubectl label` commands can run at any
point before the migration commit:

```bash
for kind+name in <list>; do
  kubectl annotate "$kind" -n "$ns" "$name" \
    meta.helm.sh/release-name=<release> \
    meta.helm.sh/release-namespace=<ns> --overwrite
  kubectl label "$kind" -n "$ns" "$name" \
    app.kubernetes.io/managed-by=Helm --overwrite
done
```

This is operator-ran, not GitOps — Flux's controllers don't write
these labels. **Do NOT bake the labels into the raw YAML files** —
that would force a Kustomization re-apply which can racing again.

Verify: `kubectl get <kind> <name> -o yaml | grep -E 'release-name|managed-by'`.

### Commit C — the migration commit

Single atomic commit:

* Remove raw manifest files from the Kustomization source.
* Add the `HelmRelease` (or update an existing one).
* Set `prune` back to `true` on the Kustomization (or remove the
  per-resource prune-disabled annotations) — but keep the resource
  out of the Kustomization source so it isn't a target.

Push, wait for both controllers to reconcile (helm-controller's
`helm install` adopts cleanly because of the pre-applied metadata;
kustomize-controller no longer sees the resources so doesn't try
to prune them).

### Commit D — verification

After both controllers report success:

```bash
helm get manifest -n <ns> <release> | grep ^kind
kubectl get <kind> <name> -o yaml | grep meta.helm.sh
```

Confirm Helm owns the manifest set + lock-in any cleanup (e.g.
remove dead directories from the Kustomization base).

## When the four-commit dance is overkill

* The chart emits resources at *different names* than the raw ones —
  no collision, no race. Single commit is fine.
* The Kustomization has `prune: false` already (some Kustomizations
  do, e.g. for CRD lifecycle reasons).
* You're migrating in an environment where you can afford a brief
  reconcile gap (i.e., not a stateful workload).

Any time the raw resource is **stateful** (CNPG Cluster, Postgres
StatefulSet, anything backed by a PVC where deletion = data loss),
**always** do the four-commit dance.

## Defense-in-depth at the chart level

Stateful CNPG clusters use `pleme-cnpg` (helmworks) which adds
`helm.sh/resource-policy: keep` on the `Cluster` and
`ScheduledBackup` resources. This blocks `helm uninstall` from
deleting them — the PVCs survive. Recovery from a stray uninstall
becomes at-most a re-install, never a re-init from empty PVCs.

This annotation only protects against `helm uninstall`, not against
the prune race or against `helm upgrade` issuing a destructive patch.
The four-commit playbook is still required.

## Reference incident

`pleme-io/k8s/clusters/rio/infrastructure/pangea-database/POSTMORTEM-2026-05-03.md`

End-to-end timeline of the destructive event + recovery via the WAL
archive. Useful as concrete evidence of why the race is real and
what the recovery path looks like when the playbook is missed.
