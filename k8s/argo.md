## Install Argo

Using a LoadBalancer service is much simpler than setting up the traefik specific ingressroute (as two protocols are on the same port), which allows simple running on https and suport for both protocols on same pot.

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Get the `admin` password for the UI

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

Install the CLI

```
brew install argocd  # or 'ark get argocd'
```

Log in the CLI as `admin`

```
argocd login --username admin --insecure \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) \
  $(kubectl get svc argocd-server -n argocd -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
```

## Notes

For applications released as helm charts, can get values to override

```
helm show values <repo>/<chart>
```
