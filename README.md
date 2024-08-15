# Lab Lan Config

Homelab Gitops Config Repo

## Integrated Registry

```shell
# test, should be nothing
oc get pod -n openshift-image-registry -l docker-registry=default

oc apply -f ./integrated-registry-storage.yaml
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"rolloutStrategy":"Recreate","replicas":1}}'
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"pvc":{"claim": "image-registry-storage"}}}}'

# test again, should now be something
oc get pod -n openshift-image-registry -l docker-registry=default
```

## Deployment

Use ArgoCD.

### Post Deployment

```shell
ISO_FILE=$HOME/win2k19.iso
POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
oc cp ./httpd-server/index.html $POD_NAME:/opt/app-root/src -n cluster-services
oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
```
