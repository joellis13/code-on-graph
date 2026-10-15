# ADR-0018: cog runs in WSL2 on Windows; Implementation executes in containers

- **Status:** Accepted
- **Date:** 2026-10-15
- **Supersedes, in part:** [ADR-0011](0011-platform-support-and-implementation-execution-environment.md): the platform tiers, where Planning runs on Windows, and the Implementation execution environment. Everything not listed here stays in force (see "Unchanged from ADR-0011").

## Context

ADR-0011 chose native Windows for Planning and WSL2 or containers for Implementation. Owner review (2026-10-13 to 2026-10-15) found that this keeps two environments alive: two data directories and credential setups, or `cog` on Windows dispatching into WSL2 across the Windows/Linux file boundary, which is slow and brings CRLF, permission, and worktree problems. It also keeps a growing list of Windows-specific engineering in the design.

Two facts settle it. Implementation needs a Linux environment anyway. And Planning reads **managed clones** ([ADR-0008](0008-standalone-command-center-managed-clones.md)), never the owner's working copies, so where those working copies live does not constrain where `cog` runs.

## Decision

- **Windows:** `cog` runs **inside a WSL2 Linux distribution**. Ubuntu LTS is recommended, ideally a dedicated instance for `cog`. Git Bash, cmd, and PowerShell reach it through `wsl`, or the user works in a WSL terminal. The data directory (clones, indexes, mirrors, databases) lives on the distro's own filesystem, never under `/mnt/*`.
- **Linux and macOS:** `cog` runs natively.
- **Native Windows is best-effort.** A smoke job in CI keeps the code portable. Nothing more is promised.
- **Implementation** runs repo commands and agent edits in a **container per task**. Planning needs no container because it executes no repo code.
- **WSL2 is an environment, not a sandbox.** By default it can reach Windows drives. Containment comes from containers.
- **`ExecutionEnvironment`** has two implementations, `LocalProcess` and `Container`. The `Wsl` implementation from ADR-0011 is dropped because `cog` already runs inside WSL2.
- The risks of this choice (data loss, resource limits, network quirks) are accepted with the mitigations in [Platforms and environments](../explanation/platforms-and-environments.md). That page also holds the platform tiers, the table of what runs where, and per-OS usage.

## Unchanged from ADR-0011

Commands are argument lists, never shell strings. Stored paths are POSIX-style, and Git Bash-style `/c/...` paths are never accepted internally. The path deny-list compares case-folded paths. Files are read through git blobs. Managed clone directory names are short and hash-based. The data directory is not in a cloud-synced folder. Every runner has a live contract test. Timeouts kill the whole process tree. All execution goes through `ExecutionEnvironment`.

## Alternatives considered

- **Keep ADR-0011** (native Windows for Planning, WSL2 for Implementation): two environments to maintain, and a file-boundary problem or duplicated state.
- **Native Windows everywhere:** leaves Implementation without containment.
- **`cog` itself in a container:** adds credential, volume, and nested-container complexity for little gain. The CLI is normally installed natively; the sandbox is the container.
- **Docker Desktop as the only path on Windows:** works, but has licensing terms for larger organizations; Docker Engine inside WSL2 avoids it **(verify)**.

## Consequences

- Windows-specific machinery shrinks. Linux becomes the primary CI target.
- Windows users need WSL2. The docs must say how this differs from Git Bash, cmd, and PowerShell use.
- Still open, and carried into the Implementation design: container image strategy, network profile enforcement, and the needs of UI-test repositories.

## Revisit when

WSL2 proves unreliable for daily use, collaborators on native Windows need first-class support, or a container-native workflow replaces the WSL2 host.
