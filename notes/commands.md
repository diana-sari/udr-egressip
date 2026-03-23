# Commands used during testing

## Cluster access

```bash
sshuttle --dns -r aro@20.245.86.105 10.0.0.0/20
oc login https://api.dianasari-udr.westus.aroapp.io:6443 -u kubeadmin -p '<password>'