# Stability-Aware Documentation Automation

## Problem

The homelab documentation changes frequently as systems are built, moved, tested, and reconfigured.

Manually committing every documentation session would make version history dependent on remembering a separate Git workflow. Committing automatically on every file save would solve that problem but create the opposite one: large numbers of low-value commits representing partial edits.

The tracker was built to automate version history while preserving meaningful commit boundaries.

## Workflow

The tracker runs every 10 minutes through a systemd timer.

```text
Documentation change
        |
        v
Git detects modified files
        |
        v
Current change state recorded
        |
        v
Stability period begins
        |
        +---- additional change ----+
        |                           |
        |                           v
        |                    stability resets
        |                           |
        +---------------------------+
        |
        | ~1 hour unchanged
        v
Automated Git commit
        |
        v
Push to Forgejo
```

## Stability Logic

A change is not pushed immediately after detection.

Instead:

1. The scheduled tracker checks the repository for changes.
2. When a changed state is found, the tracker records that state locally.
3. The documentation must remain unchanged for approximately one hour.
4. If another change appears, the stability period resets.
5. Once the changed state remains stable for the full window, the tracker creates a Git commit.
6. The commit is pushed to Forgejo through SSH.

The one-hour window is long enough to group an editing session into a useful snapshot while still capturing changes automatically without requiring a manual commit step.

## Scheduling

The automation uses native systemd services rather than a continuously running custom daemon.

Relevant units:

```text
doc-tracker-homelab.timer
doc-tracker-homelab.service
```

The timer runs the tracking workflow every 10 minutes.

This keeps the implementation simple:

- the operating system owns scheduling
- the script can run, evaluate state, and exit
- service status and execution history remain visible through normal Linux tooling
- the automation does not require a separate orchestration platform

## Git Authentication

Automated pushes use SSH key authentication to the Forgejo SSH service.

This allows non-interactive pushes from `doc-tracker-01` without embedding interactive credentials into the workflow.

## Ignored Editor State

The repository excludes Obsidian workspace metadata and trash directories:

```gitignore
.obsidian/
**/.obsidian/
.trash/
**/.trash/
```

The purpose of the repository is to preserve infrastructure documentation, not editor-specific state or discarded files.

## Result

The Git history becomes a sequence of stable documentation snapshots rather than a record of every small save.

That makes the repository useful for:

- reviewing how infrastructure changed over time
- comparing previous and current documentation
- recovering older documentation states
- understanding when architectural decisions were introduced
- maintaining long-term infrastructure history without adding a manual Git step to normal documentation work
