# JD-CSP CSI drivers 

## Overview

The JD-CSP(JD Cloud Storage Platform) CSI plugins support Block, File, Object.


## Block CSI plugin

CBS(Cloud Block Storage) is type of block storage, can only be used as ReadWriteOnce mode.

The [CBS CSI plugin](./cbs/README.md) implements the K8S CSI interface, supporting dynamic volume creation and mounting into workloads.

## File CSI plugin
UFS(Unified File Storage) is type of file storage, can be used as ReadWriteMany/ReadWriteOnce mode.

The [UFS CSI plugin](./ufs/README.md) implements the K8S CSI interface, supporting dynamic volume creation and mounting into workloads.

## Object CSI plugin


