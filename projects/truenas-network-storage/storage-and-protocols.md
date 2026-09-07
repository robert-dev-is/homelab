# Storage and Protocols

## ZFS Pool Design

The NAS currently uses two storage tiers with different goals.

### primary-nas

`primary-nas` is the main capacity-oriented pool.

| Item | Configuration |
|---|---|
| Drives | 2x 14TB WD Ultrastar HDDs |
| Layout | ZFS mirror |
| Role | Bulk storage, media, backups, general shares |
| Drive spindown | Disabled |

The mirror prioritizes simple redundancy and predictable recovery characteristics over maximum usable capacity.

### game-pool

`game-pool` is an SSD-backed pool intended for workloads where random access and metadata performance matter more than bulk capacity.

| Item | Configuration |
|---|---|
| Drive | Samsung 860 QVO 2TB SATA SSD |
| Layout | Single-disk ZFS vdev |
| Role | Network game library |
| Redundancy | None currently |

The data stored here is largely replaceable. If a second compatible SSD is added later, the existing single-disk vdev can be converted into a two-way mirror by attaching the second disk to the existing vdev.

## Why Separate HDD and SSD Storage?

The HDD mirror is well suited for large media files, backups, and general storage. Game installations can contain large numbers of small files and can benefit substantially from SSD random-access performance.

Separating those workloads avoids using expensive SSD capacity for bulk data while still providing low-latency storage where it matters.

## SMB and NFS Strategy

The protocol choice is intentionally client-oriented:

| Protocol | Primary Clients |
|---|---|
| NFS | Linux systems and containers |
| SMB | Windows systems |

NFS is used heavily on Linux because it integrates cleanly with Linux permissions, mounts, and filesystem semantics.

SMB remains the natural choice for Windows because it integrates directly with Explorer, Windows credentials, and Windows network-share behavior.

## Share Examples

| Share | Pool | Purpose |
|---|---|---|
| media | primary-nas | Jellyfin and media storage |
| backups | primary-nas | Proxmox and general backups |
| games | game-pool | SSD-backed network game library |

The production NAS contains additional shares beyond this short project-oriented list.

## Snapshots

ZFS periodic snapshots are configured recursively so child datasets receive their own snapshots.

Current retention includes multiple weekly snapshots and longer-retained weekly snapshots. Because recursive snapshots create entries for each child dataset, the total snapshot count can appear much larger than the number of scheduled snapshot events.

## Backup Boundaries

The TrueNAS VM is excluded from routine Proxmox Backup Server jobs because VM-level backup operations previously interfered with the storage VM.

The data protection model therefore treats the TrueNAS data itself separately from ordinary VM/CT backups:

- ZFS mirror for disk redundancy on the primary HDD pool
- ZFS snapshots for recovery from logical changes
- Separate Proxmox Backup Server protection for the rest of the virtual environment
- NAS storage can also serve as a secondary Proxmox backup destination
