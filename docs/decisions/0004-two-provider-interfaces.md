# ADR-0004: Two provider interfaces, `LLMClient` and `AgentRunner`

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

"Inference provider" covers two different things: a model endpoint that returns completions, and an agent product that runs its own loop with its own tools (Copilot, Antigravity, Claude Code). Planning needs schema-constrained output, a small read-only tool budget, full logging of every call, and untrusted text treated as data. An agent product controls its own loop and tools, so it cannot guarantee that.

## Decision

- **`LLMClient`**: completions with optional structured output and a bounded read-only tool loop. Used by the planner, query generation, and review.
- **`AgentRunner`**: executes a task packet in a working directory under a policy. Used by the Implementation workflow.
- Workflows depend only on these interfaces. Adapters sit behind them.
- Config maps **tiers** (small, medium, large) to `(adapter, model)` pairs. The same tier names appear in plans as `executor_tier`, and the escalation ladder is an ordered list of tiers.
- Adapters declare **capabilities** (structured output, sessions, MCP, cost reporting, takeover, native shell) so callers can check before calling.

## Alternatives considered

- **One abstraction for both:** forces the lowest common denominator, or leaks agent behavior into planning.
- **Using an agent CLI for planning:** loses principle 8 (outputs are proposals) and per-call logging.

## Consequences

- Two sets of fakes for tests.
- A `NativeRunner` built on `LLMClient` is possible and is the planned first `AgentRunner`.

## Revisit when

A provider offers a hybrid that cannot be cleanly split.

## Addendum (2026-10-06, Round 2): roles

Planning nodes request an LLM **role** (`planner`, `query_gen`, `verdict`, `reviewer`), and config resolves roles to tiers, then tiers to `(adapter, model)` pairs. This keeps model choices out of workflows and lets config enforce rules such as "the reviewer must not resolve to the same model as the planner." See the Planning reference, Section 7.
