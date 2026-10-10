# ADR-0003: LangGraph orchestration, typed state, and a node contract

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Workflows must be resumable (human-in-the-loop pauses), inspectable, and testable without LLMs. The Planning workflow has parallel branches, bounded loops, and interrupts. See [the LangGraph primer](../explanation/langgraph-primer.md).

## Decision

- **LangGraph 1.x** (`langgraph>=1.1,<2`) orchestrates workflows. One compiled graph per workflow.
- **State is a TypedDict whose values are Pydantic models.** Large payloads are stored in the blob store and referenced by id or hash, so checkpoints stay small.
- Every node is registered through a **node decorator** that enforces the contract from Planning principle 3: declared input and output models, an explicit failure status instead of exceptions for expected conditions, timing and usage logging, and output validation. LangGraph does not do this itself. **(verify)** whether Pydantic state schemas validate node outputs; do not rely on it.
- Dependencies (GitHub client, index, stores, provider router, clock) are **injected** when the graph is built, never imported inside nodes. Tests substitute fakes.
- `thread_id` equals `run_id`.
- A registry holds node metadata (number, name, kind, tier, status). Reference tables and diagram drift checks are generated from it.

## Alternatives considered

- **Plain asyncio pipeline:** simpler, but we would rebuild checkpointing, interrupts, and parallel joins.
- **Other agent frameworks:** more opinionated about the LLM loop, which is exactly what Planning deliberately constrains.
- **Pydantic as the graph state itself:** conflicts with how reducers and partial updates are expressed. **(verify)**

## Consequences

- Contributors must learn a small vocabulary. The primer exists for this.
- A version pin plus a "checkpointer round-trip" test guards against upgrade surprises.

## Revisit when

LangGraph's API changes materially, or a workflow needs a shape the graph model handles poorly.
