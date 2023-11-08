# UFS CSI Driver 

## Overview

The UFS CSI plugin implements the K8S CSI interface, supporting dynamic volume creation, mounting into workloads and extend volume.

The UFS CSI plugin has been tested in a Kubernetes v1.20.0 environment.

This document provides more detailed information on configuring and deploying the UFS driver, as well as detailed instructions on how to use the block storage driver.

You can find other csi plugins in [JD-CSP CSI](../README.md)


## Prerequisites

Before getting started, make sure you have done the following:

- Deployed Kubernetes v1.20.0 and it is running properly.
- Obtained the latest version of the JD-CSP CSI container image.



## Deploying the UFS CSI Driver

In this section, you will learn how to deploy the driver and some necessary sidecar containers.

### 1. Prepare the Cluster

You need to prepare a compatible version of the cluster.

| **Cluster**         | **Version** |
|:--------------------|-------------|
| Kubernetes          | 1.20.0+     |
| JD-CSP File Storage | 1.4+        |

### 2. Configure the UFS Storage

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

### 4. Deploy UFS CSI

You need to obtain the image, deploy and verify csi plugin pod.

#### 4.1 Obtain the Image

The CSI images download urls:
- [csi-node-driver-registrar.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-node-driver-registrar.tar.gz)
- [csi-plugin.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-plugin.tar.gz)
- [csi-provisioner.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-provisioner.tar.gz)
- [csi-resizer.tar.gz](https://jdcsp-csi.s3.cn-north-1.jdcloud-oss.com/csi-resizer.tar.gz)

You can download them to the [../images/](../images) directory.

The UFS CSI 1.0.0 version image contains:

- harbor.jdts.com/k8s-csi/csi-provisioner:v3.5.0
- harbor.jdts.com/k8s-csi/csi-resizer:v1.7.0
- harbor.jdts.com/k8s-csi/csi-plugin:amd64-v1.0-74a71e8
- harbor.jdts.com/k8s-csi/csi-node-driver-registrar:v2.7.0

**you should load images manually**
```sh
docker load -i ../images/csi-node-driver-registrar.tar.gz
docker load -i ../images/csi-plugin.tar.gz
docker load -i ../images/csi-provisioner.tar.gz
docker load -i ../images/csi-resizer.tar.gz
```

NOTE: 
1. You can upload images to a private hub through `docker tag`, `docker push` commands.
2. Some images may already exist, if you have used other JD-CSP csi plugin.


#### 4.2 Deploy csi pod
The yaml located at [./deploy/](./deploy)

_Change SERVER_ADDR to JD-CSP Management Server Address._
```sh
sed -i 's/JDCSP_MANAGEMENT_SERVER_ADDR/ip:port/g' ./deploy/csi-plugin.yaml
sed -i 's/JDCSP_MANAGEMENT_SERVER_ADDR/ip:port/g' ./deploy/csi-provisioner.yaml
```

```sh
kubectl apply -f ./deploy/rabc.yaml
kubectl apply -f ./deploy/csi-driver.yaml
kubectl apply -f ./deploy/csi-plugin.yaml
kubectl apply -f ./deploy/csi-provisioner.yaml
```

#### 4.3 Verify

```sh
kubectl get pods | grep jdcsp-file

jdcsp-file-csi-node-7th9r                     2/2     Running       0          3s
jdcsp-file-csi-node-drmjl                     2/2     Running       0          3s
jdcsp-file-csi-provisioner-68f4bdc4c8-4x5xh   3/3     Running       0          8s
jdcsp-file-csi-provisioner-68f4bdc4c8-54rlw   3/3     Running       0          8s
```

The Kubernetes Cluster has two nodes.

## IV. Usage of UFS CSI

After csi plugin deployment, you can create volume and extend volume.

The demo yaml located at [./demo/](./demo)

### 1. Create Volume

#### 1.1 Create StorageClass

```sh
$ cat ./demo/storage-class.yaml 
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: jdcsp-ufs-sc
provisioner: jdcsp-file.csi.jdcloud.com
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```sh
$ kubectl apply -f ./demo/storage-class.yaml
storageclass.storage.k8s.io/jdcsp-ufs-sc created
```

#### 1.2 Create PVCs

```sh
$ cat ./demo/pvcs.yaml 
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ufs-pvc-1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
  storageClassName: jdcsp-ufs-sc
```

The pvc demo will create one volume: named "ufs-pvc-1".

NOTE:
1. ufs volume accessModes support ReadWriteMany or ReadWriteOnce.

```sh
$ kubectl apply -f ./demo/pvcs.yaml
persistentvolumeclaim/ufs-pvc-1 created
```

#### 1.3 View created PVCs.
```sh
$ kubectl get pvc
NAME        STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
ufs-pvc-1   Bound    ufs-3                100Gi       RWM            jdcsp-ufs-sc   8s
```

###  2. Volumes to POD

####  2.1 Create Pod

JD-CSP CSI demo application Dockerfile is located at [../application/](../application), you can also use your application image.

```sh
$ cat ./demo/deployment.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ufs-demo-app
spec:
  selector:
    matchLabels:
      name: ufs-demo-app
  replicas: 2
  template:
    metadata:
      labels:
        name: ufs-demo-app
    spec:
      containers:
        - name: demo
          image: harbor.jdts.com/centos/jdcsp-csi-demo-app:v1.0
          volumeMounts:
            - name: volume-1
              mountPath: /data1
      volumes:
        - name: volume-1
          persistentVolumeClaim:
            claimName: ufs-pvc-1

```

```sh
$ kubectl apply -f ./demo/deployment.yaml
```
####  2.2 Read/Write PVCs

```sh
View the pod named ufs-demo-app:
$ kubectl get pod | grep ufs-demo-app
ufs-demo-app-5f49b88448-m2rjr                  1/1     Running             0          65m
ufs-demo-app-5f49b88448-sbctx                  1/1     Running             0          67m

$ # direct write ufs-pvc-1
$ kubectl exec ufs-demo-app-5f49b88448-m2rjr -- sh -c "dd if=/dev/random of=/data1/test_file bs=1M count=4 oflag=direct;md5sum /data1/test_file"
0+4 records in
0+4 records out
350 bytes (350 B) copied, 0.01687 s, 20.7 kB/s
8c19bd0b7c4d64b094003258fbd2f523  /data1/test_file

View the content written by dd in another pod:
$ kubectl exec ufs-demo-app-5f49b88448-sbctx -- md5sum /data1/test_file
8c19bd0b7c4d64b094003258fbd2f523  /data1/test_file

```
###  3. Extend Attached Volume

####  3.1 Edit PVC Capacity
```sh
kubectl edit pvc ufs-pvc-1

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: PersistentVolumeClaim
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 150Gi # change to 150Gi
  storageClassName: jdcsp-ufs-sc
  volumeName: ufs-3
status:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 100Gi
  phase: Bound
```

####  3.2 Verify Extend Volume in POD
```sh
kubectl get pvc | grep ufs-pvc-1
ufs-pvc-1   Bound    ufs-3   150Gi      RWM            jdcsp-ufs-sc   15m
```

