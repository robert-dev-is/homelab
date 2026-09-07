# Architecture

## Overview

This project uses a split-GPU architecture inside a NixOS virtual machine running on Proxmox VE.

The Radeon GPU is dedicated to game rendering. The Intel Arc GPU owns the Wayland desktop, headless output, Sunshine capture path, and hardware encoder.

The intent is to prevent game rendering load from starving the stream encoder while keeping the VM fully remote-accessible.

## Host / Guest Layout

```text
Physical Host
┌──────────────────────────────────────────────┐
│ Proxmox VE                                   │
│                                              │
│  Ryzen 9 7950X3D                             │
│      │                                       │
│      ├── RX 6700 XT ───────────────┐         │
│      │                              │ VFIO    │
│      └── Intel Arc A380 ────────────┤         │
│                                     ▼         │
│                          NixOS Gaming VM      │
└──────────────────────────────────────────────┘
```

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

## CPU Topology

The VM uses 16 vCPUs pinned to the 7950X3D V-Cache CCD and its SMT siblings.

Example Proxmox CPU configuration:

```text
sockets: 1
cores: 16
cpu: host
affinity: 0-7,16-23
```

This keeps the gaming VM on the cache-equipped CCD rather than allowing the scheduler to move gaming threads between CCDs.

## GPU Passthrough

Both discrete GPUs are bound to `vfio-pci` on the host before VM startup.

This is preferable to dynamically detaching a GPU from its native host driver at VM launch.

A previous attempt to hot-unbind the Radeon GPU from `amdgpu` caused the host task to enter uninterruptible kernel sleep during device removal. Binding the passthrough devices to VFIO at boot removed that failure path.

Generalized VFIO configuration:

```text
options vfio-pci ids=<gpu1>,<gpu1-audio>,<gpu2>,<gpu2-audio>
```

Device IDs should be determined from the actual hardware rather than copied from this project.

## Guest GPU Selection

The Arc A380 owns the desktop, while games are launched on the Radeon GPU with Mesa PRIME selection.

Example:

```text
DRI_PRIME=pci-0000_01_00_0 %command%
```

This is intentionally applied to games rather than the entire Steam process. Forcing the whole Steam UI onto the render GPU caused instability in Chromium/CEF-based Steam components during testing.

## Wayland Desktop

The VM uses Labwc as a lightweight Wayland compositor.

Relevant environment variables:

```text
WLR_DRM_DEVICES=/dev/dri/card0
WLR_BACKENDS=drm,libinput,headless
WLR_HEADLESS_OUTPUTS=1
```

The exact DRM card number is not guaranteed to remain stable across hardware changes or boots and must be verified.

### Headless Output

A virtual output named `HEADLESS-1` is created with the wlroots headless backend.

Custom modes are applied with `wlr-randr`, for example:

```text
wlr-randr --output HEADLESS-1 --on --custom-mode 2560x1440@120Hz --pos 0,0
```

Testing has included 1440p120, 1800p120, 4K120, and 4K240 virtual modes.

## Sunshine Capture and Encoding

Sunshine is configured to use VAAPI on the Arc render node.

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

Once the Wayland session is available, Sunshine detects the wlroots screencopy protocol and uses native Wayland capture:

```text
Labwc HEADLESS-1
      ↓
zwlr_screencopy_manager_v1
      ↓
Sunshine
```

### Intel Arc Media Support

The Arc A380 uses the Intel `i915` kernel driver in the current NixOS guest and Intel's `iHD` VAAPI userspace driver.

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

Sunshine can start before Labwc has created the Wayland socket. In that state, Sunshine cannot find `WAYLAND_DISPLAY` and encoder/display probing fails.

A simple delayed restart in the Labwc autostart resolved the timing issue:

```text
(sleep 8; systemctl --user restart sunshine) &
```

The important point is not the exact delay but ensuring Sunshine probes the display after the compositor has initialized.

## Current PCIe Topology

The current motherboard exposes the second full-length slot through the chipset:

```text
Ryzen CPU
│
├── CPU PCIe 4.0 x16 → RX 6700 XT
│
└── B650 chipset
    └── PCIe 4.0 x2 → Arc A380
```

The Arc A380 itself supports a wider link, but the slot is electrically limited to x2.

The physical link was verified at the Arc's upstream bridge as PCIe 4.0 x2.

This is important because the gaming workload is cross-GPU:

```text
RX renders frame
      ↓
frame must become visible to Arc-owned compositor
      ↓
Arc / Labwc
      ↓
Sunshine capture + encode
```

At high resolution and refresh rate, this path becomes sensitive to transfer bandwidth and synchronization overhead.

## Planned Topology

The planned platform revision uses two CPU-connected slots operating at x8/x8:

```text
Ryzen CPU
│
├── PCIe 4.0 x8 → RX 6700 XT
└── PCIe 4.0 x8 → Arc A380
```

The goal is to remove the chipset from the inter-GPU path and provide substantially more bandwidth to the display/encoding GPU.

## Networking

The current host uses 2.5 GbE.

10 GbE remains desirable primarily because game storage is NAS-backed. Streaming itself does not require 10 GbE; the higher-speed network is useful for game-file access, installations, updates, and large local/NAS transfers.

A future motherboard with onboard 10GbE would also avoid consuming another PCIe slot.
