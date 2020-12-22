#!/bin/bash

RED='\033[0;31m'
GREEN='\033[0;32m'
CYAN='\033[0;36m'
NC='\033[0m' # No Color

# exit on failure
set -e
set -o pipefail

PRIMARY_RESPONSIVE=true
GARDENER_PROJECT_CONFIG="./gardenerProjectConfig.yaml"
kubectl get secret "${ROBOT_CONFIGS_SECRET}" -n "${CENTRAL_NAMESPACE}" -o json | jq -r --arg project "$GARDENER_PROJECT" ' .data.robotconfigs | @base64d | fromjson [$project]' > $GARDENER_PROJECT_CONFIG
cluster=$(kubectl get secret "${BASIC_SECRET}" -n "${CENTRAL_NAMESPACE}" -o json | jq -r --arg project "$GARDENER_PROJECT" --arg cluster "$PRIMARY_CLUSTER_NAME" ' .data.projects | @base64d | fromjson[] | select(.name == $project) | .clusters[] | select(.name == $cluster)')
BACKUP_CLUSTER_NAME=`echo $cluster | jq -r '.backup'`


# fetch primary and backup kubeconfig files
kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} get secret  ${BACKUP_CLUSTER_NAME}.kubeconfig -n garden-${GARDENER_PROJECT} -o jsonpath={.data.kubeconfig} | base64 -d > ./backupConfig.yaml
kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} get secret  ${PRIMARY_CLUSTER_NAME}.kubeconfig -n garden-${GARDENER_PROJECT} -o jsonpath={.data.kubeconfig} > /dev/null 2>&1 && echo "Fetched primary KUBECONFIG successfully" || { echo "Warning: Failed to fetch primary cluster KUBECONFIG, all primary cluster operations will be skipped"; PRIMARY_RESPONSIVE=false; }
[[ $PRIMARY_RESPONSIVE = true ]] && kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} get secret  ${PRIMARY_CLUSTER_NAME}.kubeconfig -n garden-${GARDENER_PROJECT} -o jsonpath={.data.kubeconfig} | base64 -d  > ./primaryConfig.yaml
[[ $PRIMARY_RESPONSIVE = true ]] && kubectl --kubeconfig=./primaryConfig.yaml version > /dev/null 2>&1 && echo "Connectivity to primary cluster apiserver successful"  || { echo "Warning: Failed to get response from primary kubeAPI, all primary cluster operations will be skipped"; PRIMARY_RESPONSIVE=false; }

if [[ $PRIMARY_RESPONSIVE = true ]]; then
    # stop automatic backups on primary cluster
    export KUBECONFIG=./primaryConfig.yaml
    velero delete schedule devspace-backup --confirm || echo "Warning: primary cluster devspace backup schedule deletion is skipped"
    velero delete schedule crd-backup --confirm || echo "Warning: primary cluster crd backup schedule deletion is skipped"
    velero delete schedule common-backup --confirm || echo "Warning: primary cluster common backup schedule deletion is skipped"

    #scale down all devspaces of primary cluster
    echo "scale down all devspaces of primary cluster"
    readarray -t wsnames < <(kubectl --kubeconfig="./primaryConfig.yaml" get ws -A -o custom-columns=NAME:.metadata.name | tail -n +2 )
    readarray -t devspace_ns < <(kubectl --kubeconfig="./primaryConfig.yaml" get ws -A -o custom-columns=NAMESPACE:.status.namespace | tail -n +2 )
    for i in "${!wsnames[@]}"
    do
       echo "stopping workspace ${wsnames[$i]} in namespace ${devspace_ns[$i]}"
       kubectl --kubeconfig="./primaryConfig.yaml" patch ws ${wsnames[$i]} -n workspaces --type='json' -p='[{"op": "replace", "path": "/spec/suspended", "value": true}]'
       kubectl --kubeconfig="./primaryConfig.yaml" patch Deployment workspaces-${wsnames[$i]}-deployment -n ${devspace_ns[$i]} -p '{"spec":{"replicas": 0}}'
    done

    # hibernate primary cluster
    echo "hibernate primary cluster"
    kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} patch shoot ${PRIMARY_CLUSTER_NAME} -p '{"spec":{"hibernation":{"enabled":true}}}'
fi

# wake-up backup cluster
kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} patch shoot ${BACKUP_CLUSTER_NAME} -p '{"spec":{"hibernation":{"enabled":false}}}'
echo -n 'Waiting for cluster to wake...'
sleep 3s
for i in {1..40}
do
  state=`kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} get shoot ${BACKUP_CLUSTER_NAME} -o json | jq '.status.lastOperation.state' | tr -d '[:space:]' | tr -d '"'`
  if [ "$state" = "Succeeded" ]; then
    echo && break
  fi
  sleep 10s
  echo -n '.'
done
echo 'Cluster is up'

# get latest backup versions
export KUBECONFIG=./backupConfig.yaml
kubectl get backups.velero.io -A --sort-by="metadata.creationTimestamp" -o name || { echo "No backups found! Exiting" ; exit 1; }
kubectl get backups.velero.io -A --sort-by="metadata.creationTimestamp" -o name | cut -d'/' -f2  > backups
CRD_BACKUP=`cat backups | grep crd-backup | tail -n1`
DEVSPACE_BACKUP=`cat backups | grep devspace-backup | tail -n1`
COMMON_BACKUP=`cat backups | grep common-backup | tail -n1`
if [ -z "$CRD_BACKUP" ] || [ -z "$DEVSPACE_BACKUP" ] || [ -z "$COMMON_BACKUP" ]; then
    echo "No backups found! Exiting"
    exit 1
fi
echo "Crd backup: ${CRD_BACKUP}"
echo "Devspace backup: ${DEVSPACE_BACKUP}"
echo "Common backup: ${COMMON_BACKUP}"

# create devspace restore
velero delete restore devspace-restore --confirm || echo "There is no velero restore, creating devspace restore"
velero create restore devspace-restore --from-backup $DEVSPACE_BACKUP
# create common restore
velero delete restore common-restore --confirm || echo "There is no velero restore, creating common restore"
velero create restore common-restore --from-backup $COMMON_BACKUP

# fetch s3 data
IAAS_ACCESS_KEY=`kubectl get secret iaasaccesskey -n webide-system -o jsonpath={.data.IaaSAccessKeyID} | base64 -d`
IAAS_ACCESS_SECRET=`kubectl get secret iaasaccesskey -n webide-system -o jsonpath={.data.IaaSAccessKeySecret} | base64 -d`
BUCKET_NAME=`kubectl get deployment fs-infra-manager -n webide-system -o jsonpath='{.spec.template.spec.containers[0].env[?(@.name=="BUCKET_NAME")].value}'`

# create crd restore
SHOOT_JSON=$( kubectl --kubeconfig "${GARDENER_PROJECT_CONFIG}" get shoot "${BACKUP_CLUSTER_NAME}" -o json )
CLOUD_PROFILE=$( echo "$SHOOT_JSON" | jq -rM '.spec.cloudProfileName' )
echo "cloud profile: $CLOUD_PROFILE"
case "$CLOUD_PROFILE" in
"aws")
    AWS_ACCESS_KEY_ID=$IAAS_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$IAAS_ACCESS_SECRET aws s3 cp s3://$BUCKET_NAME/velero/backups/${CRD_BACKUP}/${CRD_BACKUP}.tar.gz .
;;
"alicloud")
    REGION=`kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} get shoot ${PRIMARY_CLUSTER_NAME} -o json | jq '.spec.cloud.region' | tr -d '[:space:]' | tr -d '"'`
    aliyun oss cp oss://$BUCKET_NAME/velero/backups/${CRD_BACKUP}/${CRD_BACKUP}.tar.gz . --region ${REGION} --access-key-id ${IAAS_ACCESS_KEY} --access-key-secret ${IAAS_ACCESS_SECRET}
    # aliyun recreates original object path
    mv velero/backups/${CRD_BACKUP}/${CRD_BACKUP}.tar.gz .
    rm -rf velero
;;
"az")
    CONTAINER_NAME=`kubectl get backupstoragelocations.velero.io default -n velero -o jsonpath={.spec.objectStorage.bucket}`
    AZ_TENANT_ID=`kubectl get secret iaasaccesskey -n webide-system -o jsonpath={.data.AzTenantID} | base64 -d`
    az login --service-principal --username "$IAAS_ACCESS_KEY" --password "$IAAS_ACCESS_SECRET" --tenant "$AZ_TENANT_ID" --output table
    az storage blob download --account-name ${BUCKET_NAME} --container-name ${CONTAINER_NAME} -f ./${CRD_BACKUP}.tar.gz -n velero/backups/${CRD_BACKUP}/${CRD_BACKUP}.tar.gz --output table
;;
*)
    echo "Cloud profile $CLOUD_PROFILE not supported!"
    exit 1
esac
echo

# manually restore crd-backup
mkdir -p ${CRD_BACKUP}
tar -xzvf ${CRD_BACKUP}.tar.gz -C ${CRD_BACKUP}
[ -d "${CRD_BACKUP}/resources/workspaces.devx.sap.com/namespaces/workspaces" ] && \
files=$(ls ${CRD_BACKUP}/resources/workspaces.devx.sap.com/namespaces/workspaces | grep ws-)  || \
{ echo "No workspaces found in primary backup, skipping workspace restore" ; files=(); }

for i in $files; do
    cat ${CRD_BACKUP}/resources/workspaces.devx.sap.com/namespaces/workspaces/$i | jq 'del(.metadata.creationTimestamp, .metadata.resourceVersion, .metadata.uid, .metadata.selfLink)' > tmp.json
    mv -f tmp.json ${CRD_BACKUP}/resources/workspaces.devx.sap.com/namespaces/workspaces/$i
done
[ -d "${CRD_BACKUP}/resources/workspaces.devx.sap.com/namespaces/workspaces" ] && kubectl apply -f ${CRD_BACKUP}/resources/workspaces.devx.sap.com/namespaces/workspaces/

# setup Backup cluster as the Primary one
# change isBackupCluster label to false
kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} patch shoot ${BACKUP_CLUSTER_NAME} -p '{"metadata":{"labels":{"isBackupCluster":"false"}}}'
echo -n 'Waiting for the label of backup-cluster to change...'
for i in {1..40}
do
  state=`kubectl --kubeconfig=${GARDENER_PROJECT_CONFIG} get shoot ${BACKUP_CLUSTER_NAME} -o json | jq '.status.lastOperation.state' | tr -d '[:space:]' | tr -d '"'`
  if [ "$state" = "Succeeded" ]; then
    echo && break
  fi
  sleep 10s
  echo -n '.'
done
echo 'Cluster update finished!'

# update velero values.yaml
kubectl --kubeconfig="./backupConfig.yaml" patch deployment velero -n velero -p '{"spec":{"template":{"spec":{"containers":[{"name": "velero","args":["server"]}]}}}}'

# create schedule for backup cluster
velero create schedule devspace-backup --schedule="${BACKUP_INTERVAL}" --ttl=${BACKUP_TTL} --selector='workspace.devx.sap.com/backup=true' || echo "devspace-backup already exists!"
velero create schedule common-backup --schedule="${BACKUP_INTERVAL}" --ttl=${BACKUP_TTL} --selector='common.devx.sap.com/backup=true' || echo "devspace-backup already exists!"
velero create schedule crd-backup --schedule="${BACKUP_INTERVAL}" --ttl=${BACKUP_TTL} --include-resources='workspaces.devx.sap.com' || echo "crd-backup already exists!"

#TODO - switch monitoring and logging tools to the backup cluster
