
# ARO private ingress, storage, and EgressIP walkthrough

## Goal

Validate a private ARO design pattern for a team-specific workload namespace using:

* a dedicated namespace
* a dedicated private additional ingress controller
* Azure Disk-backed application storage
* Azure Blob storage via the upstream Blob CSI driver
* namespace-scoped `EgressIP`
* private ARO with `UserDefinedRouting` outbound through Azure Firewall

This walkthrough assumes the cluster was already deployed and focuses on the application-side validation and fixes.

---

## Environment

* Platform: **ARO 4.20.15**
* Region: **West US**
* Cluster type: **private ARO**
* Network plugin: **OVNKubernetes**
* Outbound mode: **UserDefinedRouting**
* Downstream egress point: **Azure Firewall**
* Resource group: **`dianasari-udr-rg`**
* Team namespace: **`rhize-team`**

---

## 1. Cluster pre-checks

Confirmed the cluster was in a good state before workload testing.

### Verify cluster version and network type

```bash
oc get clusterversion
oc get network cluster -o yaml
````

### Verify workers

```bash
oc get nodes -o wide
```

### Verify storage classes

```bash
oc get storageclass
```

Observed:

* OpenShift version: `4.20.15`
* network plugin: `OVNKubernetes`
* 3 worker nodes present
* storage classes included `managed-csi` and `azurefile-csi`

---

## 2. Create the team namespace

Created a dedicated namespace and labeled it for later ingress and `EgressIP` selection.

```bash
oc new-project rhize-team
oc label namespace rhize-team team=rhize ingress-scope=rhize name=rhize-team --overwrite
oc get namespace rhize-team --show-labels
```

This gave a clean target for ingress, storage, and egress testing.

---

## 3. Create a private additional ingress controller

### Determine the cluster apps domain

```bash
oc get dns cluster -o jsonpath='{.spec.baseDomain}{"\n"}'
oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}{"\n"}'
```

Observed apps domain:

```text
apps.dianasari-udr.westus.aroapp.io
```

Chosen team subdomain:

```text
rhize.apps.dianasari-udr.westus.aroapp.io
```

### Create a self-signed wildcard cert and secret

```bash
openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout rhize.key \
  -out rhize.crt \
  -subj '/CN=*.rhize.apps.dianasari-udr.westus.aroapp.io' \
  -addext 'subjectAltName=DNS:*.rhize.apps.dianasari-udr.westus.aroapp.io'

oc -n openshift-ingress create secret tls rhize-ingress-tls \
  --cert=rhize.crt \
  --key=rhize.key
```

### Create the additional ingress controller

```yaml
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: rhize
  namespace: openshift-ingress-operator
spec:
  domain: rhize.apps.dianasari-udr.westus.aroapp.io
  defaultCertificate:
    name: rhize-ingress-tls
  routeSelector:
    matchLabels:
      ingress-scope: rhize
  endpointPublishingStrategy:
    type: LoadBalancerService
    loadBalancer:
      scope: Internal
  replicas: 2
```

Apply:

```bash
oc apply -f rhize-ingresscontroller.yaml
```

### Validate

```bash
oc get ingresscontroller -n openshift-ingress-operator rhize
oc get svc -n openshift-ingress router-rhize
```

Observed:

* `router-rhize` came up successfully
* the router received an **internal** Azure load balancer IP

---

## 4. Validate an Azure Disk-backed sample app

Created a sample app backed by Azure Disk storage.

### PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rhize-disk-pvc
  namespace: rhize-team
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: managed-csi
```

### Deployment and Service

A sample nginx-based app was deployed and mounted the PVC.

### Initial issue

The HTTPS route returned `403 Forbidden`.

Root cause:

* content was written to `/usr/share/nginx/html`
* the image being used effectively served content from `/opt/app-root/src`

### Fix

Updated the deployment to:

* mount the PVC at `/opt/app-root/src`
* write `index.html` there instead

### Validate

```bash
oc get pvc -n rhize-team rhize-disk-pvc
oc get pods -n rhize-team
```

Observed:

* PVC bound successfully
* pod came up successfully
* app served content correctly from the Azure Disk volume

---

## 5. Expose the app through the additional private ingress

Created a Route for the app and labeled it so only the additional ingress controller would admit it.

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: rhize-disk-app
  namespace: rhize-team
  labels:
    ingress-scope: rhize
spec:
  host: rhize-disk-app.rhize.apps.dianasari-udr.westus.aroapp.io
  to:
    kind: Service
    name: rhize-disk-app
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

### Validate

```bash
oc get route -n rhize-team rhize-disk-app
curl -k --resolve rhize-disk-app.rhize.apps.dianasari-udr.westus.aroapp.io:443:<private-rhize-lb-ip> \
  https://rhize-disk-app.rhize.apps.dianasari-udr.westus.aroapp.io/
```

Observed:

* route admitted successfully
* app reachable through the **private** team ingress controller

At this stage, the following were validated:

* dedicated namespace
* additional private ingress controller
* Azure Disk-backed sample app
* route reachable via the team-specific private ingress

---

## 6. Create Azure Blob resources

Created the Azure-side resources for Blob storage.

```bash
export AZ_RG=dianasari-udr-rg
export AZ_LOCATION=westus
export AZ_STORAGE_ACCOUNT=dsarirhize13779
export AZ_CONTAINER=rhizeblob
```

### Create storage account and container

```bash
az storage account create \
  --name "$AZ_STORAGE_ACCOUNT" \
  --resource-group "$AZ_RG" \
  --location "$AZ_LOCATION" \
  --sku Standard_LRS

AZ_STORAGE_KEY=$(az storage account keys list \
  -g "$AZ_RG" \
  -n "$AZ_STORAGE_ACCOUNT" \
  --query '[0].value' -o tsv)

az storage container create \
  --name "$AZ_CONTAINER" \
  --account-name "$AZ_STORAGE_ACCOUNT" \
  --account-key "$AZ_STORAGE_KEY"
```

Observed:

* storage account created successfully
* container `rhizeblob` created successfully

---

## 7. Install the upstream Blob CSI driver with Helm

Installed the upstream `blob-csi-driver` chart into `kube-system`.

```bash
helm repo add blob-csi-driver https://raw.githubusercontent.com/kubernetes-sigs/blob-csi-driver/master/charts
helm repo update
helm upgrade --install blob-csi-driver blob-csi-driver/blob-csi-driver \
  --namespace kube-system
```

### Initial result

* controller pods came up
* node DaemonSet pods failed

---

## 8. Fix the Blob CSI node DaemonSet on ARO/RHCOS

### Initial node failure

The node pods initially failed because the chart expected `/host/usr/local` to exist as a normal hostPath-backed directory.

### Investigation

The init container command was temporarily changed to inspect the host filesystem layout.

Observed on ARO/RHCOS:

```text
/host/usr/local -> ../var/usrlocal
```

So `/usr/local` on the host was a symlink into `/var/usrlocal`, not a normal directory layout matching the upstream chart assumption.

### Working workaround

Patched the node DaemonSet to:

* add a hostPath mount for `/var/usrlocal`
* mount it into the init container as `/host/var/usrlocal`
* create `/host/usr/local/bin` before running the original install script
* scope the node DaemonSet to workers only

After the workaround:

```bash
oc get pods -n kube-system -o wide | grep csi-blob-node
```

Observed:

* all `csi-blob-node-*` pods became healthy
* node pods ran on workers only
* blobfuse installation/init completed successfully

This was an ARO/RHCOS-specific adjustment worth documenting.

---

## 9. Dynamic Blob provisioning attempt

Created:

* a Blob secret in `rhize-team`
* a `StorageClass` using `blob.csi.azure.com`
* a PVC
* a test pod

### Result

The PVC remained `Pending`.

### Root cause

Controller logs showed the upstream driver expected Azure controller-side configuration that was not present in this lab, including:

* secret `azure-cloud-provider` in `kube-system`
* `/etc/kubernetes/azure.json`

Without usable Azure cloud config / identity, provisioning failed.

### Conclusion

Dynamic provisioning was **not validated** in this lab and remained blocked by missing controller-side Azure config / identity.

---

## 10. Static Blob provisioning

Switched to a static PV/PVC model pointing at the existing Blob container.

### Create the namespace secret

```bash
cat <<EOF | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: azure-blob-secret
  namespace: rhize-team
type: Opaque
stringData:
  azurestorageaccountname: ${AZ_STORAGE_ACCOUNT}
  azurestorageaccountkey: ${AZ_STORAGE_KEY}
EOF
```

### First static attempt and failure

The PV/PVC bound, but the pod mount failed.

Initial mount-time issue:

* BlobFuse tried to create its default work directory at `/.blobfuse2`
* the filesystem was read-only there

### Working static fix

The static PV was recreated with:

* `protocol: fuse2`
* a writable BlobFuse2 working directory under `/mnt/blobfuse2workdir`
* the correct secret reference name: `azure-blob-secret`

Combined PV/PVC manifest used:

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-rhize-blob
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: azureblob-fuse-static
  mountOptions:
    - --file-cache-timeout-in-seconds=120
    - --use-attr-cache=true
    - --cancel-list-on-mount-seconds=10
    - --log-level=LOG_DEBUG
    - --default-working-dir=/mnt/blobfuse2workdir
  csi:
    driver: blob.csi.azure.com
    readOnly: false
    volumeHandle: pv-rhize-blob-handle
    volumeAttributes:
      resourceGroup: dianasari-udr-rg
      storageAccount: dsarirhize13779
      containerName: rhizeblob
      protocol: fuse2
    nodeStageSecretRef:
      name: azure-blob-secret
      namespace: rhize-team
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-rhize-blob
  namespace: rhize-team
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: azureblob-fuse-static
  volumeName: pv-rhize-blob
EOF
```

### Validate static Blob mount

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: rhize-blob-test
  namespace: rhize-team
spec:
  containers:
  - name: test
    image: registry.access.redhat.com/ubi9/ubi-minimal
    command: ["/bin/sh", "-c"]
    args:
      - >
        echo "hello from blob" > /data/hello.txt &&
        ls -l /data &&
        cat /data/hello.txt &&
        sleep 3600
    volumeMounts:
    - name: blobvol
      mountPath: /data
  volumes:
  - name: blobvol
    persistentVolumeClaim:
      claimName: pvc-rhize-blob
EOF
```

Observed:

* pod reached `Running`
* `/data/hello.txt` was created successfully
* additional writes succeeded
* files remained present after pod deletion and recreation

### Conclusion

Static Azure Blob storage on ARO was validated successfully for:

* mount
* read
* write
* persistence across pod recreation

---

## 11. Validate namespace-scoped EgressIP

This reused the same private ARO + UDR + Azure Firewall model from the earlier lab, but targeted the application namespace used in this walkthrough.

### Check worker node capability

```bash
oc get nodes -o wide
oc get node -l node-role.kubernetes.io/worker= -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{.metadata.annotations.cloud\.network\.openshift\.io/egress-ipconfig}{"\n\n"}{end}'
oc get nodes -L k8s.ovn.org/egress-assignable
```

Observed:

* worker subnet `10.0.2.0/23`
* workers already had `cloud.network.openshift.io/egress-ipconfig`
* workers already had the `k8s.ovn.org/egress-assignable` label

### Create the `EgressIP`

```bash
cat <<'EOF' | oc apply -f -
apiVersion: k8s.ovn.org/v1
kind: EgressIP
metadata:
  name: rhize-egressip
spec:
  egressIPs:
  - 10.0.2.50
  namespaceSelector:
    matchLabels:
      name: rhize-team
EOF
```

### Verify assignment objects

```bash
oc get egressip rhize-egressip -o yaml
oc get cloudprivateipconfig 10.0.2.50 -o yaml
```

Observed:

* private EgressIP `10.0.2.50` assigned successfully
* `CloudPrivateIPConfig` showed the IP assigned to worker `dianasari-udr-xcr45-worker-westus-qqhk4`
* status showed `Assigned=True`

### Validate outbound behavior from the namespace

```bash
oc -n rhize-team create deployment curl --image=registry.access.redhat.com/ubi9/ubi-minimal -- sleep infinity
oc -n rhize-team rollout status deploy/curl
oc -n rhize-team exec deploy/curl -- bash -lc 'curl -s --connect-timeout 5 ifconfig.me; echo'
oc -n rhize-team exec deploy/curl -- bash -lc 'while true; do date; curl -s --connect-timeout 2 ifconfig.me || echo FAIL; echo; sleep 2; done'
```

Observed externally visible source IP:

```text
172.184.103.46
```

This matched the Azure Firewall public IP, which was expected because outbound traffic still followed the UDR → Azure Firewall path.

### Conclusion

`EgressIP` validation for `rhize-team` was successful:

* namespace-scoped EgressIP selection worked
* the private EgressIP was assigned to a worker
* outbound traffic remained stable
* externally visible SNAT remained the Azure Firewall public IP

Note:

* the test reused `10.0.2.50` from an earlier lab, so the existing `CloudPrivateIPConfig` metadata may reference the earlier object name even though the assignment behavior was still valid

---

## 12. Final validation summary

### Fully validated

* team namespace creation and labeling
* additional private ingress controller scoped by route label
* Azure Disk-backed sample app
* private Route through the additional ingress controller
* Blob CSI node DaemonSet workaround on ARO/RHCOS
* static Azure Blob mount/read/write/persistence
* namespace-scoped `EgressIP` behavior for `rhize-team`

### Partially validated

* Azure Blob resources created successfully
* Blob CSI driver installed successfully

### Not fully validated

* dynamic Azure Blob provisioning with the upstream Blob CSI driver

  * blocked by missing Azure controller-side config / identity in this lab

---

## 13. Operational takeaways

* additional private ingress controllers are a practical way to isolate team/app Routes behind distinct internal load balancer IPs
* image/path assumptions in sample containers can cause false ingress errors; the Azure Disk app issue was actually a content path issue
* the upstream Blob CSI node chart needed ARO/RHCOS-specific adjustment because `/usr/local` on the host resolved via `/var/usrlocal`
* for static Blob volumes, `protocol: fuse2` plus a writable BlobFuse2 working directory resolved the read-only default workdir failure
* dynamic Blob provisioning remains a separate controller-identity/config problem, not a node-driver problem
* in private ARO with UDR + Azure Firewall, `EgressIP` can still pin namespace traffic to a private worker-side IP while the externally visible public source IP remains the Azure Firewall SNAT IP

---

## 14. Useful commands

```bash
oc get nodes -o wide
oc get nodes -L k8s.ovn.org/egress-assignable
oc get ingresscontroller -n openshift-ingress-operator
oc get svc -n openshift-ingress
oc get route -n rhize-team
oc get pvc -n rhize-team
oc get pod -n rhize-team -o wide
oc describe pod -n rhize-team rhize-blob-test
oc get pods -n kube-system -o wide | grep csi-blob-node
oc logs -n kube-system <csi-blob-node-pod> -c blob
oc get egressip rhize-egressip -o yaml
oc get cloudprivateipconfig -o yaml
oc -n rhize-team exec deploy/curl -- bash -lc 'while true; do date; curl -s --connect-timeout 2 ifconfig.me || echo FAIL; echo; sleep 2; done'
```
