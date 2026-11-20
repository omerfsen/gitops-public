# Before You Upgrade

This bundle was generated for the `argocd-helm` deployer. The steps below apply only
when this bundle is applied over an existing installation of an earlier version.
A fresh install can skip this file.

Each entry covers only the boundary this bundle's pinned version lands in. To
check your exact move, including any earlier boundary you would cross, run
`aicr upgrade-check` against the recipe you deployed from.

## nodewright-operator: manual

This bundle pins `v0.21.0`. Applies when upgrading from `<0.18.0` into `>=0.18.0 <=0.21.0`.

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

## kube-prometheus-stack: manual

This bundle pins `84.5.0`. Applies when upgrading from `<84.5.0` into `84.5.0`.

The upgrade deletes Grafana, Alertmanager, node-exporter, kube-state-metrics, the default rules and the dashboards. Prometheus stops loading PrometheusRules, Probes and ScrapeConfigs, keeps only the series AICR components query, and moves from its PVC to an emptyDir, so its history is lost.

**Before you start:** Anything that reads this release's Grafana, Alertmanager or Prometheus history has another source, such as nktp.

**Reversible:** yes. Redeploy the previous bundle. Remove nktp first: its node-exporter and this chart's both bind host port 9100.

**Steps for argocd-helm:**

1. `export`: Export any Grafana dashboards, Alertmanager configuration, and PrometheusRules you still need.\
   *Why:* The upgrade deletes them.
2. `upgrade`: Sync or reconcile the new bundle, upgrading this release before nktp.\
   *Why:* nktp's node-exporter can't start while the old one holds host port 9100. The operator recreates Prometheus on an emptyDir, so prometheus-adapter's metrics are unavailable for about 30 seconds.
3. `clean-up`: Delete what the release leaves behind: kubectl delete pvc --ignore-not-found -n monitoring prometheus-kube-prometheus-prometheus-db-prometheus-kube-prometheus-prometheus-0; kubectl delete secret --ignore-not-found -n monitoring kube-prometheus-admission; kubectl delete service --ignore-not-found -n kube-system kube-prometheus-kubelet.\
   *Why:* The deployer doesn't own them: the StatefulSet created the PVC, a hook Job created the Secret, and the operator created the Service.

**References:**

- https://github.com/NVIDIA/aicr/blob/main/docs/design/026-nktp-telemetry-collection.md
- https://github.com/NVIDIA/aicr/blob/main/docs/user/component-catalog.md#kube-prometheus-stack-8450-keeps-only-the-runtime-prometheus

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

## nktp: manual

This bundle pins `0.2.0`. Applies when upgrading from `<0.2.0` into `0.2.0`.

0.2.0 renames every Collector's OTLP exporter from otlp to otlp_grpc. An override under <collector>.otelcol.config.exporters.otlp, such as a backend endpoint or tls block, no longer reaches the exporter in use, so telemetry goes to the bundled otel-lgtm instead, or the bundle fails CheckNKTPExportersHaveBackend when otel-lgtm is disabled. The chart also moves from oci://ghcr.io/dsx-ai-factory/nvke/charts to the NGC Helm repository.

**Steps for argocd-helm:**

1. `rename-exporter-overrides`: Before regenerating the bundle, rename every nktp override path containing otelcol.config.exporters.otlp. to otelcol.config.exporters.otlp_grpc., in --set flags and in any recipe overlay.\
   *Why:* The old key defines an otlp exporter no pipeline uses, so the override is accepted and silently ignored.
2. `mirror-chart-and-images`: Only when the cluster installs from a mirror: mirror chart nktp 0.2.0 from https://helm.ngc.nvidia.com/nvidia, and the images otel/opentelemetry-collector-k8s:0.164.0, grafana/otel-lgtm:0.38.0, and registry.k8s.io/kubectl:v1.35.10, before upgrading.\
   *Why:* The bundle now names the NGC repository, and the collectors, the evaluation backend, and the chart's hook Jobs pull these images. Without them the release fails to fetch or its pods stay in ImagePullBackOff.

**References:**

- https://github.com/NVIDIA/aicr/blob/main/docs/user/component-catalog.md#nktp-telemetry-pipeline
- https://github.com/dsx-ai-factory/nvke-telemetry-pipeline/tree/main/nktp

## nvsentinel: manual

This bundle pins `v1.31.0`. Applies when upgrading from `>=1.28.0 <1.31.0` into `1.31.0`.

Three changes need operator action. A fatal health-events-analyzer event now cordons and drains its node through a new default fault-quarantine rule set, where v1.28.0 set only a node condition; it applies when the analyzer's effective processingStrategy is EXECUTE_REMEDIATION. With the remediation step and R610 or later drivers, XID 154 recovery actions now reset the GPU or reboot the node, where v1.28.0 reported CONTACT_SUPPORT. A custom generic-provider rebootImage without sh and cat no longer reboots nodes.

**Before you start:** Where health-events-analyzer is deployed, its module-level processingStrategy and the processing_strategy of every enabled rule are neither EXECUTE_REMEDIATION nor UNSPECIFIED (an absent module value is the chart default, EXECUTE_REMEDIATION, and platform-connectors converts UNSPECIFIED to it), or fault-quarantine.ruleSets is set without the "Health events analyzer fatal error ruleset", or you accept that fatal analyzer events cordon and drain nodes from v1.31.0. Where the remediation step is composed and nodes run R610 or later drivers, you accept that an XID 154 recovery action resets the GPU or reboots the node, or you keep syslog-health-monitor events out of remediation. Where janitor-provider.csp.generic.rebootImage is a custom image, it contains sh and cat.

**Reversible:** yes. helm rollback to the previous revision restores the previous workloads; the HealthEvent CRD only gains optional status fields, and the other CRDs differ only in an annotation. A cordon that the new rule set created survives the rollback: release it with kubectl uncordon once the node is safe to return. The rollback itself was not exercised in this record's lanes.

**Steps for argocd-helm:**

1. `check-analyzer-strategy`: Where health-events-analyzer is deployed, read the effective strategies from the nv-sentinel values in the bundle: the module-level processingStrategy, and the processing_strategy of each enabled rule in health-events-analyzer.config, which replaces the module value for that rule when it is set. An absent module value is the chart default, EXECUTE_REMEDIATION, not STORE_ONLY: only the nvsentinel-health-events-analyzer mixin sets STORE_ONLY. UNSPECIFIED counts as EXECUTE_REMEDIATION too, because platform-connectors converts it. Also read fault-quarantine.ruleSets: when set, it replaces the chart's list, and the v1.31.0 rule set that cordons for a fatal analyzer event applies only while the effective list still contains "Health events analyzer fatal error ruleset". If a strategy above is EXECUTE_REMEDIATION and that rule set is present, a fatal analyzer event cordons and drains its node: accept that, or keep the v1.28.0 condition-only behavior by setting fault-quarantine.ruleSets to the chart's v1.31.0 default list without that rule set (carry every other rule set, since the list replaces the default). Setting STORE_ONLY instead also removes the node condition and the Kubernetes Event that EXECUTE_REMEDIATION set, so alerts that read them stop; set it at the module level and change or remove every enabled rule's own processing_strategy: EXECUTE_REMEDIATION, because a rule that keeps it still cordons and drains.\
   *Why:* The new default rule set cordons for a fatal analyzer event, and fault-quarantine reads only EXECUTE_REMEDIATION events, so that strategy, set or inherited, is what turns a condition-only signal into a drain.
2. `check-xid-154-actions`: Where the nvsentinel-remediation step is composed and any node runs an R610 or later driver, expect an XID 154 recovery action (PF FLR, OS Reboot, Bus Reset or System Reboot) to reset that GPU or reboot the node, where v1.28.0 cordoned and drained the node for the fatal event and repaired nothing. No setting restores that: the monitor's action mapping has no per-XID switch. To stop syslog events from acting at all, regenerate the bundle with --set nv-sentinel:syslog-health-monitor.processingStrategy=STORE_ONLY, which also stops their cordon and drain and keeps them from the analyzer's rules, since STORE_ONLY events are not analyzer input, and read the processingStrategy of each NIC pattern in the monitor's config, because a pattern's own value replaces the module's.\
   *Why:* v1.31.0 maps the R610 wordings to COMPONENT_RESET and RESTART_BM, which fault-remediation acts on; v1.28.0 mapped them to CONTACT_SUPPORT, which it does not.
3. `check-reboot-image`: Where the nvsentinel-remediation step is composed with janitor-provider.csp.provider: generic, read janitor-provider.csp.generic.rebootImage in the bundle. The shipped BusyBox image is compatible. A custom image must contain sh and cat: the v1.31.0 reboot Job first reads the host boot ID with cat and refuses to reboot when it cannot, which v1.28.0 did not do. Check the image with kubectl run reboot-image-check --rm -i --restart=Never --image=<image> -- sh -c 'cat /proc/sys/kernel/random/boot_id'.\
   *Why:* A reboot image without cat makes every reboot Job exit non-zero before it reboots, so RESTART_VM and RESTART_BM repairs never reboot the node.
4. `apply`: Re-run install.sh (helm), or commit the regenerated bundle and reconcile or sync it (flux, argocd, argocd-helm).\
   *Why:* These deployers apply the chart's CRDs themselves, then roll the DaemonSets and Deployments to v1.31.0.

**References:**

- https://github.com/NVIDIA/NVSentinel/releases/tag/v1.29.0
- https://github.com/NVIDIA/NVSentinel/releases/tag/v1.30.0
- https://github.com/NVIDIA/NVSentinel/releases/tag/v1.31.0
- https://github.com/NVIDIA/NVSentinel/blob/v1.31.0/docs/deprecations/gpu-health-monitor-dcgm-3-support-removal.md
