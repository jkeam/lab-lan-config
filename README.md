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

Make sure drive is clear. Using `nvme0n1` as an example:

```shell
oc debug node/$NODE_NAME
chroot /host
wipefs -af /dev/nvme0n1
sgdisk --zap-all /dev/nvme0n1
dd if=/dev/zero of=/dev/nvme0n1 bs=1M count=100 oflag=direct,dsync
# might have to reboot to run the following:
blkdiscard /dev/nvme0n1
```

### LVM

This uses the Logical Volume Manager.

1. Install LVM Storage Operator (Don't create LVMCluster yet, we do that next)

2. Create LVM Cluster

    ```shell
    oc apply -f ./initial/lvmcluster.yaml
    ```

3. Add annotation

    ```shell
    oc patch storageclass lvms-vg1 --type='merge' \
        -p '{"metadata":{"annotations":{"storageclass.kubevirt.io/is-default-virt-class":"true"}}}'
    ```

4. Some Helpful Commands

    ```shell
    lsblk                   # list all devices
    findmnt /sysroot        # find which device is the boot device
    ls -l /dev/disk/by-id/  # find the id which is a better way to id the device
    ```

### LSO

This uses the Local Storage Operator.

1. Setup namespace

    ```shell
    oc adm new-project openshift-local-storage
    # allow to run on infra nodes
    oc annotate namespace openshift-local-storage openshift.io/node-selector=''
    # allow to run on a SNO
    oc annotate namespace openshift-local-storage workload.openshift.io/allowed='management'
    ```

2. Install Local Storage Operator, specifically into `openshift-local-storage` namespace

3. Create `LocalVolume`

    ```shell
    oc apply -f ./initial/localvolume.yaml
    ```

4. Add annotation

    ```shell
    oc patch storageclass local-sc --type='merge' \
        -p '{"metadata": {"annotations": {"storageclass.kubernetes.io/is-default-class": "true"}}}'
    oc patch storageclass local-sc --type='merge' \
        -p '{"metadata":{"annotations":{"storageclass.kubevirt.io/is-default-virt-class":"true"}}}'
    ```

## Integrated Registry

Set up integrated registry by doing the following.

```shell
# test, should be nothing
oc get pod -n openshift-image-registry -l docker-registry=default

# create registry
oc apply -f ./initial/integrated-registry-storage.yaml

# patch object
oc patch configs.imageregistry.operator.openshift.io cluster \
    --type=merge \
    --patch '{"spec":{"managementState":"Managed"}}'
oc patch config.imageregistry.operator.openshift.io/cluster \
    --type=merge \
    -p '{"spec":{"rolloutStrategy":"Recreate","replicas":1}}'
oc patch configs.imageregistry.operator.openshift.io cluster \
    --type=merge \
    --patch '{"spec":{"storage":{"pvc":{"claim": "image-registry-storage"}}}}'
oc patch configs.imageregistry.operator.openshift.io/cluster \
    --type=merge \
    --patch '{"spec":{"defaultRoute":true}}'

# test again, should now be something
oc get pod -n openshift-image-registry -l docker-registry=default

# get the route
oc get route default-route -n openshift-image-registry \
    --template='{{ .spec.host }}'
```

## Dev Spaces

Install Dev Spaces from the Operator Hub.

```shell
# create the che cluster, pay attention to my workspace configs
oc apply -f ./initial/che-cluster.yaml
```

## Virtualization

Install OpenShift Virtualization operator and create `HyperConverged` object
using all defaults.

Optionally, configure `dataImportCronTemplates` using `./initial/hyperconverged-snippet.yaml`
and putting that in `HyperConverged.spec.dataImportCronTemplates`.

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

Note: Leaving `./network/nad.yaml` for reference only, but the `./network/cudn.yaml`
will create this for us.

### Cluster Services

```shell
oc new-project cluster-services
oc apply -f ./rbac/httpd-server.yaml
oc create -f https://raw.githubusercontent.com/jkeam/lab-lan-gitops/refs/heads/main/httpd-app.yaml
  # points to `httpd-server` dir to create `httpd-server.cluster-services.svc.cluster.local`
```

## VMs

Create `images` directory.

```shell
oc exec $(oc get pods -l app=httpd-server -n cluster-services -o name) -n cluster-services -- mkdir ./images
```

### Windows

1. Download Windows ISO

2. Upload ISO

    ```shell
    ISO_FILE=$HOME/Downloads/Win10_22H2_English_x64v1.iso  # or wherever it is
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp ./httpd-server/index.html $POD_NAME:/opt/app-root/src -n cluster-services
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src/images -n cluster-services
    ```

3. Apply [https://github.com/jkeam/lab-lan-gitops/windows10.yaml](https://github.com/jkeam/lab-lan-gitops/blob/main/windows10.yaml). That uses this `vms/windows` dir.  Make sure that the `bootOrder` is set to boot from `installation-cdrom` so that Windows can install.

4. After installation, edit the `bootOrder` to switch the `rootdisk` and `installation-cdrom`

### Fedora Desktop

This is using a live disk.

1. Download Fedora ISO. I am using the LXDE Spin.
2. Upload ISO

    ```shell
    ISO_FILE=$HOME/Downloads/Fedora-LXDE-Live-x86_64-42-1.1.iso  # or wherever
    POD_NAME=$(oc get pods --selector=app=httpd-server -o jsonpath='{.items[0].metadata.name}' -n cluster-services)
    oc cp ./httpd-server/index.html $POD_NAME:/opt/app-root/src -n cluster-services
    oc cp ./httpd-server/.htaccess $POD_NAME:/opt/app-root/src -n cluster-services
    oc cp $ISO_FILE $POD_NAME:/opt/app-root/src/images -n cluster-services
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

## Setup Helm Charts

```shell
oc apply -k ./helm-repo
oc apply -f ./rbac/apps.yaml
```

### Hello Go

```shell
oc apply -f https://raw.githubusercontent.com/jkeam/lab-lan-gitops/refs/heads/main/hello-go.yaml
```

## OpenClaw

```shell
oc apply -k openclaw

# for openshell, go here: https://gist.github.com/jkeam/cfb7899f0f360d7efe502cebc838a2fb
```

Then wait for CSV status to become `Succeeded` and for pods to come up.

```shell
oc get csv -n claw-operator
oc get pods -n claw-operator
```

## Git

```shell
oc new-project forgejo
helm install forgejo oci://code.forgejo.org/forgejo-helm/forgejo \
  --set image.registry=codeberg.org \
  --set image.repository=forgejo/forgejo \
  --set image.tag=16 \
  --set image.rootless=true \
  --set clusterDomain=lab.keam.org \
  --set route.enabled=true \
  --set serviceAccount.create=true \
  --set gitea.admin.username=forgejo_admin \
  --set gitea.admin.password=qGmgz9U8PpTDm9Cyo5nBbvA5b70 \
  --set gitea.admin.email=jpkeam@gmail.com \
  --set podSecurityContext.fsGroup=1000990000 \
  --set global.compatibility.openshift.adaptSecurityContext=force
```

## S3

```shell
oc new-project s3
helm upgrade --install rustfs rustfs --repo https://charts.rustfs.com --version 0.0.85 \
  --set 'podSecurityContext.fsGroup=null' \
  --set 'podSecurityContext.runAsUser=null' \
  --set 'podSecurityContext.runAsGroup=null' \
  --set 'containerSecurityContext.readOnlyRootFilesystem=null' \
  --set 'containerSecurityContext.allowPrivilegeEscalation=null' \
  --set 'containerSecurityContext.runAsNonRoot=null' \
  --set image.rustfs.tag=1.0.0-alpha.85 \
  --set ingress.enabled=false \
  --set mode.standalone.enabled=true \
  --set mode.distributed.enabled=false \
  --set storageclass.name=lvms-vg1
```

## References

1. [Cloud init secret](https://access.redhat.com/solutions/7090471)

## Demo

To create a demo environment to do things, spin it up:

```shell
oc apply -k ./demo
```
