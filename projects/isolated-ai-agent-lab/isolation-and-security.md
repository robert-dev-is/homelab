# Isolation and Security

## Security Objective

The AI Agent Lab assumes that an autonomous agent may make incorrect decisions, execute unexpected commands, install unnecessary software, generate excessive network traffic, or modify a system in ways that require recovery.

The security model therefore focuses on **containment and recoverability** rather than trusting the agent to behave perfectly.

## Dedicated AI Network

The agent environment runs on a separate VLAN from trusted homelab systems.

The isolated network can be given:

- Internet access where required
- narrow access to the local inference endpoint
- specific access to disposable target systems
- explicitly approved access to selected services

It is not given unrestricted reachability into the trusted LAN.

## Capability-Based Access

A recurring design principle is:

> Give the agent the capability it needs, not broad network membership.

Examples include allowing only:

- SSH to a specific sandbox host
- HTTPS or SSH to a specific Git service
- the local inference API
- outbound Internet access for research

This reduces the blast radius of mistakes while still allowing useful autonomous work.

## Disposable Targets

Systems intended for autonomous administration are treated as disposable.

The agent may be allowed sudo privileges inside those targets because the environment is specifically created for experimentation, testing, development, service deployment, and problem solving.

Recovery options include:

- Proxmox snapshots
- VM restore
- reinstallation
- replacement with a clean template

Broad privileges are therefore confined to systems where destructive mistakes are acceptable.

## Credential Scope

Credentials used by the agent should be valid only for the sandbox or for narrowly scoped services.

Public project documentation must not contain passwords, tokens, API keys, or other reusable credentials.

Examples of safer patterns include:

- dedicated sandbox user accounts
- dedicated SSH keys
- repository-scoped Git credentials
- service-specific accounts
- credentials that do not work on trusted infrastructure

## Network Behavior Findings

One experiment showed that autonomous web-search behavior can generate enough outbound requests to overwhelm a small router.

That reinforced an important lesson:

> Tool autonomy needs resource limits in addition to network isolation.

Useful controls include:

- request-rate limits
- concurrency limits
- command timeouts
- maximum retries
- maximum agent turns
- explicit approval for high-impact actions

## Human Approval

The sandbox is designed so the agent can operate freely where failure is cheap while still leaving consequential changes under human control.

Examples of actions that should remain deliberate include:

- expanding firewall access
- giving the agent new credentials
- connecting new trusted services
- exposing internal services externally
- changing production infrastructure

## Filesystem Trust Model

Long-running agents can accumulate stale or compressed context.

During the autonomous coding experiment, the agent occasionally remembered earlier versions of source files or inferred symbols that no longer existed.

The recovery rule became:

> When agent memory and the filesystem disagree, the current file on disk wins.

The agent was repeatedly able to recover by re-reading the actual source instead of continuing to debug an outdated mental model.

## Why the Agent Is Not Placed on the Trusted LAN

Putting an autonomous tool-using agent directly on the trusted network would make experiments easier, but it would remove one of the most important parts of the project: designing realistic boundaries.

The isolated architecture demonstrates that useful autonomous workflows can be built without granting unrestricted visibility or control over the rest of the environment.

## Security Lessons

- Isolation should be designed before autonomy is added.
- Root or sudo can be acceptable inside intentionally disposable systems.
- Network access should be explicit and narrow.
- Credentials should be scoped to the agent's actual task.
- Tool limits matter as much as firewall rules.
- Snapshots and rebuild paths reduce the cost of experimentation.
- Agent conclusions should be verified against actual system state.
- A sandbox should make failure informative rather than dangerous.
