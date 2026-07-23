# What's the driver actually saying?
kubectl -n kohl-uni-np describe volumesnapshot velero-unica-pvc-np-bbsxq
kubectl describe volumesnapshotcontent snapcontent-f086fed2-d307-49f8-a69d-15ab013e7442

# Snapshot class params — Filestore CSI needs type: backup + a valid location
kubectl get volumesnapshotclass snapshotclass-csi-2 -o yaml

# Map PVC -> Filestore instance
kubectl -n kohl-uni-np get pvc unica-pvc-np -o jsonpath='{.spec.volumeName}{"\n"}'
kubectl get pv <pv-name> -o jsonpath='{.spec.csi.volumeHandle}{"\n"}'

gcloud filestore instances list
gcloud filestore backups list \
  --format="table(name,state,sourceInstance,createTime,capacityGb,downloadBytes)"

# Anything stuck in flight?
gcloud filestore operations list --filter="done=false" \
  --format="table(name,metadata.verb,metadata.target,metadata.createTime)"

kubectl get volumesnapshotcontent -o json | jq -r '
  .items[] | select(.status.readyToUse != true) |
  [.metadata.name, .spec.volumeSnapshotRef.namespace, .spec.volumeSnapshotRef.name, .metadata.creationTimestamp] | @tsv'

gsutil ls gs://hclsw-hss-bkt-kohl-np-velero/kopia/
gsutil ls gs://hclsw-hss-bkt-kohl-np-velero/kopia/kohl-uni-np/
gsutil lifecycle get gs://hclsw-hss-bkt-kohl-np-velero

kubectl -n velero get backuprepository kohl-uni-np-kohl-np-velero-kopia -o yaml

kubectl -n velero delete backuprepository kohl-uni-np-kohl-np-velero-kopia

kubectl -n velero get podvolumebackup kohl-np-velero-2-hours-20260723141620-hg7rf -o yaml | grep -A6 -E "pod:|volume:"

kubectl -n kohl-uni-np get pods -o json | jq -r '
  .items[] | select(.metadata.annotations["backup.velero.io/backup-volumes"] != null or
                    .metadata.annotations["backup.velero.io/backup-volumes-excludes"] != null) |
  [.metadata.name, .metadata.annotations["backup.velero.io/backup-volumes"]] | @tsv'


---

gcloud storage buckets describe gs://hclsw-hss-bkt-kohl-np-velero \
  --format="default(lifecycle,versioning,retentionPolicy,softDeletePolicy)"

# What prefixes actually survive?
gcloud storage ls gs://hclsw-hss-bkt-kohl-np-velero/

# Soft-delete may still hold the objects — this is the fastest recovery path
gcloud storage ls --soft-deleted gs://hclsw-hss-bkt-kohl-np-velero/kopia/ 2>/dev/null | head

# Who deleted them? (needs Data Access audit logs enabled)
gcloud logging read \
  'resource.type="gcs_bucket" AND resource.labels.bucket_name="hclsw-hss-bkt-kohl-np-velero"
   AND protoPayload.methodName=~"storage.objects.delete"' \
  --freshness=90d --limit=20 \
  --format="table(timestamp,protoPayload.authenticationInfo.principalEmail,protoPayload.resourceName)"

kubectl -n kohl-uni-np get deploy velero-fsb-test -o yaml | grep -A5 annotations
kubectl -n velero get schedule kohl-np-velero-2-hours -o yaml | grep -iE "defaultVolumes|snapshotVolumes|labelSelector"

# Was it one CR or all five?
kubectl -n velero get backuprepository

# Did the maintenance error spam stop?
kubectl -n velero logs deployment/velero -c velero --since=1h | grep -i "maintenance\|prune"

# Still nothing in the bucket? (expected at this point)
gcloud storage ls gs://hclsw-hss-bkt-kohl-np-velero/

kubectl create ns velero-fsb-validate

velero backup create fsb-validate-$(date +%H%M) \
  --include-namespaces velero-fsb-validate --wait

kubectl -n velero logs -l name=node-agent --since=10m | grep -iE "initializ|connect|repo"
gcloud storage ls -r gs://hclsw-hss-bkt-kohl-np-velero/kopia/

kubectl -n velero get ds node-agent -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'
kubectl -n velero get sa <sa-name> -o yaml | grep -i "iam.gke.io/gcp-service-account"
