Stuff I work on

## Home Lab

I have a [Kubernetes Cluster](./base-cluster.md) that I run on some old NUCs I bought off ebay.

Applications are installed using [Argo CD](https://argo-cd.readthedocs.io/en/stable/), manifest is [applications](applications.yaml), and secrets are managed with [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets).

Persistent Volumes are backed up with [Velero](https://velero.io/docs/v1.9/), here is [how](backups.md)
