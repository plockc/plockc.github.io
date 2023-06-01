## Kubernetes Home Lab

I have a [Kubernetes Cluster](./base-cluster.md) that I run on some old NUCs I bought off ebay.

Nodes are based on [talos](https://www.talos.dev) linux images, and installation is covered in [base-cluster.md](base-cluster.md).


## Persistent Volumes

### Cluster Storage Options

* For the old NUCs in use, Mayastor has too many CPU requirements
* Longhorn requires iSCSI and touches the host too much, does not play well with read only root system
* Rook (Ceph) requires way too much memory (only have 8GB) and resources (only 4 virtual thread CPU) to run.
* Longhorn requires a regular OS worth of utilities installed.
* Mayastor from OpenEBS requires CPU instructions that are not available on the NUCs being used (very old)
* There is [Talos documentation](https://www.talos.dev/v1.2/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/) using OpenEBS Jiva (NUC has single disk), and OpenEBS uses user space only

### Updgrade Warning

**Make sure to use `--preserve` when upgrading talos linux**

### OpenEBS Jiva Preparation

Following the [Talos Local Storage Guide](https://www.talos.dev/v1.4/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/) to install OpenEBS Jiva, some of the steps have already been done in [base-cluster.md](base-cluser.md) guide above.

Make sure some of the iscsi configuration has been done: https://www.talos.dev/v1.4/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/#patching-the-jiva-installation
If this is forgotten, apply, the restart the jiva-ctrl pods (maybe more than once) and the mounting pod to get pods to attach to volume.

### Update Talos Config

```
cat >patch.yaml <<EOF
- op: add
  path: /machine/install/extensions
  value:
    - image: ghcr.io/siderolabs/iscsi-tools:v0.1.4
- op: add
  path: /machine/kubelet/extraMounts
  value:
    - destination: /var/openebs/local
      type: bind
      source: /var/openebs/local
      options:
        - bind
        - rshared
        - rw
EOF
talosctl -n t1,t2,t3 patch mc -p @patch.yaml
talosctl upgrade --nodes t1 --image ghcr.io/siderolabs/installer:v1.4.4 --wait
```

---

## Argo CD Managed Applications

After bringing up the cluster and preparing for iscsi, install [Argo CD](https://argo-cd.readthedocs.io/en/stable/), directions in [argo.md](argo.md).

Next apply the [infra applications yaml manifest](infra-apps.yaml) and the [applications.yaml manifest](applications.yaml) for Argo "Application of Applications" 

```
kubectl apply -f infra-apps.yaml
# wait for the storage class to appear
while ! kubectl get -o name sc openebs-jiva-csi-default; do
   echo retying...; sleep 2;
done
kubectl apply -f applications.yaml
```

### Uninstalling

```
kubectl delete -f applications.yaml
kubectl delete -f infra-apps.yaml
```

* openenbs jiva - Using latest version found in [releases](https://github.com/openebs/jiva/releases)
* Secrets - [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets), managed with [argo-apps/sealed-secrets.yaml](argo-apps/sealed-secrets.yaml)
* Backups - Persistent Volumes are backed up with [Velero](https://velero.io/docs/v1.9/), here is [how](backups.md)
* LoadBalancer - PureLB deployment managed by [argo-apps/purelb.yaml](argo-apps/purelb.yaml), and the service group (available IPs) is a [ServiceGroup resource](argo-apps/purelb-servicegroup.yaml) using the 192.168.14.0/23 subnet
* Ingress - Traefik deployment is managed at [argo-apps/traefik.yaml](argo-apps/traefik.yaml)
* [TiddlyWiki manifest](manifests/tiddly.yaml) managed by [argo-apps/tiddly.yaml](argo-apps/tiddly.yaml)
* [Minecraft](minecraft.md)
