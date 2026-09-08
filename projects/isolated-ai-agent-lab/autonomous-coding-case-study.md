# Autonomous Coding Case Study: VoxelCraft

## Objective

To stress-test a locally hosted Qwen model running through Hermes Agent, I provided an intentionally excessive software-development prompt:

> Build a fully playable browser-based Minecraft-style voxel game using Three.js and vanilla JavaScript in a single self-contained HTML file.

The prompt required far more than a visual demo. It included procedural world generation, biomes, caves, rivers, mining, placing blocks, inventory, crafting, tools, mobs, lighting, chunk streaming, day/night behavior, and other game systems.

The goal was not to prove that the model could reproduce Minecraft perfectly.

The goal was to see how a local autonomous agent behaved when given a project large enough to exceed a normal single-response coding task.

## Environment

The project ran through:

- Hermes Agent
- local Qwen 3.8 27B-class inference
- llama.cpp
- AMD Instinct MI60 with 32 GB HBM2
- isolated Debian agent VM
- `tmux` for persistent sessions

The work took place primarily under:

```text
/tmp/vg
```

## Development Strategy

Although the final requirement was a single self-contained HTML file, the agent chose to develop the game as a set of intermediate modules and assemble them later.

The working project was split into sections for areas such as:

- HTML/bootstrap
- core definitions
- textures
- world generation
- lighting and meshing
- player physics
- block interaction
- inventory and crafting
- mobs
- effects
- main game loop

This turned out to be one of the most important decisions in the project because it let the agent persist state outside the conversation.

## Context and Output Limits

The experiment quickly exposed practical limits of long-running agent sessions.

Observed problems included:

- model responses hitting the maximum output-token limit
- large `write_file` tool calls being truncated
- Hermes retrying truncated responses
- incomplete tool calls being correctly refused
- repeated context compression
- compression-summary timeouts
- context histories growing extremely large
- sessions reaching the maximum agent-iteration budget

At one point a session accumulated roughly 220K tokens of working history.

Trying to preserve that entire conversation became counterproductive.

## Filesystem-Based Project State

The project files under `/tmp/vg` became the durable source of truth.

Instead of relying on the model to remember every previous decision, later sessions were instructed to:

1. read the original project requirements
2. inspect the current files in `/tmp/vg`
3. determine what had already been implemented
4. continue from the current source tree
5. avoid rewriting completed work unnecessarily

This allowed the project to continue across fresh Hermes sessions without restarting development.

## Multi-Session Handoffs

The project eventually behaved like a sequence of developer handoffs.

### Session 1

Built a large portion of the core game systems but accumulated enough context and output pressure that continuation became increasingly unreliable.

### Session 2

Started fresh from the original requirements plus the existing filesystem.

It inspected the current project, verified module structure, assembled the final HTML, and moved into runtime testing and debugging.

### Later Sessions

Continued testing, profiling, and debugging until they reached Hermes's iteration limits.

Each new session inherited:

- original requirements
- current project state
- previous handoff notes where available

The filesystem, not the conversation, became the persistent project memory.

## Runtime Debugging

One of the first major runtime failures was particularly instructive.

The static start screen rendered correctly, but clicking did not enter the game.

The agent traced the failure through the browser and found a bug in the procedural cobblestone texture generator.

The code created an outer array but attempted to write to an inner row before that row had been initialized.

After patching the texture generator and reassembling the game, the engine progressed much further.

## First Successful Render

The game eventually rendered a real Three.js voxel world.

Observed runtime state included:

- a 3D block world
- chunk generation
- a player spawn
- a crosshair
- health display
- hotbar UI
- biome information
- runtime telemetry

At that stage, the world was visibly much flatter and sandier than intended.

That visual result later matched an automated finding in the terrain-generation tests.

## Automated Test Harness

Browser testing alone was not sufficient.

The agent encountered unreliable vision tooling and headless GPU behavior inside the VM, so it created additional testing methods.

These included:

- runtime telemetry
- browser-console checks
- Node-based game-logic tests
- mocked DOM / Three.js behavior
- feature-specific test cases
- performance probes

The Node test suite eventually covered behavior such as:

- block placement
- deterministic block drops
- torch placement and lighting
- passive-mob spawning
- zombie burning in sunlight
- creeper explosion behavior
- particles and torch embers

One late handoff reported:

```text
25 tests passed
10 tests failed
```

with a full suite runtime of roughly 68 seconds after test-harness optimizations.

## Performance Profiling

The agent did not stop at functional testing.

It also profiled parts of the engine and found that chunk meshing itself was relatively fast in one test while lighting computation dominated more of the workload.

This shifted later investigation toward lighting and world-generation behavior rather than assuming the mesh builder was the primary bottleneck.

## Terrain-Generation Root Cause

One of the most important late findings was that terrain generation was nearly flat.

The agent traced this to the 2D noise path.

Observed noise values remained in a very narrow range around 0.5, producing terrain heights clustered around roughly the same elevation.

That explained multiple downstream symptoms:

- mostly beach/ocean classification
- missing plains and forests
- missing mountains
- little or no grass
- no trees in expected regions
- cave tests behaving strangely because terrain depth was too uniform

The agent identified the 2D noise implementation as a likely upstream cause rather than treating each failed biome test as an unrelated problem.

## Self-Correction

The project also exposed failure modes in the agent itself.

During long sessions, the model sometimes remembered earlier versions of files or assumed that symbols existed when they did not.

A useful behavior emerged:

> when memory and the filesystem disagreed, the agent went back to disk and re-read the current source.

This prevented several debugging paths from turning into patches against imaginary or outdated code.

## What Worked

The experiment demonstrated that a local model and agent harness can:

- sustain multi-hour and multi-day development
- build a modular software project
- persist work outside the model context
- resume from fresh sessions
- inspect inherited code
- find runtime bugs
- patch targeted problems
- create its own test harness
- instrument the application
- profile performance
- correlate test failures with upstream root causes

## What Did Not Work Perfectly

The project also exposed important limitations:

- output truncation during large tool calls
- context bloat
- repeated compression
- stale model assumptions
- overly long validation loops
- premature declarations of success
- unreliable headless/vision testing
- test harnesses that sometimes contained incorrect assumptions
- iteration-budget exhaustion
- remaining gameplay and performance issues

## Why This Case Study Matters

The most interesting result was not the generated voxel game itself.

The important result was the workflow that emerged around it.

A single prompt turned into:

```text
requirements
   ↓
agent implementation
   ↓
filesystem state
   ↓
runtime failures
   ↓
instrumentation
   ↓
automated tests
   ↓
profiling
   ↓
fresh-session handoff
   ↓
continued debugging
```

That is much closer to software engineering than a normal one-shot code-generation demo.

## Current Status

The game has rendered a real voxel world and multiple gameplay systems have been implemented and exercised through automated tests.

The project is still being refined, particularly around terrain generation, performance, and full interactive gameplay verification.

The case study therefore remains intentionally presented as an engineering experiment rather than a claim of a perfect finished Minecraft clone.
