# Agent Stack and Tooling

## Overview

The AI Agent Lab separates the autonomous agent harness from the model-serving infrastructure.

```text
Hermes Agent
     │
     │ OpenAI-compatible API
     ▼
llama.cpp
     │
     ▼
Qwen model
     │
     ▼
AMD Instinct MI60
```

This keeps the agent runtime lightweight while allowing inference to run on dedicated GPU compute.

## Hermes Agent

Hermes Agent is the main autonomous execution harness.

It provides the model with access to tools such as:

- terminal commands
- file reads and writes
- patch operations
- browser interaction
- web research
- long-running process management
- project and task workflows

The value of the harness became especially clear during long-running software-development experiments. The same model could continue working across tool failures, inspect files, patch bugs, build tests, and recover from stale assumptions.

## Local Model Serving

The agent uses a locally served Qwen model through llama.cpp.

The inference endpoint is exposed through an OpenAI-compatible API, allowing Hermes to use the local model as a custom provider.

Benefits of local inference include:

- no per-token API bill
- control over model choice and quantization
- control over context size
- ability to run extremely long experiments
- privacy for local project data
- freedom to test inefficient or failure-prone workflows without financial pressure

## GPU Compute

The inference node uses an AMD Instinct MI60 with 32 GB of HBM2.

For the long-running coding experiment, context size was pushed high enough to use nearly all available VRAM while keeping inference on the GPU.

This made the experiment useful not only as an agent test but also as a practical stress test for:

- local inference
- context-window sizing
- VRAM allocation
- long-duration GPU workloads
- agent/model interaction under sustained load

## tmux

Hermes is run inside `tmux` for persistence.

Typical workflow:

```bash
tmux new -s hermes
```

Detach:

```text
Ctrl+B, D
```

Resume:

```bash
tmux attach -t hermes
```

This allows the agent to continue working even when the SSH client disconnects.

It also makes it practical to start a task on one client and inspect it later from another.

## SSH

The agent can SSH into disposable systems in the isolated lab.

This allows experiments that look much more like real administration than a simulated terminal:

- package management
- service configuration
- scripting
- log inspection
- file transfers
- cron/system automation
- troubleshooting

An early lab target was intentionally provided with SSH and sudo access so the agent could modify the environment while pursuing assigned goals.

## Browser Tooling

Hermes can use browser automation for:

- loading generated applications
- checking browser-console output
- interacting with pages
- runtime verification
- screenshots
- web research

The voxel-game case study exposed an important limitation: browser/headless GPU behavior in a VM can differ from a normal hardware-accelerated client.

This required the agent to distinguish between:

- application bugs
- browser automation failures
- headless GPU limitations
- actual user-visible behavior

## Telemetry and Test Harnesses

When browser vision and rendering checks became unreliable, the agent created alternative validation methods.

These included:

- runtime logging
- telemetry beacons
- Node-based test harnesses
- mocked Three.js / DOM environments
- direct execution of game logic
- performance timing
- targeted feature tests

This allowed the agent to verify internal behavior even when it could not visually inspect the result.

## Context Compression

Long-running Hermes sessions periodically compress context.

This became a major constraint during multi-hour and multi-day work.

Observed behaviors included:

- repeated context compression
- compression-summary timeouts
- stale remembered source
- context growing beyond the displayed working budget
- output truncation during large tool calls

The project eventually adopted fresh-session handoffs rather than trying to preserve one indefinitely growing conversation.

## Fresh-Session Handoffs

For large projects, a new Hermes session can be given:

1. the original requirements
2. current project files
3. a concise handoff summary

The new agent then inspects the current filesystem and continues.

This keeps context cleaner while preserving actual project state.

## Tooling Lessons

- The harness matters almost as much as the model.
- Filesystem persistence is critical for long-running projects.
- Tool calls should be small enough to avoid output truncation.
- Long-running jobs need terminal/session persistence.
- Browser automation alone is not sufficient for verification.
- Automated tests give agents a much stronger feedback loop.
- The agent should be encouraged to re-read current files rather than trust stale context.
