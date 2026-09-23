# Installing Velero and Connecting It to MinIO on Kubernetes (RKE2)

A simple guide for installing Velero via Helm and connecting it to MinIO as backup storage, plus performing and testing an actual Backup and Restore.

Once this setup is done, Velero can back up **any namespace or workload** in the cluster — not just the one used as an example here. Just swap `<namespace>` / `<backup-name>` in the commands below with whatever you want to back up (an app, a database, ArgoCD, an entire environment, etc.), and the same Backup/Restore flow applies.

---

## Prerequisites

- A running Kubernetes cluster (tested on RKE2)
- Helm installed
- MinIO running and reachable (in-cluster or external) with an access key and secret key
- `cluster-admin` permissions on the cluster

---

## 1. Create the MinIO Credentials File

```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF
```

> Replace `minio` and `minio123` with your actual MinIO credentials.

---

## 2. Create the Namespace and Secret

```bash
kubectl create namespace velero

kubectl create secret generic cloud-credentials \
  --namespace velero \
  --from-file=cloud=credentials-velero
```

---

## 3. Add the Velero Helm Repository

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
```

---

## 4. `values.yaml` File

```yaml
image:
  repository: velero/velero
  tag: v1.18.2   # make sure this matches the chart's default Velero version

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.12.1   # make sure it's compatible with the Velero version
    volumeMounts:
      - mountPath: /target
        name: plugins

credentials:
  useSecret: true
  existingSecret: cloud-credentials

configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: velero
      config:
        region: minio
        s3ForcePathStyle: "true"
        s3Url: http://<MINIO_IP>:9000   # set this to your MinIO URL/IP

  volumeSnapshotLocation: []   # MinIO has no native snapshot API

deployNodeAgent: true   # enables backing up Persistent Volume data
snapshotsEnabled: false
```

> ⚠️ **Important:** Make sure `image.tag` and `velero-plugin-for-aws` are compatible with the chart's default Velero version (Chart v12.2.0 pulls Velero v1.18.2 by default). A mismatch causes the container to crash on startup with `unknown flag` errors.

---

## 5. Install Velero via Helm

```bash
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --set upgradeCRDs=false \
  -f values.yaml
```

> `upgradeCRDs=false` is used here to bypass a failing pre-install hook that updates the CRDs. If this is the cluster's first-ever Velero install, make sure to install the CRDs manually (step 6).

---

## 6. Install CRDs Manually (if needed)

```bash
kubectl get crds | grep velero.io
```

If the result is empty:

```bash
kubectl -n velero run velero-crds-install \
  --image=velero/velero:v1.18.2 \
  --restart=Never \
  --overrides='{"spec": {"serviceAccountName": "velero-server-upgrade-crds"}}' \
  --command -- /velero install --crds-only --apply

kubectl -n velero logs velero-crds-install
kubectl -n velero delete pod velero-crds-install --ignore-not-found
```

---

## 7. Create the Bucket in MinIO

Using the MinIO web console (port `9001`):

1. Open `http://<MINIO_IP>:9001` and log in with your MinIO credentials.
2. From the sidebar, select **Buckets**.
3. Click **Create Bucket** and name it `velero` (must match the name in `values.yaml`).
4. Click **Create**.

---

## 8. Verify the Installation

```bash
kubectl -n velero get pods
```

All pods should be `Running` and `1/1` (or `2/2`).

```bash
kubectl -n velero get backupstoragelocation
```

`PHASE` should be `Available`.

---

## 9. Install the Velero CLI

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.18.2/velero-v1.18.2-linux-amd64.tar.gz
tar -xvf velero-v1.18.2-linux-amd64.tar.gz
mv velero-v1.18.2-linux-amd64/velero /usr/local/bin/
velero version
```

---

## 10. Create a Backup

```bash
velero backup create <backup-name> --include-namespaces=<namespace>
```

Example:

```bash
velero backup create argocd-backup --include-namespaces=argocd
```

### Check status

```bash
velero backup get
velero backup describe <backup-name>
velero backup logs <backup-name>
```

`STATUS` should be `Completed` with `ERRORS = 0`.

---

## 11. Restore

### Restore into the same namespace (after it's deleted or lost)

```bash
kubectl delete namespace <namespace>
# wait until it's fully gone
kubectl get namespace <namespace>
```

```bash
velero restore create <restore-name> --from-backup <backup-name>
```

Example:

```bash
velero restore create argocd-restore --from-backup argocd-backup
```

### Restore into a different namespace (test without affecting the current one)

```bash
velero restore create <restore-name> \
  --from-backup <backup-name> \
  --namespace-mappings <old-namespace>:<new-namespace>
```

### Check status

```bash
velero restore describe <restore-name>
kubectl get all -n <namespace>
```

---

## Common Issues and Fixes (Troubleshooting)

| Issue | Cause | Fix |
|---|---|---|
| `Job/velero-upgrade-crds not ready` | The pre-install hook failed | Use `--set upgradeCRDs=false` and install the CRDs manually |
| `cannot reuse a name that is still in use` | An old failed release with the same name still exists | `helm uninstall velero -n velero` before reinstalling |
| `unknown flag: --repo-maintenance-job-configmap` | The image version in `values.yaml` doesn't match the chart's Velero version | Align `image.tag` with the chart's default Velero version |
| `secret "cloud-credentials" not found` | The secret is missing or was deleted | Recreate it using the command in step 2 |
| Pods stuck in `Evicted` | The node has `DiskPressure` (disk space exhausted) | Grow the disk/partition, or clean up unused images and logs |
| `NoSuchBucket` in the BackupStorageLocation status | The bucket doesn't exist in MinIO | Create the bucket with the exact name set in `values.yaml` |

---

## Quick Architecture Summary

- **Velero (server + node-agent):** runs **inside** the cluster and is responsible for taking snapshots of cluster resources and uploading them.
- **MinIO:** acts as **external** (S3-compatible) storage for backup data.
- If the entire cluster is lost, Velero can be installed on a new cluster and pointed at the same MinIO bucket to restore everything.

---

## Quick Reference Commands

```bash
# Overall status
kubectl -n velero get pods
kubectl -n velero get backupstoragelocation

# Backup
velero backup create <name> --include-namespaces=<ns>
velero backup get
velero backup describe <name>

# Restore
velero restore create <name> --from-backup <backup-name>
velero restore describe <name>

# Update configuration
helm upgrade velero vmware-tanzu/velero -n velero --set upgradeCRDs=false -f values.yaml
```
