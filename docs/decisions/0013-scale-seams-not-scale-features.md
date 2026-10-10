# ADR-0013: Build the seams for scale, not the scale features

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

The tool is personal now and should be able to grow into something shared if it succeeds.

## Decision

Build interfaces with local implementations now. Do not build multi-user features.

| Seam | Local implementation | Possible scale implementation |
|---|---|---|
| `PlanStore`, `ArtifactStore`, `RunStore` | Files + SQLite | Object storage + Postgres |
| `VCSHost` auth | Token / `gh` | GitHub App |
| `ExecutionEnvironment` | Local process, WSL2 | Containers, remote workers |
| Interfaces | CLI | Web review UI, MCP server, webhook trigger |
| Config layers | Defaults to flags | Organization policy layer |

Everything the CLI does goes through a service layer that other interfaces can call.

## Alternatives considered

- **Build for multi-user now:** premature; adds auth, tenancy, and deployment concerns with no user.
- **Ignore scale:** risks coupling the CLI to storage and process details.

## Consequences

- Some ceremony (interfaces with one implementation). Judged worth it at the storage, VCS, and execution boundaries only.
- Subscription-based runners become riskier as the audience grows (see ADR-0005).

## Revisit when

A second user or a remote deployment is planned.
