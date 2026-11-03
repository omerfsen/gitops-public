# gitops-public

What Argo CD deploys on my lab clusters, one directory per cluster. Public:
nothing here is secret.

## nvke: the NVKE lab cluster

The NVIDIA Kubernetes Engine management cluster that
[omerfsen/nvke](https://github.com/omerfsen/nvke) builds on KVM (three node
VMs, one GeForce RTX 4080 SUPER passed through to one of them).

| Path | What |
|---|---|
| `nvke/argocd/values.yaml` | Argo CD itself: Helm values and the pinned chart version |
| `nvke/apps/aicr-lab.yaml` | The Application that watches `nvke/aicr/lab` |
| `nvke/aicr/lab/` | The GPU stack: NVIDIA AI Cluster Runtime's `nvke` recipe as an Argo CD Helm chart. Generated, do not edit by hand |

### Bootstrap

```shell
export KUBECONFIG=~/.kube/nvke-mgmt.yaml
helm upgrade --install argocd argo-cd \
  --repo https://argoproj.github.io/argo-helm --version 10.11.6 \
  --namespace argocd --create-namespace -f nvke/argocd/values.yaml
kubectl apply -f nvke/apps/aicr-lab.yaml
```

The UI: `kubectl -n argocd port-forward svc/argocd-server 8080:80`, user
`admin`, password from
`kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`.

### Changing the GPU stack

`nvke/aicr/lab` is rendered by
[omerfsen.aicr](https://github.com/omerfsen/aicr)'s `gitops` playbook from
AICR's `nvke` recipe plus the overrides NVKE and the lab need (driver
615.71.09, the containerd toolkit setting, no NIC health monitor):

```shell
cd ~/projects/aicr
ansible-playbook playbooks/gitops.yml \
  -e aicr_gitops_dir=~/projects/gitops-public/nvke/aicr/lab -e @examples/nvke-kvm.yml
```

Then commit the change through a pull request. Argo CD syncs it from `main`.
Check the cluster against the recipe afterwards with omerfsen.aicr's `nvke`
playbook and `-e aicr_apply=false` (it validates without applying).
