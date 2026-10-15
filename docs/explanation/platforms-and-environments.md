# Platforms and Environments

**Type:** Explanation · **Status:** Draft v1 · **Last updated:** 2026-10-15
**Decision:** [ADR-0018](../decisions/0018-wsl2-home-and-container-execution.md)
Items marked **(verify)** are tracked in the [roadmap's verification log](../roadmap.md#4-verification-log). Commands and settings named here are illustrative until the CLI exists.

Where `cog` runs, where its data lives, and what runs in a container. This page is the one place for platform tiers and the "what runs where" chart; other pages link here.

---

## 1. Platform tiers

| Tier | Platform                                   | Commitment                                                               |
| ---- | ------------------------------------------ | ------------------------------------------------------------------------ |
| 1    | Linux, and Windows through WSL2            | Primary. Built, tested in CI, and used daily                             |
| 2    | macOS                                      | Supported. Best-effort CI                                                |
| 3    | Native Windows (Git Bash, cmd, PowerShell) | Best-effort. A smoke job in CI keeps the code portable; nothing promised |

On Windows, `cog` is a Linux program that happens to be started from Windows. It is not ported to Windows.

---

## 2. What runs where

| Activity                                                    | Windows                                                            | Linux                   | macOS                                                 |
| ----------------------------------------------------------- | ------------------------------------------------------------------ | ----------------------- | ----------------------------------------------------- |
| `cog` CLI, Python, uv                                       | Inside the WSL2 distro                                             | Native                  | Native                                                |
| Data directory (plans, databases, indexes, mirrors, clones) | The distro's own filesystem, `~/.local/share/cog`                  | `~/.local/share/cog`    | `~/Library/Application Support/cog`                   |
| GitHub and model-provider calls                             | From WSL2                                                          | Native                  | Native                                                |
| Planning (every node)                                       | WSL2, no container (no repo code executes)                         | Native                  | Native                                                |
| Review page                                                 | Opened in the Windows browser through a path or helper             | `xdg-open`              | `open`                                                |
| Backups                                                     | Opt-in backup directory; `wsl --export` for the whole distro       | Backup directory        | Backup directory                                      |
| Implementation: the `NativeRunner` loop and its model calls | In the `cog` process, in WSL2                                      | Native                  | Native                                                |
| Implementation: repo edits, tests, lint, build              | **Container per task**                                             | **Container per task**  | **Container per task** (the runtime hides a Linux VM) |
| Container runtime                                           | Docker Engine inside WSL2, or Docker Desktop with WSL2 integration | Docker Engine or Podman | Docker Desktop or another Docker-compatible runtime   |

Containers are an Implementation concern only. A container gives each task the repo's own toolchain (JDK and Gradle versions, Node, Python), resource limits, a network profile, and a scrubbed environment ([Policy and sandbox](policy-and-sandbox.md), Section 11).

---

## 3. Using `cog` on each system

### Windows

You have two equivalent ways to work:

1. **Open a WSL terminal** (a Windows Terminal tab for the distro, or VS Code's WSL integration) and run `cog …` as on Linux.
2. **Call it from your usual shell** with `wsl cog …`. This works from Git Bash, cmd, and PowerShell, and standard output and exit codes pass through, so scripts keep working.

| Usual tool            | What changes                                                                                                                                            |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Git Bash              | `cog` is not installed there. Use `wsl cog …`, or a WSL terminal. Paths are Linux paths inside WSL2; Git Bash `/c/...` paths are never accepted         |
| cmd and PowerShell    | Use `wsl cog …`. Quoting is applied by your shell first and then by WSL, so prefer simple arguments and files over long quoted strings                  |
| Editor                | Your own working copies stay where they are. `cog` does not read them; it works on managed clones inside WSL2                                           |
| Review page           | `cog` prints a path inside WSL2. Open it from Windows through the distro's network share, or with an opener helper **(verify the exact path form)**     |
| Credentials           | `gh auth login` and provider keys are set up inside WSL2 (environment variables in the shell profile). Windows-side logins are not shared automatically |
| Agent CLIs (optional) | Copilot and Antigravity CLIs, if used, are installed and signed in inside WSL2                                                                          |
| Adding a repo         | Use a repository URL. A Windows path would be read through the slow `/mnt/c` mount                                                                      |

### Linux

Install and run natively. Any mainstream distribution works; CI uses Ubuntu LTS.

### macOS

Install and run natively. Planning needs nothing else. Implementation needs a Docker-compatible runtime for task containers.

---

## 4. WSL2 setup recommendations

- **Distribution:** the current Ubuntu LTS release (it has the widest tool and documentation coverage, and CI uses it). Prefer a **dedicated distro instance for `cog`**, so backups, resource settings, and isolation settings apply to `cog` alone **(verify how to create a second instance on the installed WSL version)**.
- **Resources:** set explicit `memory` and `processors` limits for the WSL2 VM in the Windows-side `.wslconfig`, sized to the machine, so index builds and parallel tasks cannot starve Windows **(verify defaults and keys)**.
- **Data directory:** keep it on the distro's own filesystem. Do not point `COG_HOME` at `/mnt/c/...`: SQLite's write-ahead mode is known to be unreliable on that mount **(verify)**.
- **Isolation (optional hardening for Implementation):** WSL2 can reach Windows drives and launch Windows programs by default. A dedicated distro can turn automount and interop off in `/etc/wsl.conf`, at the cost of the helper that opens the review page and of writing backups to a Windows folder **(verify the settings)**. Containers remain the real boundary.

---

## 5. Risks and mitigations

| Risk                                                              | Mitigation                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Losing the distro (unregister, reset, corrupt disk)               | Only `plans/` and approvals are irreplaceable; everything else rebuilds from GitHub or git. **Opt-in backup directory:** each approved plan version is copied there (plans are immutable files, so a synced folder is safe), and `cog backup` writes a consistent snapshot of `cog.db`. `cog doctor` warns when no backup is configured. Document `wsl --export` for whole-distro backups. `cog plan export` is another copy. `cog gc` never deletes `plans/` |
| WSL2 starving Windows of memory or CPU                            | Recommended `.wslconfig` limits (Section 4). `cog doctor` reports memory, CPUs, and free disk inside WSL2. Index builds use a bounded worker count and a time budget ([Indexing](indexing.md)); parallel tasks are bounded by config                                                                                                                                                                                                                          |
| Disk growth (the virtual disk grows and may not shrink by itself) | `cog doctor` reports data-directory size by category. `cog gc` prunes indexes, clones, and worktrees with size accounting **(verify shrink behavior)**                                                                                                                                                                                                                                                                                                        |
| Network quirks (VPN, proxy, DNS inside WSL2)                      | `cog doctor` checks that GitHub and each provider are reachable from inside WSL2 and names the failing host. Known workarounds are documented in the install how-to once it exists                                                                                                                                                                                                                                                                            |
| Agents reaching Windows files from WSL2                           | Containers are the sandbox for Implementation. Optional automount and interop hardening (Section 4)                                                                                                                                                                                                                                                                                                                                                           |
| Native Windows drifting out of working order                      | A smoke job runs the unit tests on Windows in CI. Portability rules (Section 6) stay in force                                                                                                                                                                                                                                                                                                                                                                 |

---

## 6. Portability rules the design keeps

These cost little and keep native Windows, macOS, and the Windows-authored repos `cog` reads working.

| Area           | Rule                                                                                                                                                                 |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Paths          | `pathlib` everywhere. Stored paths are POSIX-style. Git Bash `/c/...` paths are never accepted. The deny-list compares case-folded paths                             |
| Repo content   | Read through git blobs, so line-ending conversion cannot change spans or hashes. Repos written on Windows and macOS (CRLF, case-insensitive names) are still handled |
| Subprocesses   | Argument lists, never shell strings. Timeouts kill the whole process tree                                                                                            |
| Clone paths    | Short, hash-based names; no reliance on symlinks                                                                                                                     |
| Terminal       | Plain prompts and an explicit `cog resume`; no dependence on a rich terminal interface                                                                               |
| Data directory | Resolved with `platformdirs`; never in a cloud-synced folder                                                                                                         |
| CI             | Linux is primary; a Windows smoke job; macOS best-effort                                                                                                             |

Windows-only measures (`core.longpaths`, resolving `mvnw.cmd`, `gradlew.bat`, and `npm.cmd`, the full list of Windows path traps) apply only where `cog` runs natively on Windows. They are kept where cheap and are not promised.

---

## 7. `ExecutionEnvironment`

All command execution goes through one interface.

| Implementation | Used for                                                                 |
| -------------- | ------------------------------------------------------------------------ |
| `LocalProcess` | Planning-side git and tooling; development; running inside WSL2 or Linux |
| `Container`    | Implementation tasks; later, remote workers                              |

---

## 8. Containers for Implementation: what is decided and what is open

Decided ([ADR-0018](../decisions/0018-wsl2-home-and-container-execution.md)): one container per task; Planning uses none; `cog` itself is not containerized.

Open, for the Implementation design ([outline](../workflows/implementation/outline.md)):

- Image strategy: use a repo's own dev container definition when it has one, or `cog`-provided base images per ecosystem.
- Network profile enforcement (`none`, `package-registry-only`, `full`).
- Repos whose tests need more than a toolchain: browser-driven suites (Selenium, Playwright) need browsers and display handling in the image, and a system under test that may be on the network, with credentials passed by name from the user's environment.
- Docker Desktop licensing for larger organizations, versus Docker Engine inside WSL2 **(verify)**.
- Bind-mount performance on macOS.
