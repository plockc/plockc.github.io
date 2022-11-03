# Minecraft on K8s

[itzg](https://github.com/itzg/docker-minecraft-server/blob/master/README.md) did all the heavy lifting for making minecraft into a container, and creating [charts](https://github.com/itzg/minecraft-server-charts/tree/master/charts/minecraft-proxy).

## Create vcluster

Get versions

```
helm search repo vcluster/vcluster --versions
```

Currently 0.12.3

```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minecraft-vcluster
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

## Install with Argo in the Vcluster

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
