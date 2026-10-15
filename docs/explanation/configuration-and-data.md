# Configuration and Data

**Type:** Explanation · **Status:** Draft v1 · **Last updated:** 2026-10-15
**Decisions:** [ADR-0006](../decisions/0006-local-first-storage-no-logging-service.md), [ADR-0007](../decisions/0007-plans-as-immutable-json.md), [ADR-0008](../decisions/0008-standalone-command-center-managed-clones.md), [ADR-0009](../decisions/0009-layered-configuration.md)

Config snippets are illustrative. Directory names are the plan, not an implemented layout.

---

## 1. Layered configuration

Precedence, lowest to highest:

| #   | Layer                                   | Typical owner | Example content                                               |
| --- | --------------------------------------- | ------------- | ------------------------------------------------------------- |
| 1   | Built-in defaults                       | cog           | Sensible limits, empty tiers                                  |
| 2   | User config                             | You           | Providers, tiers, escalation ladder, budgets, preferences     |
| 3   | In-repo `.cog/config.toml` _(optional)_ | The team      | Test, lint, and build commands; paths to deny; language hints |
| 4   | Local per-repo override                 | You           | Personal tweaks for one repo, stored in the data directory    |
| 5   | Environment variables                   | You / CI      | Secrets and one-off switches                                  |
| 6   | CLI flags                               | You           | One-time overrides                                            |

Rules:

- **Secrets are never in config files.** Config names the environment variable (or OS keyring entry) that holds each secret. **(verify)** keyring behavior on Windows Credential Manager, macOS Keychain, and Linux.
- **The resolved config is saved in every run record**, so any run is reproducible and explainable. `cog config show --explain` shows which layer set each value.
- **Repo-supplied config is untrusted input.** It can only _tighten_ policy, never loosen it. Command allow-list entries from a repo require **trust on first use**: cog shows them, you approve, the approval is stored in user data, and any change re-prompts.
- **Repo facts beat user defaults** (repo conventions outrank harness preferences, [ADR-0014](../decisions/0014-repo-convention-precedence.md)); **your local override beats the shared repo file.**
- Style notes ("we prefer loops over streams here") may appear in repo config as free text for the planner, never as code in cog.
- **Commands are argument lists, never strings,** and each carries what a non-interactive run needs. They form the **command table** that acceptance checks reference ([ADR-0019](../decisions/0019-acceptance-checks-reference-the-command-table.md)). Node 9 also discovers entries from build files; config declares or overrides them.

```toml
# illustrative repo config (in-repo or local override)
[commands.test]
argv = ["./gradlew", "test"]
env = { CI = "true" }                            # keeps watch-mode runners non-interactive
timeout_s = 900
selectors = { class = ["--tests", "{value}"] }   # appended to argv for a class selector

[commands.lint]
argv = ["./gradlew", "spotlessCheck"]

[policy]
deny_paths = [".env*", ".github/workflows/**"]

[notes]
style = "Plain loops over streams in the billing module; keep existing patterns."
```

```toml
# illustrative user config additions
[backup]
dir = "/mnt/c/Users/me/cog-backup"     # opt-in; approved plans are copied here (not the data directory)

[acceptance]
allow_custom_argv = false              # ADR-0019: gated escape for commands the table lacks

[scope]                                # ADR-0020; a repo's config may only tighten the mode
mode = "expand"                        # strict, expand, or open
pause_after_expansions = 5
always_allow = ["**/*Test.java", "**/*.test.ts"]
never_allow = ["**/migrations/**"]
```

---

## 2. The data directory

Location comes from `platformdirs` (on Linux and inside WSL2, `~/.local/share/cog`; on macOS, `~/Library/Application Support/cog`) and can be overridden with a `COG_HOME` environment variable. It must not be in a cloud-synced folder. Inside WSL2 it must stay on the distro's own filesystem, not under `/mnt/`, where SQLite's write-ahead mode is unreliable **(verify)** ([Platforms and environments](platforms-and-environments.md)). All of cog's state is here; target repos are untouched.

```text
<data>/
  cog.db                       run records, plan metadata, the event store (SQLite, WAL)
  checkpoints.db               LangGraph checkpoints (SQLite)
  config/
    user.toml
    repos/<owner>__<name>.toml     local per-repo overrides
  repos/<short-hash>/          managed clones (short, hash-based names)
  mirrors/<owner>__<name>/     issue and PR mirror (SQLite + full-text index)
  indexes/<owner>__<name>/<base_sha>/    code index, keyed by commit
  runs/<run_id>/
    resolved-config.json       (events are rows in cog.db; `cog runs show --json` exports them)
  plans/<owner>__<name>/<issue>/<plan_id>/
    v1/plan.json
    v1/plan.html               derived, disposable
    v1/approval.json           present once approved
    v2/...                     a new version for each edit
  blobs/<aa>/<hash>            content-addressed payloads (prompts, responses, context packs, raw issues)
  worktrees/<run_id>/<task>/   Implementation (future)
  logs/cog.log                 structured text log
  trust/                       trust-on-first-use approvals
```

**What is the system of record for what**

| Data                      | Record                       | Rebuildable from                |
| ------------------------- | ---------------------------- | ------------------------------- |
| Plans and approvals       | Files in `plans/`            | n/a (these are the record)      |
| Plan metadata for queries | `cog.db`                     | Rescanning `plans/`             |
| In-flight run state       | `checkpoints.db`             | n/a (lost state means a re-run) |
| Run history, events       | `cog.db` (payloads in blobs) | Partly; keep until pruned       |
| Debug log                 | Not a record: `logs/cog.log` | n/a (diagnostics only)          |
| Code index                | `indexes/`                   | The clone at that commit        |
| Issue mirror              | `mirrors/`                   | GitHub (an incremental sync)    |
| Clones                    | `repos/`                     | The remote                      |

Never make a database the only record of an approval.

**Backups.** Only `plans/` and approvals are irreplaceable; everything else rebuilds or means a re-run. An optional backup directory receives each approved plan version (plans are immutable files, so a synced folder is safe there), and `cog backup` adds a consistent snapshot of `cog.db`. `cog gc` never deletes `plans/`. Whole-environment backup on Windows is covered in [Platforms and environments](platforms-and-environments.md), Section 5.

---

## 3. The event store (logging)

All runs are logged locally, in three layers, with no separate service.

1. **Checkpoints:** step-by-step state, via LangGraph.
2. **Event store:** SQLite in WAL mode. One row per node execution and one per LLM call, tool call, GitHub call, and budget event. Large payloads (prompts, responses, context packs) are stored in the blob store and referenced by hash.
3. **Structured text logs** (JSON lines in `logs/cog.log`) for debugging. They are diagnostics, not a record: the event store is the one record of what a run did, and anything a report or eval needs is read from it.

### What each event records

| Field                                                        | Notes                                                                  |
| ------------------------------------------------------------ | ---------------------------------------------------------------------- |
| `run_id`, `event_id`, `parent_id`                            | Run = trace, node = span, call = child span (OpenTelemetry-shaped)     |
| `kind`                                                       | `node`, `llm`, `tool`, `github`, `git`, `budget`, `interrupt`, `error` |
| `node`, `status`                                             | For node events, including the explicit failure statuses               |
| `started_at`, `duration_ms`                                  |                                                                        |
| `model`, `tier`, `adapter`, `prompt_version`                 | For LLM events                                                         |
| `input_tokens`, `output_tokens`, `cached_tokens`, `cost_usd` | `cost_usd` is `NULL` when unknown, never `0`                           |
| `payload_ref`                                                | Hash of the stored request/response blob                               |
| `error_class`                                                | From the provider error taxonomy                                       |
| `api_calls`, `api_bucket`                                    | For GitHub events, to account against rate limits                      |

### Using the data

- Plain SQL against `cog.db`. Datasette or DuckDB can be pointed at it for ad-hoc analysis.
- `cog runs show <run_id>` reproduces a run's story. Evals use the same data.
- Hosted tracing (for example, LangSmith) is **opt-in** only, because events contain code and issue text. Export is an adapter on the OpenTelemetry-shaped events.
- Logs pass through a **redaction filter**. Secrets never appear in events, blobs, or logs.

---

## 4. Concurrency and locking

- SQLite runs in WAL mode with short write transactions.
- Per-repo locks guard `git fetch` and index builds. Index builds are keyed by commit, so two runs on the same commit share one build.
- Runs on different repos or issues may execute in parallel. A second run on the same issue warns and gets a distinct run id.
- Worktrees are named by run and task to avoid collisions.

---

## 5. Data lifecycle

| Concern            | Plan                                                                                                                                                                                                 |
| ------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Retention**      | `cog gc` prunes old runs, blobs, indexes, and orphaned worktrees by age and size. Plans and approvals are **never** pruned automatically. Default thresholds are decided when `cog gc` is specified. |
| **Crash recovery** | A crashed run's last checkpoint is intact. `cog resume` can continue it. `cog gc` finds orphaned worktrees.                                                                                          |
| **Migrations**     | Each SQLite file records its schema version (`PRAGMA user_version`) and applies forward-only SQL migrations on startup. Plan JSON carries `schema_version` and is upgraded on read.                  |
| **Backup**         | `plans/` and `config/` are what matter. Everything else is rebuildable or disposable.                                                                                                                |
| **Disk use**       | Managed clones may use partial-clone options to save space. **(verify)** at build time.                                                                                                              |
| **Export**         | `cog plan export` is the only path from the data directory into a target repo.                                                                                                                       |

---

## 6. Open questions

- Default retention thresholds.
- Whether the user config and the data directory share a root on every platform, or follow each platform's config-versus-data convention.
- One data directory per environment (Windows and WSL2) or a shared one, decided with the Implementation design ([ADR-0011](../decisions/0011-platform-support-and-implementation-execution-environment.md)).
