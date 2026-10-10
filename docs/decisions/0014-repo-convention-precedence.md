# ADR-0014: Repo conventions outrank harness preferences

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Repos differ in style and practice. One may always use `for` loops where another prefers streams. A harness that imposes its own taste would produce plans and code that fight each repo's norms.

## Decision

The harness has opinions about **process** (typed contracts, bounded loops, validated LLM output) and **none about code style**. Precedence, highest first:

1. Explicit repo rules: AGENTS.md, CONTRIBUTING, lint and format configs, and cog repo config style notes.
2. Patterns observed in nearby code: sibling files, the module being changed, retrieved exemplars.
3. Nothing. There is no harness default style.

Mechanisms:

- `repo_conventions` (node 9) harvests rules and tool configs deterministically and also retrieves **exemplars**.
- Planner and executor prompts say "match the surrounding code" and carry no style preferences. Any prompt-level default is labeled as overridable.
- Acceptance checks run the **repo's own** lint, format, and test commands.
- Where a repo is internally inconsistent, follow the nearest file or module.
- Optionally, an eval metric tracks convention adherence.

## Alternatives considered

- **Harness-wide style guide:** simple, but wrong for a tool meant to span repos.

## Consequences

- Becomes **Planning principle 9** in Round 2. Prompt reviews must check for smuggled style preferences.

## Revisit when

Evals show convention adherence is poor even with exemplars.

## Addendum (2026-10-06, Round 2): where exemplars are selected

Exemplars (sibling code to imitate) depend on code hits, so they are selected by `code_retriever`'s neighbor tier (node 6) and emitted with `kind="exemplar"`. `repo_conventions` (node 9) supplies the rules, tool configs, and style notes. The decision above is unchanged; this records the implementation placement.
