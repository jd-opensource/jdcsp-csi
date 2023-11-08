# CBS CSI Driver 

## Overview

The CBS CSI plugin implements the K8S CSI interface, supporting dynamic volume creation and mounting into workloads.

The CBS CSI plugin has been tested in a Kubernetes v1.20.0 environment.

This document provides more detailed information on configuring and deploying the CBS driver, as well as detailed instructions on how to use the block storage driver.

You can find other csi plugins in [JD-CSP CSI](../README.md)


## Prerequisites

Before getting started, make sure you have done the following:

- Deployed Kubernetes v1.20.0 and it is running properly.
- Obtained the latest version of the JD-CSP CSI container image.



## Deploying the CBS CSI Driver

In this section, you will learn how to deploy the driver and some necessary sidecar containers.

### 1. Prepare the Cluster

You need to prepare a compatible version of the cluster.

| **Cluster**          | **Version** |
|:---------------------|-------------|
| Kubernetes           | 1.20.0+     |
| JD-CSP Block Storage | 1.4+        |

### 2. Configure the CBS Storage

You need to create the storage cluster in the JD-CSP Management System.

### 3. Configure Kubernetes 

You need to obtain the account information to access the JD-CSP Management System.

#### 3.1 CSI Secret

##### 3.1.1 Config account information

```sh
$ cat ./deploy/secret.yaml

---
apiVersion: v1
kind: Secret
metadata:
  name: jdcsp-csi-secrets
  namespace: default
data:
  USER_NAME: <USER_NAME>
  USER_PASSWORD: <USER_PASSWORD>
```

The following table provides data definitions:

| **Term**      | **Definition**                                            |
|---------------|-----------------------------------------------------------|
| USER_NAME     | Base64ed login username for the JD-CSP Management System. |
| USER_PASSWORD | Base64ed login password for the JD-CSP Management System. |

##### 3.1.2 Deploy the CSI Secret

```sh
kubectl apply -f ./deploy/secret.yaml
```

### 4. Deploy CBS CSI

You need to obtain the image, deploy and verify csi plugin pod.

#### 4.1 Obtain the Image

The CSI images download urls:
- [csi-attacher.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-attacher.tar.gz)
- [csi-node-driver-registrar.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-node-driver-registrar.tar.gz)
- [csi-plugin.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-plugin.tar.gz)
- [csi-provisioner.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-provisioner.tar.gz)
- [csi-resizer.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-resizer.tar.gz)
- [csi-snapshotter.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-snapshotter.tar.gz)
- [snapshot-controller.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/snapshot-controller.tar.gz)

You can download them to the [../images/](../images) directory.

The CBS CSI 1.0.0 version image contains:

- harbor.jdts.com/k8s-csi/csi-provisioner:v3.5.0
- harbor.jdts.com/k8s-csi/csi-attacher:v4.2.0
- harbor.jdts.com/k8s-csi/csi-resizer:v1.7.0
- harbor.jdts.com/k8s-csi/csi-snapshotter:v6.2.1
- harbor.jdts.com/k8s-csi/snapshot-controller:v6.2.1
- harbor.jdts.com/k8s-csi/csi-plugin:amd64-v1.0-74a71e8
- harbor.jdts.com/k8s-csi/csi-node-driver-registrar:v2.7.0

**you should load images manually**
```sh
docker load -i ../images/csi-attacher.tar.gz
docker load -i ../images/csi-node-driver-registrar.tar.gz
docker load -i ../images/csi-plugin.tar.gz
docker load -i ../images/csi-provisioner.tar.gz
docker load -i ../images/csi-resizer.tar.gz
docker load -i ../images/csi-snapshotter.tar.gz
docker load -i ../images/snapshot-controller.tar.gz
```

NOTE: 
1. You can upload images to a private hub through `docker tag`, `docker push` commands.
2. Some images may already exist, if you have used other JD-CSP csi plugin.


#### 4.2 Deploy snapshot crd and csi pod 
The yaml located at [./deploy/](./deploy)

_Change SERVER_ADDR to JD-CSP Management Server Address._
```sh
sed -i 's/JDCSP_MANAGEMENT_SERVER_ADDR/ip:port/g' ./deploy/csi-plugin.yaml
sed -i 's/JDCSP_MANAGEMENT_SERVER_ADDR/ip:port/g' ./deploy/csi-provisioner.yaml
```

```sh
kubectl apply -f ./deploy/snapshot-crd.yaml
kubectl apply -f ./deploy/rabc.yaml
kubectl apply -f ./deploy/csi-driver.yaml
kubectl apply -f ./deploy/csi-plugin.yaml
kubectl apply -f ./deploy/csi-provisioner.yaml
```

#### 4.3 Verify

```sh
kubectl get pods | grep jdcsp-block

jdcsp-block-csi-node-7th9r                     2/2     Running       0          3s
jdcsp-block-csi-node-drmjl                     2/2     Running       0          3s
jdcsp-block-csi-provisioner-68f4bdc4c8-4x5xh   6/6     Running       0          8s
jdcsp-block-csi-provisioner-68f4bdc4c8-54rlw   6/6     Running       0          8s
```

The Kubernetes Cluster has two nodes.

## IV. Usage of CBS CSI

After csi plugin deployment, you can create volume, create snapshot, clone volume, attach volume to POD, and extend volume.

The demo yaml located at [./demo/](./demo)

### 1. Create Volume

#### 1.1 Create StorageClass

```sh
$ cat ./demo/storage-class.yaml 

---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jdcsp-cbs-sc
provisioner: jdcsp-block.csi.jdcloud.com
parameters:
  media: hdd
  csi.storage.k8s.io/fstype: xfs
reclaimPolicy: Delete
allowVolumeExpansion: true
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: jdcsp-cbs-snapshot-sc
driver: jdcsp-block.csi.jdcloud.com
deletionPolicy: Delete
parameters:
```

The following table provides sc parameters definitions:

| **Term**                  | **Definition**                              |
|---------------------------|---------------------------------------------|
| media                     | cbs storage media. eg:hdd, ssd              |
| csi.storage.k8s.io/fstype | volume mount fstype. only support xfs, ext4 |

```sh
$ kubectl apply -f ./demo/storage-class.yaml

storageclass.storage.k8s.io/jdcsp-cbs-sc created
volumesnapshotclass.snapshot.storage.k8s.io/jdcsp-cbs-snapshot-sc created
```

#### 1.2 Create PVCs

```sh
$ cat ./demo/pvcs.yaml 

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-1
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block
  resources:
    requests:
      storage: 10Gi
  storageClassName: jdcsp-cbs-sc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-2
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 20Gi
  storageClassName: jdcsp-cbs-sc
```

The pvc demo will create two volumes: one named "cbs-pvc-1" with mode ```block```; one named "cbs-pvc-2" with mode ```Filesystem```.

NOTE: 
1. cbs volume size must be aligned to 10Gi.
2. cbs volume accessModes only support ReadWriteOnce.

```sh
$ kubectl apply -f ./demo/pvcs.yaml

persistentvolumeclaim/cbs-pvc-1 created
persistentvolumeclaim/cbs-pvc-2 created
```



#### 1.3 View created PVCs.
```sh
$ kubectl get pvc

NAME        STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cbs-pvc-1   Bound    cbs-vol-9dt6lh3hnk   10Gi       RWO            jdcsp-cbs-sc   8s
cbs-pvc-2   Bound    cbs-vol-qlomfvwejj   20Gi       RWO            jdcsp-cbs-sc   8s
```


###  2. Attach Volumes to POD

####  2.1 Create Pod

JD-CSP CSI demo application Dockerfile is located at [../application/](../application), you can also use your application image.

```sh
$ cat ./demo/deployment.yaml

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cbs-demo-app
spec:
  selector:
    matchLabels:
      name: cbs-demo-app
  template:
    metadata:
      labels:
        name: cbs-demo-app
    spec:
      containers:
        - name: demo
          image: harbor.jdts.com/centos/jdcsp-csi-demo-app:v1.0
          volumeDevices:
            - name: volume-1
              devicePath: /block1
          volumeMounts:
            - name: volume-2
              mountPath: /data1
      volumes:
        - name: volume-1
          persistentVolumeClaim:
            claimName: cbs-pvc-1
            readOnly: false
        - name: volume-2
          persistentVolumeClaim:
            claimName: cbs-pvc-2
            readOnly: false
```

```sh
$ kubectl apply -f ./demo/deployment.yaml
```
####  2.2 Read/Write PVCs

```sh
$ docker ps | grep demo

957b156fde2b        b7490c0c901b                                        "/sleep.sh" ...

$ docker exec -it 957b156fde2b bash
$ # direct write cbs-pvc-1
$ dd if=/dev/random of=/block1 bs=1M count=4 oflag=direct

$ # direct write cbs-pvc-2
$ dd if=/dev/random of=/data1/test_file bs=1M count=4 oflag=direct

$ # direct read cbs-pvc-1 and calculate md5sum
$ dd if=/block1 bs=1M count=4 iflag=direct | md5sum
a204851d2cf482eecf3ba80f94895d0e

$ # direct read cbs-pvc-2 and calculate md5sum
$ dd if=/data1/test_file bs=1M count=4 iflag=direct | md5sum
046e76e328093d1b75311e98759e959f
```

### 3. Create Volume Snapshot

#### 3.2 Create Volume Snapshot

```sh
$ cat ./demo/snapshots.yaml 

---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cbs-pvc-1-snapshot-1
spec:
  volumeSnapshotClassName: jdcsp-cbs-snapshot-sc
  source:
    persistentVolumeClaimName: cbs-pvc-1
---
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: cbs-pvc-2-snapshot-1
spec:
  volumeSnapshotClassName: jdcsp-cbs-snapshot-sc
  source:
    persistentVolumeClaimName: cbs-pvc-2
```
    
```sh
$ kubectl apply -f ./demo/snapshots.yaml 

volumesnapshot.snapshot.storage.k8s.io/cbs-snapshot-1 created
volumesnapshot.snapshot.storage.k8s.io/cbs-snapshot-2 created
```

```sh
$ kubectl get vs

NAME             READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS           SNAPSHOTCONTENT                                    CREATIONTIME   AGE
cbs-snapshot-1   true         cbs-pvc-1                           10Gi          jdcsp-cbs-snapshot-sc   snapcontent-3aa21adb-9f7a-4bc6-8319-297a155da085   2m           2m
cbs-snapshot-2   true         cbs-pvc-2                           20Gi          jdcsp-cbs-snapshot-sc   snapcontent-7612f275-8ddc-4cd1-a1da-d22fa2674927   2m           2m
```

### 4. Clone Volumes

#### 4.1 Create Cloned Volumes

```sh
$ cat ./demo/pvcs-from-snap.yaml 

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-3
spec:
  dataSource:
    name: cbs-snapshot-1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: jdcsp-cbs-sc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-4
spec:
  dataSource:
    name: cbs-snapshot-2
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: jdcsp-cbs-sc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-5
spec:
  dataSource:
    name: cbs-snapshot-1
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: jdcsp-cbs-sc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cbs-pvc-6
spec:
  dataSource:
    name: cbs-snapshot-2
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  storageClassName: jdcsp-cbs-sc
```  

We will create "cbs-pvc-3/cbs-pvc-4" from snapshot "cbs-snapshot-1/cbs-snapshot-2" and they should be the same as "cbs-pvc-1/cbs-pvc-2" .

"cbs-pvc-5/cbs-pvc-6" are also created from snapshot "cbs-snapshot-1/cbs-snapshot-2" but the size is extended to 50Gi.

NOTE: 

1. The volume capacity must be greater than or equal to the snapshot capacity.
2. The CSI plugin will automatically expand the file system, if the snapshot capacity is greater than the Volume capacity.


```sh 
$ kubectl apply -f ./demo/pvcs-from-snap.yaml 

persistentvolumeclaim/cbs-pvc-3 created
persistentvolumeclaim/cbs-pvc-4 created
persistentvolumeclaim/cbs-pvc-5 created
persistentvolumeclaim/cbs-pvc-6 created
```  

```sh 
$ kubectl get pvc 

NAME        STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
cbs-pvc-1   Bound    cbs-vol-3n8yykoc3n   10Gi       RWO            jdcsp-cbs-sc   5m
cbs-pvc-2   Bound    cbs-vol-rxilbpzzlq   20Gi       RWO            jdcsp-cbs-sc   5m
cbs-pvc-3   Bound    cbs-vol-rewebg33ma   10Gi       RWO            jdcsp-cbs-sc   5s
cbs-pvc-4   Bound    cbs-vol-akikbeabhp   20Gi       RWO            jdcsp-cbs-sc   5s
cbs-pvc-5   Bound    cbs-vol-33qdnl1luo   50Gi       RWO            jdcsp-cbs-sc   5s
cbs-pvc-6   Bound    cbs-vol-v17i9lke4p   50Gi       RWO            jdcsp-cbs-sc   5s
```

#### 4.2 Attach Volumes to Pod
```sh 
$ kubectl apply -f ./demo/depolyment-pvc-from-snap.yaml 
```

#### 4.3 Verify PVCs data

```sh 
$ # direct read cbs-pvc-3 and calculate md5sum
$ dd if=/block2 bs=1M count=4 iflag=direct | md5sum
a204851d2cf482eecf3ba80f94895d0e

$ # direct read cbs-pvc-4 and calculate md5sum
$ dd if=/data2/test_file bs=1M count=4 iflag=direct | md5sum
046e76e328093d1b75311e98759e959f

$ # direct read cbs-pvc-5 and calculate md5sum
$ dd if=/block3 bs=1M count=4 iflag=direct | md5sum
a204851d2cf482eecf3ba80f94895d0e

$ # direct read cbs-pvc-6 and calculate md5sum
$ dd if=/data3/test_file bs=1M count=4 iflag=direct | md5sum
046e76e328093d1b75311e98759e959f
```


###  5. Extend Attached Volume

####  5.1 Edit PVC Capacity
```sh
kubectl edit pvc cbs-pvc-1

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi # change to 100Gi
  storageClassName: jdcsp-cbs-sc
  volumeMode: Block
  volumeName: cbs-vol-3n8yykoc3n
status:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  phase: Bound
```

####  5.2 Verify Extend Volume in POD
```sh
kubectl get pvc | grep cbs-pvc-1

cbs-pvc-1   Bound    cbs-vol-3n8yykoc3n   100Gi      RWO            jdcsp-cbs-sc   15m
```

