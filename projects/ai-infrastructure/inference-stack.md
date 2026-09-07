# LLM Inference Stack

## Overview

The LLM inference layer is built around llama.cpp running on `ai-node-01` with the AMD Instinct MI60 accelerated through Vulkan.

The frontend and inference engine are separate services.

Open WebUI sends requests to llama.cpp over an API, while llama.cpp performs the actual model loading, memory allocation, prompt processing, and token generation.

```text
Open WebUI
    |
    v
llama.cpp API
    |
    v
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

## Why llama.cpp

llama.cpp fits this platform well because it provides:

- strong GGUF support
- quantized model inference
- a relatively lightweight dependency footprint
- Vulkan acceleration
- an OpenAI-compatible serving interface
- direct control over context size and memory behavior
- broad support for local model experimentation

A major advantage is that the inference stack does not need PyTorch or ROCm just to serve an LLM.

That reduces software complexity for the workload.

## GGUF Models

The platform primarily uses GGUF models with llama.cpp.

GGUF allows models to be stored in quantized formats that reduce VRAM requirements while preserving useful model quality.

Common considerations include:

- parameter count
- architecture
- active parameter count for mixture-of-experts models
- quantization level
- model file size
- context length
- expected KV-cache growth
- prompt-processing performance
- token-generation performance

The "largest model that fits" is not automatically the best model for every use case.

A useful model must also provide acceptable:

- response speed
- context size
- output quality
- stability
- VRAM headroom

## Vulkan Backend

The llama.cpp environment uses Vulkan rather than ROCm.

The software path is:

```text
llama.cpp
   |
   v
Vulkan API
   |
   v
RADV
   |
   v
Mesa
   |
   v
AMDGPU
   |
   v
AMD Instinct MI60
```

This is one of the most important findings from the project.

An AMD datacenter GPU does not automatically require ROCm for every AI workload.

For llama.cpp, Vulkan has proven to be a practical and performant path on the MI60.

## Why Vulkan Is Useful Here

Vulkan provides several advantages for this specific environment:

- simpler dependency chain than a full PyTorch/ROCm stack
- good compatibility with llama.cpp
- direct access through the standard Linux graphics stack
- avoids tying LLM inference to ROCm release compatibility
- works well with the MI60 for tested quantized models

This does not mean Vulkan is universally better than ROCm.

It means Vulkan is a strong fit for this particular application.

## Model Loading

llama.cpp loads model weights from centralized storage and allocates the required model data into GPU memory.

The important stages are:

```text
Model file
   |
   v
Storage read
   |
   v
llama.cpp model initialization
   |
   v
GPU allocation
   |
   v
Context / KV-cache allocation
   |
   v
Inference ready
```

Storage performance is most visible during model loading.

Once the model is resident in VRAM, steady-state token generation is dominated by the inference workload rather than model-file storage.

## GPU Offload

The goal for supported models is to keep as much of the workload on the MI60 as practical.

When a model fits fully in VRAM, the system avoids the performance penalty associated with frequent CPU/GPU movement or system-memory spillover.

The 32 GB VRAM capacity is therefore particularly valuable.

## VRAM Budget

VRAM is consumed by more than just model weights.

A simplified view is:

```text
Total VRAM
  |
  +-- model weights
  |
  +-- KV cache
  |
  +-- compute buffers
  |
  +-- backend overhead
```

This means a model that appears to fit based only on file size may still fail or leave too little room for a useful context window.

## Context Size

Context size is one of the most important tunable parameters in local inference.

Increasing context allows the model to retain more conversation or input data, but it also increases memory consumption.

The platform has been used for large-context experimentation, including context sizes well beyond typical short-chat workloads.

Testing has included very large context configurations, including 100K and 150K context settings during Qwen experiments and an approximately 131K context setting during a Hermes experiment.

These values were used for specific experiments rather than as a standard context setting for every model.

The purpose is to observe how context size changes:

- VRAM consumption
- prompt processing time
- model responsiveness
- long-conversation usability
- spillover risk

## Quantization

Quantization is one of the main tools for fitting larger models into available VRAM.

Lower-bit quantization reduces memory requirements but may affect quality.

Testing therefore considers a balance between:

- quality
- VRAM use
- model size
- context headroom
- speed

Q4-class quantization has proven useful for larger models where capacity matters more than running the highest-precision variant.

## API Serving

llama.cpp is not only a local command-line inference tool in this architecture.

It acts as a service.

```text
Client / Open WebUI
       |
       v
HTTP API
       |
       v
llama-server
       |
       v
Loaded model
       |
       v
MI60
```

This service-oriented model is what allows the frontend to run independently on another physical node.

## Open WebUI Integration

Open WebUI is the primary interface used to interact with models.

Open WebUI handles the user experience while llama.cpp handles the compute.

This division is intentional:

```text
Open WebUI:
- conversations
- interface
- knowledge
- model selection
- application state

llama.cpp:
- model loading
- context allocation
- inference
- token generation
- GPU execution
```

The two services communicate through the inference API.

## SearXNG Integration

SearXNG provides search functionality for workflows that need current information.

This is also kept outside the GPU compute environment.

The resulting conceptual flow is:

```text
User
 |
 v
Open WebUI
 | \
 |  \--> SearXNG
 |
 v
llama.cpp
 |
 v
MI60
```

Search therefore enriches the application layer without becoming part of the inference engine itself.

## Model Switching

Because models are stored centrally, the inference environment can load different models without duplicating the entire model library inside the LXC.

Model switching requires consideration of:

- model unload time
- model load time
- VRAM allocation
- context settings
- backend options
- service startup configuration


## Service Management

The inference environment is intended to behave like infrastructure, not a manually launched desktop application.

That means service management should support:

- predictable startup
- predictable shutdown
- clear logs
- repeatable model arguments
- controlled environment variables
- easy restart after configuration changes

The long-term goal is that rebuilding the inference environment should be straightforward because models and application state are stored elsewhere.

## Troubleshooting Approach

The separation between services makes troubleshooting easier.

For example:

If Open WebUI fails:

- test llama.cpp directly

If llama.cpp fails:

- test Vulkan device access

If Vulkan fails:

- inspect Mesa/RADV and AMDGPU

If the model fails to load:

- inspect VRAM, context, and quantization requirements

This keeps the troubleshooting chain narrow.

## Current Limitations

The current design still has practical limits:

- only 32 GB of VRAM is available on the MI60
- the MI60 is an older architecture
- model load times still depend on storage and model size
- very large context windows consume substantial VRAM
- not every new model architecture performs equally well in llama.cpp
- frontend availability does not provide inference availability if the compute node is offline

These limitations are accepted because the project prioritizes modularity, learning value, and useful local performance over enterprise-scale availability.
