# Performance

## Overview

This file records only performance results that have actually been observed and documented on the Local AI Infrastructure Platform.

The current inference environment uses:

- `ai-node-01`
- AMD Instinct MI60 32 GB
- llama.cpp
- Vulkan

No broader benchmark matrix has been collected yet, so this document intentionally avoids estimated values, placeholder tables, or unmeasured comparisons.

## Observed LLM Inference Performance

| Model | Quantization | Generation Speed |
|---|---|---:|
| Qwen 3.8 27B | Q6 | ~17-19 tokens/sec |
| Qwen 3.6 35B A3B | Q4 | ~83-84 tokens/sec |

These results reflect observed generation throughput on the current MI60-based llama.cpp/Vulkan inference stack.

The large difference between the two models should not be interpreted as a direct model-to-model efficiency comparison. They use different architectures and quantizations, so the numbers are best treated as individual observed results rather than a controlled benchmark comparison.

