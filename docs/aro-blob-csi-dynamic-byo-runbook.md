# ARO Blob CSI dynamic provisioning runbook (BYO storage account)

## Overview

This runbook captures the dynamic Azure Blob CSI validation steps performed on the private ARO lab in this repository.

It focuses specifically on the **working dynamic provisioning path**:

* Azure Blob CSI on ARO
* **Bring your own storage account**
* BlobFuse2 (`protocol: fuse2`)
* PVC dynamically provisioned by the CSI driver
* Pod successfully mounted and accessed the volume

It also records the important troubleshooting findings that were required to get this working, including controller credential fixes and the BlobFuse2 working-directory fix.

## What was validated

Validated successfully:

* Blob CSI controller pods healthy on ARO after wiring `azure-cred-file` and a valid `azure-cloud-provider` secret
* Dynamic provisioning using a **pre-existing Azure storage account** and secret-based StorageClass
* RWX PVC created dynamically by the CSI driver
* Test pod mounted the dynamically provisioned volume successfully
* Read access inside the pod validated with `df -h` and `ls -la`

Not validated in this runbook:

* Driver-managed storage account creation path (the path where the controller creates or discovers storage accounts automatically)

That driver-managed path still hit Azure control-plane access issues against the managed resource group and is documented here as a known limitation/next-step area

## Lab context

Cluster and resources used during testing:

* ARO cluster from this repo's private ingress / storage / EgressIP lab
* Worker node example: `dianasari-udr-xcr45-worker-westus-x6t7d`
* VNet resource group: `dianasari-udr-rg`
* Storage account: `dsarirhize13779`
* Test namespace: `rhize-team`
* Blob container used for the BYO dynamic test: `rhizeblobdyn`

## Key findings

1. **Static Blob PV/PVC worked first** and helped prove that the node-side Blob CSI path was basically functional.
2. Initial **dynamic provisioning failed** because the Blob CSI controller lacked usable Azure cloud config and credentials.
3. Adding the `azure-cred-file` ConfigMap to point the controller to `/etc/kubernetes/cloud.conf` fixed the first controller bootstrap problem.
4. Dynamic provisioning still failed until a real `kube-system/azure-cloud-provider` secret was created with a valid service principal-backed cloud config.
5. The **BYO storage account** dynamic path still worked after controller credentials were fixed.
6. The test pod initially failed to mount with BlobFuse2 because it tried to create `/.blobfuse2` on a read-only filesystem.
7. Adding an explicit writable BlobFuse2 working directory fixed the mount issue:

   * `--default-working-dir=/tmp/blobfuse2`
8. Final result: **dynamic Blob provisioning on ARO was validated using a BYO storage account + secret-based StorageClass + explicit BlobFuse2 working directory**.

## Troubleshooting summary

### 1. Controller had no usable Azure cloud config initially

Symptoms seen earlier in testing:

* controller looked for `kube-system/azure-cloud-provider`
* fallback behavior referenced missing `/etc/kubernetes/azure.json`
* dynamic PVCs failed during `CreateVolume`
* earlier logs showed `clientFactory is nil`

### 2. `azure-cred-file` was required on ARO

The controller eventually worked better after creating:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: azure-cred-file
  namespace: kube-system
data:
  path: /etc/kubernetes/cloud.conf
  path-windows: C:\\k\\cloud.conf
EOF

oc rollout restart deploy/csi-blob-controller -n kube-system
oc rollout status deploy/csi-blob-controller -n kube-system
```

### 3. The controller still needed real Azure credentials

The host had `/etc/kubernetes/cloud.conf`, but that was not sufficient by itself for the controller-side Azure operations required by dynamic provisioning.

A real `azure-cloud-provider` secret had to be created in `kube-system` using valid service principal values.


### 4. BlobFuse2 default work dir broke the pod mount

Even after the PVC bound successfully, the pod initially stayed in `ContainerCreating` because the node mount failed with:

```text
failed to create default work dir [mkdir /.blobfuse2: read-only file system]
```

This was fixed by explicitly setting:

```text
--default-working-dir=/tmp/blobfuse2
```

## Step 1: export Azure variables

Use the subscription and tenant from the active Azure CLI context:

```bash
export SUBSCRIPTION_ID="$(az account show --query id -o tsv)"
export TENANT_ID="$(az account show --query tenantId -o tsv)"

export VNET_RG="dianasari-udr-rg"
export STORAGE_RG="dianasari-udr-rg"
export STORAGE_ACCOUNT="dsarirhize13779"
export AZ_CONTAINER="rhizeblobdyn"

export VNET_RG_SCOPE="/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${VNET_RG}"
export STORAGE_SCOPE="/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${STORAGE_RG}/providers/Microsoft.Storage/storageAccounts/${STORAGE_ACCOUNT}"
```

Optional sanity checks:

```bash
echo "$SUBSCRIPTION_ID"
echo "$TENANT_ID"
echo "$VNET_RG_SCOPE"
echo "$STORAGE_SCOPE"
```

## Step 2: create a dedicated service principal for Blob CSI testing

Create a new test-specific service principal:

```bash
SP_JSON="$(az ad sp create-for-rbac --name "blob-csi-aro-test-$(date +%Y%m%d%H%M%S)" --skip-assignment)"

export APP_ID="$(printf '%s' "$SP_JSON" | python3 -c 'import sys, json; print(json.load(sys.stdin)["appId"])')"
export APP_SECRET="$(printf '%s' "$SP_JSON" | python3 -c 'import sys, json; print(json.load(sys.stdin)["password"])')"

echo "$APP_ID"
echo "${APP_SECRET:0:8}"
```

## Step 3: assign Azure roles

The service principal was successfully granted:

* `Contributor` on the VNet resource group
* `Storage Account Contributor` on the storage account


### Successful assignments

```bash
az role assignment create --assignee "$APP_ID" --role "Contributor" --scope "$VNET_RG_SCOPE"
az role assignment create --assignee "$APP_ID" --role "Storage Account Contributor" --scope "$STORAGE_SCOPE"
```

### Verification

Use `--all`, otherwise resource-group and resource-scoped assignments may not show up:

```bash
az role assignment list --assignee "$APP_ID" --all -o table
```

You can also check the scopes directly:

```bash
az role assignment list --assignee "$APP_ID" --scope "$VNET_RG_SCOPE" -o table
az role assignment list --assignee "$APP_ID" --scope "$STORAGE_SCOPE" -o table
```


## Step 4: wire controller cloud config and credentials

### 4.1 Create `azure-cred-file`

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: azure-cred-file
  namespace: kube-system
data:
  path: /etc/kubernetes/cloud.conf
  path-windows: C:\\k\\cloud.conf
EOF
```

### 4.2 Create a real `azure-cloud-provider` secret

Create a cloud-config JSON using the real service principal values:

```bash
cat <<EOF > azure-cloud-provider.json
{
  "cloud":"AzurePublicCloud",
  "tenantId":"${TENANT_ID}",
  "subscriptionId":"${SUBSCRIPTION_ID}",
  "aadClientId":"${APP_ID}",
  "aadClientSecret":"${APP_SECRET}",
  "resourceGroup":"dianasari-udr-rg-managed",
  "location":"westus",
  "vnetName":"dianasari-udr-vnet",
  "vnetResourceGroup":"dianasari-udr-rg",
  "subnetName":"dianasari-udr-machine-subnet",
  "securityGroupName":"dianasari-udr-xcr45-nsg",
  "routeTableName":"dianasari-udr-xcr45-node-routetable",
  "useManagedIdentityExtension": false,
  "useInstanceMetadata": true,
  "loadBalancerSku":"standard"
}
EOF
```

Apply it:

```bash
oc -n kube-system create secret generic azure-cloud-provider \
  --from-file=cloud-config=azure-cloud-provider.json \
  --dry-run=client -o yaml | oc apply -f -
```

### 4.3 Restart the controller

```bash
oc rollout restart deploy/csi-blob-controller -n kube-system
oc rollout status deploy/csi-blob-controller -n kube-system
```

### 4.4 Verify controller health

```bash
oc get pods -n kube-system -l app=csi-blob-controller -o wide
oc logs -n kube-system deploy/csi-blob-controller -c blob --tail=100 | cat
```

Expected healthy state:

* both controller pods `4/4 Running`
* no CrashLoopBackOff
* logs show the controller reading `kube-system/azure-cloud-provider`
* ARM client factories initialize successfully

## Step 5: create the storage account key secret for BYO provisioning

Retrieve a storage account key and export it:

```bash
export AZ_STORAGE_KEY="$(az storage account keys list \
  --resource-group "$STORAGE_RG" \
  --account-name "$STORAGE_ACCOUNT" \
  --query '[0].value' \
  -o tsv)"

echo "$STORAGE_ACCOUNT"
echo "$AZ_CONTAINER"
echo "${AZ_STORAGE_KEY:0:8}"
```

Create the Kubernetes secret used by the Blob CSI driver:

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: azure-secret
  namespace: rhize-team
type: Opaque
stringData:
  azurestorageaccountname: ${STORAGE_ACCOUNT}
  azurestorageaccountkey: ${AZ_STORAGE_KEY}
EOF
```

## Step 6: create the working BYO dynamic StorageClass

This is the validated StorageClass pattern. This validated path uses BlobFuse2 via `protocol: fuse2`.

Important fix included here:

* `--default-working-dir=/tmp/blobfuse2`

That option was required to avoid BlobFuse2 trying to create `/.blobfuse2` on a read-only filesystem.

```bash
cat <<EOF | oc apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azureblob-fuse-dynamic-byo
provisioner: blob.csi.azure.com
parameters:
  containerName: ${AZ_CONTAINER}
  csi.storage.k8s.io/provisioner-secret-name: azure-secret
  csi.storage.k8s.io/provisioner-secret-namespace: rhize-team
  csi.storage.k8s.io/node-stage-secret-name: azure-secret
  csi.storage.k8s.io/node-stage-secret-namespace: rhize-team
  protocol: fuse2
reclaimPolicy: Delete
volumeBindingMode: Immediate
allowVolumeExpansion: true
mountOptions:
  - -o allow_other
  - --file-cache-timeout-in-seconds=120
  - --use-attr-cache=true
  - --cancel-list-on-mount-seconds=10
  - --default-working-dir=/tmp/blobfuse2
  - -o attr_timeout=120
  - -o entry_timeout=120
  - -o negative_timeout=120
  - --log-level=LOG_WARNING
EOF
```

## Step 7: create the dynamic PVC

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rhize-blob-dynamic-byo-pvc
  namespace: rhize-team
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azureblob-fuse-dynamic-byo
  resources:
    requests:
      storage: 5Gi
EOF
```

Validate that the PVC binds and a PV is created dynamically:

```bash
oc describe pvc rhize-blob-dynamic-byo-pvc -n rhize-team | cat
oc get pv
oc get events -n rhize-team --sort-by=.lastTimestamp
```

Expected result:

* PVC transitions to `Bound`
* a new PV is created automatically by the CSI driver

## Step 8: create a test pod and validate the mount

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: blob-dynamic-byo-test
  namespace: rhize-team
spec:
  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi-minimal
      command: ["/bin/sh", "-c", "sleep 3600"]
      volumeMounts:
        - name: blobvol
          mountPath: /mnt/blob
  volumes:
    - name: blobvol
      persistentVolumeClaim:
        claimName: rhize-blob-dynamic-byo-pvc
EOF
```

Check pod state:

```bash
oc get pod blob-dynamic-byo-test -n rhize-team
oc describe pod blob-dynamic-byo-test -n rhize-team | cat
```

Validate inside the pod:

```bash
oc exec -n rhize-team blob-dynamic-byo-test -- sh -c 'df -h /mnt/blob; ls -la /mnt/blob'
oc exec -n rhize-team blob-dynamic-byo-test -- sh -c 'echo hello > /mnt/blob/hello.txt && ls -la /mnt/blob && cat /mnt/blob/hello.txt'
```

Expected successful output:

* pod status `Running`
* `df -h /mnt/blob` shows `blobfuse2`
* listing `/mnt/blob` succeeds
* test file creation succeeds

## Useful diagnostics

### Controller logs

```bash
oc logs -n kube-system deploy/csi-blob-controller -c blob --tail=100 | cat
oc logs -n kube-system deploy/csi-blob-controller -c csi-provisioner --tail=100 | cat
```

### Check controller pods

```bash
oc get pods -n kube-system -l app=csi-blob-controller -o wide
```

### Describe PVC and pod

```bash
oc describe pvc rhize-blob-dynamic-byo-pvc -n rhize-team | cat
oc describe pod blob-dynamic-byo-test -n rhize-team | cat
```

### Confirm cloud config secret content shape

Do not share secrets, but this is useful for local validation:

```bash
oc -n kube-system get secret azure-cloud-provider -o jsonpath='{.data.cloud-config}' | base64 -d ; echo
```

### Check Azure role assignments

```bash
az role assignment list --assignee "$APP_ID" --all -o table
```

## Failure patterns seen during testing

### `clientFactory is nil`

Cause:

* controller had no usable Azure cloud config / credentials

Fix:

* create `azure-cred-file`
* create a real `azure-cloud-provider` secret
* restart controller

### Controller CrashLoop with Azure identity panic

Cause:

* controller had cloud config but no usable Azure credentials/token path

Fix:

* create a real service principal-backed `azure-cloud-provider` secret
* restart controller

### Pod stuck in `ContainerCreating` with BlobFuse2 error

Cause:

* BlobFuse2 tried to create `/.blobfuse2` on a read-only filesystem

Fix:

* add `--default-working-dir=/tmp/blobfuse2` in the StorageClass mount options

## Final validated outcome

The following path was validated successfully on this ARO cluster:

* Blob CSI controller configured with `azure-cred-file` and a valid service-principal-backed `azure-cloud-provider` secret
* BYO Azure storage account
* secret-based StorageClass
* `protocol: fuse2`
* explicit BlobFuse2 working directory
* dynamically provisioned RWX PVC
* successful mount inside a running pod

## Optional cleanup

```bash
oc delete pod blob-dynamic-byo-test -n rhize-team
oc delete pvc rhize-blob-dynamic-byo-pvc -n rhize-team
oc delete sc azureblob-fuse-dynamic-byo
oc delete secret azure-secret -n rhize-team
oc delete secret azure-cloud-provider -n kube-system
oc delete configmap azure-cred-file -n kube-system
az ad sp delete --id "$APP_ID"
az ad app delete --id "$APP_ID"