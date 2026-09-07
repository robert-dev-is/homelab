# Hardware and Design

## Compute Host

| Component | Hardware |
|---|---|
| System | HP EliteDesk 705 G5 Mini |
| CPU | AMD Ryzen 5 3400GE |
| Memory | 32GB DDR4 |
| Hypervisor | Proxmox VE |
| NAS VM | TrueNAS SCALE |

The EliteDesk Mini was selected as a compact and efficient host, but the project intentionally pushes beyond the expansion normally expected from a Mini PC.

## PCIe Expansion Through OCuLink

The system uses M.2 PCIe connectivity as external PCIe links through OCuLink.

One path connects the LSI 9300-8i HBA and another connects the 10GbE SFP+ NIC. To the operating system, these devices enumerate as normal PCIe hardware even though they are physically located outside the Mini PC.

This approach allowed the project to retain a small compute host while moving hot, physically large, and cable-heavy PCIe devices into a separate enclosure.

## Storage HBA

The LSI 9300-8i provides direct SAS/SATA connectivity for the storage devices.

The HBA is passed through to the TrueNAS VM rather than managed by the Proxmox host. This gives TrueNAS direct disk visibility for ZFS and SMART monitoring.

The HBA normally uses a wider PCIe interface than the external x4 path provides, but the available PCIe bandwidth is still substantially greater than the throughput required by the current HDD and SATA SSD storage.

## External 2U Enclosure

A generic 2U rack chassis was repurposed as a storage and PCIe enclosure.

Because there is no motherboard inside the chassis, there is significant room for storage devices, PCIe adapters, airflow, and power cabling.

The enclosure contains:

- LSI 9300-8i HBA
- OCuLink PCIe carrier boards
- 10GbE SFP+ NIC
- 2x 14TB WD Ultrastar HDDs
- Samsung 860 QVO 2TB SATA SSD
- EVGA 500W ATX PSU
- 2x 80mm Noctua intake fans
- 1x 40mm Noctua fan dedicated to the HBA heatsink

## Cooling

The HBA heatsink remained hot with only chassis airflow, so a 40mm fan was mounted directly over it. This substantially reduced HBA temperature without requiring high chassis fan speeds.

The HDDs are spaced to allow airflow between them and reduce direct vibration transfer.

Typical observed storage temperatures are in the mid-30°C range.

## Power Architecture

The enclosure uses a standard ATX PSU. The HBA carrier's 24-pin ATX interface controls PSU startup automatically based on the host PCIe state.

A second PCIe carrier for the SFP+ NIC receives SATA power once the enclosure PSU is active.

This provides one of the most important usability features of the design: the external storage hardware powers on and off with the NAS host.

## Power Consumption

Observed idle consumption has been approximately:

| Component | Approximate Idle Power |
|---|---|
| Mini PC | ~15W |
| External enclosure | ~35W |
| Combined system | ~50W |

The enclosure figure includes the always-spinning HDDs, SSD, HBA, SFP+ NIC, PCIe carriers, cooling fans, and PSU losses.
