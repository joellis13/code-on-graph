# ADR-0006: Local-first storage; no logging service

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

All runs must be logged and tracked so the system can be improved. The tool is personal now but should be able to scale. Logs contain code and issue text, so sending them to a third party has a privacy cost.

## Decision

Three local layers, all under the data directory (see [configuration and data](../explanation/configuration-and-data.md)):

1. **LangGraph checkpoints** in SQLite: step-by-step state per run.
2. **Run event store** in SQLite (WAL mode): one row per node execution and per LLM, tool, and GitHub call. Large payloads go to a **content-addressed blob store** on disk and are referenced by hash.
3. **Structured text logs** (JSON lines) for debugging.

Event shapes follow OpenTelemetry concepts (run = trace, node = span) so exporting later is an adapter. LangSmith or similar is opt-in only.

## Alternatives considered

- **Hosted tracing (LangSmith, Langfuse cloud):** richer UI, but sends code off-machine by default.
- **Self-hosted observability stack:** operational burden unjustified for one user.
- **Files only:** poor queryability.

## Consequences

- Querying is plain SQL. Datasette or DuckDB can be pointed at the files.
- We own retention, migration, and pruning (see the data lifecycle section of the configuration doc).

## Revisit when

Multiple users or remote runs appear, or dashboards become a requirement.
