# Platform Architecture

## Overview

The Local AI Infrastructure Platform is designed as a set of cooperating services rather than one all-in-one AI host.

The architecture separates four major concerns:

1. user-facing applications
2. supporting application services
3. GPU compute
4. model storage

The GPU is therefore one part of the platform rather than the entire platform.

## Architectural Goals

The design was built around several goals:

- keep GPU compute independent from user-facing services
- allow individual services to be rebuilt without rebuilding the whole stack
- avoid duplicating large model files inside containers
- use separate environments for workloads with different dependency requirements
- keep changes to the GPU compute layer independent from frontend services
- keep the Proxmox host itself as clean as practical
- preserve clear service boundaries for troubleshooting

## High-Level Topology

```text
                           +------------------+
                           |      Users       |
                           +---------+--------+
                                     |
                                     v
                           +------------------+
                           |    Open WebUI    |
                           |    elite-node    |
                           +----+--------+----+
                                |        |
                                |        +----------------+
                                |                         |
                                v                         v
                      +------------------+       +------------------+
                      |    llama.cpp     |       |     SearXNG      |
                      |   ai-node-01     |       |    elite-node    |
                      +---------+--------+       +------------------+
                                |
                                v
                      +------------------+
                      |      Vulkan      |
                      +---------+--------+
                                |
                                v
                      +------------------+
                      | Instinct MI60    |
                      |    32 GB HBM2    |
                      +------------------+
```

Image generation is a separate compute path:

```text
+------------------+
|     ComfyUI      |
|   ai-node-01     |
+---------+--------+
          |
          v
+------------------+
| PyTorch / ROCm   |
+---------+--------+
          |
          v
+------------------+
| Instinct MI60    |
|    32 GB HBM2    |
+------------------+
```

Both compute paths can consume models from centralized storage.

## Application Layer

### Open WebUI

Open WebUI is the primary user-facing interface for LLM use.

Its responsibilities include:

- chat interface
- conversation history
- model selection
- knowledge/document workflows
- application settings
- interaction with external inference endpoints
- integration with search services

Open WebUI is hosted on `elite-node`, not on the GPU compute node.

That separation is intentional. Open WebUI does not need local GPU access because it communicates with the inference service over an API.

### PostgreSQL

Open WebUI stores persistent application data in PostgreSQL inside the Open WebUI container.

The database is part of the application layer, not the GPU layer.

This means model-serving changes do not require migration of conversation history or application state.

### SearXNG

SearXNG is also hosted on `elite-node`.

It provides a search service that can be consumed by Open WebUI for workflows that benefit from current web information.

Its failure does not prevent ordinary local model inference from working.

## Compute Layer

The primary GPU compute system is `ai-node-01`.

It runs Proxmox VE and provides isolated environments for AI workloads.

The two major workload classes are:

- LLM inference
- image generation

The same physical MI60 is used for both workload classes, but the software stacks and workload environments are separate. This does not require the two workloads to run simultaneously.

### LLM Compute Path

```text
Open WebUI
   |
   | API request
   v
llama-server
   |
   v
llama.cpp Vulkan backend
   |
   v
Mesa / RADV
   |
   v
AMDGPU
   |
   v
AMD Instinct MI60
```

### Image-Generation Compute Path

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

## Why the Compute Layer Is Separate

A common small-scale AI deployment puts everything on one host:

```text
GPU
+ model server
+ frontend
+ database
+ search
+ model files
```

That is simple initially, but tightly couples unrelated services.

This platform instead treats the inference engine as a backend service.

Benefits include:

- the frontend can survive compute-node maintenance
- compute-layer changes do not require migrating the application database
- llama.cpp can be rebuilt or upgraded independently
- image-generation dependencies cannot easily interfere with LLM dependencies
- compute services can be tested directly without involving Open WebUI
- the architecture can later support additional compute nodes or inference endpoints

## Service Boundaries

### Open WebUI

Responsible for:

- user interaction
- conversations
- model selection
- application state
- knowledge workflows

Not responsible for:

- loading GGUF models directly
- GPU driver management
- Vulkan configuration
- ROCm configuration
- direct GPU scheduling

### llama.cpp

Responsible for:

- loading GGUF models
- quantized inference
- GPU offload
- context allocation
- exposing the model through an API
- token generation

Not responsible for:

- long-term user accounts
- primary conversation management
- web search
- application database storage

### ComfyUI

Responsible for:

- diffusion workflows
- image-generation pipelines
- model selection for diffusion workloads
- graph-based workflow execution

Not responsible for:

- LLM serving
- Open WebUI data
- search services

### SearXNG

Responsible for:

- metasearch
- providing search results to supported workflows

Not responsible for:

- model inference
- application persistence
- GPU compute

## Storage Layer

Large AI models are stored centrally rather than being permanently tied to a particular workload container.

Conceptually:

```text
Central Model Storage
|
+-- LLM Models
|   +-- GGUF
|
+-- Diffusion Models
    +-- Checkpoints
    +-- LoRAs
    +-- VAE
```

This gives several advantages:

- containers can be rebuilt without re-downloading the full model library
- model data is easier to back up independently
- multiple workloads can consume the same storage
- local container disks remain smaller
- model organization is consistent across the platform

## Failure Domains

The platform is not highly available, but service separation creates useful failure boundaries.

### If `ai-node-01` is unavailable

Open WebUI, PostgreSQL, and SearXNG can remain online.

The interface may still load, but local inference requests cannot complete until the compute endpoint returns.

### If Open WebUI is unavailable

llama.cpp can continue running.

The inference API remains independently testable and usable by another compatible client.

### If SearXNG is unavailable

Local LLM inference still works.

Only search-enhanced workflows are affected.

### If ComfyUI is unavailable

LLM inference remains unaffected.

The image-generation environment is isolated from the main chat workflow.

### If centralized model storage is unavailable

Already-loaded models may continue to run, but model loading, switching, or new workload startup may fail depending on what data is already local.

## Proxmox and LXC Role

Proxmox provides the virtualization layer for `ai-node-01`.

LXC is used to keep workload-specific software separate while avoiding the overhead of a full virtual machine where it is unnecessary.

The general model is:

```text
Physical Hardware
    |
    v
Proxmox VE
    |
    v
LXC Workload Environment
    |
    v
AI Application
    |
    v
GPU Device Access
```

The Proxmox host is kept focused on virtualization and hardware management.

Workload-specific software belongs inside the workload environment whenever practical.

## Extensibility

Because the application layer communicates with compute services over APIs, the architecture is not tightly bound to one frontend-to-backend implementation.

Additional inference endpoints or AI services could be integrated if needed without requiring the application layer to be redesigned. This is an architectural capability rather than a statement of planned hardware changes.
