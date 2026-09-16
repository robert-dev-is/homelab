# Lessons Learned

This project produced several reusable infrastructure and troubleshooting lessons beyond the specific hardware involved.

The most valuable lessons came from cases where an initially reasonable explanation turned out to be incomplete or wrong after deeper testing.

## Follow the Entire Data Path

A component can look healthy while the overall workload still performs poorly.

The useful model for this system was the complete path:

```text
game
 ↓
render GPU
 ↓
GPU scheduler
 ↓
cross-GPU presentation / synchronization
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
client render
 ↓
final presentation
```

Each stage can fail independently.

The Arc A380's encoder retained substantial unused capacity during some of the worst streams. The network showed no meaningful packet loss. The client could decode frames quickly.

None of those facts guaranteed that frames were reaching those stages on time.

Troubleshooting became much more productive once the complete pipeline was treated as a sequence of dependencies rather than as a collection of individual components.

## Utilization Does Not Explain Latency

A resource does not need to be saturated for the system to experience severe latency.

The most important scheduler trace showed stream-critical RX graphics work waiting roughly 40 ms before execution even though the actual work itself took only a fraction of a millisecond.

```text
~40 ms waiting
     ↓
~0.15 ms executing
```

Looking only at GPU utilization would not have revealed that distinction.

For latency-sensitive systems, queue time, dependency wait time, frame pacing, and scheduling behavior can be more useful than utilization percentages alone.

## Queue Depth Can Be a Performance Variable

The AMDGPU software queue depth became one of the most important tuning variables in the project.

The stock setting:

```text
amdgpu.sched_jobs=32
```

allowed substantially more work to accumulate ahead of stream-critical dependencies.

Reducing the value to `8` or `4` applied backpressure sooner and significantly improved frame delivery.

The lesson is broader than this specific kernel parameter:

**More queued work can improve throughput-oriented workloads while making latency-sensitive workloads worse.**

A deep queue is not automatically better simply because the hardware can accept more outstanding work.

## High FPS Does Not Guarantee Smooth Motion

Average FPS is only one performance metric.

A game can report a high average frame rate while producing inconsistent frame intervals.

Testing with the stock AMDGPU queue depth showed noticeably more erratic frametime behavior. Reducing the queue depth produced much smoother frame pacing in several workloads.

This reinforced the distinction between:

```text
frames per second
```

and:

```text
how consistently those frames arrive
```

A lower but evenly paced frame rate can look substantially smoother than a higher average frame rate with unstable frametimes.

For this project, game FPS, frametime behavior, Moonlight incoming FPS, and subjective smoothness all proved useful.

## Separate Render and Encode Carefully

Using one GPU for rendering and another for display, capture, and encoding solved one architectural problem while creating another.

The split prevents the encoder from directly competing with a fully loaded game-rendering GPU:

```text
RX 6700 XT
└── game rendering

Arc A380
├── compositor
├── virtual display
├── capture
└── encode
```

However, completed frames and synchronization dependencies now have to cross GPU boundaries before encoding can begin.

Splitting responsibilities can eliminate resource contention at one stage while introducing synchronization and data-movement requirements elsewhere.

Component isolation is useful, but the interfaces between components become part of the performance design.

## Hardware Topology Is a Hypothesis, Not a Conclusion

The original B650 design placed the GPUs on very different PCIe paths:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x16
Arc A380   → chipset PCIe 4.0 x2
```

High-resolution streaming became substantially worse as workload increased, making the narrow chipset-connected Arc path a reasonable suspect.

The motherboard was replaced with an ASRock X870 Taichi Creator:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x8
Arc A380   → CPU-direct PCIe 4.0 x8
```

The new topology removed the suspected limitation.

The streaming problem remained.

That result was extremely useful because it rejected the original explanation and redirected the investigation toward GPU scheduling and synchronization.

A hardware change can be valuable even when it does not fix the problem if it cleanly eliminates a major variable.

## PCIe Topology Still Matters

Rejecting PCIe bandwidth as the primary cause does not make PCIe topology irrelevant.

A second full-length slot does not automatically mean a second high-bandwidth, CPU-connected path.

Lane ownership, chipset routing, bifurcation, IOMMU grouping, slot spacing, and onboard devices all influence what a multi-GPU system can actually do.

The X870 x8/x8 layout remains a substantially cleaner architecture even though it did not solve the scheduler problem.

Hardware design and root-cause diagnosis are related, but they are not the same thing.

## Verify the Entire Physical Device Path

The Arc endpoint initially reported an internal PCIe link that did not accurately describe the motherboard-to-card connection.

Walking the PCIe hierarchy and inspecting the upstream bridge exposed the actual external link.

When a device contains internal PCIe bridges, a single `lspci` line may not describe the physical connection that matters.

The complete hierarchy should be checked before drawing conclusions about negotiated link width or routing.

## Render Headroom Can Matter Even When Game FPS Looks Healthy

Red Dead Redemption 2 demonstrated that a game can remain fast while the streaming path lacks enough scheduling headroom.

At a heavy 1800p resolution workload, the game could remain around 70 FPS while Moonlight incoming FPS fell to roughly 35 FPS.

Reducing the workload with VSync and lower resolution progressively improved stream delivery.

This showed that:

```text
healthy game FPS
        ≠
available scheduling headroom
```

A GPU can still be producing frames quickly while latency-sensitive work is being serviced too late.

## Tune for the Delivered Result, Not the Largest Number

4K240 sounds better than 4K144 on paper.

In practice, repeated 4K240 testing produced worse incoming frame delivery.

4K144 currently provides a better balance between responsiveness and stream consistency.

This reinforced a general engineering lesson: the largest configurable number is not necessarily the best operating point.

The useful target is the configuration that produces the best end-to-end behavior.

## Stable Device Ownership Is Better Than Hot-Unbinding

Attempting to dynamically detach the RX 6700 XT from `amdgpu` caused the Proxmox task to hang in uninterruptible kernel sleep.

Binding dedicated passthrough GPUs to `vfio-pci` at host boot produced a cleaner and more predictable ownership model.

For a machine intended to behave as a dedicated passthrough appliance, stable hardware ownership was more valuable than dynamically switching a GPU between host and guest roles.

## Startup Ordering Matters in Headless Systems

Sunshine could be running successfully as a process while still being unable to stream because Labwc had not yet created the Wayland session it depended on.

Restarting Sunshine after the compositor became available solved the problem.

Service state and service readiness are different concepts.

A dependency may need to be functionally ready before a consumer starts probing it.

## Stable Device Names Can Still Be Incompatible

A `/dev/dri/by-path` device name appeared useful for stable GPU selection.

However, `WLR_DRM_DEVICES` treats colons as separators between multiple devices, while PCI-address-based paths themselves contain colons.

The naming method was stable, but the consuming software could not use the format directly.

A stable identifier is only useful if every layer that consumes it understands its syntax.

## Do Not Add Layers Without Evidence

Gamescope was tested because compositor and presentation behavior were possible causes of the streaming issue.

It did not materially improve the problem, matter of fact it made no difference.

That was still a useful result because it eliminated another potential explanation.

Additional layers should solve a measured problem rather than being introduced simply because they are commonly associated with a particular workload or platform.

## Failed Experiments Are Useful When They Remove Variables

Several tests produced no improvement:

```text
SDMA scheduler-mask changes
VKD3D swapchain latency tuning
VKD3D staggered-submit changes
moving Labwc rendering onto the RX
moving the entire compositor onto the RX
Gamescope
```

Those tests were not wasted effort.

Each removed another plausible explanation and narrowed the investigation until RX scheduler queue latency became visible as the strongest measured cause.

An experiment does not need to fix the system to provide useful information.

## Client Presentation Can Fail Independently

One Moonlight client initially appeared to have a host-side streaming problem.

Decode and render times were healthy, but borderless/windowed presentation produced approximately 15-20 ms of frame-queue delay and visible choppiness.

Switching Moonlight to true fullscreen immediately made presentation smooth.

This demonstrated why host processing, network transport, client decoding, client rendering, and final frame presentation should be measured separately.

"The stream stutters" is a symptom, not a diagnosis.

## Bandwidth Is Not the Same as Latency

Fallout 4 performed extremely poorly from NAS storage despite relatively low network throughput.

The workload generated large amounts of filesystem metadata activity rather than sustained sequential bandwidth.

Moving the game and mod environment to local NVMe dramatically reduced launch time.

The Witcher 3 demonstrated an even more selective storage strategy: the main game files can remain on the NAS while save-game files in '/mnt/local-games/SteamLibrary/steamapps/compatdata/292030/pfx/drive_c/users/steamuser/Documents/The Witcher 3' are kept locally.

Storage placement does not need to be all-or-nothing.

Large, mostly static files may work well remotely while small latency-sensitive files from the same application perform better locally.

## Faster Networking Does Not Fix Every Storage Problem

The gaming node now has 10GbE connectivity, which is useful for large game installs, updates, NAS-backed libraries, and bulk transfers.

However, additional bandwidth does not eliminate filesystem metadata latency.

A workload dominated by many small operations can remain slow even when almost none of the available network bandwidth is being consumed.

Throughput and operation latency need to be evaluated separately.

## Declarative Configuration Makes Repeated Testing Easier

NixOS made the gaming VM easier to treat as an appliance rather than as a manually configured desktop.

GPU support, Intel media packages, Sunshine, PipeWire, Labwc, Home Manager, kernel parameters, and other permanent configuration could be tracked declaratively.

That made repeated experiments safer because changes could be reproduced, reverted, and documented without relying on memory of one-off commands.

Reducing configuration drift became increasingly valuable as both hardware and software variables changed.

## CPU Affinity Is Placement, Not Reservation

The gaming VM is constrained to the 3D V-Cache CCD of the Ryzen 9 7950X3D.

That improves placement predictability, but affinity alone does not reserve those host CPUs exclusively for the VM.

Other workloads can still execute on those processors unless they are separately constrained.

CPU affinity should therefore be understood as:

```text
where this workload may run
```

rather than:

```text
these CPUs now belong only to this workload
```

That distinction matters when interpreting host-wide monitoring and when placing additional workloads.

Therefore future vms or lxc containers should have affinity for the non 3D V-Cache CCD

## Keep Hypotheses Separate From Conclusions

The PCIe investigation became one of the best examples of why this matters.

The evidence initially supported a reasonable hypothesis:

```text
high resolution makes streaming worse
        +
Arc is on PCIe 4.0 x2 through the chipset
        ↓
PCIe topology may be the bottleneck
```

Instead of promoting that explanation directly into a conclusion, the architecture was changed specifically to test it.

```text
replace motherboard
        ↓
CPU-direct x8/x8
        ↓
repeat workload
        ↓
problem remains
```

The hypothesis was rejected.

Later scheduler tracing produced much stronger evidence for a different explanation.

Good troubleshooting is not about finding the first explanation that fits the symptoms.

It is about designing tests that can prove that explanation wrong.

## Main Takeaway

The most important lesson from the project is that end-to-end performance problems are often caused by interactions between otherwise healthy components.

The game could be fast.

The render GPU could be working correctly.

The encoder could have available capacity.

The network could be clean.

The client could decode quickly.

And the stream could still perform badly because a tiny piece of GPU work spent tens of milliseconds waiting in a scheduler queue.

The eventual improvement came from measuring the pipeline deeply enough to distinguish:

```text
execution time
from
queue time

throughput
from
latency

average FPS
from
frame pacing

component health
from
end-to-end behavior

hypothesis
from
conclusion
```

That troubleshooting process became more valuable than any single hardware or kernel setting discovered along the way.