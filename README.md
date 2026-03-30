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
- 2x Western Digital 14TB Ultrastar HDD

AI-Node-01
- AMD Ryzen 3600
- AMD Radeon Instinct Mi60 32GB
- 32GB DDR4
- 1TB Crucial P310 NVME
- Proxmox VE 9.1.6
- Hosts llama and diffusion LXC containers with Mi60 exposed to them
