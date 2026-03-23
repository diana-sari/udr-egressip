# udr-egressip

Testing notes, manifests, and findings for private ARO with:

- UserDefinedRouting (UDR)
- Azure Firewall in the downstream egress path
- OVN Kubernetes EgressIP failover behavior

## Scope

This repo documents a lab validation of:

- private ARO deployment with Terraform
- EgressIP assignment on Azure via CloudPrivateIPConfig
- controlled EgressIP reassignment
- worker node-loss simulation
- observed outbound behavior through Azure Firewall

## Key findings

- EgressIP controlled reassignment between eligible workers completed within seconds.
- A simple cordon/drain did not by itself trigger EgressIP reassignment.
- Deleting the owning worker Machine caused temporary EgressIP unassignment.
- Reassignment required another worker already labeled `k8s.ovn.org/egress-assignable`.
- Replacement workers did not automatically inherit the `egress-assignable` label.
- The externally visible source IP remained the Azure Firewall public IP throughout testing.
- During actual node-loss recovery, one brief traffic blip was observed.

## Repo layout

- `docs/aro-private-udr-egressip-test.md`: full runbook
- `docs/findings.md`: concise findings and caveats
- `manifests/egressip-test.yaml`: test EgressIP manifest
- `notes/commands.md`: useful command snippets