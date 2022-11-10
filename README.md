Stuff I work on


## Home Lab

I have a [Kubernetes Cluster](./base-cluster.md) that I run on some old NUCs I bought off ebay.

Nodes are based on [talos](https://www.talos.dev) linux images, and installation is covered in [base-cluster.md](base-cluster.md).


## Argo CD

After bringing up the cluster, install [Argo CD](https://argo-cd.readthedocs.io/en/stable/), directions in [argo.md](argo.md).

Next apply the [applications.yaml manifest](applications.yaml) for Argo "Application of Applications" 


## Persistent Volumes

Follow the [Talos Local Storage Guide](https://www.talos.dev/v1.2/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/) to install OpenEBS Jiva, some of the steps have already been done above.

It was chosen because it does so much in userspace, which plays well with Talos.

Make sure to set `storageClass.isDefaultClass: true` for the chart, if you forget:

```
helm upgrade --namespace openebs --set storageClass.isDefaultClass=true openebs-jiva openebs-jiva/jiva
```

Want to switch to installing via ArgoCD


## Argo CD Managed Applications

* Secrets - [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), managed with [argo-apps/sealed-secrets.yaml](argo-apps/sealed-secrets.yaml).
* Backups - Persistent Volumes are backed up with [Velero](https://velero.io/docs/v1.9/), here is [how](backups.md).
* LoadBalancer - PureLB deployment managed by [argo-apps/purelb.yaml](argo-apps/purelb.yaml), and the service group (available IPs) is a [ServiceGroup resource](argo-apps/purelb-servicegroup.yaml) using the 192.168.14.0/23 subnet.
* Ingress - Traefik deployment is managed at [argo-apps/traefik.yaml](argo-apps/traefik.yaml).
* [TiddlyWiki](argo-apps/tiddly.yaml)
* [Minecraft](minecraft.md)
