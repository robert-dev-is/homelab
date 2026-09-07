# TrueNAS and Network Storage Architecture

## Project Overview

This project documents the design and evolution of a compact homelab NAS built around TrueNAS SCALE, Proxmox VE, ZFS, direct HBA passthrough, external PCIe expansion, and 10GbE networking.

The project began as a way to centralize bulk storage and backups, then expanded into a broader network-storage platform supporting media, virtualization, Linux clients, Windows clients, and SSD-backed game storage.

Rather than using a traditional full-size server chassis, the design uses a small HP EliteDesk 705 G5 Mini as the compute host and a separate 2U enclosure for storage devices, PCIe cards, power, and cooling.

## Goals

- Build reliable centralized storage for the homelab.
- Keep the compute node compact and power efficient.
- Give TrueNAS direct access to physical storage through HBA passthrough.
- Support both SMB and NFS clients.
- Remove 2.5GbE as a network bottleneck by moving to 10GbE SFP+.
- Add SSD-backed network storage for latency-sensitive workloads.
- Preserve a modular architecture that can be expanded or replaced one component at a time.

## Current Architecture

```text
                    +---------------------------+
                    |   HP EliteDesk 705 G5     |
                    |       nas-node-01         |
                    |                           |
                    | Proxmox VE                |
                    | TrueNAS SCALE VM          |
                    +-------------+-------------+
                                  |
                  M.2 / OCuLink PCIe expansion
                                  |
             +--------------------+--------------------+
             |                                         |
             v                                         v
   +-------------------+                     +-------------------+
   | LSI 9300-8i HBA   |                     | 10GbE SFP+ NIC    |
   | PCIe passthrough  |                     | Proxmox vmbr0     |
   +---------+---------+                     +---------+---------+
             |                                         |
             v                                         v
   +-------------------+                         10GbE network
   | External 2U       |
   | storage enclosure |
   +---------+---------+
             |
       +-----+-----+
       |           |
       v           v
  2x 14TB HDD   2TB SATA SSD
  ZFS mirror    game-pool
```

## Storage Roles

| Storage | Layout | Primary Use |
|---|---|---|
| 2x 14TB WD Ultrastar HDDs | ZFS mirror | Bulk storage, media, backups, general shares |
| 2TB Samsung 860 QVO | Single-disk ZFS vdev | SSD-backed game storage |

## Network Storage Protocols

- **NFS** is used primarily by Linux systems and containers.
- **SMB** is used primarily by Windows systems.

This keeps client configuration simple and uses the protocol that fits each operating system most naturally.

## Project Documents

- [architecture.md](/projects/truenas-network-storage-project/architecture.md) — system layout and data paths
- [hardware-and-design.md](/projects/truenas-network-storage-project/hardware-and-design.md) — hardware choices and external PCIe design
- [storage-and-protocols.md](/projects/truenas-network-storage-project/storage-and-protocols.md) — ZFS pools, shares, SMB, and NFS
- [performance-and-testing.md](/projects/truenas-network-storage-project/performance-and-testing.md) — observed storage and network performance
- [lessons-learned.md](/projects/truenas-network-storage-project/lessons-learned.md) — design decisions, limitations, and takeaways

## Status

The platform is operational and currently serves as the homelab's primary NAS. The architecture has been expanded from 2.5GbE to 10GbE and from HDD-only storage to a mixed HDD/SSD design.
