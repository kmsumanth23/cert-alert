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
