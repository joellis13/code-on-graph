# ADR-0009: Layered configuration

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Settings change at different rates and have different owners: shipped defaults, personal preferences, repo facts, and one-off overrides. Repo-supplied config is also untrusted input.

## Decision

Precedence, lowest to highest:

1. Built-in defaults
2. User config
3. In-repo `.cog/config.toml` (optional, shared by the team)
4. Local per-repo override in the data directory
5. Environment variables
6. CLI flags

Rules:

- **Secrets are never in config files.** Config names the environment variable (or keyring entry) that holds them.
- The **resolved config is persisted in every run record** for reproducibility.
- **Repo-supplied config can only tighten policy**, never loosen it. Command allow-list entries from a repo require **trust on first use**, recorded in user data, and re-prompted if they change.
- Model-tier mappings and the escalation ladder live in user config.

## Alternatives considered

- **Single user file:** cannot express repo facts.
- **Config in the target repo only:** conflicts with ADR-0008 and is a policy-weakening vector.

## Consequences

- A `cog config show --explain` view is needed to show which layer set each value.

## Revisit when

Multi-user deployment needs organization-level policy (add a layer above user config).
