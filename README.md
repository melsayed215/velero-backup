# دليل تثبيت وربط Velero مع MinIO على Kubernetes (RKE2)

دليل مبسط لتثبيت Velero عبر Helm وربطه بـ MinIO كمكان تخزين للنسخ الاحتياطية، بالإضافة لعمل وتجربة Backup و Restore فعلي.

---

## المتطلبات

- كلاستر Kubernetes شغال (تم الاختبار على RKE2)
- Helm مثبت
- MinIO شغال ومتاح (داخل الكلاستر أو خارجه) مع access key و secret key
- صلاحيات `cluster-admin` على الكلاستر

---

## 1. إنشاء ملف الـ Credentials الخاص بـ MinIO

```bash
cat > credentials-velero <<EOF
[default]
aws_access_key_id=minio
aws_secret_access_key=minio123
EOF
```

> استبدل `minio` و `minio123` ببيانات الدخول الفعلية لـ MinIO عندك.

---

## 2. إنشاء الـ Namespace والـ Secret

```bash
kubectl create namespace velero

kubectl create secret generic cloud-credentials \
  --namespace velero \
  --from-file=cloud=credentials-velero
```

---

## 3. إضافة مستودع Helm الخاص بـ Velero

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
helm repo update
```

---

## 4. ملف `values.yaml`

```yaml
image:
  repository: velero/velero
  tag: v1.18.2   # تأكد من توافق النسخة مع الـ chart المستخدم

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.12.1   # تأكد من توافقها مع نسخة Velero
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
        s3Url: http://<MINIO_IP>:9000   # عدّل الـ IP/URL الخاص بـ MinIO عندك

  volumeSnapshotLocation: []   # MinIO لا يدعم snapshot API أصلي

deployNodeAgent: true   # لتفعيل نسخ بيانات الـ Persistent Volumes
snapshotsEnabled: false
```

> ⚠️ **مهم:** تأكد أن نسخة `image.tag` و `velero-plugin-for-aws` متوافقة مع نسخة الـ chart الافتراضية (Chart v12.2.0 يجلب Velero v1.18.2 تلقائيًا). عدم التطابق يسبب فشل الحاوية عند بدء التشغيل (`unknown flag` errors).

---

## 5. تثبيت Velero عبر Helm

```bash
helm install velero vmware-tanzu/velero \
  --namespace velero \
  --set upgradeCRDs=false \
  -f values.yaml
```

> استخدمنا `upgradeCRDs=false` لتجاوز مشكلة فشل الـ pre-install hook الخاص بتحديث الـ CRDs. إذا كانت هذه أول عملية تثبيت على الكلاستر، تأكد من تثبيت الـ CRDs يدويًا (خطوة 6).

---

## 6. تثبيت الـ CRDs يدويًا (إذا لزم الأمر)

```bash
kubectl get crds | grep velero.io
```

إذا كانت النتيجة فارغة:

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

## 7. إنشاء الـ Bucket في MinIO

من خلال واجهة MinIO الرسومية (Console) على البورت `9001`:

1. افتح `http://<MINIO_IP>:9001` وسجل الدخول ببيانات MinIO.
2. من القائمة الجانبية اختر **Buckets**.
3. اضغط **Create Bucket** واكتب الاسم `velero` (يجب أن يطابق الاسم في `values.yaml`).
4. اضغط **Create**.

---

## 8. التحقق من نجاح التثبيت

```bash
kubectl -n velero get pods
```

يجب أن تكون كل الـ pods بحالة `Running` و `1/1` أو `2/2`.

```bash
kubectl -n velero get backupstoragelocation
```

يجب أن يكون `PHASE` = `Available`.

---

## 9. تثبيت Velero CLI

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.18.2/velero-v1.18.2-linux-amd64.tar.gz
tar -xvf velero-v1.18.2-linux-amd64.tar.gz
mv velero-v1.18.2-linux-amd64/velero /usr/local/bin/
velero version
```

---

## 10. عمل Backup

```bash
velero backup create <backup-name> --include-namespaces=<namespace>
```

مثال:

```bash
velero backup create argocd-backup --include-namespaces=argocd
```

### متابعة الحالة

```bash
velero backup get
velero backup describe <backup-name>
velero backup logs <backup-name>
```

يجب أن تكون `STATUS = Completed` مع `ERRORS = 0`.

---

## 11. عمل Restore

### استرجاع في نفس الـ namespace (بعد حذفه أو فقدانه)

```bash
kubectl delete namespace <namespace>
# انتظر حتى يختفي تمامًا
kubectl get namespace <namespace>
```

```bash
velero restore create <restore-name> --from-backup <backup-name>
```

مثال:

```bash
velero restore create argocd-restore --from-backup argocd-backup
```

### استرجاع في namespace مختلف (اختبار بدون التأثير على النسخة الحالية)

```bash
velero restore create <restore-name> \
  --from-backup <backup-name> \
  --namespace-mappings <old-namespace>:<new-namespace>
```

### متابعة الحالة

```bash
velero restore describe <restore-name>
kubectl get all -n <namespace>
```

---

## مشاكل شائعة وحلولها (Troubleshooting)

| المشكلة | السبب | الحل |
|---|---|---|
| `Job/velero-upgrade-crds not ready` | فشل الـ pre-install hook | استخدم `--set upgradeCRDs=false` وثبّت الـ CRDs يدويًا |
| `cannot reuse a name that is still in use` | يوجد release قديم فاشل بنفس الاسم | `helm uninstall velero -n velero` قبل إعادة التثبيت |
| `unknown flag: --repo-maintenance-job-configmap` | نسخة الصورة في `values.yaml` غير متوافقة مع نسخة الـ chart | وحّد نسخة `image.tag` مع نسخة Velero الافتراضية للـ chart |
| `secret "cloud-credentials" not found` | الـ secret غير موجود أو انمسح | أعد إنشاءه بالأمر في الخطوة 2 |
| Pods بحالة `Evicted` باستمرار | الـ node يعاني من `DiskPressure` (نفاد مساحة القرص) | كبّر الـ disk/partition، أو نظّف الصور واللوجز غير المستخدمة |
| `NoSuchBucket` في حالة الـ BackupStorageLocation | الـ bucket غير موجود في MinIO | أنشئ الـ bucket بنفس الاسم المحدد في `values.yaml` |

---

## ملخص سريع لبنية النظام

- **Velero (server + node-agent):** يعمل **داخل** الكلاستر، ومسؤول عن أخذ نسخ من موارد الكلاستر ورفعها.
- **MinIO:** يعمل كمكان تخزين **خارجي** (S3-compatible) لحفظ بيانات النسخ الاحتياطية.
- عند فقدان الكلاستر بالكامل، يمكن تثبيت Velero على كلاستر جديد وربطه بنفس bucket في MinIO لاسترجاع كل البيانات.

---

## أوامر مرجعية سريعة

```bash
# حالة كل شيء
kubectl -n velero get pods
kubectl -n velero get backupstoragelocation

# Backup
velero backup create <name> --include-namespaces=<ns>
velero backup get
velero backup describe <name>

# Restore
velero restore create <name> --from-backup <backup-name>
velero restore describe <name>

# تحديث الإعدادات
helm upgrade velero vmware-tanzu/velero -n velero --set upgradeCRDs=false -f values.yaml
```
