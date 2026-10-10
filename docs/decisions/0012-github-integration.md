# ADR-0012: GitHub integration: github.com, token chain, GraphQL, local mirror

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Code, docs, and issues all live on GitHub (github.com). The owner reports hitting a limit of roughly 9 calls per minute, which does not match the standard limits: authenticated REST gets 5,000 requests per hour, search is a separate 30 per minute, code search has its own bucket, and GraphQL has a separate cost-based 5,000 points per hour. Secondary limits apply to bursts. The source of the owner's figure is still to be diagnosed with `gh api rate_limit`.

## Decision

**Scope:** github.com only for v1. Access sits behind a `VCSHost` interface.

**Authentication**, resolved in order: `GITHUB_TOKEN` environment variable, then `gh auth token`, then a clear error. Use **read-only** permissions until Implementation needs to write. A GitHub App provider can be added behind the interface for scale.

**Transport:** a real HTTP client using that token, not a `gh` subprocess per call. This gives control over ETag caching and rate-limit headers and avoids per-call process overhead. `gh` remains a fallback transport and a diagnostic tool.

**Minimize API use by design:**

| Need                                  | Source                                                                | API cost |
| ------------------------------------- | --------------------------------------------------------------------- | -------- |
| Code retrieval and search             | Managed clone + local index                                           | None     |
| `related_prs` history                 | Local `git log`                                                       | None     |
| `repo_conventions`, `docs_retriever`  | Files in the clone (a wiki is a separate git repo)                    | None     |
| Issue, comments, timeline, linked PRs | One GraphQL query, cached by `(repo, issue, updated_at)`              | Low      |
| `related_issues`                      | **Local issue/PR mirror** with full-text search, synced incrementally | Low      |

**Client behavior:** conditional REST requests (ETag; GraphQL does not support them), header-aware handling that treats primary and secondary limits as different mechanisms (waiting out the primary reset does not clear a secondary limit), a concurrency cap, per-run API budget accounting in the event store, `Retry-After` respected, and an `--offline` mode that replays from the mirror.

## Alternatives considered

- **`gh` for everything:** convenient, but process-per-call and opaque rate-limit handling.
- **Search API for related issues:** limited to 30 per minute, and it is the first thing to throttle.

## Consequences

- Resolves Planning open decision 3: **keep a local mirror**.
- Planning nodes 2, 7, and 8 change in Round 2; `preflight` (node 1) checks remaining API budget.

## Revisit when

The diagnosis of the owner's rate limit shows a different constraint, or GitHub Enterprise Server or another host is needed.

## Addendum (2026-10-06, Round 2): diagnosis of the rate limit

`gh api rate_limit` on the owner's account shows `core` 5,000 per hour, `graphql` 5,000 points per hour, `search` 30 per minute, and **`code_search` 10 per minute**. The owner's observed limit of roughly 9 calls per minute therefore matches the **code-search bucket**, the tightest of all. The decision stands and is confirmed by the numbers: the pipeline never calls the code-search or search APIs. Code retrieval uses the managed clone and local index, and related issues use the local mirror. The `preflight` node records all buckets for diagnostics.
