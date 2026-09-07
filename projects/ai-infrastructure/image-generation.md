# Image Generation

## Overview

Image generation is handled separately from LLM inference because the software requirements are different.

The platform uses ComfyUI for diffusion workflows and ROCm for GPU acceleration on the AMD Instinct MI60.

The stack is:

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
AMD Instinct MI60
```

This differs from the LLM stack, which uses llama.cpp and Vulkan.

## Why ComfyUI

ComfyUI is useful for infrastructure experimentation because it exposes the image-generation workflow as a graph rather than hiding the pipeline behind a single prompt box.

This makes it easier to understand and modify:

- model loading
- text encoding
- sampling
- VAE decode
- LoRA application
- image dimensions
- conditioning
- multi-stage workflows

It also makes the workload more transparent when troubleshooting GPU compatibility or performance.

## Why ROCm Is Used Here

ComfyUI is built around the PyTorch ecosystem.

For AMD GPUs, that makes ROCm the natural compute stack.

The important distinction is:

```text
LLM inference:
llama.cpp -> Vulkan -> MI60

Image generation:
ComfyUI -> PyTorch -> ROCm -> MI60
```

The platform therefore uses two different AMD GPU software stacks on the same physical accelerator.

This is not accidental duplication.

It reflects the requirements of the applications.

## MI60 Compatibility

The MI60 is an older datacenter accelerator, which creates additional compatibility considerations for modern ROCm and PyTorch software.

Potential challenges include:

- GPU architecture support
- ROCm version compatibility
- PyTorch build compatibility
- environment variables or compatibility overrides
- package dependency conflicts
- application updates that assume newer GPU generations

This is one reason the image-generation environment is isolated from the llama.cpp environment.

A ROCm experiment should not be able to break the working Vulkan LLM stack.

## Containerized Workload

ComfyUI runs in its own workload environment rather than sharing every dependency with llama.cpp.

Benefits include:

- independent package versions
- separate troubleshooting
- simpler rollback
- reduced cross-application dependency conflicts
- easier rebuild or replacement of the workload environment

The physical MI60 remains the common compute device, but the userspace environments are kept separate.

## Centralized Diffusion Model Storage

Diffusion models are stored centrally rather than permanently inside the ComfyUI container.

A representative organization is:

```text
/ai-models/diffusion/
|
+-- checkpoints/
|
+-- loras/
|
+-- vae/
```

This allows the ComfyUI environment to be rebuilt without re-downloading a large model collection.

## Checkpoints

Checkpoint models are the primary diffusion model files.

They can be significantly larger than ordinary application files, so treating them as persistent infrastructure data rather than disposable container data simplifies maintenance.

## LoRAs

LoRA files are also centralized.

Benefits include:

- one consistent library
- easier backup
- easier testing across workflows
- no dependence on one ComfyUI container filesystem

## VAE Files

VAE models are stored alongside the rest of the diffusion model library.

Keeping them separate from the application environment makes it easier to understand what is application code and what is model data.

## Workflow Data

ComfyUI workflows are much smaller than model files but are still valuable configuration artifacts.

They should be treated as reusable infrastructure/application configuration rather than temporary browser state.


## GPU Memory Considerations

Image-generation workloads use VRAM differently from llama.cpp.

Important variables include:

- checkpoint size
- image resolution
- batch size
- VAE behavior
- LoRA use
- model architecture
- precision
- intermediate tensors

The same 32 GB of VRAM that is valuable for large LLMs also provides useful headroom for diffusion workloads.

## Performance Considerations

Image-generation performance should be measured separately from LLM performance.

Useful metrics include:

- total generation time
- time per sampling step
- VRAM usage
- GPU power draw
- GPU temperature
- maximum practical resolution
- stability across repeated generations

Because the software stack is different, performance observations from Vulkan inference should not be assumed to apply directly to ROCm image generation.

## Cooling and Sustained Load

ComfyUI can keep the GPU under sustained load for extended periods.

The same cooling requirements documented in `gpu-compute.md` apply here.

A stable configuration must account for:

- passive MI60 heatsink design
- directed airflow
- case intake
- exhaust path
- power limit
- sustained rather than burst temperature

## Separation From LLM Inference

Keeping image generation and LLM inference separate provides a clean operational boundary.

If ComfyUI or ROCm needs to be rebuilt:

- llama.cpp does not need to change
- Open WebUI does not need to change
- centralized model storage remains intact

If llama.cpp is upgraded:

- the ROCm environment does not need to change

This is a direct example of why the overall platform uses service separation rather than one large AI software environment.

## Current Limitations

The current image-generation stack inherits the limitations of using an older AMD datacenter GPU:

- modern ROCm support may require more care
- some packages assume newer GPU architectures
- some community workflows are primarily tested on NVIDIA
- troubleshooting information may be less mature for MI60-specific setups

Despite those limitations, the MI60's memory capacity makes it a useful platform for learning and experimentation.
