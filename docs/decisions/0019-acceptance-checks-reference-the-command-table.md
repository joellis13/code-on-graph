# ADR-0019: Acceptance checks reference the resolved command table

- **Status:** Accepted
- **Date:** 2026-10-15

## Context

The Planning reference sketched an acceptance check as a command string (`cmd: str`). That conflicts with the rule that commands are argument lists, never shell strings, because a string field forces a parse step whose quoting rules differ by platform. It also lets the planner (an LLM reading untrusted issue text) author executables, which breaks Planning principle 8: LLM outputs are proposals, verified by code.

## Decision

- A task's acceptance check **names a command and does not contain one**. It holds a `command_id` that is a key in the **resolved command table** built by `repo_conventions` (node 9), an optional typed **selector**, a working directory, and the expected result. `cog` builds the argument list from the table entry. The planner never writes an executable or a flag.
- **Command ids are extensible.** `test`, `lint`, `format`, `typecheck`, and `build` are conventional names. A repo can declare more (for example `e2e` or `compile`) in its config.
- **Table entries are argument lists** and carry what a non-interactive run needs: environment additions (for example `CI=true`, so watch-mode test runners exit), a timeout, and the selector kinds the command supports with the argument fragment each appends. Commands in repo and user config are therefore argument lists too, not strings.
- **Selectors are typed** (`class`, `file`, `tag`, `name`). A value is validated against a conservative pattern, and class and file values must resolve in the index.
- The resolved argument list is stored in the plan and shown at review.
- **No automatic fallback to planner-authored commands.** When a repo has no entry for a needed command, the validator reports it and the planner records an `open_question`. The owner adds the command to config once.
- **A gated escape exists:** an optional `custom_argv` field. It is **off by default**, enabled by a config setting, shown at review as untrusted, and subject to trust on first use. Enabling it needs no code change.
- **Discovery is static.** Planning executes no repo code, so node 9 reads files and never invokes a build tool to ask what it can do.
- **Discovery is measured, not assumed.** In Phase 2, discovery runs on the owner's actual repositories. The bar is that `test` and `build` are found or declared for each, and one selector works per ecosystem.

The schema is in the [Planning reference](../workflows/planning/reference.md), Section 3. Discovery sources are in node 9.

## Alternatives considered

- **Keep a command string:** most expressive, but needs parsing and lets the planner author commands.
- **Planner-authored argument list:** no parsing, but the planner still chooses executables and flags, and the validator must prefix-match against the table.

## Consequences

- Principle 8 holds for commands by construction.
- Node 9's discovery quality becomes load-bearing. Weak spots are expected for lint and format detection when build logic hides in convention plugins, for selector templates per test runner, and for multi-module builds. The mitigations are explicit config overrides, provenance and confidence on every discovered entry, and the Phase 2 measurement.
- Widening the schema later (for example, adding validated extra arguments) is backward compatible with `schema_version`. Narrowing a loose schema is not, which is why this starts narrow.

## Revisit when

The Phase 2 measurement misses the bar on the owner's repositories, or real plans need acceptance checks the table cannot express.
