# Baremetal Kubernetes Cluster

Instead of managing both an OS and Kubernetes, use [talos](https://www.talos.dev) linux images.  Probably don't need [Sidero Metal](https://www.sidero.dev) to build BM (baremetal) clusters on demand as [vcluster](https://www.vcluster.com) can be used for dev and testing, but if needed, it can be used.

## Local Prep

Install kubectl, talosctl, vcluster, kubectx, kube-ps1, and kubens (mostly by using [arkade](https://github.com/alexellis/arkade) but for missing things, brew/apt, etc.

## Installation Media

Boot from ISO on USB.

Talos install makes the USB stick unbootable, re-run this each time, replace the `/dev/sdX` with the disk device.

```
sudo dd if=/tmp/talos-amd64.iso of=/dev/sdX conv=fdatasync bs=4M
```

Follow the Getting Started guide for talos but make sure the VIP is configured in `controlplane.yaml`:

## Form Cluster 

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

bootstrap the first node then add the rest.

