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

Performance is evaluated using both game-side metrics and stream-side behavior. Average game FPS alone is not sufficient because a game can continue rendering quickly while frame delivery to Moonlight becomes uneven or substantially slower.

## Current Platform

The current host uses an ASRock X870 Taichi Creator with both GPUs connected directly to the Ryzen 9 7950X3D:

```text
RX 6700 XT → CPU-direct PCIe 4.0 x8
Arc A380   → CPU-direct PCIe 4.0 x8
```

The previous B650 platform placed the Arc A380 on a chipset-connected PCIe 4.0 x2 path. Moving to the symmetrical x8/x8 topology removed that limitation but did not eliminate the high-load streaming problem.

This was an important result because it moved the investigation away from PCIe bandwidth as the primary explanation and toward GPU scheduling and synchronization behavior.

## Historical High-Resolution Baseline

Before the platform and scheduler investigation, 120 FPS-oriented testing produced results such as:

| Host virtual display | Moonlight request | Approx. incoming FPS | Result |
|---|---|---:|---|
| 3840×2160 @ 120 | 3840×2160 @ 120 | ~32 FPS | Severe host-side delivery drop |
| 3200×1800 @ 120 | 2560×1440 @ 120 | ~84 FPS | Large improvement |
| 2560×1440 @ 120 | 2560×1440 @ 120 | ~80–100 FPS | Much better, but not consistently 120 |

In one 4K120 failure case, Moonlight reported approximately:

- ~32 incoming FPS
- 0 network drops
- 0 jitter drops
- ~1 ms network latency
- ~0.64 ms decode time
- ~0.56 ms render time
- ~34.6 ms average host processing time

These results showed that the network and Moonlight client decoder were not responsible for the majority of the delay.

## Intel Arc Encoder Headroom

The Arc A380 has consistently retained substantial headroom during poor streams.

Earlier `intel_gpu_top` testing showed approximately:

```text
Render/3D     ~56%
Blitter       ~36%
Video         ~43%
VideoEnhance    0%
```

More detailed later testing measured approximately 2.94 seconds of Intel VIDEO-engine execution during a 10-second bad-case sample.

This indicates that the Arc hardware encoder is not saturated when incoming stream FPS collapses. The primary delay occurs before encoding.

## RX-Side Scheduling Findings

Tracing showed that the most important delay occurs in the AMDGPU graphics scheduling path.

A representative bad case showed approximately:

```text
RX GFX work queued
        ↓
~39.9 ms scheduler wait
        ↓
~0.15 ms actual GFX execution
        ↓
SDMA handoff begins almost immediately
        ↓
~2.3 ms SDMA execution
        ↓
Arc / Labwc dependency can continue
        ↓
Sunshine capture and encode
```

The important result is that the stream-critical RX work itself was extremely fast once scheduled. Most of the latency came from waiting behind work that had already accumulated in the AMDGPU software queue.

## AMDGPU Software Queue Depth

The stock AMDGPU setting is:

```text
amdgpu.sched_jobs=32
```

For this workload, a queue depth of `32` allows the game to build substantial queue-ahead under heavy render load.

Testing has therefore focused on:

```text
amdgpu.sched_jobs=8
amdgpu.sched_jobs=4
```

Lowering `sched_jobs` does not directly cap game FPS. It reduces how much work a scheduling entity can have outstanding, applying backpressure sooner and reducing how far GPU work can accumulate ahead of frame-handoff dependencies.

### Frame Pacing

The difference is visible not only in Moonlight incoming FPS but also in game frametime behavior.

With the stock value of `32`, frametime graphs can show frequent closely spaced oscillation even while average FPS remains high.

Reducing the queue depth to `8` or `4` produces substantially smoother frametime behavior in tested workloads.

This is important because a high average frame rate does not guarantee smooth motion. Consistent frame delivery and frametime stability are often more important than the headline FPS number.

## Current Game Results

### The Witcher 3

The Witcher 3 remains one of the most useful heavy-load tests because it can place sustained pressure on the RX 6700 XT.

With `amdgpu.sched_jobs=8`:

- Game FPS: approximately 55-63 FPS
- Moonlight incoming FPS: approximately 45-50 FPS
- Subjectively much smoother than the stock queue-depth behavior

With `amdgpu.sched_jobs=4`:

- Moonlight incoming FPS has reached approximately 65-70 FPS
- Frametime behavior is noticeably smoother
- The game feels smoother than with the stock `32` setting and, in current testing, smoother than `8`

The main game installation remains on NAS storage while its latency-sensitive save-game data under the Proton Windows `Documents` environment is kept on local NVMe.

### Fallout 4

Fallout 4 performs extremely well on the 7950X3D and RX 6700 XT.

Current 4K testing in Boston has produced approximately 80-100 game FPS.

With `amdgpu.sched_jobs=8`:

- Game FPS: approximately 80-100 FPS
- Moonlight incoming FPS: generally in the 50s
- Stream behavior is substantially smoother than with the stock queue depth

With `amdgpu.sched_jobs=4`:

- Game FPS can remain around 100 FPS
- Moonlight incoming FPS reaches approximately 60-70 FPS
- Frametime behavior is smoother than with `sched_jobs=32`

Fallout 4 also remains an important storage-performance example because its metadata-heavy workload performs dramatically better from local NVMe than from NFS.

### Red Dead Redemption 2

Red Dead Redemption 2 demonstrates the remaining limitation most clearly.

Current NixOS testing at approximately 1800p produces around 70 game FPS, compared with a previous Windows 11 reference point of roughly 50–60 FPS at approximately 1768p.

Raw game performance is therefore strong, but stream delivery can still fall substantially behind the game.

With `amdgpu.sched_jobs=8` at approximately 1800p:

- Game FPS: ~70 FPS
- Moonlight incoming FPS: ~35 FPS

Testing `sched_jobs=4` at the same heavy workload can feel choppier than `8`.

Reducing game workload demonstrates that the streaming pipeline itself is capable of delivering higher frame rates:

| RDR2 workload | Approx. Moonlight incoming FPS |
|---|---:|
| 1800p, ~70 FPS game load | ~35 FPS |
| 1800p, 60 FPS diagnostic cap | ~43-50 FPS |
| 1620p, 60 FPS diagnostic cap | ~60 FPS |

These caps and resolution reductions are diagnostic tests rather than the intended permanent solution. They demonstrate that additional RX render headroom allows the cross-GPU handoff to complete more consistently.

## GPU Utilization vs Stream Performance

One of the strongest findings is that high game FPS does not imply that the stream has enough scheduling headroom.

A game can remain fast and responsive while the RX is heavily loaded, yet stream-critical work may still be delayed enough for Moonlight incoming FPS to fall sharply.

The relevant distinction is:

```text
Game rendering performance
        ≠
Frame-delivery latency
```

This is why both MangoHud frametime data and Moonlight stream statistics are used when evaluating changes.

## Moonlight Refresh Rate

The current practical Moonlight target is:

```text
3840x2160 @ 144 Hz
```

4K240 is functional, but repeated testing causes incoming frame delivery to fall again.

4K144 provides a better balance between high-refresh responsiveness and reliable frame delivery, especially because current game workloads generally do not approach 240 FPS at 4K.

The project therefore currently favors stable 4K144 operation rather than maximizing the configured refresh-rate number.

## Client-Side Presentation

A separate client-side issue was discovered during earlier 120 FPS testing.

One Moonlight client showed approximately 15-20 ms of average frame queue delay in borderless/windowed mode despite low decode and render times.

Switching Moonlight to true fullscreen immediately restored smooth presentation.

This demonstrated that client presentation can create its own frame-pacing problem independently of host delivery, network performance, or hardware decoding.

## Network Capacity

The gaming host now uses onboard 10GbE.

The Sunshine stream itself does not require anywhere near 10GbE bandwidth, and problematic streaming tests have not shown meaningful packet loss or network latency.

10GbE is primarily useful for NAS-backed game storage, large installations, updates, and bulk transfers rather than increasing Sunshine stream performance.

## Current Performance Conclusions

The current results support several conclusions:

1. Raw Linux/NixOS game performance is strong and is not the primary limitation.
2. The Intel Arc media engine has significant unused encoding capacity during bad streams.
3. Moving from chipset PCIe 4.0 x2 to CPU-direct PCIe 4.0 x8 did not eliminate the problem.
4. The strongest measured delay occurs in RX-side GPU scheduling before the cross-GPU frame handoff.
5. Reducing `amdgpu.sched_jobs` from `32` to `8` or `4` substantially improves frame pacing and Moonlight delivery.
6. The best queue depth is workload-dependent.
7. Extremely heavy RX workloads can still reduce stream delivery even with a shallow software queue.
8. 4K144 is currently a more effective streaming target than 4K240.
9. Average FPS alone is insufficient; frametime consistency and delivered stream FPS are equally important.