# ARO private UDR + EgressIP test runbook

## Goal

Validate how `EgressIP` behaves on a private ARO cluster using:

* private API / ingress
* `UserDefinedRouting` outbound
* Azure Firewall in the downstream egress path

The main questions tested were:

1. how quickly OVN reassigns the `EgressIP`
2. whether outbound traffic continues through the UDR + Azure Firewall path
3. whether the externally visible public SNAT identity remains stable
4. whether there is any visible interruption during reassignment

---

## Environment

* Platform: **ARO 4.20.15**
* Region: **West US**
* Deployment method: **Terraform** using `rh-mobb/terraform-aro`
* Cluster type: **private ARO**
* Outbound mode: **UserDefinedRouting**
* Downstream egress point: **Azure Firewall**

---

## 1. Terraform preparation

Started from the `rh-mobb/terraform-aro` repo.

### Create new branch

```bash
git checkout -b andersen-udr-test
```

### Terraform version

The repo required Terraform `>= 1.12`, so Terraform was updated with `tfenv`.

```bash
tfenv install 1.12.0
tfenv use 1.12.0
terraform version
```

### Reinitialize providers

```bash
rm -rf .terraform
terraform init -upgrade
```

### Clean start

An old ARO state / resource group existed from prior testing, so the environment was reset before building the new test cluster.

---

## 2. Terraform variables used

```bash
export TF_VAR_pull_secret_path="$HOME/Downloads/pull-secret.txt"
export TF_VAR_subscription_id="$(az account show --query id --output tsv)"
export TF_VAR_cluster_name="$(whoami)-udr"
export TF_VAR_resource_group_name="$(whoami)-udr-rg"
export TF_VAR_location="westus"
export TF_VAR_aro_version="$(az aro get-versions -l westus --query '[-1]' -o tsv)"
export TF_VAR_api_server_profile="Private"
export TF_VAR_ingress_profile="Private"
export TF_VAR_outbound_type="UserDefinedRouting"
export TF_VAR_restrict_egress_traffic="true"
export TF_VAR_domain="$(whoami)-udr"
```

### Deploy

```bash
terraform init -upgrade
terraform plan -out tf.plan
terraform apply tf.plan
```

---

## 3. Post-deploy validation

Terraform outputs confirmed:

* private API IP
* private ingress IP
* jumphost public IP
* Azure Firewall present in the environment

### Confirm jumphost

```bash
az vm list -g dianasari-udr-rg -o table
```

### Confirm public IPs

```bash
az vm list-ip-addresses -g dianasari-udr-rg -n dianasari-udr-jumphost -o yaml
az network public-ip list -g dianasari-udr-rg -o table
```

Observed:

* **jumphost public IP:** `20.245.86.105`
* **Azure Firewall public IP:** `172.184.103.46`

---

## 4. Access the private cluster

Because the cluster API was private, direct `oc login` from a laptop failed.

### Find jumphost username

```bash
az vm show -g dianasari-udr-rg -n dianasari-udr-jumphost --query osProfile.adminUsername -o tsv
```

Observed username:

```text
aro
```

### Reach the VNet with sshuttle

```bash
sshuttle --dns -r aro@20.245.86.105 10.0.0.0/20
```

### Log in to OpenShift

```bash
oc login https://api.dianasari-udr.westus.aroapp.io:6443 -u kubeadmin -p '<kubeadmin-password>'
```

---

## 5. Inspect worker nodes and subnet

### List nodes

```bash
oc get nodes -o wide
```

Observed worker nodes:

* `dianasari-udr-xcr45-worker-westus-2rt49` -> `10.0.2.4`
* `dianasari-udr-xcr45-worker-westus-qqhk4` -> `10.0.2.5`
* `dianasari-udr-xcr45-worker-westus-zqp7s` -> `10.0.2.6`

### Check Azure EgressIP capability annotation

```bash
oc get node -l node-role.kubernetes.io/worker= -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{.metadata.annotations.cloud\.network\.openshift\.io/egress-ipconfig}{"\n\n"}{end}'
```

Observed worker subnet:

```text
10.0.2.0/23
```

---

## 6. Create test namespace and workload

```bash
oc new-project egress-test
oc create deployment curl --image=registry.access.redhat.com/ubi9/ubi-minimal -- sleep infinity
oc -n egress-test rollout status deploy/curl
```

---

## 7. Mark workers as EgressIP-assignable

```bash
oc label node dianasari-udr-xcr45-worker-westus-2rt49 k8s.ovn.org/egress-assignable=""
oc label node dianasari-udr-xcr45-worker-westus-qqhk4 k8s.ovn.org/egress-assignable=""
```

---

## 8. Create the EgressIP object

Selected test EgressIP:

```text
10.0.2.50
```

### Manifest

```yaml
apiVersion: k8s.ovn.org/v1
kind: EgressIP
metadata:
  name: egressip-test
spec:
  egressIPs:
  - 10.0.2.50
  namespaceSelector:
    matchLabels:
      kubernetes.io/metadata.name: egress-test
```

### Apply

```bash
oc apply -f egressip-test.yaml
```

### Verify assignment

```bash
oc get egressip egressip-test -o yaml
oc get cloudprivateipconfig
```

Observed:

* EgressIP assigned to `dianasari-udr-xcr45-worker-westus-2rt49`
* `CloudPrivateIPConfig` created for `10.0.2.50`

---

## 9. Verify outbound path

### Continuous curl loop

```bash
oc -n egress-test exec deploy/curl -- bash -lc 'while true; do date; curl -s --connect-timeout 2 ifconfig.me || echo FAIL; echo; sleep 2; done'
```

Observed externally visible source IP:

```text
172.184.103.46
```

This matched the Azure Firewall public IP, confirming outbound traffic was following the UDR → Azure Firewall path.

---

## 10. Controlled reassignment test

### 10.1 Drain-only behavior

```bash
oc adm cordon dianasari-udr-xcr45-worker-westus-2rt49
oc adm drain dianasari-udr-xcr45-worker-westus-2rt49 --ignore-daemonsets --delete-emptydir-data --force
```

Observed:

* EgressIP ownership did **not** move
* `CloudPrivateIPConfig` remained on `2rt49`
* curl stayed steady

### Finding

A simple cordon + drain did **not** make the node unavailable enough to trigger EgressIP reassignment.

### 10.2 Forced eligibility change

Remove EgressIP eligibility from the current owner:

```bash
oc label node dianasari-udr-xcr45-worker-westus-2rt49 k8s.ovn.org/egress-assignable-
```

Watch reassignment:

```bash
oc get egressip egressip-test -w
oc get cloudprivateipconfig -w
```

Observed:

* ownership moved from `2rt49` to `qqhk4`
* handoff happened within seconds
* curl remained steady
* externally visible IP stayed `172.184.103.46`

### Finding

Controlled EgressIP reassignment was fast and non-disruptive in this test.

---

## 11. Actual node-loss simulation

### 11.1 Identify the owning worker Machine

```bash
oc get machine -n openshift-machine-api -o wide
```

### 11.2 Delete the owning Machine

```bash
oc delete machine dianasari-udr-xcr45-worker-westus-2rt49 -n openshift-machine-api
```

### Observed initial behavior

* EgressIP became unassigned
* `CloudPrivateIPConfig` disappeared
* replacement worker `x6t7d` was provisioned
* curl continued briefly through the firewall path

### 11.3 Check eligible nodes

```bash
oc get nodes -L k8s.ovn.org/egress-assignable
```

Observed:

* no remaining workers were labeled `egress-assignable`
* replacement worker did **not** inherit the label automatically

### 11.4 Restore eligibility

```bash
oc label node dianasari-udr-xcr45-worker-westus-qqhk4 k8s.ovn.org/egress-assignable=""
oc label node dianasari-udr-xcr45-worker-westus-x6t7d k8s.ovn.org/egress-assignable=""
```

### Observed recovery

* EgressIP reassigned to `qqhk4`
* new `CloudPrivateIPConfig` created for `10.0.2.50`
* curl showed **one failed attempt**
* traffic immediately resumed
* externally visible source IP remained `172.184.103.46`

Sample observed curl blip:

```text
Thu Mar 19 22:47:35 UTC 2026
172.184.103.46
Thu Mar 19 22:47:37 UTC 2026
FAIL
Thu Mar 19 22:47:40 UTC 2026
172.184.103.46
```

Approximate visible interruption: **~3 seconds**

---

## 12. Findings summary

### Controlled reassignment

* removing `egress-assignable` from the owning worker caused fast reassignment
* reassignment occurred within seconds
* no visible curl interruption
* external public SNAT remained the Azure Firewall public IP

### Actual node-loss simulation

* deleting the owning Machine caused temporary EgressIP unassignment
* reassignment was not automatic unless another worker was already labeled `egress-assignable`
* replacement workers do **not** inherit the label automatically
* once eligibility was restored, reassignment completed
* observed about one failed curl attempt during recovery
* external public SNAT still remained the Azure Firewall public IP

---

## 13. Operational takeaways

* EgressIP failover depends on having other workers already labeled:

  ```text
  k8s.ovn.org/egress-assignable
  ```

* A replacement worker does **not** automatically inherit that label.

* Draining a worker is not the same as true node unavailability for EgressIP reassignment.

* In this private ARO + UDR + Azure Firewall setup:

  * EgressIP ownership can move between workers
  * Azure Firewall public IP can remain the stable externally visible source IP
  * an actual node-loss event may still introduce a brief interruption during reassignment

---

## 14. Most useful commands used during testing

```bash
oc get nodes -o wide
oc get nodes -L k8s.ovn.org/egress-assignable
oc get egressip egressip-test -o yaml
oc get egressip egressip-test -w
oc get cloudprivateipconfig
oc get cloudprivateipconfig -o yaml
oc get cloudprivateipconfig -w
oc get machine -n openshift-machine-api -o wide
oc -n egress-test exec deploy/curl -- bash -lc 'while true; do date; curl -s --connect-timeout 2 ifconfig.me || echo FAIL; echo; sleep 2; done'
```

---

