# ADR-0007: Plans are immutable JSON files; SQLite indexes them; HTML is derived

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Planning principle 6 says the structured plan is the source of truth and HTML is only a view. The v2 Planning reference stored approved plans inside the target repo (`.harness/plans/`), which conflicts with the standalone command-center model (ADR-0008).

## Decision

- The **plan JSON file** is the record. It carries a `schema_version` and is validated by Pydantic.
- Plans are **immutable**. A human edit or rejection-driven re-plan creates a new version with a parent hash. Approval is a separate small file recording the approved plan hash, time, and edits.
- **SQLite holds metadata only** (plan id, repo, issue, status, hash, path, timestamps) and is rebuildable by rescanning files.
- **HTML is generated** from the JSON by an auto-escaping template. It is disposable and may be cached beside the JSON. Because it renders issue-derived and LLM-derived text, the template must escape everything, never inject raw HTML, and carry a restrictive content-security policy.
- Plans live in the **data directory**. `cog plan export` writes them into a target repo (`.cog/plans/`) only on explicit request, optionally as a commit or PR.
- Access goes through a `PlanStore` interface.

## Alternatives considered

- **Plans only in the database:** not diffable, and a corrupted DB loses approvals.
- **Plans in the target repo by default:** reviewable in PRs, but gives every repo a footprint and requires write access.

## Consequences

- Approvals survive DB loss. The Planning reference's Storage section must be revised (Round 2).

## Revisit when

A team workflow needs plans reviewed in pull requests by default (make export the default for that repo).
