# Kubernetes Storage

## Overview

Persistent Kubernetes storage is provided by TrueNAS over NFS using the Kubernetes NFS CSI driver.

## StorageClass

| Item | Value |
|---|---|
| StorageClass | `shared-storage` |
| Provisioner | `nfs.csi.k8s.io` |
| Access Mode | `ReadWriteMany` |
| Reclaim Policy | `Retain` |
| Volume Binding | `Immediate` |
| Volume Expansion | Enabled |
| NFS Version | 4.1 |

## Backend

- NFS Server: `10.10.10.43`
- Export: `/mnt/primary-nas/kubernetes`
- Allowed Network: `10.10.10.0/24`

TrueNAS NFS Maproot is configured as:

- User: `root`
- Group: `wheel`

This is required so the CSI provisioner can dynamically create PVC directories.

## Architecture

```text
PVC
↓
StorageClass: shared-storage
↓
NFS CSI
↓
TrueNAS NFS
↓
/mnt/primary-nas/kubernetes
```

## Validation

Dynamic provisioning was tested successfully.

A temporary 1 GiB RWX PVC was created, dynamically bound to a PV, mounted into a pod, and successfully used for file read/write operations.

## Notes

`reclaimPolicy: Retain` is intentionally used so deleting a PVC does not automatically delete the underlying persistent data.