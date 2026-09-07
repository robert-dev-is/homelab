# Homelab Infrastructure

A multi-node homelab built for hands-on experience with Linux infrastructure,
virtualization, networking, storage, Kubernetes, local AI, observability,
automation, and systems engineering.

The environment currently includes eight Proxmox VE hosts, a dedicated
Proxmox Backup Server, TrueNAS/ZFS network storage, 10GbE networking,
a Talos Kubernetes cluster, AMD Instinct GPU compute, centralized monitoring,
and isolated environments for infrastructure and AI experimentation.

## Lab at a Glance

- 8 Proxmox VE hosts
- Dedicated Proxmox Backup Server
- TrueNAS SCALE with ZFS and HBA passthrough
- 10GbE SFP+ networking
- Zyxel NWA240BE Wi-Fi 7 access point
- Talos Kubernetes with Flux GitOps
- AMD Instinct MI60 32GB AI compute
- Prometheus + Grafana observability
- NUT-monitored rack UPS
- Forgejo-backed infrastructure documentation

## Featured Projects

### TrueNAS Network Storage Architecture
Compact virtualized storage system using PCIe passthrough, an LSI HBA,
external OCuLink expansion, ZFS, SMB/NFS, and 10GbE networking.

[View project →](projects/truenas-network-storage/)

### Talos Kubernetes Platform
Three-node Talos Kubernetes environment using Flux and Forgejo for GitOps,
MetalLB, Traefik, NFS CSI-backed TrueNAS storage, and internal TLS.
Vaultwarden currently runs as a production-like internal workload.

[View project →](projects/kubernetes/)

### Local AI Infrastructure
Self-hosted AI platform built around an AMD Instinct MI60 32GB accelerator,
with llama.cpp, ComfyUI, Open WebUI, centralized model storage, and separate
frontend and GPU-compute services.

[View project →](projects/ai-infrastructure/)

### Isolated AI Agent Lab
Network-segmented environment for experimenting with autonomous AI agents
while providing only specifically permitted access to services on the primary
homelab network.

[View project →](projects/isolated-ai-agent-lab/)

### Game Streaming Infrastructure
Virtualized gaming environment built on Proxmox, with NixOS as the VM OS,
using GPU passthrough, dedicated AMD rendering and Intel Arc hardware encoding,
CPU pinning, and Sunshine/Moonlight streaming.

[View project →](projects/game-streaming-infrastructure/)

### Observability Platform
Centralized Prometheus and Grafana monitoring for Proxmox, Linux hosts,
virtual machines, containers, storage, and UPS telemetry.

[View project →](projects/observability/)

### Automated Documentation Pipeline
NAS-hosted Markdown documentation automatically tracked in Git and pushed
to Forgejo after a one-hour stability window to preserve useful history
without generating excessive commits.

[View project →](projects/documentation-automation/)

## Architecture

The lab separates physical roles for compute, storage, AI, media,
infrastructure control, and gaming.

[View architecture documentation →](architecture/)

## Core Technologies

**Virtualization:** Proxmox VE, LXC, KVM/QEMU, VFIO  
**Linux:** Debian, NixOS, Talos Linux  
**Storage:** TrueNAS SCALE, ZFS, NFS, SMB, PCIe/HBA passthrough  
**Networking:** OpenWrt, 2.5GbE, 10GbE SFP+, Wi-Fi 7, Tailscale  
**Kubernetes:** Talos, Flux, Kustomize, Helm, MetalLB, Traefik, NFS CSI  
**AI:** AMD Instinct MI60, llama.cpp, Vulkan, ROCm, ComfyUI, Open WebUI  
**Observability:** Prometheus, Grafana, node_exporter, NUT  
**DevOps / Documentation:** Git, Forgejo, Markdown, Obsidian

## Design Principles

- Prefer service separation and clear infrastructure roles
- Keep persistent data separate from disposable compute
- Treat experimental systems as isolated environments
- Use snapshots and backups to make experimentation recoverable
- Prefer transparent infrastructure that exposes how the underlying
  technology works
- Document design decisions, troubleshooting, and lessons learned

## Current Development

Current areas of continued development include:

- Kubernetes workloads and GitOps
- Internal PKI and automated certificate trust
- Power-loss shutdown and startup orchestration
- Managed 10GbE networking
- Local AI and agent infrastructure