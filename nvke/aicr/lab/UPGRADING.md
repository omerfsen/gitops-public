# Before You Upgrade

This bundle was generated for the `argocd-helm` deployer. The steps below apply only
when this bundle is applied over an existing installation of an earlier version.
A fresh install can skip this file.

Each entry covers only the boundary this bundle's pinned version lands in. To
check your exact move, including any earlier boundary you would cross, run
`aicr upgrade-check` against the recipe you deployed from.

## nodewright-operator: manual

This bundle pins `v0.20.0`. Applies when upgrading from `<0.18.0` into `>=0.18.0 <=0.20.0`.

v0.18.0 renames skyhook.nvidia.com/v1alpha1 Skyhook to nodewright.nvidia.com/v1alpha1 NodeWright, moves DeploymentPolicy to the same group, and shifts the on-node annotation, label and finalizer prefix. An operator-side mirror migrates the objects for you, but completion status is then written only on the new kind and the attempt to mirror it back to the legacy object fails in a reconcile conflict loop, so Skyhook.status stays empty forever on a cluster where tuning has genuinely finished. Anything still reading the legacy status waits on a value that never arrives. The legacy object also goes read-only: the post-rename admission webhook rejects any spec, pause or disable change to it, so the first attempt to alter tuning after the upgrade fails rather than the cluster merely reporting the wrong status. Deletions and identical re-applies are still accepted, which is why a steady-state sync keeps working and the break only surfaces on a real edit.

**Before you start:** Every Skyhook is complete with no nodes in progress. Upstream makes this a requirement rather than a recommendation: the migration relabels the operator's package and per-node ConfigMaps so the post-rename operator adopts them instead of recreating them, and that flow assumes no in-flight package work to disrupt. Check with `kubectl get skyhooks.skyhook.nvidia.com -o custom-columns=NAME:.metadata.name,STATUS:.status.status,INPROGRESS:.status.nodesInProgress` and proceed only when nothing is actively rolling out. paused and disabled objects are fine to leave as they are — the mirror carries that state onto the NodeWright — but do not unpause or enable one until its pre-upgrade package pods are gone.

**Reversible:** no.

**Steps for argocd-helm:**

1. `expect-operator-downtime-during-the-hook`: The chart runs a selector-migration pre-upgrade hook that DELETES the operator Deployment before it is recreated, because spec.selector is immutable and the rename changes it. If that hook fails, the sync can stop with the Deployment already gone and no operator running. Re-sync to recover.\
   *Why:* Verified on the helm path, where a hook Job that tripped its deadline left the namespace with zero operator pods and the release in `failed`. The deletion itself is correct: it avoids a release wedged on "Pending termination" when the running operator predates label-based webhook discovery. The hook ships in the chart rather than the deployer, so any path that runs the chart's hooks carries the same gap between delete and recreate; the failure was not reproduced on argocd, flux, or fleet specifically.
2. `upgrade-operator`: Bump the operator's chart version in its own Application, HelmRelease, or Fleet bundle and sync it, before touching any CR.\
   *Why:* The chart ships both groups' CRDs and RBAC, and the mirror imports each Skyhook into a NodeWright. Per-node state is copied to the nodewright.nvidia.com/* prefix, so packages are not re-run. The new objects are outside the reconciler's inventory at this point, so it neither prunes nor flags them.
3. `rename-crs-in-one-commit`: In a single git commit, remove each Skyhook manifest and add the NodeWright equivalent: apiVersion skyhook.nvidia.com/v1alpha1 becomes nodewright.nvidia.com/v1alpha1, kind Skyhook becomes NodeWright, metadata.name is unchanged. DeploymentPolicy changes apiVersion only. Then sync or reconcile.\
   *Why:* The reconciler adopts the mirror-created NodeWright and prunes the Skyhook in one pass. Leaving both in git opens a window where the mirror and the reconciler both write the NodeWright and a stale Skyhook re-import stomps the git edit; do not reach for kubectl here, since anything applied by hand is reverted on the next reconcile. Rewrite apiVersion and kind only: a blanket substitution also rewrites nodeSelectors and podNonInterruptLabels, which name your labels rather than the operator's, and pointing them at keys nothing carries makes the CR match no node.
4. `verify-migration`: kubectl get nodewrights.nodewright.nvidia.com lists a NodeWright for each former Skyhook, node annotations are present under the nodewright.nvidia.com/ prefix, and no package re-ran.\
   *Why:* Confirms the mirror adopted live lifecycle position rather than handing the new object a fresh rollout.

**References:**

- https://github.com/NVIDIA/nodewright/blob/main/docs/getting-started/migration.md
- https://github.com/NVIDIA/nodewright/blob/main/docs/getting-started/migration.md#prerequisite-all-skyhooks-must-be-complete
- https://github.com/NVIDIA/aicr/blob/main/docs/user/component-catalog.md#nodewright-operator-v0180-renames-skyhook-to-nodewright
- https://github.com/NVIDIA/aicr/issues/2593
- https://github.com/NVIDIA/aicr/issues/2594

## nfd: manual

This bundle pins `0.20.1`. Applies when upgrading from `>=0.19.0 <0.20.1` into `0.20.1`.

0.20.0 moves nfd-topology-updater to server-side apply, so its ClusterRole needs create and patch on noderesourcetopologies instead of get and update. The chart ships that rule, so chart-managed RBAC (AICR's default) needs nothing; RBAC managed outside the chart without patch crash-loops every 0.20.x topology-updater pod on its first NodeResourceTopology update.

**Before you start:** Know who manages the topology-updater RBAC. The steps that change it apply only when your nfd override sets topologyUpdater.rbac.create=false (check the bundle's nfd values.yaml and any --set nfd: flags). Without that override the chart manages the rule and those steps do not apply.

**Reversible:** yes. Rolling back to 0.19.0 works while the topology-updater rule still grants get and update: a chart rollback re-applies the 0.19.0 rule, and a rule you manage keeps them until narrow-nrt-rbac.

**Steps for argocd-helm:**

1. `widen-nrt-rbac`: Only with topologyUpdater.rbac.create=false: before the nfd release moves to 0.20.1, add create and patch to the noderesourcetopologies (topology.node.k8s.io) rule of the ClusterRole bound to the topology-updater service account, keeping get and update.\
   *Why:* 0.20.x pods write NodeResourceTopology with server-side apply and crash-loop without patch. 0.19.0 pods still running during the DaemonSet rollout, or after a rollback, need get and update.
2. `sync-nfd`: Sync or reconcile the nfd release.\
   *Why:* Moves nfd to 0.20.1.
3. `narrow-nrt-rbac`: Only if you ran widen-nrt-rbac: once every topology-updater pod runs 0.20.x and you no longer need a rollback path, drop get and update from that rule.\
   *Why:* 0.20.x needs only create and patch, which is all the chart's own rule grants.

**References:**

- https://github.com/kubernetes-sigs/node-feature-discovery/releases/tag/v0.20.0
- https://github.com/kubernetes-sigs/node-feature-discovery/releases/tag/v0.20.1

## nvsentinel: manual

This bundle pins `v1.28.0`. Applies when upgrading from `>=1.25.0 <1.28.0` into `1.28.0`.

From v1.27.0 janitor records in GPUReset status.managedServices which GPU Operator services it stopped before a reset, and restores only those. The v1.26.0 GPUReset CRD does not declare the field, so the API server prunes it; janitor then rewrites the record every second and the reset never reaches its Job, leaving the node cordoned. It matters only where janitor runs, which is the nvsentinel-remediation step. On the observe step, the quarantine annotations v1.26.0 wrote carry no dry-run marker, so the new release reads each once as a manual uncordon and cancels the faults observed before the upgrade; and v1.26.0's dry run fed the circuit breaker, so a trip it persisted halts fault-quarantine once the cluster moves to a live step.

**Before you start:** Where janitor runs (kubectl -n nvsentinel get deployment janitor finds it, as on the nvsentinel-remediation step), no GPUReset is in progress: kubectl get gpuresets.janitor.dgxc.nvidia.com -o jsonpath='{range .items[*]}{.metadata.name}{" "}{.status.conditions[?(@.type=="Complete")].status}{"\n"}{end}' prints True for every GPUReset.

**Reversible:** yes. helm rollback to the previous revision restores the previous workloads. The newer GPUReset CRD stays applied; it only adds an optional status field, which earlier janitor versions do not read. First remove the dry-run marker from every node (kubectl annotate nodes --all quarantineHealthEventDryRun-): earlier versions leave it in place, so a real quarantine it makes would carry the marker, and the next upgrade would discard that quarantine as a dry run and leave the node cordoned with no record. This rests on the CRD diff and upstream's notes; the rollback itself was not exercised.

**Steps for argocd-helm:**

1. `apply`: Re-run install.sh (helm), or commit the regenerated bundle and reconcile or sync it (flux, argocd, argocd-helm).\
   *Why:* These deployers apply the chart's CRDs themselves, then roll the DaemonSets and Deployments to v1.28.0.
2. `reset-breaker`: If the cluster ran the observe step before v1.27.0 and this bundle moves it to a live step (quarantine or remediation), check kubectl get cm circuit-breaker -n nvsentinel -o jsonpath='{.data.status}'. If it prints TRIPPED, reset it with upstream's circuit-breaker runbook (docs/runbooks/circuit-breaker.md at v1.28.0), which patches the ConfigMap to CLOSED and restarts fault-quarantine.\
   *Why:* v1.26.0's dry run fed the breaker, and a trip it persisted halts fault-quarantine at its first live start, so no fault is quarantined or released until the breaker is reset.
3. `verify`: If janitor runs (kubectl -n nvsentinel get deployment janitor), kubectl get crd gpuresets.janitor.dgxc.nvidia.com -o jsonpath='{.spec.versions[0].schema.openAPIV3Schema.properties.status.properties.managedServices.type}' prints object. Without janitor the chart renders no GPUReset CRD and there is nothing to check.\
   *Why:* Confirms the deployer applied the new CRD; the stall shows only when a reset runs.

**References:**

- https://github.com/NVIDIA/NVSentinel/releases/tag/v1.27.0
- https://github.com/NVIDIA/NVSentinel/releases/tag/v1.28.0
- https://github.com/NVIDIA/NVSentinel/pull/1989
- https://github.com/NVIDIA/NVSentinel/blob/v1.28.0/docs/configuration/fault-quarantine.md
