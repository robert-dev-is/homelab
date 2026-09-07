# Lessons Learned

## Overview

The Local AI Infrastructure Platform started as a way to run local AI models on an AMD Instinct MI60, but the more important outcome has been learning how AI workloads interact with virtualization, GPU software stacks, storage, power, cooling, and service architecture.

The lessons below are based on operating the platform rather than on theoretical design alone.

## Datacenter GPUs Are Not Desktop GPUs

The MI60 is physically a PCIe accelerator, but that does not make it equivalent to installing a desktop graphics card.

A consumer GPU usually includes:

- its own fans
- a cooler designed around desktop airflow
- display outputs
- firmware and software assumptions aimed at workstations

The MI60 instead assumes:

- forced server airflow
- no local display requirement
- datacenter cooling
- compute-focused software use

That difference affects the entire deployment.

The cooling problem has to be solved deliberately.

## Cooling Must Match the Original Hardware Assumptions

A passive datacenter heatsink is not "fanless" in the usual desktop sense.

It expects an external airflow system.

The important lesson is:

> Cooling design should match the hardware's intended airflow path.

A case full of fans is not automatically enough if air is not being pushed through the GPU heatsink.

The dedicated fan attached to the MI60 is therefore functionally replacing part of the server airflow environment for which the card was designed.

## A Proper Rackmount Chassis Improved Rack Use and Serviceability

`ai-node-01` was previously housed in an ATX mid-tower case placed in the rack. The tower required approximately 3U of open clearance above it so the system could be moved in and out of the rack.

A 3U rackmount replacement was considered, but the available options created unnecessary constraints around:

- GPU clearance
- cooling
- CPU cooler height
- fan placement
- serviceability

Moving the node into a SilverStone 4U rackmount chassis eliminated the extra service clearance required by the tower while also improving airflow and access around the MI60.

The result was better thermal management and more efficient use of the rack, including enough usable space for another 4U node.

## VRAM Capacity Can Matter More Than Gaming Performance

The MI60 is old compared with modern gaming GPUs.

That does not make it irrelevant for AI.

Its 32 GB of HBM2 allows it to run workloads that simply do not fit on many newer GPUs with 12, 16, 20, or 24 GB of VRAM.

For local AI, the useful question is often not:

> Which GPU has the highest gaming benchmark?

It is:

> Can the model, context, and runtime buffers fit entirely in GPU memory?

This changes the way GPU value is evaluated.

## Memory Bandwidth Matters

Large-language-model token generation frequently moves large amounts of model data through memory.

That makes memory bandwidth highly relevant.

The MI60's HBM2 is one reason it can remain useful despite its age.

A newer GPU with less memory or less suitable memory characteristics may not automatically be better for every local inference workload.

## AMD AI Does Not Automatically Mean ROCm

One of the most important practical lessons was that an AMD GPU does not require ROCm for every AI application.

The LLM path works well as:

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

Meanwhile, ComfyUI uses:

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
MI60
```

The correct compute stack depends on the application.

## Vulkan Is a Serious LLM Inference Backend

Vulkan is sometimes treated as a fallback backend compared with vendor-specific compute APIs.

On this platform, it has been a practical production-like solution for llama.cpp.

Advantages include:

- simpler dependency management
- strong compatibility with llama.cpp
- less dependence on ROCm release support for the MI60
- good observed performance

The broader lesson is to evaluate the actual workload instead of assuming one compute API is always superior.

## Different Workloads Should Be Allowed to Use Different Stacks

Trying to force llama.cpp and ComfyUI into one identical software environment would add unnecessary complexity.

The better design is:

```text
LLM workload
  -> llama.cpp
  -> Vulkan

Image workload
  -> ComfyUI
  -> PyTorch
  -> ROCm
```

Both ultimately use the same MI60.

Service isolation makes this possible without dependency conflicts becoming a platform-wide problem.

## Separate Applications From Compute

Open WebUI does not need to live on the GPU node.

This became one of the central architectural principles of the platform.

The user-facing layer and compute layer have different responsibilities.

```text
Application Layer
- Open WebUI
- PostgreSQL
- SearXNG

Compute Layer
- llama.cpp
- ComfyUI
- MI60
```

The benefit is that the GPU node can be restarted, reconfigured, or rebuilt without moving the application database or redesigning the frontend.

## An AI Platform Is More Than the GPU

It is easy to call one machine "the AI server."

In practice, the working platform includes:

- user interface
- database
- search
- inference API
- model runtime
- GPU drivers
- model storage
- networking
- virtualization
- backups
- monitoring
- power and cooling

The GPU is a critical resource, but it is not the whole system.

Thinking in service layers produces a more flexible design.

## Centralize Large Model Data

Model files are too large to treat like ordinary application dependencies.

Keeping them inside disposable containers creates several problems:

- duplicate storage use
- slow rebuilds
- difficult backup
- inconsistent model organization

Centralized model storage allows workload containers to be treated as replaceable compute environments.

This is much closer to how infrastructure should behave.

## Keep the Hypervisor Clean

Installing every AI dependency directly on the Proxmox host would make the node harder to understand and harder to recover.

Using isolated workload environments keeps the host focused on:

- hardware
- virtualization
- storage presentation
- networking
- device access

AI software belongs in the workload layer whenever practical.

## LXC Is Well Suited to This Type of Experimentation

LXC provides a useful middle ground:

- lower overhead than a full VM
- clear filesystem and service boundaries
- easy snapshots
- easier rebuilds
- direct access to required devices

For Linux-native AI services, this makes LXC a strong fit.

It also matches the broader design principle of running one clear service or workload role per environment.

## Power Limits Should Be Measured, Not Guessed

Running the GPU at maximum possible power is not automatically optimal.

Tests on the MI60 showed that some LLM workloads retained most of their token-generation performance at significantly reduced power limits.

The correct lesson is not simply "lower power is better."

It is:

> Measure application throughput while changing power, then choose the operating point based on efficiency and stability.

## Thermal Testing Must Be Sustained

A GPU surviving a two-minute test does not prove the cooling system is adequate.

AI workloads can run for:

- tens of minutes
- hours
- long interactive sessions

Cooling validation therefore needs sustained load.

This is especially important for a passive datacenter accelerator.

## Very High VRAM Utilization Is Useful but Leaves Little Margin

The platform has successfully operated workloads using roughly 94% of the MI60's VRAM.

That demonstrates how much value can be extracted from 32 GB.

It also demonstrates the danger of planning only around model file size.

Context changes, runtime buffers, and backend behavior can consume the remaining margin quickly.

The practical lesson is to leave headroom when possible.

## Context Size Is an Infrastructure Variable

Context size is often discussed only as a model feature.

Locally, it is also a resource-planning variable.

Larger context changes:

- VRAM consumption
- prompt-processing time
- responsiveness
- model capacity planning

Context should therefore be treated like a configurable infrastructure resource.

## Model Performance Requires Context

A tokens-per-second result is not meaningful by itself.

Performance reporting should include:

- exact model
- quantization
- context
- backend
- power setting
- VRAM use
- prompt characteristics
- software version

Without that information, benchmark comparison becomes misleading.

## Troubleshooting Is Easier When Layers Are Separate

The architecture provides a useful diagnostic chain.

If the frontend fails, test the inference endpoint.

If inference fails, test the GPU backend.

If the backend fails, test the driver stack.

If the model fails to load, inspect model size, context, and VRAM.

This is much easier than troubleshooting a single container that includes every component.

## Rebuildability Is More Important Than Preserving One Working Install

A system that only works because it has never been touched is fragile.

The platform is designed so that application environments can be rebuilt while preserving:

- model data
- Open WebUI application data
- architecture
- service boundaries

That makes experimentation safer.

## Older Hardware Can Still Be Valuable When the Workload Fits

The MI60 is not a current-generation accelerator.

However, AI infrastructure value is workload-dependent.

For the right workload, an older accelerator with:

- sufficient VRAM
- high memory bandwidth
- workable software support
- acceptable power consumption

can remain extremely useful.

## Areas for Further Development

Several areas can continue to be developed without changing the core architecture:

- more formal GPU monitoring
- more repeatable performance data collection
- more automated startup/shutdown behavior
- expanded public documentation and diagrams

The current architecture allows those operational improvements to be made without redesigning the application and compute layers.

## Final Takeaway

The project reinforced a broader infrastructure principle:

> Good local AI infrastructure is not just about installing a model. It is about designing the compute, software, storage, networking, thermals, and service boundaries so the system remains understandable and maintainable.

That has been the main value of building the platform.
