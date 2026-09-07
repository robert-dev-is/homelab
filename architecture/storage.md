# Storage Architecture

Centralized network storage is provided by a TrueNAS SCALE VM on `nas-node-01`.

The design keeps persistent data separate from individual application and compute workloads while providing shared storage to Linux, Windows, Proxmox, Kubernetes, media, documentation, and backup systems.

## TrueNAS Platform

TrueNAS runs as VM 103 with:

- 24GB fixed RAM
- LSI 9300-8i HBA passed directly through to the VM
- 10GbE SFP+ connectivity
- SMB and NFS services

The HBA and network adapter are attached through external PCIe expansion using M.2 to OCuLink from the host.

Detailed hardware design, cooling, performance testing, and lessons learned are documented in the TrueNAS network-storage project.

## Storage Pools

### `primary-nas`

- 2x 14TB Western Digital Ultrastar HDDs
- ZFS mirror
- Bulk storage for media, backups, documentation, software, and general lab data

### `game-pool`

- 2TB Samsung 860 QVO SATA SSD
- Single-disk ZFS vdev
- No redundancy
- SSD-backed network game storage

The TrueNAS system contains additional datasets and shares beyond these two high-level storage roles.

## Protocols and Consumers

TrueNAS provides both SMB and NFS.

SMB is used where Windows-compatible file access is needed, while NFS is used heavily by Linux, Proxmox, and Kubernetes workloads.

Major consumers include:

- Proxmox hosts
- Linux and Windows clients
- containers and VMs
- media services
- documentation infrastructure
- Kubernetes
- backup workflows

## Kubernetes Storage

The Kubernetes cluster uses TrueNAS over NFS through the Kubernetes NFS CSI driver.

The current `shared-storage` StorageClass supports dynamic persistent-volume provisioning and uses `ReadWriteMany` with a `Retain` reclaim policy.

Persistent Kubernetes data therefore remains on TrueNAS rather than depending on the lifecycle of an individual Kubernetes node.

## Backup Relationship

A dedicated Proxmox Backup Server provides VM and container backups for the Proxmox environment.

Current PBS retention:

- 4 daily
- 2 weekly
- 2 monthly

`primary-nas` can also be used as a less-frequent secondary Proxmox backup target.

TrueNAS uses periodic recursive ZFS snapshots for dataset protection.

The TrueNAS VM is excluded from normal PBS jobs because backup operations previously caused reliability problems with the VM.

## Design Principles

- Keep persistent data separate from disposable compute
- Use ZFS for storage integrity and snapshots
- Provide both SMB and NFS for mixed-platform workloads
- Use 10GbE to avoid unnecessary network bottlenecks
- Keep VM/container backups separate from NAS dataset protection
