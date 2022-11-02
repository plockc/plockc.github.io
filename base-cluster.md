# Baremetal Kubernetes Cluster

Instead of managing both an OS and Kubernetes, use [talos](https://www.talos.dev) linux images.  Probably don't need [Sidero Metal](https://www.sidero.dev) to build BM (baremetal) clusters on demand as [vcluster](https://www.vcluster.com) can be used for dev and testing, but if needed, it can be used.

Assumption is that the cluster will be on a layer 2 isolated subnet (to allow for firewalling enhancements later) and a gateway with two interfaces (one for LAN and one for the cluster network).  Arguably this could be done with VLANs as well.

## Cluster Storage Options

* For the old NUCs in use, Mayastor has too many CPU requirements
* Longhorn requires iSCSI and touches the host too much, does not play well with read only root system
* There is [Talos documentation](https://www.talos.dev/v1.2/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/) using OpenEBS Jiva (NUC has single disk), and OpenEBS uses user space only

## Local Prep

Install kubectl, talosctl, vcluster, kubectx, k9s, kube-ps1, and kubens (mostly by using [arkade](https://github.com/alexellis/arkade) but for missing things, brew/apt, etc.

For Mac OS, to add a route to your gateway for the node and LoadBalancer CIDRs, where `gw` is the name/IP of the gateway:

```
sudo route -n add -net 192.168.8.0/24 gw
sudo route -n add -net 192.168.14.0/23 gw
```

## Installation Media

Boot from ISO on USB.

Talos install makes the USB stick unbootable, re-run this each time, replace the `/dev/sdX` with the disk device.

```
sudo dd if=/tmp/talos-amd64.iso of=/dev/sdX conv=fdatasync bs=4M
```

Follow the [Getting Started](https://www.talos.dev/v1.2/introduction/getting-started/) guide for talos but

* make sure the VIP is configured in `controlplane.yaml` at `machine.network.interfaces`:
  ```
  - interface: eth0
    dhcp: true
    vip:
      ip: 192.168.8.19 # Specifies the IP address to be used.
  ```
* append the iscsi extension (for CSI) at `machine.install.extensions`:
  ```
  - image: ghcr.io/siderolabs/iscsi-tools:v0.1.1
  ```
* append the mount configuration needed for CSI node storage at `machine.kubelet.extraMounts`:
  ```
  - destination: /var/openebs/local
    type: bind
    source: /var/openebs/local
    options: [bind, rshared, rw]
  ```
* allow workloads to be scheduled on the control plane in the `cluster` section:
  ```
  allowSchedulingOnControlPlanes: true
  ```
* allow the openebs namespace to have some privileges by updating cluster.apiServer.admissionControl[0].configuration.exemptions.namespaces to also include `openebs` and `purelb`
  

## Prepare Network 

Dnsmasq can be configured with each of the MAC addresses for the machines and assign a host name to it, add a static leases file in /etc/dnsmasq.d/

```
dhcp-host=dc:a6:32:31:82:a5,cp1,192.168.8.10
```

and an example config for dnsmasq for a subnet 192.168.8.0/24 behind a dedicated nic (eth1)

```
cache-size=1000
no-dhcp-interface=eth0
no-dhcp-interface=wlan0
dhcp-option=121,0.0.0.0/0,192.168.8.1
dhcp-range=192.168.8.100,192.168.8.250,100d
log-dhcp
bogus-priv
domain-needed
resolv-file=/etc/resolv.dnsmasq.conf
addn-hosts=/etc/hosts.dnsmasq
domain=k8s.local
```

## Create Cluster

bootstrap the first node then add the rest.

## Install OpenEBS for CSI Storage

Follow the [Talos Local Storage Guide](https://www.talos.dev/v1.2/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/), some of the steps have already been done above.

It was chosen because it does so much in userspace, which plays well with Talos.

Want to switch to installing via ArgoCD

## Install Argo

Install Argo, and make it insecure, the incoming traffic will need to go through the LoadBalancer which will terminate the SSL session.  Might want to wireguard between pods . . .

Also install the ingress route (traefik specific -- see [docs](https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/#ingressroute-crd)

This has a race with the configmap vs starting the pod, might need to restart the server pods for argo, and also should use jsonnet to edit the manifest inline to applying.

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cmd-params-cm
  namespace: argocd
data:
  server.insecure: "true"
---
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: argocd-server
  namespace: argocd
spec:
  entryPoints:
    - websecure
  routes:
    - kind: Rule
      match: Host(\`argocd.k8s.local\`)
      priority: 10
      services:
        - name: argocd-server
          port: 80
    - kind: Rule
      match: Host(\`argocd.k8s.local\`) && Headers(\`Content-Type\`, \`application/grpc\`)
      priority: 11
      services:
        - name: argocd-server
          port: 80
          scheme: h2c
  tls:
    certResolver: default
EOF
```

Get the `admin` password

```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## Install LoadBalancer

Uses Argo to install, and add a service group for available IPs
```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: purelb
  namespace: argocd
spec:
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
  destination:
    server: "https://kubernetes.default.svc"
    namespace: purelb
---
apiVersion: purelb.io/v1
kind: ServiceGroup
metadata:
  name: default
  namespace: purelb
spec:
  local:
    v4pool:
      aggregation: default
      pool: 192.168.14.10-192.168.15.254
      subnet: 192.168.14.0/23
EOF
```

## Install Ingress

Uses Argo to install
```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: traefik
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

## Notes

For the helm charts, can get values to override

```
helm show values <repo>/<chart>
```
