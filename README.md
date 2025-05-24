# Lab Lan Config

Homelab Gitops Config Repo

## User

Creates new admin user that is not kubeadmin. Make sure `httpd-tools` is installed to get `htpasswd` cli.

```shell
# creates file named openshift.htpasswd for username admin and password openshift
htpasswd -c -B -b ./openshift.htpasswd admin openshift

# creates secret from said file
oc create secret generic htpass-users --from-file=htpasswd=./openshift.htpasswd -n openshift-config

# add username/password auth method
oc patch oauth/cluster --type merge --patch '{"spec":{"identityProviders":[{"name": "htpasswd", "mappingMethod": "claim", "type": "HTPasswd", "htpasswd": {"fileData": {"name": "htpass-users"}}}]}}'

# make user an admin
oc adm policy add-cluster-role-to-user cluster-admin admin

# watch pods rollout
oc get pods -n openshift-authentication -w
```

## Storage

Will be using local storage via LVM.

1. Install LVM Storage Operator
2. Create LVM Cluster

    ```shell
    oc annotate namespace openshift-local-storage openshift.io/node-selector=''
    oc annotate namespace openshift-local-storage workload.openshift.io/allowed='management'
    oc debug node/master1 -- sgdisk --zap-all /dev/nvme1n1
    oc apply -f ./initial/lvmcluster.yaml
    ```

## Integrated Registry

Set up integrated registry by doing the following.

```shell
# test, should be nothing
oc get pod -n openshift-image-registry -l docker-registry=default

oc apply -f ./initial/integrated-registry-storage.yaml
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"managementState":"Managed"}}'
oc patch config.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"rolloutStrategy":"Recreate","replicas":1}}'
oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"pvc":{"claim": "image-registry-storage"}}}}'

# test again, should now be something
oc get pod -n openshift-image-registry -l docker-registry=default
```

## Virtualization

Install OpenShift Virtualization operator and create `HyperConvered` object using all defaults.

## Deployment

Everything will be installed via GitOps.

1. Install RH GitOps (ArgoCD)
2. Get password with

    ```shell
    oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
    ```

### Apps

#### Cluster Services

1. Create rbac oc apply -f ./rbac/httpd-server.yaml
2. Apply github.com/jkeam/lab-lan-gitops/http-app.yaml. That uses this `httpd-server` dir to create `httpd-server.cluster-services.svc.cluster.local`

#### Windows

1. Download Windows ISO
2. Upload ISO
    ```shell
    ISO_FILE=$HOME/win2k19.iso  # or whatever you named your iso
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp ./httpd-server/index.html $POD_NAME:/opt/app-root/src -n cluster-services
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
    ```
3. Apply github.com/jkeam/lab-lan-gitops/windows10.yaml. That uses this `vms/windows` dir to create `httpd-server.cluster-services.svc.cluster.local`

#### Other

1. vms/fedora - create fedora vm. update `sudo vi /etc/passwd` to use `/bin/zsh` as default shell

## References

1. [Cloud init secret](https://access.redhat.com/solutions/7090471)
