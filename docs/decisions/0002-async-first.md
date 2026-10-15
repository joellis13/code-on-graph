# ADR-0002: Async-first

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Most of the work is I/O: GitHub calls, git subprocesses, LLM requests, and later agent sessions. The agent SDKs under consideration (Copilot, Antigravity, Claude) are async. LangGraph supports both sync and async nodes, but mixing them later is painful.

## Decision

- Nodes, services, and provider adapters are `async`.
- Subprocesses use `asyncio` subprocess APIs behind a single wrapper that handles timeouts and **kills the whole process tree** on Windows and POSIX.
- The LangGraph checkpointer is the async SQLite variant. **(verify)** exact class and package at build time.
- Blocking libraries (SQLite access outside the checkpointer, tree-sitter parsing) run in a thread pool through one helper, never ad hoc.
- Windows uses the default Proactor event loop. Nothing in the project may require the Selector loop.

## Alternatives considered

- **Sync-first with threads:** simpler to read, but fights the async SDKs and complicates cancellation and timeouts.

## Consequences

- Tests use `pytest-asyncio`.
- Every blocking call must be consciously wrapped. The wrapper is the one place that policy applies.

## Revisit when

Never expected. Changing this later would touch every node.

## Addendum (2026-10-15, Round 4a): CPU-bound parsing

The Decision above names tree-sitter parsing among the blocking work that runs in a thread pool. During index builds, parsing is CPU-bound and runs in a **process pool** instead ([Indexing](../explanation/indexing.md), Section 8). The thread-pool helper remains the one path for blocking I/O and for small, one-off parses outside an index build. The async-first decision is unchanged.
