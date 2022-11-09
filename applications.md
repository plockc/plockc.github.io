These will install applications into the local cluster

## kubeseal

```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
  namespace: argocd
spec:
  destination:
    namespace: sealed-secrets
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: sealed-secrets
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    targetRevision: 2.7.0
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```



## purelb

```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: purelb
  namespace: argocd
spec:
  destination:
    namespace: purelb
    server: https://kubernetes.default.svc
  project: default
  source:
    chart: purelb
    repoURL: https://gitlab.com/api/v4/projects/20400619/packages/helm/stable
    targetRevision: 0.6.4
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
EOF
```

