# Architecture

## Design Summary

The storage platform separates compute from the physical storage enclosure.

The compute host is an HP EliteDesk 705 G5 Mini running Proxmox VE. TrueNAS SCALE runs as a virtual machine. An LSI 9300-8i HBA is connected externally through an M.2-to-OCuLink PCIe path and passed directly through to TrueNAS.

A second M.2-to-OCuLink path connects a 10GbE SFP+ NIC. Proxmox and the TrueNAS VM share the network interface through the host bridge.

## Compute Layer

```text
HP EliteDesk 705 G5 Mini
├── Proxmox VE
│   └── TrueNAS SCALE VM
├── 32GB DDR4
├── NVMe boot storage
├── M.2 -> OCuLink -> HBA
└── M.2 -> OCuLink -> 10GbE NIC
```

TrueNAS receives a fixed RAM allocation and direct access to the storage HBA. This avoids presenting the storage disks as virtual disks and allows TrueNAS/ZFS to interact with the physical drives directly.

## External Storage Layer

The PCIe cards and storage devices are installed in a separate 2U enclosure rather than inside the Mini PC.

```text
External 2U enclosure
├── LSI 9300-8i HBA
├── 10GbE SFP+ NIC carrier
├── 2x 14TB WD Ultrastar HDDs
├── 2TB Samsung 860 QVO SSD
├── 500W EVGA ATX PSU
├── 2x 80mm Noctua intake fans
└── 1x 40mm Noctua HBA fan
```

The HBA's PCIe carrier uses a 24-pin ATX connection and automatically controls the enclosure PSU based on the host's PCIe power state. This allows the external enclosure to start and stop with the compute node instead of requiring a separate manual power sequence.

## Storage Data Path

```text
Application / Client
        |
   SMB or NFS
        |
      10GbE
        |
Proxmox Linux bridge
        |
 TrueNAS SCALE VM
        |
PCIe passthrough
        |
  LSI 9300-8i
        |
   SAS / SATA
        |
   ZFS storage
```

## Why Virtualize TrueNAS?

Virtualizing TrueNAS allows the storage server to participate in the broader Proxmox environment while retaining direct access to the disks through PCIe passthrough.

The important design boundary is that the HBA is passed directly to TrueNAS. Proxmox manages the VM, while TrueNAS manages the disks and ZFS pools.

## Network Architecture

The original design used 2.5GbE and achieved roughly the practical limit of that connection. The system was later upgraded to 10GbE SFP+ so the network would no longer cap SSD-backed workloads and cached ZFS reads.

The TrueNAS VM uses a virtual NIC attached to the Proxmox bridge rather than requiring a dedicated physical NIC exclusively for the VM. This keeps the network layout simple while still allowing high throughput.
