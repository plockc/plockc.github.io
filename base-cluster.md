# Baremetal Kubernetes Cluster

Instead of managing both an OS and Kubernetes, use [talos](https://www.talos.dev) linux images.  Probably don't need [Sidero Metal](https://www.sidero.dev) to build BM (baremetal) clusters on demand as [vcluster](https://www.vcluster.com) can be used for dev and testing, but if needed, it can be used.

Assumption is that the cluster will be on a layer 2 isolated subnet (to allow for firewalling enhancements later) and a gateway with two interfaces (one for LAN and one for the cluster network).  Arguably this could be done with VLANs as well.

## Cluster Storage Options

* For the old NUCs in use, Mayastor has too many CPU requirements
* Longhorn requires iSCSI and touches the host too much, does not play well with read only root system
* There is [Talos documentation](https://www.talos.dev/v1.2/kubernetes-guides/configuration/replicated-local-storage-with-openebs-jiva/) using OpenEBS Jiva (NUC has single disk), and OpenEBS uses user space only

## Local Prep

Install kubectl, talosctl, vcluster, kubectx, k9s, kube-ps1, and kubens (mostly by using [arkade](https://github.com/alexellis/arkade) but for missing things, brew/apt, etc.

For Mac OS, to add a route to your gateway for the subnet:

```
sudo route -n add -net 192.168.8.0/24 <ip of gateway on the LAN>
```

## Installation Media

Boot from ISO on USB.

Talos install makes the USB stick unbootable, re-run this each time, replace the `/dev/sdX` with the disk device.

```
sudo dd if=/tmp/talos-amd64.iso of=/dev/sdX conv=fdatasync bs=4M
```

Follow the [Getting Started](|https://www.talos.dev/v1.2/introduction/getting-started/) guide for talos but make sure the VIP is configured in `controlplane.yaml`:

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

