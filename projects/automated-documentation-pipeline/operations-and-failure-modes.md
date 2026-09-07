# Operations and Failure Modes

## Dependency Chain

The pipeline depends on several independent services:

```text
TrueNAS documentation share
        |
        v
Proxmox host mount
        |
        v
doc-tracker-01
        |
        v
SSH connectivity
        |
        v
git-service-01 / Forgejo
```

The design intentionally keeps failures in the tracking path from becoming failures in the primary documentation store.

## Failure Behavior

### Documentation Share Unavailable

If the NAS-backed documentation mount becomes unavailable:

- the tracker cannot access the repository
- automated snapshots cannot occur
- the existing Forgejo history remains intact

The working documentation depends on the storage platform, not Forgejo.

### Forgejo Unavailable

If `git-service-01` is unavailable:

- Forgejo web access is unavailable
- automated pushes cannot complete
- the NAS-hosted Markdown documentation remains unaffected

The tracker and Git host are separate containers, so repository hosting can be serviced independently of the tracking logic.

## Validation Workflow

A concise validation sequence is enough to isolate most failures in the pipeline.

### 1. Confirm the documentation mount

```bash
ls -la /mnt/docs/homelab
```

### 2. Confirm the Git working tree

```bash
cd /mnt/docs/homelab/Homelab
git status
git remote -v
```

### 3. Confirm the scheduler

```bash
systemctl status doc-tracker-homelab.timer
systemctl list-timers | grep doc-tracker
```

### 4. Confirm the tracking service

```bash
systemctl status doc-tracker-homelab.service
tail -f /var/log/doc-tracker.log
```

### 5. Confirm repository history and remote access

```bash
git log --oneline -5
git push
```

On the Forgejo side, service health can be checked with standard systemd and socket tooling:

```bash
systemctl status forgejo
ss -tulpn | grep 2222
```

## Operational Design

The pipeline deliberately uses conventional Linux components:

- filesystem mounts
- LXC bind mounts
- Bash
- Git
- systemd services and timers
- SSH authentication
- Forgejo

That keeps each layer observable with familiar tools and avoids hiding the workflow behind a larger automation framework.

The system is small enough that a failed snapshot can be traced from storage, to the tracker, to Git state, to network authentication, to Forgejo without requiring a separate control plane.
