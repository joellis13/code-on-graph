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

## Addendum (2026-10-15, Round 4): who owns which quality gate

The decision above is unchanged. This records how responsibility splits between the harness and the repos it works on, so the harness stays generic.

- **The repo owns code quality:** style, formatting, lint rules, test frameworks, architecture patterns, commit and PR conventions. The repo expresses them in its own files (AGENTS.md, CONTRIBUTING, tool configs, instruction files).
- **The harness owns process quality:** contracts, bounded loops, verification, safety, and evidence. It never decides what good code is. It decides whether the repo's own gates were found, run, and passed.

Gates apply in this order:

1. **Repo gates (mandatory when they exist):** the repo's lint, format, test, typecheck, and build commands, found by `repo_conventions` (node 9) and run by `cog`. The bar is no new failures compared with a baseline run before the change.
2. **Harness safety gates (always on):** path deny-list, task file scope ([ADR-0020](0020-task-file-scope-detect-and-review.md)), secret checks, no edits to CI definitions.
3. **Harness advisories (warn only):** change size, tests added for new behavior, and similar signals about the change, never about style.

Language knowledge in the harness is limited to what indexing and command discovery need: grammars, import syntax, test-file patterns, and where each ecosystem declares its commands. These ship as overridable defaults. If a repo has no gates, `cog` says so and does not invent any.

## Addendum (2026-10-15, Round 4a): repo instruction files in prompts

The decision above is unchanged. Repo instruction files (`AGENTS.md`, `CONTRIBUTING.md`, assistant-instruction files, and repo-config style notes) outrank the harness on code and conventions, and never on the harness's process or safety rules. How they are framed in prompts, and what happens when they conflict with a process rule, is defined once in [Policy and sandbox](../explanation/policy-and-sandbox.md), Section 9.
