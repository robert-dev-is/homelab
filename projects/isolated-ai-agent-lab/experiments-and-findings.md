# Experiments and Findings

## Purpose

The AI Agent Lab is used to test how a locally hosted autonomous agent behaves when it can interact with real systems rather than only answer questions.

The focus is not on one benchmark score. It is on observing practical behavior:

- Can the agent administer systems?
- Can it recover from failed commands?
- Can it troubleshoot?
- Can it maintain long-running work?
- Can it validate its own results?
- What limits appear when a task lasts for hours or days?

## Experiment: Dedicated Linux Lab Host

A dedicated Debian VM was created specifically for agent experimentation.

The agent was given:

- SSH access
- a sandbox user account
- sudo privileges
- permission to install software
- permission to modify the system
- permission to use the environment for testing and development

The agent successfully connected over SSH and created its own SSH key for continued access.

### Finding

Giving an agent real shell access changes the nature of the experiment. The model must work with actual package managers, permissions, services, files, and errors rather than an imagined environment.

It also reinforces why the target must be isolated and disposable.

## Experiment: Cross-Platform Administration

The agent was tested against more than one Linux environment.

Tasks included:

- package installation
- service configuration
- script creation
- cron-style automation
- log handling
- storage-related administration
- troubleshooting failed commands

### Finding

The agent can adapt between environments, but it still benefits from clear system boundaries and verification.

Different distributions expose different tooling and conventions, so the agent must inspect the system rather than assume that commands from one distribution apply everywhere.

## Experiment: SMART-Test Automation

One experiment asked the agent to implement recurring SMART testing and logging.

The agent:

- created a test workflow
- scheduled execution
- wrote logs
- copied results to another target
- attempted to verify the result

The verification phase became overly persistent and consumed much more time than necessary.

### Finding

Autonomous agents may need explicit limits around:

- command runtime
- retries
- repeated verification
- maximum turns
- web requests
- concurrency

A prompt telling the agent to "be quick" is not as reliable as real harness-level limits.

## Experiment: Web Research Load

Autonomous web research created enough outbound traffic to overload a small router during testing.

### Finding

Network isolation alone is not sufficient.

An agent may remain inside allowed boundaries while still creating excessive load.

Useful controls include:

- request rate limiting
- concurrency limits
- retry limits
- network monitoring

## Experiment: Long-Running tmux Workflow

Hermes was run inside `tmux` while accessed over SSH.

The user could disconnect and reconnect later while the agent continued operating.

### Finding

This provides a practical model for centralized autonomous compute:

```text
thin client / laptop
        │
       SSH
        │
        ▼
agent VM + tmux
        │
        ▼
local GPU inference
```

The client device becomes primarily an interface while the lab performs the work.

## Experiment: Context Exhaustion

The autonomous coding case study grew large enough to hit several limits:

- per-response output truncation
- repeated context compression
- context histories exceeding the agent's comfortable working budget
- compression timeouts
- maximum iteration limits

### Finding

Large autonomous projects should not depend on one immortal conversation.

The more reliable pattern became:

```text
requirements
     +
current filesystem state
     +
fresh agent session
```

The filesystem serves as durable state while conversation context remains disposable.

## Experiment: Agent Self-Correction

During long-running debugging, the agent sometimes remembered outdated source code or inferred symbols that did not actually exist.

In several cases it responded by:

1. searching the current filesystem
2. recognizing that its assumption was wrong
3. re-reading the authoritative source
4. continuing from the current implementation

### Finding

A capable agent still needs an explicit source of truth.

For software development, the working tree should be treated as authoritative over conversation memory.

## Overall Findings

The project has shown that a local autonomous agent can perform surprisingly substantial work when given:

- real tools
- time
- a durable filesystem
- a controlled network
- disposable systems
- feedback from tests and telemetry

It has also shown that autonomy does not remove the need for engineering controls.

The most reliable results come from combining model capability with:

- isolation
- observability
- repeatable testing
- scoped permissions
- durable state
- human verification
