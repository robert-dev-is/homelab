# Hardware and Design

## Design Goals

The system is intended to combine server-style virtualization with the physical simplicity of a conventional gaming PC.

Key design goals are:

- keep the gaming environment virtualized under Proxmox VE,
- dedicate one GPU to rendering and another to display/capture/encoding,
- provide enough CPU resources for gaming while leaving capacity for additional game-server workloads,
- use local storage where latency matters and NAS storage where capacity matters,
- provide 10GbE connectivity,
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

The gaming VM is allowed to execute only on the V-Cache CCD through Proxmox CPU affinity. This constrains VM placement but does not exclusively reserve those host threads; other substantial workloads can be placed on the opposite CCD when stronger workload separation is desired.

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

The host now uses an ASRock X870 Taichi Creator, which provides CPU-connected PCIe lanes to both GPUs:

```text
Ryzen 9 7950X3D
│
├── CPU PCIe 4.0 x8 → RX 6700 XT
└── CPU PCIe 4.0 x8 → Intel Arc A380
```

Both GPUs therefore operate through symmetrical CPU-direct PCIe 4.0 x8 links.

The motherboard change removed the previous chipset-connected x2 path from the cross-GPU streaming architecture and also provided onboard 10GbE without consuming another expansion slot.

## Previous B650 Topology

The previous B650 motherboard exposed the GPUs through different paths:

```text
Ryzen CPU
│
├── CPU PCIe 4.0 x16 → RX 6700 XT
│
└── B650 chipset
    └── PCIe 4.0 x2 → Arc A380
```

The Arc supported a wider link, but the motherboard's second full-length slot was electrically limited to PCIe 4.0 x2 through the chipset.

Because the streaming problem became substantially worse at high resolutions, this asymmetric topology was initially suspected as the primary cross-GPU bottleneck.

Moving to the X870 x8/x8 platform removed that limitation, but the high-load streaming problem remained. The old PCIe topology is therefore no longer considered the root cause.

Later tracing instead showed that stream-critical AMDGPU graphics work could spend tens of milliseconds waiting in the RX software queue before the frame-handoff dependency completed.

## OCuLink Experiment

Before replacing the motherboard, an M.2-to-OCuLink-to-PCIe path was tested as a way to give the Arc access to a CPU-connected M.2 slot while retaining the existing network adapter.

The Arc received power but did not enumerate through that path. Moving the same card directly into the motherboard's secondary PCIe slot caused it to enumerate immediately.

The experiment confirmed that the Arc itself was functional and reinforced the preference for a native motherboard solution rather than permanent external PCIe adapters or risers.

## Platform Revision Result

The ASRock X870 Taichi Creator was installed to provide:

CPU-direct PCIe 4.0 x8 links to both GPUs,
onboard 10GbE,
improved physical spacing between the GPUs,
a cleaner self-contained expansion layout.

The platform revision successfully achieved those goals.

It also served as an important diagnostic test. Eliminating the previous chipset-connected Arc path did not eliminate the high-load streaming problem, demonstrating that PCIe topology was not the primary cause.

The current investigation instead points to RX-side scheduling and synchronization latency during heavily loaded gaming workloads.

## Networking

The host now uses the ASRock X870 Taichi Creator's onboard Marvell/Aquantia AQC113 10GbE interface.

The previous configuration temporarily fell back to onboard 2.5GbE because installing the Arc A380 displaced the Intel X520 adapter. Moving to the X870 platform restored 10GbE without requiring another PCIe expansion slot.

The Sunshine video stream itself does not require 10GbE. Typical streaming bitrates remain far below even 2.5GbE capacity, and the observed incoming-frame-rate problems have not correlated with packet loss or network latency.

10GbE is primarily valuable because much of the game library is NAS-backed. The additional bandwidth benefits:

- large game installations,
- updates,
- bulk file transfers,
- NAS-backed game libraries,
- movement between local and network storage.

## Storage Design

The system uses both local NVMe and NAS-backed storage.

```text
NAS storage
└── default location for games and bulk storage

Local NVMe
└── latency-sensitive or metadata-heavy games and mod environments
```

Heavily modded Fallout 4 demonstrated that high sequential network throughput does not eliminate filesystem metadata latency. Moving that workload from NAS storage to local storage dramatically reduced launch time.

The Witcher 3 uses a mixed approach. The main game installation remains on NAS storage, while the save-game data normally stored under the Windows Documents path is kept locally through the Proton compatibility environment. This preserves the capacity and convenience of NAS-hosted game files while keeping latency-sensitive save data on local NVMe.

The resulting design is intentionally selective rather than all-or-nothing: large game installations can remain on the NAS when they perform well there, while individual components that are sensitive to latency, metadata access, or filesystem behavior can be moved to local storage.

## Thermal Design

Despite the dual-GPU configuration, both GPUs remain comfortably cooled during sustained gaming and streaming workloads.

Recent heavy-load observations include:

- RX 6700 XT: approximately 67°C near full utilization at roughly 190-200 W
- Intel Arc A380: approximately 64°C during display/capture/encoding workloads

The RX 6700 XT uses a large triple-fan cooler, while the Arc occupies the lower expansion position. The North XL provides ventilation beneath the lower GPU, and the X870 motherboard provides additional spacing between the two cards compared with the previous layout.

Current temperatures do not indicate a thermal limitation in the streaming architecture.

## Physical Design

The system is housed in a Fractal Design North XL.

Maintaining a conventional, self-contained gaming-PC layout is an intentional design requirement. The project avoids permanent exposed risers or external PCIe hardware even though those approaches could make experimentation easier.

This constraint influenced the decision to replace the motherboard with one that natively supports the required CPU-direct x8/x8 topology.
