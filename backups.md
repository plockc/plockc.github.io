# Velero

Given that Jiva is used from OpenEBS, there is no snapshot support, however velero leverages restic and snapshots the mounted volumes directly.

## Install Client

```
brew install velero
```

## Create Credentials

```sh
touch ~/.velero-minio-creds
chmod 600 ~/.velero-minio-creds
cat > ~/.velero-minio-creds <<EOF
[default]
aws_access_key_id = minio
aws_secret_access_key = minio123
EOF
```

update the credentials to match those on minio

```
vim ~/.velero-minio-creds
```

## Install valero

TODO: create secret . . and use sealed secrets
TODO: install via helm

```sh
velero install \
    --provider aws \
    --plugins velero/velero-plugin-for-aws:v1.5.1 \
    --bucket minecraft \
    --secret-file ~/.velero-minio-creds \
    --use-volume-snapshots=false \
    --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://minio:9000 \
    --use-restic \
    --default-volumes-to-restic \
    --wait
```

## Define Backup

Single shot

```
velero backup create nginx-backup --default-volumes-to-restic --selector app=tiddly
```

Recurring backup manifest is in [velero-backups](velero-backups/tiddly.yaml)
