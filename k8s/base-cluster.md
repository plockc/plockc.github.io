# Baremetal Kubernetes Cluster

Instead of managing both an OS and Kubernetes, use [talos](https://www.talos.dev) linux images.  Probably don't need [Sidero Metal](https://www.sidero.dev) to build BM (baremetal) clusters on demand as [vcluster](https://www.vcluster.com) can be used for dev and testing, but if needed, it can be used.

DHCP is assumed with static leases.

## Local Prep

Install kubectl, talosctl, vcluster, kubectx, k9s, kube-ps1, and kubens (mostly by using [arkade](https://github.com/alexellis/arkade) but for missing things, brew/apt, etc.

For Mac OS, to add a (temporary) route to your gateway for the node and LoadBalancer CIDRs, where `gw` is the name/IP of the gateway:

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
* also add velero to cluster.apiServer.admissionControl[0].configuration.exemptions.namespaces  

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

## Upgrade cluster

Find the latest [release](https://github.com/siderolabs/talos/releases) then upgrade a node at a time.

Make sure to use `--preserve` if the CSI is writing to the node like Jiva
```
talosctl upgrade --preserve --nodes t1 --image ghcr.io/siderolabs/installer:v1.4.4 --wait
```

