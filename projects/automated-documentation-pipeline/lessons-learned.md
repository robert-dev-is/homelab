# Documentation Automation Lessons Learned

## Automating Every Save Is Not the Same as Preserving Useful History

The objective was not to produce the largest possible number of commits.

A useful infrastructure history should capture meaningful documentation states. Adding a stability window between change detection and commit creation prevents an active editing session from turning into a stream of intermediate snapshots.

The resulting history is easier to review because commits represent groups of completed changes rather than individual saves.

## Separate the Automation Engine From the Repository Service

Forgejo and the tracker have different responsibilities.

Forgejo hosts repositories and provides history, diffs, and repository access. The tracker decides when documentation is stable enough to commit and performs the automated Git workflow.

Keeping those roles in separate containers makes the architecture easier to reason about and allows either service to be maintained without combining unrelated responsibilities.

## Keep Primary Documentation Independent of Git Hosting

The working Markdown files live on TrueNAS rather than inside the Forgejo container.

This prevents repository-service availability from becoming a requirement for reading or editing the documentation. Forgejo preserves history, but it is not the primary document store.

That distinction also creates a useful failure boundary: a Forgejo outage interrupts automated pushes without making the documentation itself unavailable.

## Native Linux Scheduling Is Enough for This Workload

The tracking problem does not require a large orchestration platform.

A Bash script combined with a systemd timer and service provides:

- predictable scheduling
- normal service status reporting
- centralized logs
- straightforward restart and troubleshooting behavior
- no permanently running custom process

Using smaller, familiar components keeps the workflow transparent.

## Track Documentation, Not Editor Noise

Obsidian is useful as the editing interface, but its workspace metadata does not belong in infrastructure history.

Ignoring `.obsidian/`, nested Obsidian metadata, and trash directories keeps the repository focused on the Markdown content that describes the lab.

## Private History and Public Portfolio Serve Different Purposes

The automated private repository is intentionally detailed because it acts as long-term infrastructure memory.

The public portfolio is intentionally selective. Projects are extracted from that private history and rewritten around architecture, design decisions, troubleshooting, technologies used, and measurable results.

Keeping those repositories separate allows automation to preserve operational depth without turning the public portfolio into a mirror of every internal note.

## Main Takeaway

The most important design decision was not the choice of Forgejo or systemd by itself. It was treating documentation as infrastructure with its own storage, automation, version history, dependencies, and failure boundaries.

The pipeline removes the need to remember routine Git snapshots while still producing history that is useful enough to revisit later.
