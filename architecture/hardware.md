# Homelab Hardware

Most server infrastructure is installed in a 20U rack. The main environment consists of eight Proxmox VE hosts, a dedicated Proxmox Backup Server, network and power infrastructure, and several Linux client systems.

## Proxmox VE Hosts

### `elite-node`

**Role:** General application and documentation services

- HP EliteDesk 705 G4 Mini
- AMD Ryzen 5 2400GE
- 16GB DDR4
- 500GB NVMe
- 2.5GbE
- Local ZFS storage

### `ai-node-01`

**Role:** Dedicated AI compute

- AMD Ryzen 5 3600
- MSI B450 Tomahawk MAX
- 32GB DDR4
- 1TB Crucial P310 NVMe
- AMD Radeon Instinct MI60 32GB
  - Custom 92mm fan shroud
  - 170W power limit
  - Vulkan for llama.cpp
  - ROCm 6.3 for ComfyUI
- Local ZFS storage
- Centralized AI model storage

### `nas-node-01`

**Role:** TrueNAS storage host

- HP EliteDesk 705 G5 Mini
- AMD Ryzen 5 3400GE
- 32GB DDR4
- 256GB NVMe boot SSD
- 10GbE SFP+
- Local ZFS storage

#### External PCIe / Storage Enclosure

- M.2 / OCuLink PCIe expansion
- LSI 9300-8i HBA
- 10GbE SFP+ NIC
- 500W EVGA ATX PSU
- 2U chassis
- 2x 80mm Noctua intake fans
- 1x 40mm Noctua HBA fan

#### Attached Storage

- 2x 14TB Western Digital Ultrastar HDDs
  - ZFS mirror under TrueNAS
- 2TB Samsung 860 QVO SATA SSD
  - SSD-backed game pool

### `encoder-node-01`

**Role:** Media encoding, transcoding, capture, and LAN streaming

- Dell OptiPlex 5040 SFF
- Intel Core i5-6500
- Intel HD Graphics 530 / Quick Sync
- NVIDIA Quadro P1000
- 16GB DDR3
- 250GB SATA SSD
- Magewell XI200DE-HDMI capture card
- 1GbE
- Local ZFS storage

### `usff-node-01`

**Role:** Utility services and experimental VMs

- Dell OptiPlex 9020 USFF
- Intel Core i7-4790S
- 16GB DDR3
- 250GB SATA SSD
- 2.5GbE
- Local ZFS storage

### `infra-control-node-01`

**Role:** Always-on networking, monitoring, UPS, and infrastructure control

- Dell Latitude 5280
- Intel Core i5-7300U
- 8GB DDR4
- 256GB M.2 SATA SSD
- 1GbE
- Internal battery
- Power-on with AC
- Local ZFS storage

### `compute-node-01`

**Role:** General-purpose VM compute and Kubernetes workloads

- AMD Ryzen 9 5900XT (16C / 32T)
- ASUS ROG Strix B450-F
- 64GB DDR4
- 512GB NVMe SSD
- AMD Radeon RX 570 8GB
- 10GbE
- Local ZFS storage

### `gaming-node-01`

**Role:** GPU-passthrough gaming and Sunshine/Moonlight streaming

- AMD Ryzen 9 7950X3D (16C / 32T)
- MSI MAG B650 Tomahawk WiFi
- 32GB DDR5
- 1TB NVMe SSD
- AMD Radeon RX 6700 XT 12GB
  - Primary render GPU
- Intel Arc A380 6GB
  - Display/compositor and hardware-encoding GPU
- Realtek RTL8125 2.5GbE
- Local ZFS storage

The system previously used an Intel X520 10GbE adapter, but the adapter is currently displaced by the Arc A380.

## Proxmox Backup Server

### `pbs-node-01`

**Role:** Dedicated Proxmox Backup Server

- Dell Latitude 3400
- Intel Core i3-8145U
- 8GB DDR4
- 128GB M.2 SATA SSD boot drive
- 500GB Western Digital Black 2.5-inch HDD datastore
- 1GbE
- ZFS on boot and datastore storage

## Network Hardware

### GL.iNet GL-SFT1200

- OpenWrt router/firewall

### SODOLA 8-Port 10Gb L3 Managed Switch

- 8x 10Gb SFP+ ports
- Management and Layer 3 features not yet configured

### 10-Port Unmanaged 2.5GbE Switch

- 8x 2.5GbE RJ45 ports
- 2x 10Gb SFP+ ports

### NICGIGA 6-Port 2.5GbE Switch

- 4x 2.5GbE RJ45 ports
- 2x 10Gb SFP+ ports

### Zyxel NWA240BE

- Wi-Fi 7 access point

## Power Hardware

### CyberPower CP1500PFCRM2U-Series UPS

- 1500VA / 1000W
- Pure sine wave
- 8 outlets
- USB monitoring

### WattBox WB-800-IPVM-12

- 12 network-managed outlets

## Other Computers

These systems are not part of the Proxmox cluster but are used to access and administer the lab.

### `linux-desktop-01`

**Role:** Primary Linux desktop and homelab client

- Dell OptiPlex 9020 SFF
- NixOS 26.05
- Intel Core i5-4570
- 16GB DDR3
- AMD Radeon RX 6300
- 250GB PNY CS900 SATA SSD
- 2.5GbE

### `proto-node`

**Role:** Mobile Linux client and administration system

- HP Pavilion 15z-cw100 laptop
- NixOS 26.05
- AMD Ryzen 7 3700U
- 16GB DDR4
- 500GB Samsung 850 EVO SATA SSD
- Previously served as an early Linux/NAS prototype before the dedicated TrueNAS architecture was built

### `think-node`

**Role:** Mobile homelab command station

- Lenovo ThinkPad W500
- Debian 13 with XFCE
- Intel Core 2 Duo T9400
- 8GB DDR3
- 250GB Crucial MX250 SATA SSD
