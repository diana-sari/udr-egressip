# udr-egressip

Testing notes, manifests, and findings for private ARO with:

- UserDefinedRouting (UDR)
- Azure Firewall in the downstream egress path
- OVN Kubernetes EgressIP behavior
- private ingress and storage validation patterns

## Scope

This repo documents lab validation for private ARO patterns including:

- private ARO deployment with Terraform
- EgressIP assignment on Azure via CloudPrivateIPConfig
- controlled EgressIP reassignment
- worker node-loss simulation
- observed outbound behavior through Azure Firewall
- additional private ingress controller patterns
- internal load balancer to router to Route validation
- Azure Disk-backed application validation
- Azure Blob CSI validation on ARO/RHCOS

## Key findings

- EgressIP controlled reassignment between eligible workers completed within seconds.
- A simple cordon/drain did not by itself trigger EgressIP reassignment.
- Deleting the owning worker Machine caused temporary EgressIP unassignment.
- Reassignment required another worker already labeled `k8s.ovn.org/egress-assignable`.
- Replacement workers did not automatically inherit the `egress-assignable` label.
- The externally visible source IP remained the Azure Firewall public IP throughout testing.
- During actual node-loss recovery, one brief traffic blip was observed.
- Additional private ingress controllers can be used to isolate team or app Routes behind distinct internal load balancer IPs.
- Internal ingress was validated end to end through the flow: Azure internal load balancer → OpenShift router → Route → ClusterIP service → pod.
- Static Azure Blob storage worked on ARO after adjusting the upstream Blob CSI node setup for the RHCOS `/var/usrlocal` layout and using `protocol: fuse2` with a writable BlobFuse2 working directory.
- Dynamic Azure Blob provisioning was validated on ARO for the BYO storage account path after wiring `azure-cred-file` and a valid `azure-cloud-provider` secret, using `protocol: fuse2` and an explicit BlobFuse2 working directory.
- The driver-managed storage account creation path was **not** validated in this lab.

## Repo layout

- `docs/aro-private-udr-egressip-test.md`: focused runbook for private ARO UDR + EgressIP assignment, reassignment, and failover behavior
- `docs/aro-private-ingress-storage-egressip-walkthrough.md`: broader end-to-end walkthrough covering a team namespace, additional private ingress, Azure Disk validation, Blob CSI fixes, static Blob validation, and namespace-scoped EgressIP
- `docs/aro-internal-ingress-validation.md`: focused validation of Azure internal load balancer → OpenShift router → Route → ClusterIP service → pod using a dedicated IngressController
- `docs/aro-blob-csi-dynamic-byo-runbook.md`: validated runbook for Azure Blob CSI dynamic provisioning on ARO using a bring-your-own storage account, secret-based `StorageClass`, `protocol: fuse2`, and explicit BlobFuse2 working directory
- `docs/findings.md`: concise summary of the original EgressIP failover lab findings and caveats
- `manifests/egressip-test.yaml`: test EgressIP manifest
- `notes/commands.md`: useful command snippets