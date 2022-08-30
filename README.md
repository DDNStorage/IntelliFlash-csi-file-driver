# IntelliFlash CSI File Driver

## Overview
The Intelliflash Container Storage Interface (CSI) Driver provides a CSI interface used by Container Orchestrators (CO) to manage the lifecycle of Intelliflash volumes over NFS and SMB protocols.

## Supported kubernetes versions matrix

## Feature List
|Feature|Feature Status|CSI Driver Version|CSI Spec Version|Kubernetes Version|Intelliflash Version|
|--- |--- |--- |--- |--- |--- |
|Static Provisioning|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|Dynamic Provisioning|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|RW mode|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|RO mode|GA|>= v1.0.0|>= v1.0.0|>=1.13|>=3.11.2|
|Creating and deleting snapshot|GA|>= v1.2.0|>= v1.0.0|>=1.17|>=3.11.2|
|Provision volume from snapshot|GA|>= v1.2.0|>= v1.0.0|>=1.17|>=3.11.2|
|Provision volume from another volume|GA|>= v1.3.0|>= v1.0.0|>=1.17|>=3.11.2|
|List snapshots of a volume|Beta|>= v1.2.0|>= v1.0.0|>=1.17|>=3.11.2|
|Expand volume|GA|>= v1.3.0|>= v1.1.0|>=1.16|>=3.11.2|
|Access list for volume (NFS only)|GA|>= v1.3.0|>= v1.0.0|>=1.13|>=3.11.2|
|Topology|Beta|>= v1.4.0|>= v1.0.0|>=1.17|>=3.11.2|
|Raw block device|In development|future|>= v1.0.0|>=1.14|>=3.11.2|
|StorageClass Secrets|Beta|>= v1.3.0|>=1.0.0|>=1.13|>=3.11.2|
|Mount options|GA|>=v1.0.0|>=v1.0.0|>=v1.13|>=3.11.2|

## Requirements

- Kubernetes cluster must allow privileged pods, this flag must be set for the API server and the kubelet
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enable-privileged-pods)):
  ```
  --allow-privileged=true
  ```
- Required the API server and the kubelet feature gates
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-features)):
  ```
  --feature-gates=VolumeSnapshotDataSource=true,VolumePVCDataSource=true,ExpandInUsePersistentVolumes=true,ExpandCSIVolumes=true,ExpandPersistentVolumes=true,Topology=true,CSINodeInfo=true
  ```
  If you are planning on using topology, the following feature-gates are required
  ```
  ServiceTopology=true,CSINodeInfo=true
  ```
- Mount propagation must be enabled, the Docker daemon for the cluster must allow shared mounts
  ([instructions](https://github.com/kubernetes-csi/docs/blob/735f1ef4adfcb157afce47c64d750b71012c8151/book/src/Setup.md#enabling-mount-propagation))

  ## Installation

1. Create Intelliflash project for the driver, example: `csi-file`.
   By default, the driver will create filesystems in this dataset and mount them to use as Kubernetes volumes.
2. Clone driver repository
   ```bash
   git clone https://github.com/DDNStorage/intelliflash-csi-file-driver.git
   cd IntelliFlash-csi-file-driver
   ```
3. Load docker image
  ```bash
  docker load -i bin/intelliflash-csi-file-driver.tar
  ```
4. Edit `deploy/kubernetes/intelliflash-csi-file-driver-config.yaml` file. Driver configuration example:
   ```yaml
   arrays:
	array:
    restIp: https://172.27.10.30:443   # [required] IntelliFlash REST API endpoint
    username: admin                    # [required] IntelliFlash REST API username
    password: t                        # [required] IntelliFlash REST API password
    defaultProject: csi-block          # default project name for driver's volume
    defaultDataIp: 172.27.10.30        # default IntelliFlash data IP
    defaultMountFsType: nfs            # default mount fs type [nfs|cifs]
    defaultMountOptions: vers=4        # default mount options (mount -o ...)
     # for CIFS mounts:
     #defaultMountFsType: cifs                               # default mount fs type [nfs|cifs]
     #defaultMountOptions: username=admin,password=secrete   # username/password must be defined for CIFS
   debug: true
   ```
5. Create Kubernetes secret from the file:
   ```bash
   kubectl create secret generic intelliflash-csi-file-driver-config --from-file=deploy/kubernetes/intelliflash-csi-file-driver-config.yaml
   ```
6. Register driver to Kubernetes:
   ```bash
   kubectl apply -f deploy/kubernetes/intelliflash-csi-file-driver.yaml
   ```

## Usage

### Dynamically provisioned volumes

For dynamic volume provisioning, the administrator needs to set up a _StorageClass_ pointing to the driver.
In this case Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8`).
Default driver configuration may be overwritten in `parameters` section:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: intelliflash-csi-file-driver-sc-nginx-dynamic
provisioner: intelliflash-csi-file-driver.intelliflash.com
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
#- matchLabelExpressions:            # use to following lines to configure topology by zones
#  - key: topology.kubernetes.io/zone
#    values:
#    - us-east
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f examples/kubernetes/nginx-dynamic-volume.yaml

# to delete this pod:
kubectl delete -f examples/kubernetes/nginx-dynamic-volume.yaml
```

### Pre-provisioned volumes

The driver can use already existing Intelliflash filesystem,
in this case, _StorageClass_, _PersistentVolume_ and _PersistentVolumeClaim_ should be configured.

#### _StorageClass_ configuration

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: intelliflash-csi-driver-cs-nginx-persistent
provisioner: intelliflash-csi-driver.intelliflash.com
mountOptions:                        # list of options for `mount -o ...` command
#  - noatime                         #
```

#### _PersistentVolume_ configuration

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: intelliflash-csi-driver-pv-nginx-persistent
  labels:
    name: intelliflash-csi-driver-pv-nginx-persistent
spec:
  storageClassName: intelliflash-csi-driver-cs-nginx-persistent
  accessModes:
    - ReadWriteMany
  capacity:
    storage: 1Gi
  csi:
    driver: intelliflash-csi-driver.intelliflash.com
    volumeHandle: array1:csi-file-test1:nginx-persistent
  #mountOptions:  # list of options for `mount` command
  #  - noatime    #
```

CSI Parameters:

| Name           | Description                                                       | Example                              |
|----------------|-------------------------------------------------------------------|--------------------------------------|
| `driver`       | installed driver name "intelliflash-csi-file-driver.intelliflash.com"        | `intelliflash-csi-file-driver.intelliflash.com` |
| `volumeHandle` | NS appliance name from config and path to existing Intelliflash filesystem [configName:project:filesystem] | `array1:csi-file-test1:nginx-persistent`               |

#### _PersistentVolumeClaim_ (pointed to created _PersistentVolume_)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: intelliflash-csi-driver-pvc-nginx-persistent
spec:
  storageClassName: intelliflash-csi-driver-cs-nginx-persistent
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      # to create 1-1 relationship for pod - persistent volume use unique labels
      name: intelliflash-csi-file-driver-pv-nginx-persistent
```

#### Example

Run nginx server using PersistentVolume.

**Note:** Pre-configured filesystem should exist on the Intelliflash:
`csi-file-test1:nginx-persistent`.

```bash
kubectl apply -f examples/kubernetes/nginx-persistent-volume.yaml

# to delete this pod:
kubectl delete -f examples/kubernetes/nginx-persistent-volume.yaml
```

### Cloned volumes

We can create a clone of an existing csi volume.
To do so, we need to create a _PersistentVolumeClaim_ with _dataSource_ spec pointing to an existing PVC that we want to clone.
In this case Kubernetes generates volume name automatically (for example `pvc-ns-cfc67950-fe3c-11e8-a3ca-005056b857f8`).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: intelliflash-csi-file-driver-pvc-nginx-dynamic-clone
spec:
  storageClassName: intelliflash-csi-file-driver-cs-nginx-dynamic
  dataSource:
    kind: PersistentVolumeClaim
    apiGroup: ""
    name: intelliflash-csi-file-driver-pvc-nginx-dynamic # pvc name
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

#### Example

Run Nginx pod with dynamically provisioned volume:

```bash
kubectl apply -f examples/kubernetes/nginx-clone-volume.yaml

# to delete this pod:
kubectl delete -f examples/kubernetes/nginx-clone-volume.yaml
```

## Snapshots

**Note**: this feature is an
[alpha feature](https://kubernetes-csi.github.io/docs/snapshot-restore-feature.html#status).

```bash
# create snapshot class
kubectl apply -f examples/kubernetes/snapshot-class.yaml

# take a snapshot
kubectl apply -f examples/kubernetes/take-snapshot.yaml

# deploy nginx pod with volume restored from a snapshot
kubectl apply -f examples/kubernetes/nginx-snapshot-volume.yaml

# snapshot classes
kubectl get volumesnapshotclasses.snapshot.storage.k8s.io

# snapshot list
kubectl get volumesnapshots.snapshot.storage.k8s.io

# snapshot content list
kubectl get volumesnapshotcontents.snapshot.storage.k8s.io
```


## No Rest API tier

No Rest API tier can be useful if you do not need advanced functionality such as snapshots and clones and volume creation/deletion time is a priority. This tier does not use any API calls for volume management and operates with the NFS mount on the filesystem level.
In order to use this tier, you need to manually create a filesystem on Intelliflash and set appropriate ACLs for it to be mountable on all nodes of k8s cluster where the driver is running. All the volumes will be represented as separate folders in this filesystem.
It is required to pass `lowTierVolume: true` as a parameter in the storageClass. Also, `parentShareMountPoint` that represents the mount point on Intelliflash side needs to be passed in config section.

Examples:
config file
```bash
arrays:
  array:
    restIp: https://10.204.86.70:443
    username: admin
    password: t
    defaultProject: csi-file
    defaultDataIp: 10.204.86.71
    defaultMountFsType: nfs
    defaultMountOptions: nolock,vers=3 # only vers=4 works in container
    parentShareMountPoint: "export/csi-file/parentfs"      # used for lowTierVolume

debug: true

```

storage class
```bash

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: intelliflash-csi-file-driver-cs-nginx-dynamic
provisioner: intelliflash-csi-file-driver.intelliflash.com
parameters:
  lowTierVolume: "true"  # a tier that does not use any Rest API calls to create/delete a volume. Does not have snapshot and clone capabilities.
```

## Uninstall

Using the same files as for installation:

```bash
# delete driver
kubectl delete -f deploy/kubernetes/intelliflash-csi-file-driver.yaml

# delete secret
kubectl delete secret intelliflash-csi-file-driver-config
```

## Troubleshooting

- Show installed drivers:
  ```bash
  kubectl get csidrivers
  kubectl describe csidrivers
  ```
- Error:
  ```
  MountVolume.MountDevice failed for volume "pvc-ns-<...>" :
  driver name intelliflash-csi-file-driver.intelliflash.com not found in the list of registered CSI drivers
  ```
  Make sure _kubelet_ configured with `--root-dir=/var/lib/kubelet`, otherwise update paths in the driver yaml file
  ([all requirements](https://github.com/kubernetes-csi/docs/blob/387dce893e59c1fcf3f4192cbea254440b6f0f07/book/src/Setup.md#enabling-features)).
- "VolumeSnapshotDataSource" feature gate is disabled:
  ```bash
  vim /var/lib/kubelet/config.yaml
  # ```
  # featureGates:
  #   VolumeSnapshotDataSource: true
  # ```
  vim /etc/kubernetes/manifests/kube-apiserver.yaml
  # ```
  #     - --feature-gates=VolumeSnapshotDataSource=true
  # ```
  ```
- Driver logs
  ```bash
  kubectl logs -f intelliflash-csi-controller-0 driver
  kubectl logs -f $(kubectl get pods | awk '/intelliflash-csi-node/ {print $1;exit}') driver
  ```
- Show termination message in case driver failed to run:
  ```bash
  kubectl get pod intelliflash-csi-controller-0 -o go-template="{{range .status.containerStatuses}}{{.lastState.terminated.message}}{{end}}"
  ```
- Configure Docker to trust insecure registries:
  ```bash
  # add `{"insecure-registries":["10.3.199.92:5000"]}` to:
  vim /etc/docker/daemon.json
  service docker restart
  ```

## Development

Commits should follow [Conventional Commits Spec](https://conventionalcommits.org).
Commit messages which include `feat:` and `fix:` prefixes will be included in CHANGELOG automatically.

### Build

```bash
# print variables and help
make

# build go app on local machine
make build

# build container (+ using build container)
make container-build

# update deps
~/go/bin/dep ensure
```

### Run

Without installation to k8s cluster only version command works:

```bash
./bin/nexentastor-csi-driver --version
```

### Publish

```bash
# push the latest built container to the local registry (see `Makefile`)
make container-push-local

# push the latest built container to hub.docker.com
make container-push-remote
```

  

