# Streaming Findings

This document records the most useful observations from testing and troubleshooting the dual-GPU Sunshine/Moonlight architecture.

These are **experimental observations from one system**, not universal performance claims.

## Test Architecture

```text
RX 6700 XT
└── game rendering
      ↓
PRIME / cross-GPU presentation
      ↓
Arc A380
├── Labwc
├── HEADLESS-1 virtual display
├── Sunshine capture
└── VAAPI hardware encoding
      ↓
Moonlight client
```

The unusual part of this design is that the GPU rendering the game is not the GPU owning the desktop, virtual display, capture path, or encoder.

That separation works, but it makes cross-GPU frame delivery and synchronization important parts of the streaming pipeline.

A recurring observation throughout testing has been:

```text
Game remains smooth
        ≠
Stream receives frames smoothly
```

The RX 6700 XT can continue rendering a game at a healthy frame rate while Moonlight incoming FPS drops substantially.

## Baseline High-Load Failure

Before scheduler tuning, the failure appeared most clearly when the RX 6700 XT was heavily loaded.

Typical behavior included:

- the game continuing to render normally,
- Moonlight incoming FPS falling well below game FPS,
- Sunshine host processing time increasing substantially,
- no corresponding network packet loss,
- low client decode and render latency,
- substantial unused Intel Arc media-engine capacity.

The Witcher 3 provided one particularly useful example. The game could remain around 60 FPS while Moonlight incoming FPS fell into roughly the 20-30 FPS range.

This established that game FPS alone was not a useful indicator of whether the streaming path was healthy.

## Encoder Validation

The Intel Arc A380 was straightforward to enable under NixOS.

After installing `intel-media-driver`, VAAPI exposed:

- H.264 encode
- HEVC encode
- HEVC Main10 encode
- AV1 encode

Sunshine successfully uses Intel's `iHD` VAAPI driver.

Early `intel_gpu_top` observations already showed that the media engine was not fully saturated during poor streams.

Later measurement made this clearer: during a 10-second bad-case sample, the Intel VIDEO engine accumulated approximately 2.94 seconds of execution time.

That is substantial activity, but it is far from continuous saturation of one video engine.

The current evidence therefore places the primary delay **before the encoder receives frames**, rather than inside the hardware encoding stage itself.

## The PCIe Topology Hypothesis

The original B650 platform had an asymmetric GPU topology:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x16
Arc A380   → chipset PCIe 4.0 x2
```

Because stream performance became dramatically worse at high resolutions, this narrow chipset-connected path was initially the strongest architectural suspect.

A simple 32-bit 4K framebuffer is approximately 33 MB:

```text
3840 x 2160 x 4 bytes ≈ 33 MB
```

For scale, transferring one equivalent raw framebuffer for every frame would represent approximately:

```text
4K60  ≈ 2.0 GB/s
4K120 ≈ 4.0 GB/s
4K240 ≈ 8.0 GB/s
```

Real PRIME/dma-buf behavior is more complicated than this simplified calculation and does not imply that every frame literally crosses PCIe as one uncompressed 32-bit copy.

The calculation nevertheless made the PCIe 4.0 x2 topology worth testing.

## x8/x8 Platform Validation

The motherboard was replaced with an ASRock X870 Taichi Creator, giving both GPUs CPU-direct links:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x8
Arc A380   → CPU-direct PCIe 4.0 x8
```

This was an important controlled architectural test.

The narrow chipset path was eliminated, but the high-load streaming problem remained.

The motherboard change therefore disproved the original hypothesis that the old Arc PCIe path was the primary cause of the frame-delivery collapse.

The x8/x8 topology remains preferable for this architecture, but PCIe bandwidth alone does not explain the observed behavior.

## RX Scheduler Trace Findings

Low-level tracing eventually narrowed the major delay to the AMDGPU graphics scheduler and the synchronization chain leading into the cross-GPU handoff.

A representative bad case looked approximately like this:

```text
RX GFX job queued
        ↓
~39.9 ms waiting before execution
        ↓
~0.15 ms actual GFX execution
        ↓
RX dependency signals
        ↓
SDMA handoff begins ~19 µs later
        ↓
~2.3 ms SDMA execution
        ↓
Arc / Labwc dependency can continue
        ↓
Sunshine capture / encode path proceeds
```

The important result is the difference between **queue wait time** and **execution time**.

The stream-critical RX graphics work was not computationally expensive. Once the scheduler allowed it to run, it completed in only a fraction of a millisecond.

The majority of the observed delay came from waiting behind work that was already queued on the RX.

Additional tracing showed that:

- Labwc / i915 external-fence waits could be associated with AMDGPU `drm_sched` SDMA fences.
- Some SDMA handoff work depended on both Arc-side and RX-side fences.
- SDMA execution itself was fast once its dependencies were satisfied.
- Moving handoff work between SDMA engines did not materially improve the stream.
- Conventional CPU `dma_fence_wait()` calls observed during tracing were only microseconds long and did not explain the tens-of-milliseconds delay.
- Sunshine successfully creates HIGH-priority EGL contexts, ruling out the previously suspected `CAP_SYS_NICE` / EGL-priority failure mode.

The strongest measured delay was therefore RX-side scheduling latency before the frame could continue through the cross-GPU path.

## AMDGPU Software Queue Depth

The default AMDGPU software queue setting is:

```text
amdgpu.sched_jobs=32
```

For this workload, the stock value allows game work to build substantial queue-ahead.

The game is not literally cutting in front of already queued stream work. Instead, it can continuously fill the available queue deeply enough that work required for frame handoff spends too long waiting before it reaches execution.

Conceptually:

```text
sched_jobs=32
        ↓
deep RX software queue
        ↓
substantial game work queued ahead
        ↓
stream-critical dependency waits
        ↓
cross-GPU handoff occurs late
        ↓
Moonlight incoming FPS collapses
```

Reducing the software queue depth applies backpressure sooner:

```text
sched_jobs=8 or 4
        ↓
less queue-ahead
        ↓
handoff dependency reaches execution sooner
        ↓
Arc receives frames more consistently
        ↓
Moonlight delivery improves
```

Lowering `sched_jobs` does **not** directly cap game FPS.

It limits how much work can remain outstanding in the AMDGPU software queue before earlier work must complete.

## Scheduler Tuning Results

The useful values tested so far are:

```text
amdgpu.sched_jobs=8
amdgpu.sched_jobs=4
```

Both are major improvements over the stock value of `32`.

`8` has proven to be a strong general-purpose improvement across tested games.

`4` further improves some workloads, particularly The Witcher 3 and Fallout 4, but the best setting is not universal.

Representative observations include:

| Workload | Observed behavior |
|---|---|
| Witcher 3, `sched_jobs=8` | ~45-50 Moonlight incoming FPS with much smoother delivery than stock |
| Witcher 3, `sched_jobs=4` | ~65-70 incoming FPS and noticeably smoother frame pacing |
| Fallout 4, `sched_jobs=8` | Incoming FPS generally in the 50s while game FPS can reach ~80-100 |
| Fallout 4, `sched_jobs=4` | ~60-70 incoming FPS while the game can remain near ~100 FPS |
| RDR2, heavy 1800p load | Remains more sensitive to RX saturation and can behave better with additional render headroom |

The important result is not simply that one numeric queue depth is "best."

The important result is that the stock queue depth of `32` is poorly suited to this latency-sensitive split-GPU workload.

## Frame Pacing Matters

Scheduler tuning also produced a visible improvement in game frametime behavior.

With `sched_jobs=32`, the frametime graph could oscillate rapidly even when average FPS remained high.

Reducing the queue depth to `8` or `4` produced much smoother frametime behavior in tested workloads.

This reinforces an important performance lesson:

```text
High average FPS
        ≠
consistent frame delivery
```

A game running at a high average frame rate can still look or feel poor if individual frames arrive at inconsistent intervals.

For this project, average game FPS, game frametime behavior, Moonlight incoming FPS, and subjective smoothness all need to be considered together.

## Render-GPU Headroom Still Matters

Reducing software queue depth substantially improves the problem, but it does not make RX load irrelevant.

Red Dead Redemption 2 demonstrates this particularly well.

At approximately 1800p, the game can render around 70 FPS while Moonlight incoming FPS falls to roughly 35 FPS.

Diagnostic reductions in RX workload produced progressively better stream delivery:

```text
1800p / ~70 FPS game load
→ ~35 incoming FPS

1800p / VSync set to 60 FPS
→ ~43-50 incoming FPS

1620p / VSync set to 60 FPS
→ ~60 incoming FPS
```

These caps and resolution changes were diagnostic tests, not the desired permanent solution.

They demonstrated that the streaming pipeline is capable of delivering a full 60 FPS when the RX has enough scheduling headroom.

The remaining issue is therefore not a fixed stream-FPS ceiling. Extremely heavy rendering workloads can still consume enough RX resources to delay the cross-GPU handoff even with a much shallower software queue.

## Moonlight Refresh-Rate Findings

4K240 is functional, but repeated testing has shown lower incoming frame delivery when Moonlight is configured for 240 Hz.

The current practical target is:

```text
3840x2160 @ 144 Hz
```

4K144 still provides high-refresh desktop and gaming behavior while placing less pressure on the frame-delivery pipeline.

This should not be interpreted as 144 Hz solving the underlying scheduling problem. Rather, it is currently the better operating point for the system.

4K240 remains useful as a stress test.

## Rejected or Ineffective Tests

Several changes were tested but did not solve the host-side frame-delivery problem:

- `amdgpu.sched_hw_submission=1`
  - values below `2` are clamped back to `2`
  - this controls hardware submissions rather than the software queue depth that was backing up

- SDMA scheduler-mask changes
  - moving handoff work between SDMA engines did not materially improve streaming

- `VKD3D_SWAPCHAIN_LATENCY_FRAMES=2`
  - no useful improvement

- `VKD3D_CONFIG=no_staggered_submit`
  - no useful improvement

- Gamescope
  - did not materially improve the streaming problem

- Labwc renderer forced onto the RX while keeping the Arc DRM backend
  - desktop streaming fell to roughly 7 FPS

- Entire Labwc compositor/display moved to the RX while Sunshine continued encoding on the Arc
  - desktop streaming initially worked
  - incoming FPS fell to roughly 5 FPS once a game heavily loaded the RX

The preferred topology therefore remains:

```text
RX 6700 XT
└── game rendering

Arc A380
├── Labwc compositor
├── HEADLESS-1 virtual display
├── Sunshine capture
└── VAAPI hardware encoding
```

## Client-Side Frame Pacing Can Fail Independently

A separate Moonlight client issue initially looked like poor host streaming performance.

The stream arrived with healthy decode and render times, but borderless/windowed presentation produced approximately 15–20 ms of average frame queue delay and visible choppiness.

Switching Moonlight to true fullscreen made the stream smooth immediately.

This established that the following stages should be treated independently when troubleshooting:

1. game rendering and frame pacing,
2. host frame delivery,
3. network transport,
4. client decoding,
5. client rendering,
6. client frame queue and final presentation.

A problem at one stage can look very similar to a problem at another.

## Network Observations

The gaming node now uses onboard 10GbE.

The Sunshine stream itself requires nowhere near 10GbE capacity, and problematic tests have consistently shown low network latency with no meaningful packet loss.

The network is therefore not considered the cause of the high-load frame-delivery collapse.

10GbE remains valuable for NAS-backed game storage, installations, updates, and bulk transfers.

## Storage Lesson: Workload Placement Matters

Storage performance produced a separate but related lesson: high bandwidth alone does not guarantee good application behavior.

Fallout 4 generated large amounts of filesystem metadata activity while using relatively little sequential bandwidth. Moving the game and mod environment from NFS storage to local NVMe dramatically reduced launch time.

The Witcher 3 uses a more selective approach. Its main game installation remains on NAS storage, while the save-game files normally stored under the Windows `Documents` path are kept on local storage.

The resulting storage policy is intentionally workload-specific:

```text
NAS
└── large game installations that behave well remotely

Local NVMe
├── metadata-heavy game installations
├── mod environments
└── latency-sensitive save / application data
```

## Current Conclusions

### Confirmed

- The RX 6700 XT works correctly as the dedicated game-rendering GPU under passthrough.
- The Arc A380 works correctly as the Labwc display, capture, and VAAPI encoding GPU.
- NixOS + Intel `iHD` provides working H.264, HEVC, and AV1 hardware encoding.
- Sunshine successfully captures the Labwc headless Wayland output.
- The Arc media engine retains substantial headroom during known bad streaming cases.
- Network loss and bandwidth are not responsible for the observed host-side collapse.
- Replacing the B650 x16/x2 topology with CPU-direct x8/x8 did not eliminate the problem.
- RX-side scheduler wait time can dominate actual execution time for work required by the cross-GPU handoff.
- Reducing `amdgpu.sched_jobs` from `32` to `8` or `4` substantially improves stream delivery and frame pacing.
- The best scheduler queue depth varies somewhat by game workload.
- Heavy RX utilization can still reduce stream delivery even with a shallow software queue.
- Client presentation mode can independently create visible stutter.
- 4K144 currently provides a better practical operating point than 4K240.

### Current Interpretation

The primary high-load limitation is associated with **RX-side scheduling and synchronization latency before frames reach the Arc-owned capture and encoding path**.

The current model is:

```text
heavy game workload
        ↓
RX software queue accumulates work
        ↓
frame-handoff dependency waits too long
        ↓
cross-GPU presentation occurs late
        ↓
Sunshine receives frames late
        ↓
Moonlight incoming FPS falls
```

Reducing AMDGPU software queue depth substantially shortens this delay.

The remaining work is to determine how far queue latency can be reduced across different games without unnecessarily restricting render throughput, and how much RX headroom is required for consistently high-rate streaming under the heaviest workloads.