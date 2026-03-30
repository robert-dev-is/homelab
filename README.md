# Homelab Infrastructure
This repository documents the design, implementation, and ongoing development of my personal homelab environment.

# Overview
This lab is designed to simulate real-world enterprise infrastructure while exploring modern technologies in virtualization, networking, storage, and local AI workloads.

Key capabilities include:

- Proxmox VE 9.1.6 virtualization cluster
- Fully local/offline-capable services
- Local AI inference (llama.cpp + OpenWebUI)
- Local AI image generation (Stable Diffusion via ComfyUI)
- LXC based service deployment
- NAS with direct HBA passthrough (TrueNAS)
- Isolated lab network with multi-device access

# Hardware
Elite-Node (Primary Service Node):
- HP EliteDesk Mini 705 G4
- AMD Ryzen 5 2400GE
- 12GB DDR4
- 512GB NVME
- Proxmox VE 9.1.6
- Primary service node in cluster that runs OpenWebUI, Unbound, AdGuard

NAS-Node-01 (Storage Node):
- HP EliteDesk Mini 705 G5
- AMD Ryzen 5 3400GE
- 16GB DDR4
- 256GB NVME
- Proxmox VE 9.1.6
- M.2 Oculink to Oculink eGPU dock used for HBA
- 9300-8i LSI HBA passed through to TrueNAS vm in Proxmox
- 2x Western Digital 14TB Ultrastar HDD in ZFS Mirror

AI-Node-01 (AI Compute Node):
- AMD Ryzen 3600
- AMD Radeon Instinct Mi60 32GB (175w power limit)
- 32GB DDR4
- 1TB Crucial P310 NVME
- Proxmox VE 9.1.6
- Hosts llama and diffusion LXC containers with Mi60 exposed to them

# Network Design

- 2.5GbE internal network
- Static IP scheme (192.168.4.x to 10.10.10.x)
- OpenWRT-based router for network control and segmentation
- Unbound service used to manage DNS
- AdGuard Home service used to filter out unwanted ads, monitoring, and telemetry from WAN

# Key Projects

Virtualization:
- Multi node cluster using Proxmox VE 9.1.6
- Ability to easily create and manage virtual machines and containers to run services
- Transitioned old services, such as nas and ai inference, from bare-metal Ubuntu to Proxmox for better management
- Snapshot-based workflows for safe testing and rollback
- Use Proxmox monitoring/logs for basic statistics and Netdata for more in depth monitoring

NAS/Storage:
- TrueNAS VM with HBA passed through to allow direct access to hdds
- Network file shares via SMB for storage and backups (such as Proxmox backups)

AI Infrastructure:
- Models are stored centrally on AI-Node-01 and mounted into LXC containers
  - Prevents duplicate downloads
  - Protects models from container rollbacks or corruption

- Local LLM hosting via llama.cpp
  - Ubuntu 24.04 LXC Container
  - Uses Vulkan for inference (comparable performance to ROCm as of 2026)

- OpenWebUI hosted on Elite-Node
  - Debian 12.13 LXC container
  - Uses PostgreSQL for persistent chat storage
  - Allows for chats to be accessed across devices
  - Connects to llama.cpp service hosted on AI-Node-01

- This architecture seperates compute (AI-Node-01) from user facing services (Elite-Node), improving scalability and resource efficiency.

- Automated OpenWebUI backups
  - Daily at 4AM
  - Retains last 14 backups
  - Stored on NAS

- Image Generation service hosted by ComfyUI
  - Ubuntu 22.04 LXC container
  - Uses ROCm 6.3 (last supported version for Mi60)
  - Uses Stable Diffusion XL

# Goals

- Build a scalable, local infrastructure for services such as ai, streaming, storage
- Gain hands-on experience with enterprise systems
- Document real-world troubleshooting and experience
- Transition into more advanced infrastructure or AI engineering roles

# Notes

- This lab is continuously evolving, with frequent testing, troubleshooting, and iteration to refine architecture and performance

# Future Plans

- Expand Proxmox cluster nodes
- Implement VLAN segmentation into lab network
- Implement real-time model switching for ai inference
- Develop lan based streaming for live streams and media streams
