# AI Agent Lab Architecture

## Design Goal

The AI Agent Lab was designed to let an autonomous local AI agent interact with real systems while keeping the experiment separated from trusted homelab infrastructure.

The core design principle is:

> Give the agent useful capabilities inside a controlled environment rather than broad membership in the trusted network.

## High-Level Architecture

```text
                              TRUSTED HOMELAB
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  ┌────────────────────┐       ┌────────────────────────────────┐   │
│  │ Local AI Compute   │       │ Other Homelab Infrastructure  │   │
│  │ Qwen + llama.cpp   │       │ Proxmox / NAS / services      │   │
│  │ AMD Instinct MI60  │       │                                │   │
│  └─────────▲──────────┘       └────────────────────────────────┘   │
│            │                                                       │
└────────────┼───────────────────────────────────────────────────────┘
             │ narrowly allowed inference traffic
             │
     ┌───────┴────────────────────────────────────────────────┐
     │                 ISOLATED AI NETWORK                    │
     │                                                        │
     │  ┌──────────────────────┐      ┌─────────────────────┐ │
     │  │     ai-agent-01      │ SSH  │  Disposable Targets │ │
     │  │ Debian + Hermes      ├─────►│ Linux / test VMs    │ │
     │  └──────────────────────┘      └─────────────────────┘ │
     │                                                        │
     └────────────────────────────────────────────────────────┘
```

## Components

### Agent VM

`ai-agent-01` is the main execution environment for Hermes Agent.

Responsibilities:

- run Hermes Agent
- hold agent configuration and project state
- provide terminal and browser tooling
- maintain long-running sessions
- initiate SSH connections to disposable test systems
- call the local model-serving endpoint

The agent VM is separate from the GPU inference system. This keeps the agent harness and the model-serving stack independently manageable.

### Local AI Compute

Inference is hosted on a dedicated AI node using an AMD Instinct MI60 with 32 GB of HBM2.

The model is served through a llama.cpp OpenAI-compatible endpoint. Hermes connects to that endpoint as a custom model provider.

This separation provides several advantages:

- the agent VM can be rebuilt without rebuilding the inference stack
- the inference node can be tuned independently
- multiple clients can potentially use the same model endpoint
- the agent does not need direct hardware access to the GPU

### Disposable Target Systems

The sandbox includes systems intended specifically for the agent to modify.

A typical target can provide:

- SSH access
- a dedicated sandbox account
- sudo privileges
- permission to install software
- permission to change configuration
- permission to deploy and break services

These systems are intentionally disposable. If an experiment goes badly, they can be reverted, restored, or rebuilt.

## Network Segmentation

The agent environment is placed on a dedicated VLAN rather than the normal trusted LAN.

Traffic between the AI network and the trusted lab is controlled by firewall policy.

The design favors **specific allowed capabilities** over broad access. For example, an agent may be allowed to reach:

- the local inference API
- a specific Git service
- a specific SSH target
- the public Internet

without receiving unrestricted access to other homelab systems.

## Physical and Virtual Separation

The AI network is connected through a dedicated physical interface into a dedicated Proxmox bridge. The agent VM is attached only to that isolated bridge rather than the normal trusted bridge.

This keeps network separation understandable at both the physical and virtual layers.

## Long-Running Session Architecture

Hermes is commonly run inside `tmux`.

```text
SSH client
   │
   ▼
tmux session
   │
   ▼
Hermes Agent
   │
   ├── terminal tools
   ├── browser tools
   ├── filesystem
   └── local model API
```

This allows:

- SSH clients to disconnect without terminating the agent
- a session to be resumed from another machine
- multi-hour jobs to continue unattended
- long experiments to remain visible and inspectable

## Project-State Persistence

One of the most important findings from long-running work was that the filesystem is a better source of truth than an indefinitely growing conversation.

For large projects, the workflow evolved into:

```text
Agent Session 1
      │
      ▼
Project files on disk
      │
      ▼
Agent Session 2 reads:
  - original requirements
  - current project files
      │
      ▼
continues from current state
```

This became necessary when long sessions accumulated very large context histories and repeated context compression.

The current files on disk are treated as authoritative when they disagree with an agent's remembered version of the project.

## Why This Architecture Matters

The project is not intended to prove that an AI agent should be trusted with unrestricted infrastructure access.

It is intended to test autonomous behavior **inside deliberate boundaries** while preserving:

- recoverability
- observability
- network isolation
- human control
- reproducibility

That makes it possible to explore aggressive levels of agent autonomy without turning the experiment into a risk to the rest of the homelab.
