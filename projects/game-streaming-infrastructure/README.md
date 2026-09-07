# Game Streaming Infrastructure

A homelab project focused on building a clean, high-performance game streaming server with **Proxmox VE**, **NixOS**, **Sunshine/Moonlight**, and dedicated GPUs for rendering and encoding.

The goal is to make the system behave like a remotely accessible gaming appliance while still retaining the flexibility of a virtualized Linux host.

## Project Goals

- Run the gaming environment as a VM under Proxmox VE.
- Pass through a dedicated render GPU directly to the VM.
- Use a second GPU for the Wayland compositor, virtual display, capture, and hardware encoding.
- Keep the guest configuration declarative with NixOS.
- Stream games to lightweight clients through Sunshine and Moonlight.
- Support both local NVMe storage and NAS-hosted game libraries.
- Preserve a clean, conventional gaming-PC form factor rather than relying on exposed risers or external PCIe hardware.

## Current Hardware

| Component | Role |
|---|---|
| AMD Ryzen 9 7950X3D | Host CPU |
| AMD Radeon RX 6700 XT 12 GB | Game rendering |
| Intel Arc A380 6 GB | Compositor, virtual display, capture, and hardware encoding |
| 32 GB DDR5 | Host memory |
| 1 TB NVMe SSD | Local VM/game storage |
| 2.5 GbE | Current network connection |

The VM is pinned to the 7950X3D's V-Cache CCD and receives both discrete GPUs through PCIe passthrough.

## Software Stack

```text
Proxmox VE
└── NixOS gaming VM
    ├── Labwc
    ├── Steam + Proton
    ├── Mesa RADV
    ├── MangoHud
    ├── Sunshine
    └── Intel VAAPI / iHD
```

### GPU Roles

```text
Radeon RX 6700 XT
└── Game rendering

Intel Arc A380
├── Labwc compositor
├── Headless Wayland display
├── Sunshine capture
└── H.264 / HEVC / AV1 hardware encoding
```

Games are explicitly directed to the Radeon GPU while the desktop remains on the Arc GPU.

Example Steam launch option:

```text
DRI_PRIME=pci-0000_01_00_0 %command%
```

The exact PCI address is system-specific and should always be verified before reuse.

## Why NixOS?

NixOS works well for this project because the gaming VM is intended to behave more like an appliance than a general-purpose desktop.

Most permanent configuration lives in declarative files under `/etc/nixos`, including:

- Steam
- GPU support
- Sunshine
- PipeWire
- Labwc
- Home Manager
- MangoHud

Intel Arc hardware encoding required only the Intel media driver to be added to the NixOS graphics configuration. After that, VAAPI exposed hardware H.264, HEVC, HEVC Main10, and AV1 encoding successfully.

## Streaming Design

The active path is:

```text
Game
 ↓
RX 6700 XT
 ↓
PRIME / cross-GPU presentation
 ↓
Arc A380 / Labwc
 ↓
Wayland screencopy
 ↓
Sunshine
 ↓
Arc VAAPI encoder
 ↓
Moonlight client
```

Sunshine captures the Labwc headless output through the Wayland screencopy protocol and uses the Arc A380's Intel media engine for encoding.

## Current Limitation

The current motherboard gives the two GPUs very different PCIe paths:

```text
CPU
├── PCIe 4.0 x16 → RX 6700 XT
└── Chipset
    └── PCIe 4.0 x2 → Arc A380
```

This has become the primary architectural limitation during high-resolution, high-refresh-rate streaming tests.

The game itself can remain smooth while Sunshine's delivered frame rate falls significantly as the host display resolution increases. The Arc media engine is not saturated during the bad case, which points toward cross-GPU transfer and synchronization rather than raw encoder performance.

See [STREAMING-FINDINGS.md](STREAMING-FINDINGS.md) for the test results.

## Planned Architecture

The next platform revision is intended to give both GPUs CPU-direct lanes:

```text
CPU
├── PCIe 4.0 x8 → RX 6700 XT
└── PCIe 4.0 x8 → Arc A380
```

This should provide substantially more bandwidth for cross-GPU presentation while keeping the machine physically clean and self-contained.

The project will then be retested at 4K120 and 4K240.

## Storage Strategy

Most games can live on NAS-backed storage, but not every workload behaves well over a network filesystem.

A useful lesson from this project was that **filesystem metadata latency can matter more than raw network bandwidth**. Heavily modded Fallout 4 generated large numbers of metadata operations and launched dramatically faster after being moved to local storage.

General policy:

```text
NAS storage
└── default location for games

Local NVMe
└── games, prefixes, or mod environments that are metadata-heavy
```

## Repository Layout

Suggested public project structure:

```text
.
├── README.md
├── ARCHITECTURE.md
└── STREAMING-FINDINGS.md
```

Internal homelab documentation contains additional host-specific details, IP addresses, service dependencies, and operational notes that are intentionally omitted from the public version.

## Status

**Active project.**

The dual-GPU architecture works, Intel Arc hardware encoding works correctly under NixOS, and the remaining high-refresh limitation is being investigated as a PCIe topology problem rather than an encoder capability problem.
