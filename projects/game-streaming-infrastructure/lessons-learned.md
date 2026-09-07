# Lessons Learned

This project produced several reusable infrastructure and troubleshooting lessons beyond the specific hardware involved.

## Follow the Entire Data Path

A component can look healthy while the overall workload still performs poorly.

The Arc A380's media engine was not saturated during the worst streaming case, so focusing only on encoder capability would have pointed at the wrong part of the system.

The useful model was the complete path:

```text
render GPU
   ↓
cross-GPU presentation
   ↓
compositor
   ↓
capture
   ↓
encoder
   ↓
network
   ↓
client decode
   ↓
client presentation
```

Each stage can fail independently.

## PCIe Topology Is Part of System Design

A second full-length PCIe slot does not automatically mean a second high-bandwidth GPU path.

The current motherboard physically accepts the Arc A380 but connects that slot through the chipset at PCIe 4.0 x2. That became important only after the system was repurposed into a cross-GPU streaming architecture.

Expansion-slot placement, lane ownership, chipset routing, and IOMMU layout should be treated as design inputs rather than details to inspect after a problem appears.

## Verify the Physical Link, Not Just the Endpoint

The Arc endpoint reported an internal x1 link that did not represent the motherboard connection.

Walking the PCIe tree and checking the upstream bridge revealed the actual PCIe 4.0 x2 external path.

When a device contains internal bridges, the complete hierarchy matters more than a single `lspci` line.

## Separate Render and Encode Carefully

Using one GPU for rendering and another for encoding solved the original problem of a fully loaded render GPU starving the encoder.

It also introduced a different constraint: frames now have to cross devices before they can be captured and encoded.

Splitting responsibilities can remove one bottleneck while creating another. The data movement between components matters as much as the components themselves.

## Stable Device Ownership Is Better Than Hot-Unbinding

Attempting to dynamically detach the Radeon GPU from `amdgpu` caused a Proxmox task to hang in uninterruptible kernel sleep.

Binding passthrough GPUs to `vfio-pci` at boot produced a cleaner and more predictable ownership model.

For a dedicated passthrough appliance, stable device ownership was more valuable than trying to make the same GPU dynamically serve both host and guest roles.

## Startup Ordering Matters in Headless Systems

Sunshine could start successfully as a user service but still fail to find a usable display if Labwc had not yet created the Wayland socket.

A simple delayed restart after compositor startup fixed the issue.

Service health is not only about whether a process is running. Dependencies may need to be functionally ready before a consumer starts probing them.

## Stable Device Names Still Have Edge Cases

A `/dev/dri/by-path` name looked attractive for stable GPU selection, but wlroots interprets colons in `WLR_DRM_DEVICES` as separators between multiple devices.

That made the PCI-address-based path unsuitable in that variable without additional indirection.

"Stable naming" is only useful if the consuming software accepts the naming format.

## Do Not Add Layers Without Evidence

Gamescope was tested because compositor and presentation behavior were possible suspects. It made no meaningful difference.

That was still a useful result because it eliminated an entire class of explanations.

Additional software layers should solve a measured problem rather than being added because they are commonly associated with gaming on Linux.

## Host, Network, Decoder, and Presentation Metrics Are Different

A Moonlight stream can have:

- healthy host rendering,
- no packet loss,
- low decode time,
- low render time,

and still look bad because final client presentation is poorly paced.

The borderless-to-fullscreen fix demonstrated why end-to-end latency and frame-pacing metrics need to be separated instead of summarized as "the stream stutters."

## Bandwidth Is Not the Same as Latency

Heavily modded Fallout 4 performed badly from NAS storage despite modest actual throughput because it generated large numbers of metadata operations.

Moving the workload to local NVMe dramatically reduced launch time.

A faster network does not automatically solve a storage workload dominated by filesystem metadata latency.

## Declarative Configuration Helped Repeated Testing

NixOS made the gaming VM easier to treat as an appliance rather than a one-off desktop installation.

GPU support, Intel media packages, Sunshine, PipeWire, Labwc, Home Manager, and other permanent configuration could be tracked in declarative files and reproduced after changes.

That reduced configuration drift while the hardware and streaming design were being changed repeatedly.

## Keep Hypotheses Separate From Conclusions

The current evidence strongly suggests that the chipset-connected PCIe 4.0 x2 Arc path is a major limitation, but the project does not yet claim that raw PCIe bandwidth explains every lost frame.

The planned CPU-direct x8/x8 retest is important because it changes the suspected bottleneck while keeping the GPUs and software stack largely constant.

Good troubleshooting is not only finding a plausible explanation. It is designing the next test so that the explanation can be confirmed or rejected.
