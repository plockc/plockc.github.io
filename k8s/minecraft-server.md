# Minecraft on K8s

[itzg](https://github.com/itzg/docker-minecraft-server/blob/master/README.md) did all the heavy lifting for making minecraft into a container, and creating [charts](https://github.com/itzg/minecraft-server-charts/tree/master/charts/minecraft-proxy).

## Install Minecraft RCON sealed secret

Created a [sealed-secret](https://docs.bitnami.com/tutorials/sealed-secrets) for the [rcon](https://developer.valvesoftware.com/wiki/Source_RCON_Protocol) (remote console) which can be controlled by various client implementations.

**Note: Change the password in the command below.**

Example is at [argo-apps/minecraft-sealed-secret.yaml](argo-apps/minecraft-sealed-secret.yaml)

```
touch minecraft-rcon-secret.json
chmod 600 minecraft-rcon-secret.json
echo -n "supersekret" \
  | kubectl create secret generic minecraft --dry-run=client --from-file=rcon-password=/dev/stdin -o yaml \
  | kubeseal -o yaml \
  > minecraft-rcon-secret.yaml
```

Install the secret after the minecraft namespace is created.

```
kubectl apply -f minecraft-rcon-secret.yaml
```

## Install RCON Web Admin sealed secret

The Web interface for RCON has a separate user / password

```
touch web-rcon-secret.json
chmod 600 web-rcon-secret.json
echo -n "supersekret" \
  | kubectl create secret generic rcon --dry-run=client --from-file=password=/dev/stdin -o yaml \
  | kubeseal -o yaml \
  > web-rcon-secret.yaml
```

Install the secret after the minecraft namespace is created.

```
kubectl apply -f web-rcon-secret.yaml
```


## Installation

The ArgoCD [Application Manifest](argo-apps/minecraft.yaml) will install Minecraft and expose LoadBalancer services for Minecraft and RCON.

## Troubleshooting

### cannot open /proc//ns/mnt: No such file or directory

Double check Talos Local Storage instructions, the config map for openebs has to be applied (and some pods restarted), and the machineconfig patch applied, and make sure _all_ the nodes are upgraded.
