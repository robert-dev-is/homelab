# Architecture

## Overview

This project uses a split-GPU architecture inside a NixOS virtual machine running on Proxmox VE.

The Radeon GPU is dedicated to game rendering. The Intel Arc GPU owns the Wayland desktop, headless output, Sunshine capture path, and hardware encoder.

The intent is to keep game rendering from directly competing with video encoding while maintaining a fully remote-accessible gaming environment.

## Host / Guest Layout

```text
Physical Host
┌──────────────────────────────────────────────┐
│ Proxmox VE                                   │
│                                              │
│ Ryzen 9 7950X3D                              │
│      │                                       │
│      ├── PCIe 4.0 x8 → RX 6700 XT ───┐       │
│      │                               │ VFIO  │
│      └── PCIe 4.0 x8 → Arc A380 ─────┤       │
│                                       ▼      │
│                            NixOS Gaming VM   │
└──────────────────────────────────────────────┘

Inside the guest:

```text
NixOS Gaming VM
│
├── RX 6700 XT
│   └── Steam games / Proton / RADV
│
└── Arc A380
    ├── Labwc
    ├── HEADLESS-1
    ├── Wayland screencopy
    └── Sunshine VAAPI encode
```

## CPU Placement

The VM uses 16 vCPUs pinned to the 7950X3D V-Cache CCD and its SMT siblings.

Example Proxmox CPU configuration:

```text
sockets: 1
cores: 16
cpu: host
affinity: 0-7,16-23
```

This keeps the gaming VM on the cache-equipped CCD rather than allowing its vCPUs to move between CCDs.

CPU affinity constrains where the gaming VM may execute, but it does not exclusively reserve those host threads. Other substantial VM or container workloads can be pinned to the opposite CCD (`8-15,24-31`) when isolation from the gaming workload is desirable.

## GPU Passthrough

Both discrete GPUs are bound to `vfio-pci` on the host before VM startup.

A previous attempt to hot-unbind the Radeon GPU from `amdgpu` caused the host task to enter uninterruptible kernel sleep during device removal. Binding passthrough devices to VFIO at boot removed that failure path and made startup behavior more predictable.

Generalized VFIO configuration:

```text
options vfio-pci ids=<gpu1>,<gpu1-audio>,<gpu2>,<gpu2-audio>
```

Device IDs should always be determined from the actual hardware rather than copied from another system.

## Guest GPU Selection

The Arc A380 owns the desktop, while games are launched on the Radeon GPU with Mesa PRIME selection.

Example:

```text
DRI_PRIME=pci-0000_01_00_0 %command%
```

This is intentionally applied to games rather than the entire Steam process. Forcing the whole Steam UI onto the render GPU caused instability in Chromium/CEF-based Steam components during testing.

## AMDGPU Queue Tuning

The split-GPU design is sensitive not only to GPU throughput but also to queue latency on the RX 6700 XT.

Under heavy game load, tracing showed that graphics work required by the cross-GPU frame handoff could remain queued for roughly 40 ms before execution even though the work itself took only a fraction of a millisecond to complete.

The AMDGPU software queue is therefore tuned with:

```text
amdgpu.sched_jobs
```

The stock value of 32 allowed excessive queue-ahead for this workload. Testing with values of 8 and 4 substantially improved frame pacing and Moonlight incoming frame rate by limiting how much unfinished RX work can accumulate ahead of frame-handoff dependencies.

Conceptually:

```text
Deep RX software queue
        ↓
Game work accumulates ahead of handoff work
        ↓
Cross-GPU dependency signals late
        ↓
Arc receives the frame late
        ↓
Moonlight incoming FPS falls
```

Reducing the queue depth applies stronger backpressure:

```text
Shallower RX software queue
        ↓
Less work queued ahead
        ↓
Frame-handoff dependencies complete sooner
        ↓
Arc capture / encode path receives frames more consistently
```

This does not directly cap game FPS. It changes how far GPU work is allowed to queue ahead of execution.

The best queue depth is workload-dependent. Values of 4 and 8 are currently being evaluated, with both providing a substantial improvement over the stock value of 32.

## Wayland Desktop

The VM uses Labwc as a lightweight Wayland compositor.

Relevant environment variables:

```text
WLR_DRM_DEVICES=/dev/dri/card0
WLR_BACKENDS=drm,libinput,headless
WLR_HEADLESS_OUTPUTS=1
```

The exact DRM card number is not guaranteed to remain stable across boots or hardware changes and must be verified.

### Headless Output

A virtual output named `HEADLESS-1` is created with the wlroots headless backend.

Custom modes are applied with `wlr-randr`, for example:

```text
wlr-randr --output HEADLESS-1 --on --custom-mode 2560x1440@120Hz --pos 0,0
```

The headless output allows the VM to operate as a streaming appliance without requiring a physical display to remain connected to the streaming GPU.

## Sunshine Capture and Encoding

Sunshine is configured to capture the Labwc headless output and use VAAPI on the Arc render node.

Example:

```text
encoder = vaapi
adapter_name = /dev/dri/renderD128
output_name = HEADLESS-1
```

The render-node number is system-specific and should be verified with:

```text
ls -l /dev/dri/by-path
```

### Capture Path

Once the Wayland session is available, Sunshine uses the wlroots screencopy protocol:

```text
Game
  ↓
RX 6700 XT rendering
  ↓
PRIME / cross-GPU presentation
  ↓
Arc A380 / Labwc / HEADLESS-1
  ↓
zwlr_screencopy_manager_v1
  ↓
Sunshine
  ↓
Intel VAAPI hardware encode
  ↓
Moonlight client
```

## Intel Arc Media Support

The Arc A380 currently uses the Intel `i915` kernel driver in the NixOS guest and Intel's `iHD` VAAPI userspace driver.

The NixOS graphics configuration includes:

```nix
hardware.graphics = {
  enable = true;
  enable32Bit = true;

  extraPackages = with pkgs; [
    intel-media-driver
  ];
};
```

Verified hardware encoders:

- H.264
- HEVC
- HEVC Main10
- AV1

## Session Startup Ordering

Sunshine can start before Labwc has created the Wayland socket. In that state, Sunshine cannot find `WAYLAND_DISPLAY` and display/encoder probing fails.

A delayed restart in Labwc autostart resolved the startup race:

```text
(sleep 8; systemctl --user restart sunshine) &
```

The important design point is not the exact delay. Sunshine must probe the display only after the compositor and Wayland session are ready.

## Related Design Documents

- [hardware-and-design.md](hardware-and-design.md) covers the physical GPU topology, motherboard revision, networking, storage, and chassis decisions.
- [performance.md](performance.md) contains the quantitative results used to evaluate the architecture.
- [streaming-findings.md](streaming-findings.md) documents the troubleshooting conclusions drawn from those tests.
