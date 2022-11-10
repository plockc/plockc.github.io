# Minecraft on K8s

[itzg](https://github.com/itzg/docker-minecraft-server/blob/master/README.md) did all the heavy lifting for making minecraft into a container, and creating [charts](https://github.com/itzg/minecraft-server-charts/tree/master/charts/minecraft-proxy).

## Install Minecraft RCON sealed secret

Created a [sealed-secret](https://docs.bitnami.com/tutorials/sealed-secrets) for the [rcon](https://developer.valvesoftware.com/wiki/Source_RCON_Protocol) (remote console) which can be controlled by various client implementations.

**Note: Change the password in the command below.**

```
touch minecraft-rcon-secret.json
chmod 600 minecraft-rcon-secret.json
echo -n "supersekret" \
  | kubectl create secret generic minecraft --dry-run=client --from-file=rcon-password=/dev/stdin -o yaml \
  | kubeseal -o yaml \
  > minecraft-rcon-secret.yaml
```

## Install Minecraft Chart

```
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: minecraft
  namespace: argocd
spec:
  project: default
  source:
    chart: minecraft
    repoURL: https://itzg.github.io/minecraft-server-charts/
    targetRevision: 4.4.0
    helm:
      parameters:
        - name: livenessProbe.initialDelaySeconds
          value: "300"
        - name: readinessProbe.initialDelaySeconds
          value: "300"
        - name: resources.requests.memory
          value: "5Gi"
        - name: resources.requests.cpu
          value: "1200m"
        - name: minecraftServer.eula
          value: "TRUE"
        - name: minecraftServer.serviceType
          value: LoadBalancer
        # check out persistence, and rcon secret
        - name: minecraftServer.rcon.enabled
          value: "true"
        - name: minecraftServer.rcon.serviceType
          value: LoadBalancer
        - name: minecraftServer.rcon.existingSecret
          value: minecraft
        - name: minecraftServer.maxPlayers
          value: "10"
        - name: minecraftServer.memory
          value: "5000M"
        - name: persistence.dataDir.enabled
          value: "true"
        - name: persistence.dataDir.Size
          value: "3Gi"
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
  destination:
    server: "https://kubernetes.default.svc"
    namespace: minecraft
EOF
```

After the minecraft namespace is created
```
kubectl apply -n minecraft -f minecraft-rcon-secret.yaml
```

## Troubleshooting

### cannot open /proc//ns/mnt: No such file or directory

Double check Talos Local Storage instructions, the config map for openebs has to be applied (and some pods restarted), and the machineconfig patch applied, and make sure _all_ the nodes are upgraded.
