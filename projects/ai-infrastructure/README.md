# Local AI Infrastructure Platform

## Overview

This project documents the design and operation of a self-hosted local AI infrastructure platform built around an AMD Instinct MI60 32 GB datacenter GPU, Proxmox VE, Linux containers, llama.cpp, Vulkan, ROCm, ComfyUI, Open WebUI, PostgreSQL, SearXNG, and centralized model storage.

The platform is intentionally split into multiple layers rather than being treated as a single "AI server." GPU-intensive workloads run on a dedicated compute node, while the user-facing application layer and supporting services run elsewhere in the virtualization environment.

The main design goal is to keep the AI stack modular:

- GPU compute can be maintained, rebuilt, or tuned independently.
- Frontend services do not need to live on the GPU node.
- LLM inference and image generation can use different AMD software stacks.
- Large model files remain centralized rather than being duplicated inside application containers.
- Individual services can be maintained independently without rebuilding the entire platform.

At a high level, the LLM path looks like this:

```text
Users
  |
  v
Open WebUI
  |
  +----> SearXNG
  |
  v
llama.cpp API
  |
  v
Vulkan
  |
  v
AMD Instinct MI60 32 GB
```

Image generation follows a separate path:

```text
Users
  |
  v
ComfyUI
  |
  v
PyTorch / ROCm
  |
  v
AMD Instinct MI60 32 GB
```

## Project Goals

The platform was built to explore what can be accomplished with self-hosted AI infrastructure using relatively accessible hardware and open-source software.

Primary goals include:

- Run modern local LLMs entirely on infrastructure under my control.
- Take advantage of 32 GB of HBM2 for models that exceed the VRAM capacity of many consumer GPUs.
- Learn how older datacenter accelerators behave in a homelab environment.
- Separate AI applications from GPU compute so the frontend does not depend on one physical machine.
- Use the most appropriate GPU software stack for each workload instead of forcing every AMD workload through ROCm.
- Centralize model storage so models are not tied to one container or application.
- Measure inference throughput, VRAM use, context scaling, thermals, and power consumption.
- Make the environment easy to modify, rebuild, and troubleshoot.
- Gain hands-on experience with Proxmox, LXC device access, Linux GPU tooling, inference APIs, model serving, and AI application integration.

## Platform Components

| Layer | Component | Role |
|---|---|---|
| Virtualization | Proxmox VE | Hosts the AI compute environment |
| Isolation | Linux Containers (LXC) | Separates AI workloads and dependencies |
| GPU Compute | AMD Instinct MI60 32 GB | LLM and image-generation acceleration |
| LLM Runtime | llama.cpp | GGUF model inference and API serving |
| LLM GPU Backend | Vulkan | GPU acceleration for llama.cpp |
| Image Generation | ComfyUI | Node-based diffusion workflow environment |
| Image GPU Backend | ROCm | PyTorch-compatible GPU compute for ComfyUI |
| AI Frontend | Open WebUI | Primary user-facing LLM interface |
| Database | PostgreSQL | Persistent Open WebUI application data |
| Search | SearXNG | Search integration for AI workflows |
| Model Storage | Centralized storage | Shared model repository independent of compute containers |

## Compute Hardware

The primary compute system is `ai-node-01`.

Current core hardware:

```text
CPU:        AMD Ryzen 5 3600
Memory:     32 GB DDR4
GPU:        AMD Instinct MI60
VRAM:       32 GB HBM2
Local Disk: 1 TB NVMe
Hypervisor: Proxmox VE
```

The MI60 is a passive datacenter accelerator rather than a conventional desktop graphics card. It has no display outputs and was designed for forced-air server environments. Adapting it for sustained use in a homelab required deliberate airflow, cooling, and power tuning.

The node was later moved from an ATX mid-tower case into a SilverStone 4U rackmount chassis. The new chassis provides better airflow, more practical clearance around the GPU and its dedicated cooling fan, and eliminates the extra rack clearance previously required to move the tower in and out of the rack. This improved both thermal management and usable rack capacity.

## Software Architecture

The project deliberately separates the user-facing application layer from the GPU compute layer.

### Application Layer

Open WebUI and SearXNG run on `elite-node`, independent of the GPU compute system.

Open WebUI stores its persistent application data in PostgreSQL within the Open WebUI container.

The application layer is responsible for:

- chat history
- model selection
- conversation management
- knowledge and document workflows
- user-facing configuration
- search-enhanced workflows

It is not responsible for performing GPU inference.

### Compute Layer

`ai-node-01` is responsible for GPU-heavy workloads.

Its primary AI workloads are:

- llama.cpp for LLM inference
- ComfyUI for image generation

These workloads use different GPU software paths.

For LLM inference:

```text
llama.cpp
  |
  v
Vulkan
  |
  v
Mesa / RADV
  |
  v
AMDGPU
  |
  v
MI60
```

For image generation:

```text
ComfyUI
  |
  v
PyTorch
  |
  v
ROCm
  |
  v
AMDGPU
  |
  v
MI60
```

This distinction is important. AMD GPU compute does not automatically mean that every workload must use ROCm.

## Design Principles

### Separate Applications From Compute

The AI frontend should not have to run on the same machine as the GPU.

Open WebUI communicates with llama.cpp over an API. This means the frontend can remain available even if the compute node is restarted or reconfigured, and changes to the compute layer do not require redesigning the application layer.

### Keep Workloads Independently Replaceable

The goal is to avoid a monolithic AI appliance.

A change to llama.cpp should not require rebuilding Open WebUI. A change to ComfyUI should not require touching the inference environment. Replacing one container should not require re-downloading the model library.

### Use the Simplest Appropriate Compute Backend

llama.cpp performs well on the MI60 using Vulkan, so ROCm is not required for that workload.

ComfyUI relies on the PyTorch ecosystem, where ROCm remains the more appropriate GPU stack.

### Centralize Large Model Data

AI models are large and should not be trapped inside disposable application containers.

Centralized model storage allows multiple workloads to consume the same model repository while keeping containers smaller and easier to rebuild.

### Treat Datacenter Hardware Like Datacenter Hardware

A passive server accelerator cannot be cooled like a typical desktop GPU.

The MI60 requires directed airflow through its heatsink. Cooling design, fan placement, chassis airflow, and power limits are therefore part of the compute platform rather than an afterthought.

## Current Capabilities

The platform currently supports:

- local GGUF LLM inference
- large-context experimentation
- OpenAI-style llama.cpp API serving
- Open WebUI as the primary user interface
- PostgreSQL-backed application data
- SearXNG-enhanced search workflows
- ComfyUI image generation
- Vulkan-based LLM acceleration
- ROCm-based image-generation acceleration
- centralized model storage
- power-limit experimentation
- GPU thermal monitoring
- Proxmox/LXC workload separation

## Representative Performance

Observed llama.cpp inference performance on the AMD Instinct MI60 using the Vulkan backend includes:

| Model | Quantization | Generation Speed |
|---|---|---:|
| Qwen 3.8 27B | Q6 | ~17-19 tokens/sec |
| Qwen 3.6 35B A3B | Q4 | ~83-84 tokens/sec |

These results are workload-specific and reflect observed generation throughput on the current `ai-node-01` inference stack. The two models use different architectures and quantizations, so the results should be treated as individual observations rather than a direct model-to-model benchmark comparison.

Additional details are available in [performance.md](performance.md).

## Key Engineering Challenges

Several parts of this project required more work than a typical consumer-GPU AI installation:

- adapting a passive datacenter GPU to non-server cooling
- determining which AMD software stack was best for each workload
- getting reliable GPU access inside LXC
- balancing context size against VRAM consumption
- keeping the GPU workload containers lightweight
- centralizing models without coupling them to one service
- tuning MI60 power consumption without unnecessarily sacrificing throughput
- keeping application services independent from GPU compute

## Documentation

- [architecture.md](architecture.md) - Overall service architecture and dependency model
- [gpu-compute.md](gpu-compute.md) - `ai-node-01`, MI60 deployment, cooling, power, and LXC GPU access
- [inference-stack.md](inference-stack.md) - llama.cpp, Vulkan, model serving, context, and VRAM behavior
- [image-generation.md](image-generation.md) - ComfyUI, ROCm, diffusion model storage, and AMD compatibility
- [performance.md](performance.md) - Inference, VRAM, thermal, and power testing
- [lessons-learned.md](lessons-learned.md) - Design lessons and practical conclusions from building the platform

## Scope

This repository documents the architecture and engineering work behind the platform.

It intentionally avoids publishing credentials, private network details, secrets, or other information that is unnecessary to understand the design.
