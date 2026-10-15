# Code-OG (`code-on-graph`)

A standalone command center for AI-assisted software work, built on Python and LangGraph. Command: **`cog`**.

**Status: documentation-first. No code yet.** Code starts after the documentation rounds are complete and signed off.

## What it will do

The first workflow, **Planning**, takes a GitHub issue and produces a validated, human-approved implementation plan: deterministic context building, bounded LLM steps, code that checks every LLM output, and explicit human review. A later **Implementation** workflow will execute approved plans task by task.

## Principles in brief

- Owns the loop: cog is a harness built and controlled by its owner, with its own prompts, tools, and verification. It does not wrap Claude Code, Copilot, or similar products, whose prompts would nest inside cog's and sometimes conflict with them. Those products stay optional runners and baselines ([Architecture, Section 1](docs/explanation/architecture.md)).
- Deterministic where checkable; LLM only where judgment is needed; LLM outputs are proposals, verified by code.
- Standalone: installed once, works on managed clones, no footprint in your repos unless you export.
- Local-first: all state, logs, and plans live in a local data directory.
- Not opinionated about code style: each repo's conventions win.
- Providers are pluggable: direct APIs first (Fireworks, OpenRouter); agent CLIs optional.
- Platforms: Linux and Windows (through WSL2) are primary; macOS is supported; native Windows is best-effort. Implementation will run repo commands in containers ([Platforms and environments](docs/explanation/platforms-and-environments.md)).

## Start here

Read [`docs/index.md`](docs/index.md). Key pages: [Architecture](docs/explanation/architecture.md), [LangGraph primer](docs/explanation/langgraph-primer.md), [Roadmap](docs/roadmap.md), [ADRs](docs/decisions/README.md).
