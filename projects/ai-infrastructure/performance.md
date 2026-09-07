# Performance Testing

## Overview

Performance testing is used to understand how the Local AI Infrastructure Platform behaves under real workloads.

The purpose is not to produce a universal benchmark for the AMD Instinct MI60.

The purpose is to answer practical questions such as:

- Which models fit entirely in VRAM?
- How does context size affect memory use?
- How much performance is lost when the GPU power limit is reduced?
- Which workloads are limited by memory bandwidth?
- How much thermal headroom does the cooling system provide?
- How does the MI60 behave during long-running inference?
- Which quantization levels provide useful quality while preserving context headroom?

## Test Environment

Primary compute platform:

```text
Node:        ai-node-01
CPU:         AMD Ryzen 5 3600
Memory:      32 GB DDR4
GPU:         AMD Instinct MI60
VRAM:        32 GB HBM2
Hypervisor:  Proxmox VE
Inference:   llama.cpp
Backend:     Vulkan
```

Image-generation tests use ComfyUI with ROCm and should be treated as a separate workload class.

## Benchmark Philosophy

A single tokens-per-second number is not enough to describe an inference platform.

Useful measurements include:

- model
- architecture
- quantization
- context size
- prompt size
- prompt-processing speed
- token-generation speed
- VRAM usage
- GPU power
- GPU temperature
- llama.cpp version
- Vulkan/Mesa version
- whether the model remains entirely on GPU

Without those details, benchmark numbers are difficult to reproduce or compare.

## Representative LLM Result

One representative test using a Qwen 3.6 35B A3B Q4_K_M model achieved approximately:

```text
Generation speed: ~83-84 tokens/sec
VRAM use:         ~21 GB
GPU:              AMD Instinct MI60 32 GB
Backend:          Vulkan
```

This result demonstrates that the MI60 can provide strong interactive performance for a model that benefits from substantial VRAM capacity.

The figure should not be interpreted as a guarantee for other models.

Mixture-of-experts models, dense models, context length, quantization, and backend changes can all produce very different results.

## Context-Size Testing

Context size can materially change VRAM use.

A useful test matrix is:

| Context | VRAM Used | Prompt Processing | Generation Speed | Notes |
|---:|---:|---:|---:|---|
| 32K | TBD | TBD | TBD | baseline |
| 64K | TBD | TBD | TBD | |
| 100K | TBD | TBD | TBD | |
| 128K | TBD | TBD | TBD | |
| 150K | TBD | TBD | TBD | high-context stress test |

The purpose of this test is to determine where increased context begins to reduce practical usability or consume too much VRAM headroom.

## VRAM Utilization

The platform has been tested at very high GPU memory utilization, including workloads around 94% VRAM use while still remaining on the GPU.

This is useful because it demonstrates the practical value of the MI60's 32 GB memory capacity.

However, running close to the limit requires care.

A small change in:

- context size
- batch parameters
- model build
- backend memory behavior

may push the workload beyond available VRAM.

For normal operation, some headroom is preferable when possible.

## Power-Limit Testing

The MI60 can draw significantly more power than is required for every LLM workload.

Power-limit testing is therefore used to find a better efficiency point.

A useful test table is:

| Power Limit | Tokens/sec | GPU Temp | VRAM | Tokens/sec/W | Notes |
|---:|---:|---:|---:|---:|---|
| 300 W | TBD | TBD | TBD | TBD | reference |
| 220 W | TBD | TBD | TBD | TBD | |
| 180 W | TBD | TBD | TBD | TBD | |
| 170 W | TBD | TBD | TBD | TBD | current low-power range |

Earlier testing showed that reducing the MI60 to roughly the 170 W range did not produce a major token-generation penalty for at least some workloads.

This suggests that those workloads were not scaling linearly with available GPU power.

## Why Power May Not Scale Linearly

LLM inference can be limited by multiple resources.

During token generation, important factors include:

- memory bandwidth
- model architecture
- memory access patterns
- compute throughput
- backend efficiency
- KV-cache behavior

If a workload is primarily constrained by memory movement, increasing GPU power does not necessarily produce a proportional increase in tokens per second.

That makes the MI60's HBM2 particularly interesting.

## Thermal Testing

Thermal testing should be performed under sustained load rather than short bursts.

Relevant values include:

- idle temperature
- model-load temperature
- steady-state inference temperature
- power draw
- fan speed
- room temperature where available

The cooling solution should be considered successful only if the GPU remains stable during extended workloads.

## Chassis Comparison

The move from the earlier ATX mid-tower case to the SilverStone 4U rackmount chassis provides an opportunity to compare:

- GPU temperature
- fan requirement
- case temperature
- noise
- airflow restriction
- serviceability

The SilverStone 4U chassis has a more open airflow path and more practical room around the MI60 and its dedicated cooling fan.

Future testing can quantify whether that produces measurable thermal improvement.

## Prompt Processing vs Token Generation

Prompt ingestion and token generation are different phases.

Prompt processing can be more compute-intensive, while sequential token generation can behave differently and may be more memory-bandwidth sensitive.

Both should be recorded when possible.

A useful result format is:

```text
Model:
Quantization:
Context:
Prompt tokens:
Generated tokens:

Prompt processing:
Generation:
VRAM:
GPU power:
GPU temperature:
```

## Quantization Testing

Quantization should be tested as an engineering tradeoff.

A useful comparison can include:

| Quantization | Model Size | VRAM | Tokens/sec | Context Headroom | Subjective Quality |
|---|---:|---:|---:|---:|---|
| Q8 | TBD | TBD | TBD | TBD | TBD |
| Q6 | TBD | TBD | TBD | TBD | TBD |
| Q5 | TBD | TBD | TBD | TBD | TBD |
| Q4_K_M | TBD | TBD | TBD | TBD | TBD |

The best choice depends on the intended workload.

## Model Load Time

Model load time is influenced by:

- model size
- storage performance
- filesystem/network path
- host memory
- initialization overhead

This matters operationally even though it does not directly determine steady-state tokens per second.

For environments where models are switched frequently, load time becomes an important usability metric.

## Image-Generation Testing

ComfyUI/ROCm performance should be recorded separately.

Useful measurements include:

| Workload | Resolution | Steps | VRAM | Time | Power | Temperature |
|---|---:|---:|---:|---:|---:|---:|
| TBD | TBD | TBD | TBD | TBD | TBD | TBD |

Different model families should not be compared without documenting the workflow.

## Data Collection

Where possible, test data should be recorded directly from:

- llama.cpp timing output
- GPU monitoring tools
- system logs
- repeatable prompt inputs
- documented model versions

Screenshots can supplement results, but text tables are more useful for long-term comparison.

## Additional Measurements

Useful measurements to add as more repeatable data is collected include:

- repeatable power-limit sweep
- dense-model vs MoE comparison
- context-size scaling
- Q4 vs Q5 vs Q6 comparison
- prompt-processing scaling
- SilverStone chassis thermal comparison
- ComfyUI resolution scaling
- GPU power efficiency by workload
- model-load time from centralized storage

The goal is to turn everyday experimentation into repeatable engineering data rather than isolated benchmark numbers.
