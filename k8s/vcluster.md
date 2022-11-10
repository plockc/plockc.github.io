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
  name: temp
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
    namespace: temp
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
vcluster connect temp -n temp --update-current=false --server=https://192.168.14.12 --kube-config temp.config.yaml
argocd --kubeconfig temp.config.yaml cluster add vcluster_temp_temp_admin@nuc
```
