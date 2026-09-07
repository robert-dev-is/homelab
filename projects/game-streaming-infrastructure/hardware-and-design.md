# Hardware and Design

## Design Goals

The system is intended to combine server-style virtualization with the physical simplicity of a conventional gaming PC.

Key design goals are:

- keep the gaming environment virtualized under Proxmox VE,
- dedicate one GPU to rendering and another to display/capture/encoding,
- provide enough CPU resources for gaming while leaving capacity for additional game-server workloads,
- use local storage where latency matters and NAS storage where capacity matters,
- restore 10GbE connectivity without consuming a GPU slot,
- avoid exposed risers or external PCIe hardware in the finished system.

## CPU Selection and Workload Placement

The host uses an AMD Ryzen 9 7950X3D.

The CPU was originally purchased for gaming because the large V-Cache can benefit cache-sensitive and poorly optimized games. The additional eight-core CCD later became useful once the system evolved into a Proxmox host.

The gaming VM is pinned to the V-Cache CCD and its SMT siblings:

```text
7950X3D
├── V-Cache CCD
│   └── gaming VM
│
└── second CCD
    ├── Proxmox host capacity
    └── additional game-server workloads
```

This is CPU pinning and workload placement rather than CPU overclocking or tuning.

## GPU Role Separation

### Radeon RX 6700 XT

The RX 6700 XT is the dedicated rendering GPU. Games run through Proton with Mesa RADV and are explicitly directed to the Radeon device.

### Intel Arc A380

The Arc A380 is used for:

- the Labwc desktop,
- the headless Wayland output,
- Sunshine capture,
- H.264 hardware encoding,
- HEVC / HEVC Main10 hardware encoding,
- AV1 hardware encoding.

Separating these roles prevents the encoder from directly competing with a fully loaded render GPU. It also creates a new requirement: completed frames must move from the Radeon rendering path into the Arc-owned presentation and capture path efficiently.

## Current PCIe Topology

The current B650 motherboard exposes the GPUs through different paths:

```text
Ryzen CPU
│
├── CPU PCIe 4.0 x16 → RX 6700 XT
│
└── B650 chipset
    └── PCIe 4.0 x2 → Arc A380
```

The Arc card supports a wider link, but the motherboard's second full-length slot is electrically limited to x2 through the chipset.

The physical link was verified at the Arc's upstream PCIe bridge as PCIe 4.0 x2. The endpoint itself can report an internal x1 link, so validating the complete PCIe hierarchy was necessary to identify the real external connection.

## OCuLink Experiment

An M.2-to-OCuLink-to-PCIe path was tested as a way to give the Arc card access to a CPU-connected M.2 slot while keeping the existing network adapter installed.

The Arc received power but did not enumerate through that path. Moving the same card directly into the motherboard's secondary PCIe slot caused it to enumerate immediately.

That test established that the GPU itself was functional and narrowed the issue to the attempted OCuLink/topology path rather than the Arc card.

A native motherboard solution was preferred over continuing to build around external adapters and risers.

## Ordered Platform Revision

An ASRock X870 Taichi Creator has been ordered to provide two CPU-connected full-length slots operating at x8/x8.

Planned GPU topology:

```text
Ryzen CPU
│
├── PCIe 4.0 x8 → RX 6700 XT
└── PCIe 4.0 x8 → Arc A380
```

PCIe 4.0 x8 provides roughly four times the lane bandwidth of the Arc's current x2 connection while also removing the chipset from the inter-GPU path.

The change is intended to test whether high-resolution streaming limitations are primarily caused by cross-GPU transport and synchronization rather than Intel Arc encoder throughput.

## Networking

The current host is using onboard 2.5GbE after the previous 10GbE adapter was displaced by the second GPU.

The video stream itself does not require 10GbE. Sunshine bitrates are far below the capacity of a 2.5GbE link, and observed streaming problems have not correlated with packet loss or network latency.

10GbE is valuable primarily because much of the game library is NAS-backed. Higher local network throughput helps with:

- large game installations,
- updates,
- bulk file transfers,
- NAS-backed game libraries,
- movement between local and network storage.

The new motherboard includes onboard high-speed networking, allowing 10GbE connectivity to return without consuming another PCIe slot.

## Storage Design

The system uses both local NVMe and NAS-backed storage.

```text
NAS storage
└── default location for games and bulk storage

Local NVMe
└── latency-sensitive or metadata-heavy games and mod environments
```

Heavily modded Fallout 4 demonstrated that high sequential network throughput does not eliminate filesystem metadata latency. Moving that workload from NAS storage to local storage dramatically reduced launch time.

The resulting design uses NAS capacity by default while keeping local storage available for workloads that are sensitive to metadata or latency.

## Physical Design

The system is housed in a Fractal Design North XL.

Maintaining a conventional, self-contained gaming-PC layout is an intentional design requirement. The project avoids permanent exposed risers or external PCIe hardware even though those approaches could make experimentation easier.

This constraint influenced the decision to replace the motherboard with one that natively supports the required CPU-direct x8/x8 topology.
