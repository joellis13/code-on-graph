# ADR-0011: Platform support and the Implementation execution environment

- **Status:** Accepted (Planning); **Open** (Implementation environment details)
- **Date:** 2026-10-06

## Context

The owner works primarily on Windows, usually from Git Bash, and wants macOS and Linux to work too. Repos are Java, TypeScript, and Python (back end, front end, test automation).

Planning is Python, git, SQLite, and HTTP, and runs natively on Windows without difficulty. **Implementation** is different: it runs agents that edit files and execute commands. That needs real containment, and Windows offers fewer good options than Linux.

Observations (checked 2026-10-06):

- Copilot CLI and Antigravity CLI work interactively from Git Bash for the owner.
- Third-party guidance rates native Windows support for Copilot CLI as experimental and recommends WSL for the most stable experience. It expects PowerShell 6 or newer, and users report it executes commands through PowerShell even when started from Git Bash. This is secondary evidence; the owner's direct experience is good. The risk is _unattended_ use, where bash-versus-PowerShell command syntax and permission prompts matter and nobody is watching.

## Decision

**Platform tiers**

| Tier | Platform | Commitment                                           |
| ---- | -------- | ---------------------------------------------------- |
| 1    | Windows  | Primary development and manual use; tested in CI     |
| 2    | Linux    | CI; WSL2 as the Implementation execution environment |
| 3    | macOS    | Supported; best-effort CI; not a primary test target |

**Planning** is built and run natively on Windows (Git Bash, PowerShell, or Windows Terminal).

**Implementation execution** moves into **WSL2 or containers**. Until Implementation is designed, the working lean is WSL2 (Linux sandbox tools, faster git and indexing, no 260-character path limits), with containers as the alternative.

To keep the move cheap:

- All command execution goes through an **`ExecutionEnvironment`** interface with `LocalProcess`, `Wsl`, and `Container` implementations. Workflows never spawn processes directly.
- Commands are **argument lists**, never shell strings. Wrapper scripts (`mvnw.cmd`, `gradlew.bat`, `npm.cmd`) are resolved explicitly.
- Timeouts **kill the whole process tree**.
- Each `AgentRunner` config records the **shell** it uses (`bash` or `pwsh`).
- Every CLI/SDK runner has a **live contract test** run on the owner's machine early, to confirm unattended runs behave like interactive ones.
- Stored paths are **POSIX-style**. Git Bash-style `/c/Users/...` paths are never accepted internally. The path deny-list compares case-folded paths (Windows is case-insensitive).
- Managed clone directory names are short and hash-based. `core.longpaths` is set. Symlinks are not relied on.
- Files are read through git blobs so CRLF conversion cannot change spans or hashes.
- The data directory must not live in a cloud-synced folder.
- CI runs a Windows and Linux matrix (macOS best-effort).

## What changes when Implementation moves to WSL2

To be detailed in the Implementation design, but expected to include:

- **Where cog runs:** either entirely inside WSL2 (simplest) or on Windows dispatching into WSL2. The former avoids path translation.
- **Where managed clones live:** in the Linux filesystem for speed.
- **Credentials:** how GitHub and provider keys reach the WSL2 environment.
- **Data directory:** one data directory per environment, or a shared one (decide with the first design pass).
- **Runner compatibility:** re-run all live contract tests inside WSL2.

## Alternatives considered

- **Native Windows for everything, no OS sandbox:** simplest, but leaves Implementation without containment.
- **Linux-only:** conflicts with the owner's primary platform.
- **Docker Desktop from day one:** strong isolation, but extra setup before Planning needs it.

## Consequences

- Planning is not blocked by sandbox decisions.
- The interface boundary costs a little ceremony now and saves a rewrite later.

## Revisit when

Implementation design starts, a runner fails its live contract test, or a native Windows sandbox option becomes viable.
