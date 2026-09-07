# Streaming Findings

This document records the most useful observations from testing the dual-GPU Sunshine/Moonlight architecture.

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
├── headless display
└── Sunshine VAAPI encode
      ↓
Moonlight
```

The game itself can remain smooth even when the stream delivery rate drops substantially.

## Encoder Validation

The Intel Arc A380 was straightforward to enable under NixOS.

After installing `intel-media-driver`, VAAPI successfully exposed:

- H.264 encode
- HEVC encode
- HEVC Main10 encode
- AV1 encode

Sunshine successfully detected all three primary codec families through Intel's `iHD` driver.

During poor high-resolution streaming performance, `intel_gpu_top` did **not** show the Arc media engine fully saturated. This was an important clue that the encoder itself was not necessarily the limiting resource.

## Resolution-Dependent Behavior

The most important result was the strong relationship between host display resolution and delivered stream frame rate.

Observed examples during 120 FPS gaming:

| Host virtual display | Moonlight request | Approx. incoming FPS | Notes |
|---|---|---:|---|
| 3840×2160 @ 120 | 3840×2160 @ 120 | ~32 FPS | Severe host-side delivery drop |
| 3200×1800 @ 120 | 2560×1440 @ 120 | ~84 FPS | Large improvement |
| 2560×1440 @ 120 | 2560×1440 @ 120 | ~80–100 FPS | Much better, but not consistently 120 |

The game itself remained capable of approximately 120 FPS during the lower incoming-FPS cases.

Desktop-only streaming could also run substantially faster than streaming while a game was being rendered on the Radeon GPU.

This makes the cross-GPU presentation path a stronger suspect than the game's raw render performance.

## Why PCIe Topology Matters

The current physical layout is asymmetric:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x16
Arc A380   → chipset PCIe 4.0 x2
```

PCIe 4.0 x2 provides roughly 4 GB/s of theoretical bandwidth per direction before overhead.

For scale, a simple 32-bit 4K framebuffer is approximately 33 MB:

```text
3840 × 2160 × 4 bytes ≈ 33 MB
```

If an implementation had to move an entire uncompressed 32-bit frame for every refresh, the raw data rate would be approximately:

```text
4K60  ≈ 2.0 GB/s
4K120 ≈ 4.0 GB/s
4K240 ≈ 8.0 GB/s
```

Real PRIME/dma-buf behavior is more complicated than this simplified calculation and may use different formats, copies, or synchronization paths. The calculation is therefore **not proof of literal bus traffic**.

It does, however, explain why a chipset-connected PCIe 4.0 x2 display GPU is a poor topology for experimenting with very high-resolution cross-GPU presentation.

## Planned Validation

The next major test is to move both GPUs onto CPU-direct lanes:

```text
RX 6700 XT → CPU PCIe 4.0 x8
Arc A380   → CPU PCIe 4.0 x8
```

PCIe 4.0 x8 provides roughly four times the lane bandwidth of the current x2 Arc connection.

The same Sunshine/Moonlight tests will then be repeated at:

- 1440p120
- 1800p120
- 4K120
- 4K240

If delivered frame rate improves dramatically with the same GPUs and software stack, that will provide much stronger evidence that the current limitation is inter-GPU topology rather than Arc encoding capability.

## Client-Side Frame Pacing Lesson

A separate issue was discovered on one Moonlight client.

The stream could arrive correctly while still feeling visibly choppy. The client showed approximately 15–20 ms of average frame queue delay during a 120 FPS stream.

Changing Moonlight from borderless/windowed presentation to true fullscreen made the stream feel smooth immediately.

This is an important troubleshooting lesson:

> A smooth host and healthy decoder do not guarantee smooth presentation on the client.

When diagnosing Moonlight performance, distinguish between:

1. host processing latency,
2. network loss/latency,
3. decode time,
4. render time,
5. frame queue delay / presentation pacing.

They can fail independently.

## Network Observations

The current host uses 2.5 GbE rather than the previous 10GbE adapter.

The stream itself is nowhere near saturating 2.5 GbE at typical Sunshine bitrates, so raw link capacity is not the primary concern for Moonlight traffic.

10GbE remains useful for the NAS-backed game library rather than being required by the video stream itself.

## Storage Lesson: Bandwidth Is Not Everything

Fallout 4 provided another useful performance lesson unrelated to GPU streaming.

A heavily modded installation performed poorly from NAS storage despite low actual data throughput because the workload generated very large numbers of filesystem metadata operations.

Moving the game and mod environment to local storage dramatically reduced launch time.

The resulting storage policy is:

- NAS by default.
- Local storage for metadata-heavy or latency-sensitive workloads.

This is a good reminder that a faster network does not automatically fix a workload dominated by filesystem latency.

## Current Conclusions

### Confirmed

- RX 6700 XT game rendering works correctly under passthrough.
- Arc A380 passthrough works correctly.
- NixOS + Intel `iHD` provides working H.264, HEVC, and AV1 hardware encoding.
- Sunshine can capture the Labwc headless output through Wayland screencopy.
- The Arc media engine is not necessarily saturated when high-resolution streaming performance collapses.
- Stream delivery improves substantially as host resolution decreases.
- Client presentation mode can independently create visible stutter even when decode performance is healthy.

### Strong Working Hypothesis

The current PCIe 4.0 x2 chipset path to the Arc A380 is a major limitation for high-resolution, high-refresh cross-GPU presentation and capture.

### Not Yet Proven

- That raw PCIe bandwidth alone explains all lost frames.
- That CPU-direct x8/x8 will sustain 4K240.
- That the Arc A380 media engine itself can encode the desired 4K240 workload under the final architecture.

Those are the next experiments.
