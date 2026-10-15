# AGENTS.md

Guidance for contributors and coding agents working in this repository.

## Current phase

**Documentation-first.** Do not add application code until the documentation is signed off (ADR-0015). Sign-off is accepting ADR-0021 (proposed), which ends the documentation rounds and makes the docs a living specification. Documentation changes are welcome and expected.

## Where things are

- Start at `docs/index.md`. The architecture, providers, configuration, and interaction docs are the specification.
- Decisions are in `docs/decisions/`. Never edit an accepted ADR's decision; add a new ADR that supersedes it.
- The planned file tree is `docs/reference/repo-layout.md`. Follow its layering rules when code begins.

## Documentation rules

- American English.
- Every page opens with Type, Status, and Last updated. Update Last updated when you change a page.
- Mark claims that depend on external behavior with **(verify)** and add them to the roadmap's verification log.
- Illustrative snippets are not project code. Label them as illustrative.
- Diagrams: draw.io for large ones (one file each), Mermaid inline for small ones.
- Keep one source of truth per concept; link instead of copying.

## Consistency checklist (every documentation change)

Drift between pages is the most common documentation defect here. Before you finish a change:

1. **Search before you finish.** Grep `docs/`, `README.md`, and `AGENTS.md` for every name you touched: node ids and numbers, schema fields, state keys, commands, config keys, pause reasons, package names, ADR numbers, phase numbers, and verification ids. Update each mention, or replace a copy with a link to the owning page.
2. **Superseded or changed facts.** When a new ADR or a change makes an older statement false, add a dated addendum (or the status line) to each affected ADR and update every page that cites it. An accepted ADR's decision text is never rewritten.
3. **Owning tables.** Commands and pause reasons live in the [Interaction model](docs/explanation/interaction-model.md); the data directory and event store in [Configuration and data](docs/explanation/configuration-and-data.md); import rules in [Repo layout](docs/reference/repo-layout.md); Planning schemas in the [Planning reference](docs/workflows/planning/reference.md), Section 3; platform tiers in [Platforms and environments](docs/explanation/platforms-and-environments.md). Change the owning table first, then the pages that link to it.
4. **Layering.** If a component gains a dependency, check that the import rules in the repo layout allow it.
5. **Verification log.** Each new **(verify)** gets a row in the roadmap's verification log, settled no later than the first phase that relies on it. A settled item records its evidence, and its markers are removed.
6. **Headers and index.** Update Last updated on each changed page, and the document map in `docs/index.md` when a page is added, renamed, or removed.
7. **Retired terms.** When you rename or remove a term, grep for the old name. Once the docs check exists (roadmap Phase 1), add the old name to its retired-terms list.

## Design rules to preserve

- Deterministic where checkable; LLM outputs are proposals verified by code.
- Nodes return statuses for expected conditions; they do not raise.
- Every loop and every provider call is bounded.
- Repo conventions outrank harness preferences. No code-style opinions in prompts or defaults.
- cog never extracts or reuses subscription tokens.
- Nothing is written into a target repo without an explicit request.

## Style for code (when code begins)

Python 3.12+, uv, ruff, pyright, pytest. Async-first. Commands are argument lists, never shell strings. Stored paths are POSIX-style.
