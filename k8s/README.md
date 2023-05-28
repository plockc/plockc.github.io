## Kubernetes Home Lab

I have a [Kubernetes Cluster](./base-cluster.md) that I run on some old NUCs I bought off ebay.

Nodes are based on [talos](https://www.talos.dev) linux images, and installation is covered in [base-cluster.md](base-cluster.md).


## Argo CD

After bringing up the cluster, install [Argo CD](https://argo-cd.readthedocs.io/en/stable/), directions in [argo.md](argo.md).

Next apply the [applications.yaml manifest](applications.yaml) for Argo "Application of Applications" 


## Persistent Volumes

Rook (Ceph) requires way too much memory (only have 8GB) and resources (only 4 virtual thread CPU) to run.
Longhorn requires a regular OS worth of utilities installed.
Mayastor from OpenEBS requires CPU instructions that are not available on the NUCs being used (very old)
Will use Jiva

**Make sure to use `--preserve` when upgrading talos linux**

Follow the [Talos Local Storage Guide](https://www.talos.dev/v1.4/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/) to install OpenEBS Jiva, some of the steps have already been done above.

It was chosen because it does so much in userspace, which plays well with Talos.

Make sure to set `storageClass.isDefaultClass: true` for the chart

Use latest version found in [releases](https://github.com/openebs/jiva/releases)
```
helm repo add openebs-jiva https://openebs.github.io/jiva-operator
helm repo update
helm upgrade --install --create-namespace --set storageClass.isDefaultClass=true --namespace openebs --version 3.4.0 openebs-jiva openebs-jiva/jiva
```

Want to switch to installing via ArgoCD

---

## Argo CD Managed Applications

* Secrets - [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), managed with [argo-apps/sealed-secrets.yaml](argo-apps/sealed-secrets.yaml)
* Backups - Persistent Volumes are backed up with [Velero](https://velero.io/docs/v1.9/), here is [how](backups.md)
* LoadBalancer - PureLB deployment managed by [argo-apps/purelb.yaml](argo-apps/purelb.yaml), and the service group (available IPs) is a [ServiceGroup resource](argo-apps/purelb-servicegroup.yaml) using the 192.168.14.0/23 subnet
* Ingress - Traefik deployment is managed at [argo-apps/traefik.yaml](argo-apps/traefik.yaml)
* [TiddlyWiki manifest](manifests/tiddly.yaml) managed by [argo-apps/tiddly.yaml](argo-apps/tiddly.yaml)
* [Minecraft](minecraft.md)
