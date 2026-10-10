# ADR-0005: Direct API providers first; agent CLIs optional

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

The owner will start with Fireworks.ai and OpenRouter as direct providers and holds only an Antigravity subscription. The goal is to avoid depending on agent CLIs, using them only as a stopgap or where they are materially cheaper. Subscription terms for programmatic use vary by vendor and have changed repeatedly in 2026.

## Decision

- **One OpenAI-compatible adapter** serves Fireworks, OpenRouter, and local servers (Ollama, vLLM). Per-provider differences live in config as "quirks," not in code branches.
- Structured outputs are always **validated by Pydantic** with a repair loop, regardless of provider-side schema enforcement.
- **CLI/SDK runners are optional adapters** behind `AgentRunner`: Copilot (SDK or CLI, including its BYOK mode), Antigravity (SDK or CLI), and Claude (API key, or the unmodified CLI for personal use). None is required for Planning.
- cog **never extracts or reuses subscription tokens** to call vendor APIs directly. It either shells out to the unmodified official CLI, uses an official SDK, or uses an API key.
- A `NativeRunner` (cog's own tool-calling loop) is the long-term Implementation runner.

## Alternatives considered

- **LiteLLM or similar gateway:** broad coverage, but a large surface for a narrow need. Reconsider if provider count grows.
- **Subscription-only (CLI-first):** cheapest short term, but terms and flags change under us.

## Consequences

- Prompts and tool definitions are cog's responsibility (see [providers](../explanation/providers.md)).
- Each CLI/SDK runner needs a live contract test on the owner's machine.

## Revisit when

Provider terms change, a runner proves far cheaper at equal quality, or provider count outgrows the single adapter.
