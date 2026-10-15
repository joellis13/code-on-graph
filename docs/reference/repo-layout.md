# Repo Layout (planned)

**Type:** Reference · **Status:** Draft v1 · **Last updated:** 2026-10-15

The planned file tree of the `code-on-graph` repository. **No code exists yet.** Directories appear when the roadmap phase that needs them starts. The layering rules from [Architecture](../explanation/architecture.md) govern which packages may import which.

---

## 1. Top level

```text
code-on-graph/
  README.md                  what it is, status, how to read the docs
  AGENTS.md                  contributor and coding-agent guidance
  pyproject.toml             project metadata, dependency groups, extras, tool config
  uv.lock                    committed lockfile
  .python-version            pinned Python (3.12+)
  justfile                   task runner wrapping `uv run` (same commands in every shell)
  .pre-commit-config.yaml    ruff, pyright, docs checks
  .github/workflows/         CI: Linux primary, Windows smoke job, macOS best-effort, docs build
  src/code_on_graph/         the package (Section 2)
  tests/                     test tiers (Section 3)
  docs/                      documentation (Section 4)
  scripts/                   docs generators and dev helpers (not shipped)
  LICENSE                    decided before the repo goes public
```

---

## 2. The package: `src/code_on_graph/`

Dependencies point downward in this table (a package may import those listed below it).

| Package                     | Purpose                                                                                                                   | May import                                              |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| `interfaces/cli/`           | Typer app; one module per command group (`plan`, `runs`, `repo`, `index`, `config`, `doctor`, `eval`, `gc`)               | workflows, services, runtime, core                      |
| `workflows/planning/`       | Graph factory, `nodes/` (one module per node), `prompts/`, routing functions, state definition                            | services and providers (interfaces only), runtime, core |
| `workflows/implementation/` | Later                                                                                                                     | same                                                    |
| `services/github/`          | HTTP client, GraphQL queries, cache, rate-limit handling, issue/PR mirror sync                                            | runtime, core                                           |
| `services/git/`             | Managed clones, `show`/`log`/`blame` helpers, worktrees (later)                                                           | runtime, core                                           |
| `services/index/`           | Index build and query; per-language support (Java, TypeScript, Python first)                                              | runtime, core                                           |
| `services/retrieval/`       | The retrievers (`code`, `linked_prs`, `related_issues`, `repo_conventions`, `related_prs`, `docs`) and the context packer | index, github, git, runtime, core                       |
| `services/artifacts/`       | Blob store, `PlanStore`, `RunStore`, `ArtifactStore`                                                                      | runtime, core                                           |
| `services/render/`          | Auto-escaping templates and plan HTML                                                                                     | core                                                    |
| `services/tools/`           | Typed tools exposed to LLMs (read-only first; edit tools later), each passing through policy                              | policy, runtime, core                                   |
| `providers/llm/`            | `LLMClient` interface, tier router, OpenAI-compatible adapter, `FakeLLM`, record/replay                                   | runtime, core                                           |
| `providers/agents/`         | `AgentRunner` interface, `NativeRunner`, Copilot, Antigravity, Claude runners, `FakeRunner`                               | llm, execution, runtime, core                           |
| `providers/execution/`      | `ExecutionEnvironment`: `LocalProcess`, `Container`                                                                | runtime, core                                           |
| `runtime/`                  | Node decorator, graph helpers, dependency container, run context, budgets, run registry                                   | core                                                    |
| `config/`                   | Settings models, layering, tier mapping, trust-on-first-use                                                               | core                                                    |
| `policy/`                   | Path deny-list, command allow-list, redaction, egress rules                                                               | core                                                    |
| `telemetry/`                | Event store, structured logging, OpenTelemetry-shaped events                                                              | core                                                    |
| `eval/`                     | Golden-set tooling and the context-recall eval runner                                                                     | services, core                                          |
| `core/`                     | Pydantic domain models (`Reference`, `ContextItem`, `Plan`, `Task`, ...), status types, errors, ids                       | nothing in the project                                  |

Notes:

- **Retrievers share one contract** (status, candidates, diagnostics) and live one per module.
- **Each node module** carries a templated docstring (purpose, inputs and outputs, failure handling) and a registry entry (number, kind, tier, status) from which reference tables are generated.
- **Prompts** live beside the workflow that uses them and are versioned.
- Import boundaries are checked in CI. **(verify)** tool.

---

## 3. Tests: `tests/`

```text
tests/
  unit/            fast, offline; routing functions, extractors, validators, policy
  integration/     real SQLite, real git on fixture repos, recorded GitHub responses
  graph/           whole-graph runs with FakeLLM, including invalid plans and interrupts
  live/            opt-in contract tests against real providers and CLIs (skipped by default)
  eval/            evaluation configs; the golden set (issue + PR file lists)
  fixtures/        small fixture repos (Java with Gradle, TypeScript with React, TypeScript with Playwright, Python with uv), recorded API responses
  snapshots/       CLI help, generated reference, rendered HTML
```

Live tests run on the owner's machine early, for each runner, to confirm unattended behavior.

---

## 4. Documentation: `docs/`

```text
docs/
  index.md                                 map and reading order
  roadmap.md                               phases, exit criteria, verification log, risks
  explanation/
    langgraph-primer.md
    architecture.md
    providers.md
    interaction-model.md
    configuration-and-data.md
    platforms-and-environments.md
    indexing.md
    policy-and-sandbox.md
    testing-and-evaluation.md
    documentation-strategy.md
  workflows/
    planning/
      reference.md                         Planning reference v3 (Round 2 ✔)
      workflow.drawio                      Planning diagram v3 (Round 2 ✔)
    implementation/
      outline.md                           scoped outline only (Round 3 ✔)
  decisions/                               ADRs (README.md is the index)
  reference/
    glossary.md
    repo-layout.md                         this page
    cli.md                                 (generated once the CLI exists)
    config.md                              (generated)
    schemas/                               (generated, committed: JSON Schemas for contracts)
  diagrams/
    architecture.drawio
    providers.drawio
  how-to/                                  (once the CLI exists)
  tutorials/                               (once the CLI exists)
```

Diátaxis is how the folders are sorted (explanation, reference, how-to, tutorials). It is not a storage system; these are ordinary files in the repo.

**Diagram conventions:** one `.drawio` file per large diagram (green deterministic, orange LLM, blue human, yellow decision, gray start/end/persisted); small teaching diagrams are Mermaid inline in the page they explain.

---

## 5. Target repos: what cog may write

Nothing, unless you ask:

```text
<target-repo>/.cog/
  config.toml              optional, team-shared repo facts (read by cog, never written without request)
  plans/<issue>/v<N>/...   only after `cog plan export`
```

---

## 6. The data directory

Documented in [Configuration and data](../explanation/configuration-and-data.md), Section 2.
