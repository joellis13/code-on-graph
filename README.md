# Code-OG (`code-on-graph`)

A standalone command center for AI-assisted software work, built on Python and LangGraph. Command: **`cog`**.

**Status: documentation-first. No code yet.** Code starts after the documentation rounds are complete and signed off.

## What it will do

The first workflow, **Planning**, takes a GitHub issue and produces a validated, human-approved implementation plan: deterministic context building, bounded LLM steps, code that checks every LLM output, and explicit human review. A later **Implementation** workflow will execute approved plans task by task.

## How it differs from typical agent tools

- **Planning and execution are separate.** A plan is an immutable, versioned artifact that a human approves by hash before anything runs.
- **LLM outputs are proposals that code checks.** Every path, symbol, and command is verified against the repo; requirements must quote the issue verbatim.
- **The model never writes commands.** Acceptance checks name entries discovered from the repo's own build files.
- **Context is built deterministically, recorded, and measured** against historical merged PRs.
- **Everything is bounded and recorded locally**: loops, tokens, dollars, and every call.
- **No code-style opinions.** Each repo's conventions win.
- **cog owns the loop and is model-agnostic.** Its own prompts and tools; direct providers first (Fireworks, OpenRouter); agent products (Claude Code, Copilot, Antigravity) are optional runners and baselines.

The full comparison, including where typical agent tools are stronger and what cog does about it, is in [How cog differs](docs/explanation/how-cog-differs.md).

**Platforms:** Linux and Windows (through WSL2) are primary; macOS is supported; native Windows is best-effort. Implementation will run repo commands in containers ([Platforms and environments](docs/explanation/platforms-and-environments.md)).

## Start here

Read [`docs/index.md`](docs/index.md). Key pages: [How cog differs](docs/explanation/how-cog-differs.md), [Architecture](docs/explanation/architecture.md), [LangGraph primer](docs/explanation/langgraph-primer.md), [Roadmap](docs/roadmap.md), [ADRs](docs/decisions/README.md).
