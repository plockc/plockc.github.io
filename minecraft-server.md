# Minecraft on K8s

[itzg](https://github.com/itzg/docker-minecraft-server/blob/master/README.md) did all the heavy lifting for making minecraft into a container, and creating [charts](https://github.com/itzg/minecraft-server-charts/tree/master/charts/minecraft-proxy).

## Create vcluster

Get versions, currently 0.12.3

```
helm search repo vcluster/vcluster --versions
```

Expose the vcluster with load balancer so `vcluster connect` doesn't have to do port forward

```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minecraft-cluster
  namespace: argocd
spec:
  project: default
  source:
    chart: vcluster
    repoURL: https://charts.loft.sh
    targetRevision: 0.12.3
    helm:
      parameters:
        - name: vcluster.image
          value: rancher/k3s:v1.23.5-k3s1
        - name: service.type
          value: LoadBalancer
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  destination:
    server: "https://kubernetes.default.svc"
    namespace: minecraft
EOF
```

## Install Virtual Cluster with Argo

Get vcluster >= 0.12.3 (brew has 0.12.11 at the moment)

```
ark install vcluster
```

Add the cluster to argo, I think it's OK for the kubeconfig to have admin in the vcluster, as cluster will only have minecraft installations and those are all managed solely by argo.
The first command will write out the kubeconfig needed for the second command.

```
vcluster connect minecraft-cluster -n minecraft --update-current=false --server=https://192.168.14.12 --kube-config kubeconfig.minecraft.yaml
argocd --kubeconfig kubeconfig.minecraft.yaml cluster add  vcluster_minecraft-cluster_minecraft_admin@nuc
```

Note it is in the minecraft namespace inside the vcluster which is in namespace minecraft.

```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minecraft
  namespace: argocd
spec:
  project: default
  source:
    chart: traefik
    repoURL: https://helm.traefik.io/traefik
    targetRevision: 18.3.0
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  destination:
    server: "https://kubernetes.default.svc"
    namespace: traefik
EOF
```

## Troubleshooting

### cannot open /proc//ns/mnt: No such file or directory

Double check Talos Local Storage instructions, the config map for openebs has to be applied (and some pods restarted), and the machineconfig patch applied, and make sure _all_ the nodes are upgraded.