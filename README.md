# Homelab Infrastructure
This repository documents my personal homelab environment focused on virtualization, networking, storage, and local AI workloads.

# Overview
This lab is designed to simulate real-world enterprise infrastructure while experimenting with real world technolgies such as:

- Proxmox VE 9.1.6 virtualization cluster
- Offline and local services
- Local AI inference using llama.cpp and OpenWebUI
- Local AI image generation using diffusion
- LXC Containers to run services
- NAS with HBA Passthrough
- Designated lab network
- Ability to access services from other devices on the lab's network (main pc, laptop, phone)

# Hardware
Elite-Node:
- HP EliteDesk Mini 705 G4
- AMD Ryzen 5 2400GE
- 12GB DDR4
- 512GB NVME
- Proxmox VE 9.1.6
- Primary service node in cluster that runs OpenWebUI, Unbound, AdGuard

NAS-Node-01:
- HP EliteDesk Mini 705 G5
- AMD Ryzen 5 3400GE
- 16GB DDR4
- 256GB NVME
- Proxmox VE 9.1.6
- M.2 Oculink to Oculink eGPU dock used for HBA
- 9300-8i LSI HBA passed through to TrueNAS vm in Proxmox
- 2x Western Digital 14TB Ultrastar HDD in ZFS Mirror

AI-Node-01
- AMD Ryzen 3600
- AMD Radeon Instinct Mi60 32GB
- 32GB DDR4
- 1TB Crucial P310 NVME
- Proxmox VE 9.1.6
- Hosts llama and diffusion LXC containers with Mi60 exposed to them

# Network Design

- 2.5GbE internal network
- Static IP scheme (192.168.4.x to 10.10.10.x)
- OpenWRT router

# Key Projects

Virtualization:

NAS/Storage:

AI Infrastructure:
- Models are stored on AI-Node-01's drive in dedicated folders and LXC Containers mount to the needed folders to access models
  - This ensures duplicate models are not downloaded on LXC Containers and dare secure if container needs to go back to a snapshot or container becomes corrupted
- Local LLM hosting via llama.cpp
  - LXC Container with Ubuntu 24.04 template using Vulkan as it provided similar performance to ROCm for inference
- OpenWebUI hosted on Elite-Node instead of AI-Node-01
  - LXC Container with Debian 12.13 template that also uses PostgreSQL to store chats instead of relying on browser cache
  - This allows for chats to be accessed on multiple devices
  - Models are pulled from port broadcasted by AI-Node-01's llama service
  - Daily backups are run at 4AM, keeps last 14 backups, and are saved to NAS
 - Image Generation service hosted by ComfyUI
  - LXC Container with Ubuntu 22.04 template using ROCm 6.3 as it's the last official version supported for the Mi60
  - Uses Stable Diffusion XL as main model for image generation

# Goals

# Future Plans
