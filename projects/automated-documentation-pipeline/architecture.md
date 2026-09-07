# Documentation Pipeline Architecture

## System Design

The documentation platform separates three responsibilities:

1. **Primary document storage** — TrueNAS holds the working Markdown files.
2. **Change tracking and automation** — `doc-tracker-01` watches the documentation and creates stable Git snapshots.
3. **Repository hosting** — `git-service-01` runs Forgejo and stores the resulting Git history.

This separation keeps the automation logic independent of the Git service and avoids making Forgejo the primary storage location for the documentation itself.

## Data Path

```text
Workstations / Obsidian
          |
          v
+-----------------------------+
| TrueNAS                     |
| Private Markdown documents  |
+-------------+---------------+
              |
              | SMB documentation share
              v
+-----------------------------+
| Proxmox host                |
| /mnt/pve/docs/homelab       |
+-------------+---------------+
              |
              | LXC bind mount
              v
+-----------------------------+
| doc-tracker-01              |
| Debian 13 LXC               |
|                             |
| /mnt/docs/homelab           |
| Git working tree            |
| tracking script             |
| systemd timer/service       |
+-------------+---------------+
              |
              | Git over SSH
              v
+-----------------------------+
| git-service-01              |
| Debian 13 LXC               |
| Forgejo + Git + SQLite      |
+-----------------------------+
```

The current Git repository root inside the tracker is:

```text
/mnt/docs/homelab/Homelab
```

## Service Separation

### Documentation Tracker

`doc-tracker-01` contains the automation logic rather than the Git hosting service.

Its responsibilities are:

- access the mounted documentation
- detect Git-visible changes
- track whether those changes remain stable
- create automated commits
- push completed snapshots to Forgejo

The primary tracking script is:

```text
/srv/doc-tracker/homelab-docs.sh
```

The workflow is scheduled through:

```text
doc-tracker-homelab.timer
doc-tracker-homelab.service
```

### Forgejo

`git-service-01` provides the repository backend.

Forgejo is responsible for:

- private repository hosting
- Git history
- diffs and change review
- repository web access
- accepting automated SSH pushes from the tracker

SQLite is used for Forgejo metadata to keep the service lightweight and self-contained.

## Storage Independence

The working documentation remains on TrueNAS.

That choice means the Git service is not in the live documentation data path. If Forgejo is unavailable, the Markdown files remain accessible on the NAS and documentation work can continue. The tracker may be unable to push until Forgejo returns, but the documentation itself does not depend on repository availability.

## Repository Boundaries

The automated repository contains the detailed private homelab documentation.

The public portfolio is maintained separately. Material is intentionally selected and rewritten for public presentation instead of automatically publishing the private operational repository.

That boundary allows the private documentation to remain detailed and operational while the public repository stays concise and project-focused.
