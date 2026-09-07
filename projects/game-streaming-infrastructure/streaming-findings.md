# Streaming Findings

This document records the most useful troubleshooting findings from the dual-GPU Sunshine/Moonlight architecture.

For the quantitative results behind these conclusions, see [performance.md](performance.md).

## 1. The Encoder Was Not the First Bottleneck

The Intel Arc A380 was straightforward to enable under NixOS. After installing `intel-media-driver`, VAAPI exposed H.264, HEVC, HEVC Main10, and AV1 hardware encoding, and Sunshine detected the expected codec families through Intel's `iHD` driver.

During poor high-resolution streaming, the Arc media engine was active but not fully saturated.

That changed the investigation from "can the A380 encode this workload?" to "can frames reach the Arc-owned capture path quickly enough?"

## 2. Stream Delivery Was Strongly Resolution-Dependent

The game itself could remain smooth while Sunshine's delivered frame rate fell sharply as the host virtual display resolution increased.

The largest observed example was 4K120, where incoming stream frame rate fell to roughly 32 FPS while network drops remained at zero and client decode time stayed low.

Reducing the host display resolution produced large improvements without changing the GPUs or encoder.

That behavior made the cross-GPU presentation path a stronger suspect than raw game rendering or encoder throughput.

## 3. Physical PCIe Topology Mattered

The Arc A380 is currently connected through the B650 chipset at PCIe 4.0 x2, while the RX 6700 XT is CPU-direct at PCIe 4.0 x16.

Because the Radeon renders the game while the Arc owns the desktop and Sunshine capture path, the design depends on cross-GPU presentation.

The current topology therefore places the inter-GPU path behind a narrow chipset-connected link.

This is a strong working hypothesis, not a final conclusion. PRIME/dma-buf behavior is more complicated than a simple full-frame copy, and CPU-direct x8/x8 testing is required before attributing all lost frames to raw PCIe bandwidth.

## 4. Reading the Whole PCIe Hierarchy Was Necessary

A direct `lspci -vv` query against the Arc endpoint reported a misleading internal x1 link.

Walking the PCIe tree and checking the Arc's upstream bridge showed the physical external connection actually negotiating at PCIe 4.0 x2, which matched the motherboard's electrical slot layout.

The useful lesson was to validate the entire device path rather than assuming the endpoint's first reported link width represented the motherboard slot.

## 5. Gamescope Did Not Improve the Problem

Gamescope was tested as an additional game-focused compositor layer, but it did not materially change the streaming behavior.

That result helped eliminate game presentation inside Gamescope as the primary cause and kept the investigation focused on the cross-GPU, capture, and transport path.

Gamescope may still be useful for resolution control or game-specific presentation behavior, but it is not required for the current architecture.

## 6. Client Frame Pacing Can Fail Independently

A separate Moonlight client issue initially looked like poor host streaming performance.

The stream arrived with healthy decode and render times, but borderless/windowed presentation produced a large frame queue delay and visible choppiness.

Switching Moonlight to true fullscreen made the stream smooth immediately.

When diagnosing Moonlight performance, the following should be treated as separate stages:

1. host processing latency,
2. network loss and latency,
3. client decode time,
4. client render time,
5. frame queue delay and final presentation pacing.

A failure in one stage can look similar to a failure in another.

## 7. Render-GPU Saturation Is a Separate Variable

The Witcher 3 can drive the RX 6700 XT to approximately 99–100% utilization while producing roughly 80–100 FPS at 1440p.

That makes it a useful stress workload because there is little spare render-GPU headroom while frames are also being handed off to the Arc-owned presentation path.

If the same workload remains stuttery after the move to CPU-direct x8/x8, the next tests will focus on scheduling and synchronization rather than assuming the Arc encoder is at fault.

## 8. The Network Was Not the High-Resolution Bottleneck

The current host uses 2.5GbE, but Moonlight stream bitrates remain far below that capacity.

Problematic tests showed low network latency and no meaningful packet loss, so raw Ethernet bandwidth is not a good explanation for the high-resolution frame-delivery collapse.

10GbE remains useful for NAS-backed game storage, installations, updates, and bulk transfers.

## Current Conclusions

### Confirmed

- RX 6700 XT rendering works correctly under passthrough.
- Arc A380 passthrough works correctly.
- NixOS + Intel `iHD` provides working H.264, HEVC, and AV1 hardware encoding.
- Sunshine captures the Labwc headless output through Wayland screencopy.
- Stream delivery improves substantially as host resolution decreases.
- The Arc media engine is not fully saturated during the known high-resolution failure case.
- Client presentation mode can independently cause visible stutter.
- Gamescope did not materially improve the observed host-side limitation.

### Strong Working Hypothesis

The current PCIe 4.0 x2 chipset path to the Arc A380 is a major limitation for high-resolution, high-refresh cross-GPU presentation and capture.

### Not Yet Proven

- That raw PCIe bandwidth alone explains all lost frames.
- That CPU-direct x8/x8 will eliminate all streaming stutter.
- That the Arc A380 can sustain a single 4K240 encode workload under the final architecture.

Those are the next experiments.
