# Lab Lan Config

Homelab Gitops Config Repo.

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

Will use local storage via LVM.

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

Install OpenShift Virtualization operator and create `HyperConvered` object
using all defaults.

Also create the project where all VMs and configs will live and
create the right RBAC.

```shell
oc new-project vms
oc apply -f ./rbac/vms.yaml
```

### Deployment

Everything will be installed via GitOps.

1. Install RH GitOps (ArgoCD)
2. Get password with

    ```shell
    # username is admin
    oc extract secret/openshift-gitops-cluster -n openshift-gitops --to=-
    ```

### Network

Install the NMState operator and NMState operand.

```shell
# create localnet
oc apply -k ./network
```

### Cluster Services

1. Create rbac via `oc apply -f ./rbac/httpd-server.yaml`
2. Apply github.com/jkeam/lab-lan-gitops/http-app.yaml. That uses
this `httpd-server` dir to create `httpd-server.cluster-services.svc.cluster.local`.

## VMs

### Windows

1. Download Windows ISO
2. Upload ISO

    ```shell
    ISO_FILE=$HOME/Downloads/Win10_22H2_English_x64v1.iso  # or wherever it is
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp ./httpd-server/index.html $POD_NAME:/opt/app-root/src -n cluster-services
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
    ```

3. Apply [https://github.com/jkeam/lab-lan-gitops/windows10.yaml](https://github.com/jkeam/lab-lan-gitops/blob/main/windows10.yaml). That uses this `vms/windows` dir.

### Fedora Desktop

This is using a live disk.

1. Download Fedora ISO. I am using the LXDE Spin.
2. Upload ISO

    ```shell
    ISO_FILE=$HOME/Downloads/Fedora-LXDE-Live-x86_64-42-1.1.iso  # or wherever
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp ./httpd-server/index.html $POD_NAME:/opt/app-root/src -n cluster-services
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src -n cluster-services
    ```

3. Log in and install the OS
4. Shutdown OS
5. Change boot order so that disk is before cd-rom
6. Start up machine again

To connect:

```shell
# ensure remote-viewer is installed
# ensure virtctl is installed and in path
virtctl vnc fedora-lxde
```

### Fedora Server

Use Fedora Server source already available in OpenShift.
Installs xfce desktop environment on top.

```shell
# uses `vms/fedora` dir
oc apply -f github.com/jkeam/lab-lan-gitops/fedora.yaml.
```

## Apps

### Hello Go

```shell
oc apply -f ./rbac/apps.yaml
# oc apply -f github.com/jkeam/lab-lan-gitops/hello-go.yaml
```

## References

1. [Cloud init secret](https://access.redhat.com/solutions/7090471)

## Demo

To create a demo environment to do things, spin it up:

```shell
oc apply -k ./demo
```
