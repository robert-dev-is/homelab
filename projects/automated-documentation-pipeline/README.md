# Automated Documentation Pipeline

## Project Overview

This project documents an automated versioning pipeline built to preserve the history of a large, continuously evolving homelab documentation set without creating a Git commit for every small edit.

The private homelab documentation is written in Markdown and stored centrally on TrueNAS. A dedicated Debian LXC container monitors that documentation, waits until a group of changes has remained stable for approximately one hour, then creates a Git snapshot and pushes it to a self-hosted Forgejo instance.

The result is a documentation workflow that keeps the working files on network storage while maintaining useful infrastructure history in Git.

```text
Markdown documentation
        |
        v
TrueNAS documentation share
        |
        v
Proxmox host mount
        |
        v
doc-tracker-01
        |
        | checks every 10 minutes
        v
Change detected
        |
        | ~1 hour without further changes
        v
Git commit
        |
        | SSH
        v
Forgejo
```

The public portfolio repository is separate from this private documentation repository. The automated pipeline preserves the detailed operational history, while the public repository is a curated presentation layer containing selected projects and architecture material.

## Project Goals

- Preserve the history of infrastructure documentation automatically.
- Avoid relying on manual Git commits after every documentation session.
- Prevent rapid editing activity from generating excessive low-value commits.
- Keep Markdown files on centralized NAS storage rather than tying them to one workstation.
- Separate Git hosting from documentation-tracking automation.
- Maintain a self-hosted Git history that can later support selective public publishing and other automation.
- Keep the design simple enough to understand, troubleshoot, and rebuild.

## Core Components

| Component | Role |
|---|---|
| TrueNAS | Central storage for the private Markdown documentation |
| Proxmox VE | Mounts the documentation share and hosts the automation containers |
| `doc-tracker-01` | Detects changes, applies the stability window, creates commits, and pushes updates |
| systemd timer/service | Runs the tracking workflow every 10 minutes |
| Git | Creates historical documentation snapshots |
| `git-service-01` | Hosts the private Forgejo repositories |
| Forgejo | Provides repository hosting, history, diffs, and web access |
| SSH keys | Provide non-interactive authentication for automated pushes |
| Obsidian | Used as the Markdown editing and knowledge interface |

## Why the Stability Window Exists

An immediate commit on every detected file change would preserve history, but it would also record many intermediate edits that have little long-term value.

The tracker instead checks the documentation every 10 minutes. When it detects a change, it records the changed state and starts a stability period. The documentation must remain unchanged for approximately one hour before the tracker creates and pushes a snapshot.

If another edit occurs during that period, the stability timer resets.

This produces Git history closer to completed documentation sessions than individual file-save events.

## Project Documents

- [architecture.md](architecture.md) — storage, container, and Git service architecture
- [automation-workflow.md](automation-workflow.md) — change detection and stability-aware snapshot process
- [operations-and-failure-modes.md](operations-and-failure-modes.md) — dependencies, validation, and failure behavior
- [lessons-learned.md](lessons-learned.md) — design decisions and reusable takeaways

## Status

**Operational.**

The tracker runs through systemd, monitors the NAS-hosted private homelab documentation, and automatically pushes stable documentation snapshots to Forgejo. The system is intentionally separate from the public portfolio repository.
