# ADR-0001: Python with uv, single package, `src/` layout

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Code-OG (`cog`) is a Python application built around LangGraph. It needs reproducible installs on Windows (primary), Linux, and macOS, and it must install as a standalone command (see ADR-0008), not as a library copied into each repo.

## Decision

- Python **3.12 or newer**, pinned in `.python-version`. (LangGraph itself requires 3.10+; 3.12 gives better error messages and typing features.)
- **uv** manages the environment, the lockfile (`uv.lock`, committed), and dependency groups (`dev`, `docs`, `eval`).
- A **single package** (`code_on_graph`) in a `src/` layout. No monorepo or workspace until a second deployable exists.
- **Provider SDKs are optional extras** (`code-on-graph[copilot]`, `[antigravity]`, `[claude]`) so the core runs without any of them.
- The `cog` console script is the only entry point. Users install it with `uv tool install`.
- Tooling: ruff (lint and format), pyright (types), pytest, pre-commit. A task runner (`just` or equivalent) wraps `uv run` so commands are identical across shells.

## Alternatives considered

- **pip + venv + requirements.txt:** no lockfile semantics, slower, no dependency groups.
- **Poetry / PDM:** capable, but uv is faster and simpler for this use and handles tool installation.
- **Monorepo with workspaces:** premature for one deployable.

## Consequences

- Contributors need only uv installed. Python itself is managed by uv.
- Extras keep the core small, but CI must test with and without each extra.

## Revisit when

A second deployable (for example, a server mode or web UI) needs independent versioning.
