# Homelab Architecture Overview

The homelab is a multi-node infrastructure environment built around Proxmox VE, centralized TrueNAS storage, dedicated AI compute, Kubernetes, observability, high-speed networking, and UPS-backed power infrastructure.

Most server infrastructure is installed in a 20U rack and is used for hands-on work with Linux administration, virtualization, networking, storage, local AI, automation, monitoring, and infrastructure engineering.

## At a Glance

| Area | Platform |
|---|---|
| Virtualization | 8 Proxmox VE 9.2 hosts |
| Backups | Dedicated Proxmox Backup Server 4.2 |
| Storage | TrueNAS SCALE, ZFS, SMB, NFS |
| Networking | OpenWrt, 2.5GbE, 10GbE SFP+, Wi-Fi 7 |
| Kubernetes | Talos Linux, Flux, MetalLB, Traefik, NFS CSI |
| AI | AMD Instinct MI60 32GB, llama.cpp, ComfyUI, Open WebUI |
| Monitoring | Prometheus, Grafana, node_exporter, NUT |
| Documentation | Markdown, Obsidian, Forgejo, automated Git snapshots |

## Physical Node Roles

| Node | Primary Role |
|---|---|
| `elite-node` | General application and documentation services |
| `ai-node-01` | Local AI inference and image-generation compute |
| `nas-node-01` | TrueNAS storage host |
| `encoder-node-01` | Media encoding, transcoding, capture, and LAN streaming |
| `usff-node-01` | Utility services, experimental VMs, AI-agent workloads, and Kubernetes |
| `infra-control-node-01` | Core network services, monitoring, UPS monitoring, and infrastructure control |
| `compute-node-01` | General-purpose VM compute and Kubernetes workloads |
| `gaming-node-01` | GPU-passthrough gaming and Sunshine/Moonlight streaming |
| `pbs-node-01` | Proxmox Backup Server |

Detailed specifications are in [Hardware](hardware.md).

## Virtualization

Proxmox VE is the main virtualization layer.

LXC containers are preferred for lightweight, service-focused workloads. Virtual machines are used where stronger isolation, specialized operating systems, PCIe/GPU passthrough, or appliance-style deployment make more sense.

The lab separates physical roles across general services, storage, AI, media, infrastructure control, compute, and gaming rather than concentrating everything on one host.

## Storage and Backups

TrueNAS SCALE provides centralized persistent storage using ZFS, SMB, and NFS. The NAS uses direct HBA passthrough and 10GbE connectivity.

A dedicated Proxmox Backup Server handles VM and container backups, while TrueNAS provides ZFS snapshots for dataset-level protection.

See [Storage](storage.md).

## Networking

OpenWrt handles routing and DHCP for the primary lab network. Core DNS, filtering, reverse-proxy, certificate, and remote-access services run separately.

The physical network combines 2.5GbE access switching, 10GbE SFP+ connectivity, a managed 10GbE switch, and a Zyxel NWA240BE Wi-Fi 7 access point.

A separate `10.10.20.0/24` network isolates AI-agent experimentation from the primary LAN while allowing only explicitly required access.

See [Network](network.md).

## AI Infrastructure

The primary local AI platform separates user-facing services from GPU compute.

`ai-node-01` provides AMD Instinct MI60 32GB compute for llama.cpp and ComfyUI, while Open WebUI and supporting services run separately.

A second AI environment is isolated for agent experimentation so autonomous tooling does not receive broad access to the main lab.

## Kubernetes

The `infra-services` Talos Linux Kubernetes cluster uses one control-plane VM, two worker VMs, and a dedicated NixOS administration container.

Flux and Forgejo provide GitOps management. MetalLB, Traefik, cert-manager, NFS CSI, Metrics Server, and Rancher provide supporting cluster services, with TrueNAS supplying persistent storage.

Vaultwarden is currently running as an application workload.

## Monitoring and Power

Prometheus and Grafana provide centralized observability for Proxmox, Linux systems, and UPS telemetry.

The rack is protected by a CyberPower 1500VA / 1000W UPS monitored through NUT. A WattBox provides network-managed outlet control.

Automated full-lab shutdown and startup sequencing is still in development.

## Documentation

Homelab documentation is maintained in Markdown, stored on TrueNAS, and tracked in Forgejo.

`doc-tracker-01` checks the documentation share every 10 minutes. When a change is detected, it waits for one hour with no further changes before committing and pushing a snapshot. Any additional edit during that hour resets the timer.

This keeps useful version history without generating excessive commits during active documentation sessions.

## Design Principles

- Prefer one primary service per LXC container where practical
- Separate compute, storage, AI, media, and infrastructure-control roles
- Keep experimental workloads isolated from core infrastructure
- Keep persistent data separate from disposable compute where possible
- Use snapshots and backups to make experimentation recoverable
- Track infrastructure changes in Git-backed Markdown
- Prefer systems that expose how the underlying infrastructure works
