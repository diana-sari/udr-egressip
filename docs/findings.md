# Findings

> Note: This file summarizes the focused private ARO UDR + EgressIP failover lab. For the broader walkthrough that also covers an additional private ingress controller, Azure Disk validation, Blob CSI fixes on ARO/RHCOS, static Azure Blob validation, and namespace-scoped EgressIP, see [`docs/aro-private-ingress-storage-egressip-walkthrough.md`](./aro-private-ingress-storage-egressip-walkthrough.md)

## Environment

- Platform: ARO 4.20.15
- Region: West US
- Deployment: Terraform
- Cluster type: private ARO
- Outbound mode: UserDefinedRouting
- Downstream egress point: Azure Firewall

## Summary

### Controlled reassignment
Removing `k8s.ovn.org/egress-assignable` from the owning worker caused EgressIP reassignment to another eligible worker within seconds.

Observed behavior:
- reassignment completed quickly
- no visible curl interruption
- external public source IP remained the Azure Firewall public IP

### Drain-only behavior
A simple cordon + drain of the owning worker did not move the EgressIP.

### Actual node-loss simulation
Deleting the owning worker Machine caused:
- temporary EgressIP unassignment
- CloudPrivateIPConfig removal
- later reassignment once another eligible worker existed

Observed behavior:
- one brief failed curl attempt during recovery
- external public source IP still remained the Azure Firewall public IP

## Important caveat

EgressIP failover depended on other workers already being labeled:

`k8s.ovn.org/egress-assignable`

Replacement workers did not automatically inherit that label.