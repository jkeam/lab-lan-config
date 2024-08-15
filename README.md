# Lab Lan Config

Homelab Gitops Config Repo

## Deployment

Use ArgoCD.

### Post Deployment

```shell
ISO_FILE=$HOME/win2k19.iso
POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
```
