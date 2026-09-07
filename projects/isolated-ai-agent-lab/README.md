# AI Agent Lab

## Overview

The AI Agent Lab is a dedicated, isolated environment for evaluating how far a locally hosted AI agent can go beyond normal chatbot interaction.

The project combines a local large language model, an autonomous agent harness, Linux virtual machines, controlled network access, and disposable test systems. The goal is to study how an agent behaves when it can use real tools, access a shell, modify systems, deploy software, troubleshoot failures, and work on long-running projects while remaining separated from trusted homelab infrastructure.

The environment is intentionally designed as a **sandbox rather than a production automation platform**. The agent can be given broad control inside disposable systems without granting broad access to the rest of the lab.

## What I Built

- A dedicated Proxmox VM for the AI agent
- Hermes Agent as the autonomous agent harness
- Local Qwen inference served from a separate GPU compute node
- A dedicated isolated AI VLAN
- Narrow firewall rules for only the services the agent needs
- Disposable Linux targets for administration and development experiments
- SSH-based access from the agent to sandbox systems
- Persistent terminal sessions with `tmux` for multi-hour and multi-day work
- A workflow for handing long-running projects between fresh agent sessions using filesystem state
- Automated and semi-automated testing workflows for agent-generated software

## Why I Built It

Most AI demonstrations stop at prompt-and-response testing. I wanted to evaluate a different question:

> What happens when a local model is given tools, time, a controlled environment, and real systems to work on?

The lab lets me explore that question without allowing the agent to operate freely on trusted infrastructure.

## Architecture at a Glance

```text
                     ┌──────────────────────────┐
                     │       User / SSH         │
                     └────────────┬─────────────┘
                                  │
                                  ▼
                     ┌──────────────────────────┐
                     │       ai-agent-01        │
                     │  Debian + Hermes Agent   │
                     └───────┬──────────┬───────┘
                             │          │
                inference API│          │SSH / tools
                             │          │
                             ▼          ▼
                  ┌────────────────┐  ┌────────────────────┐
                  │ Local AI Node  │  │ Disposable Targets │
                  │ Qwen + llama.cpp│  │ Linux / test VMs   │
                  │ AMD Instinct GPU│  └────────────────────┘
                  └────────────────┘
                             │
                    controlled network access
```

See [architecture.md](architecture.md) for the full design.

## Major Experiments

### Cross-Platform System Administration

The agent has been given SSH access to disposable Linux systems and allowed to:

- install packages
- configure services
- create scripts
- schedule tasks
- troubleshoot failures
- verify its own work
- adapt between different Linux environments

One early experiment involved having the agent build and troubleshoot a SMART-test workflow on an Alpine-based storage target. The experiment demonstrated both useful autonomy and the need for limits around long-running tool use and repeated verification.

### Long-Running Autonomous Coding

A deliberately excessive coding prompt asked the agent to create a browser-based Minecraft-style voxel game using Three.js and vanilla JavaScript.

The project evolved into a multi-session autonomous development exercise involving:

- procedural terrain generation
- chunk streaming
- voxel lighting
- mining and block placement
- inventory and crafting
- passive and hostile mobs
- creeper explosions
- torch lighting
- runtime telemetry
- automated Node-based testing
- browser testing
- performance profiling
- multi-session handoffs after context or iteration limits were reached

The generated project eventually rendered a real 3D voxel world and progressed into runtime debugging and performance work.

See [autonomous-coding-case-study.md](autonomous-coding-case-study.md).

## Key Engineering Themes

This project is primarily about the infrastructure and control plane around autonomous agents:

- **Isolation before autonomy**
- **Local inference**
- **Least-privilege network access**
- **Disposable infrastructure**
- **Filesystem state as durable project memory**
- **Human verification**
- **Observability and testing**
- **Context-window management**
- **Recovery from failed or stale agent assumptions**

## Technologies

- Proxmox VE
- Debian Linux
- Hermes Agent
- Qwen
- llama.cpp
- AMD Instinct MI60
- SSH
- tmux
- OpenWrt
- VLANs and firewall policy
- JavaScript / Three.js
- Python
- Node.js

## Project Documentation

- [architecture.md](architecture.md) — system architecture and design decisions
- [isolation-and-security.md](isolation-and-security.md) — sandboxing and network controls
- [agent-stack-and-tooling.md](agent-stack-and-tooling.md) — model, harness, tools, and execution workflow
- [experiments-and-findings.md](experiments-and-findings.md) — administration and agent-behavior experiments
- [autonomous-coding-case-study.md](autonomous-coding-case-study.md) — long-running voxel-game development case study
- [lessons-learned.md](lessons-learned.md) — engineering takeaways from the project

## Status

Active project. The environment is being used to continue testing local autonomous agents, long-running workflows, cross-platform administration, and software-development tasks.
