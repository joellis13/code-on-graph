# AGENTS.md

Guidance for contributors and coding agents working in this repository.

## Current phase

**Documentation-first.** Do not add application code until the documentation rounds are signed off (ADR-0015). Documentation changes are welcome and expected.

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

## Design rules to preserve

- Deterministic where checkable; LLM outputs are proposals verified by code.
- Nodes return statuses for expected conditions; they do not raise.
- Every loop and every provider call is bounded.
- Repo conventions outrank harness preferences. No code-style opinions in prompts or defaults.
- cog never extracts or reuses subscription tokens.
- Nothing is written into a target repo without an explicit request.

## Style for code (when code begins)

Python 3.12+, uv, ruff, pyright, pytest. Async-first. Commands are argument lists, never shell strings. Stored paths are POSIX-style.
