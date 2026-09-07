# Performance

## Scope

This document records quantitative observations from one game-streaming system. The results are useful for comparing changes within this project, but they are not intended as universal benchmark claims.

The current test path is:

```text
RX 6700 XT rendering
        ↓
PRIME / cross-GPU presentation
        ↓
Arc A380 / Labwc
        ↓
Wayland screencopy
        ↓
Sunshine VAAPI encode
        ↓
Moonlight client
```

## High-Resolution Streaming Results

The strongest pattern so far is that delivered stream frame rate falls as the host virtual display resolution increases.

Observed examples during 120 FPS-oriented testing:

| Host virtual display | Moonlight request | Approx. incoming FPS | Result |
|---|---|---:|---|
| 3840×2160 @ 120 | 3840×2160 @ 120 | ~32 FPS | Severe host-side delivery drop |
| 3200×1800 @ 120 | 2560×1440 @ 120 | ~84 FPS | Large improvement |
| 2560×1440 @ 120 | 2560×1440 @ 120 | ~80–100 FPS | Much better, but not consistently 120 |

In the 4K120 failure case, Moonlight reported approximately:

- ~32 incoming FPS,
- 0 network drops,
- 0 jitter drops,
- ~1 ms network latency,
- ~0.64 ms decode time,
- ~0.56 ms render time,
- ~34.6 ms average host processing time.

The network and client decoder were therefore not the primary bottlenecks in that test.

## Intel Arc Encoder Load

During a poor high-resolution stream, `intel_gpu_top` showed the Arc A380 working but not fully saturated.

Observed utilization was approximately:

```text
Render/3D     ~56%
Blitter       ~36%
Video         ~43%
VideoEnhance    0%
```

Sunshine also successfully detected H.264, HEVC, and AV1 hardware encoders through Intel's `iHD` VAAPI driver.

These results make raw encoder saturation a weaker explanation for the current 4K120 delivery collapse.

## PCIe Bandwidth Context

The current physical topology is:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x16
Arc A380   → chipset PCIe 4.0 x2
```

PCIe 4.0 x2 provides roughly 3.94 GB/s of theoretical bandwidth per direction before protocol overhead.

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

Real PRIME/dma-buf behavior is more complicated than this simplified calculation and may use different formats, copies, synchronization paths, or device-local operations. These figures are therefore context, not proof of literal bus traffic.

They do show why PCIe 4.0 x2 is an unfavorable topology for very high-resolution cross-GPU presentation.

## Game Workloads

### Fallout 4

Fallout 4 can run smoothly at 1440p120 without fully loading the RX 6700 XT, making it a useful baseline workload for separating game rendering limits from streaming-path limits.

### The Witcher 3

The Witcher 3 is a much heavier rendering workload in the current configuration.

At 1440p with a custom high/Ultra/Ultra+ configuration and ray tracing disabled, the RX 6700 XT can reach approximately 80–100 FPS while operating at or near 99–100% GPU utilization.

Because the render GPU is fully loaded, The Witcher 3 is a useful stress workload for testing whether cross-GPU transfer and presentation remain smooth when there is little spare GPU scheduling headroom.

The current x2 topology can still produce visible stream stutter at 1440p in this workload, so the same scenario will be repeated after the x8/x8 motherboard revision.

## Client-Side Presentation

One Moonlight client initially felt visibly choppy at 120 FPS despite low decode and render times.

The client showed approximately 15–20 ms of average frame queue delay in borderless/windowed mode. Switching Moonlight to true fullscreen immediately made the stream feel smooth.

This demonstrated that client presentation can create a separate frame-pacing problem even when host delivery, network latency, and hardware decode are otherwise healthy.

## Network Capacity

The gaming host currently uses 2.5GbE.

A high-bitrate Moonlight stream remains far below the raw capacity of 2.5GbE, and the problematic tests showed no meaningful packet loss. The planned return to 10GbE is primarily for NAS-backed game storage rather than video-stream bandwidth.

## Planned x8/x8 Validation

The ordered motherboard revision will move both GPUs to CPU-connected PCIe 4.0 x8 links:

```text
RX 6700 XT → CPU PCIe 4.0 x8
Arc A380   → CPU PCIe 4.0 x8
```

The retest sequence will keep the software stack and GPUs unchanged and repeat the same workloads at:

- 1440p120,
- 1800p120,
- 4K120,
- 4K240.

The Witcher 3 will also be retained as a high-load game test because it can fully saturate the render GPU.

A large improvement with the same software and GPUs would provide much stronger evidence that the current limitation is inter-GPU topology. If significant stutter remains, the next investigation will focus on GPU scheduling, synchronization, capture, and encoder behavior rather than assuming PCIe bandwidth explains everything.
