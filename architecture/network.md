# Network Architecture

The homelab network combines OpenWrt routing, dedicated network-service containers, 2.5GbE and 10GbE switching, Wi-Fi 7, Tailscale remote access, and a separately isolated AI-agent network.

## Primary LAN

- Network: `10.10.10.0/24`
- Internal domain: `home.lab`
- Router: GL.iNet GL-SFT1200 running OpenWrt
- DHCP: OpenWrt

## Physical Network

| Device | Role |
|---|---|
| SODOLA 8-port 10Gb L3 managed switch | 10GbE SFP+ switching |
| 10-port unmanaged 2.5GbE switch | 2.5GbE access switching with 10Gb SFP+ uplinks |
| NICGIGA 6-port unmanaged  2.5GbE switch | 2.5GbE access switching with 10Gb SFP+ uplinks |
| Zyxel NWA240BE | Wi-Fi 7 access point |

`nas-node-01` and `compute-node-01` currently use 10GbE connectivity.

The SODOLA switch is installed, but its management and Layer 3 features have not yet been configured.

## Core Network Services

| IP | Service | Purpose |
|---|---|---|
| `10.10.10.3` | `vpn-01` | Tailscale subnet router |
| `10.10.10.4` | `dns-01` | Unbound recursive DNS |
| `10.10.10.5` | `reverse-proxy-01` | Zoraxy reverse proxy |
| `10.10.10.6` | `ad-blocker-01` | AdGuard Home DNS filtering |
| `10.10.10.7` | `cert-authority-01` | Internal certificate authority |

## DNS and Internal Services

Client DNS queries flow through AdGuard Home for filtering and then to Unbound for recursive resolution.

```text
Client
  ↓
AdGuard Home
  ↓
Unbound
  ↓
Upstream DNS
```

Internal `home.lab` web services resolve to Zoraxy, which terminates TLS and proxies requests to the appropriate backend service.

```text
service.home.lab
      ↓
     DNS
      ↓
   Zoraxy
      ↓
Backend service
```

## Remote Access

A dedicated Tailscale subnet-router container provides remote access to the primary lab network without exposing internal services directly through public port forwarding.

## Isolated AI Network

The AI-agent environment uses a separate network:

`10.10.20.0/24`

It connects through a dedicated interface on `usff-node-01` and an isolated Proxmox bridge. OpenWrt enforces the network boundary.

Current policy:

- WAN access is allowed
- access to the llama.cpp endpoint at `10.10.10.41:8080` is allowed
- unrestricted access from VLAN 20 to the primary LAN is blocked
- direct NAS access from VLAN 20 is blocked
- TCP 22 from the primary LAN to `ai-agent-01` is allowed for SSH administration

Documented systems on the isolated network include:

- `ai-agent-lab-01` — `10.10.20.11`
- `ai-agent-nas-01` — `10.10.20.12`
- `ai-agent-01` — isolated Hermes agent VM

## Current Development

The next major networking area is using more of the SODOLA switch's managed and Layer 3 capabilities, including additional VLAN and routing experimentation.
