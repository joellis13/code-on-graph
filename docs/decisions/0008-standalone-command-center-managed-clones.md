# ADR-0008: Standalone "command center" app with managed clones

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

The tool should run from anywhere and operate on many repos. Nothing should need installing in each repo, and working copies must not be disturbed.

## Decision

- `cog` is a **standalone application**, installed once. State lives in its data directory.
- Repos are added with `cog repo add <url-or-path>`. cog keeps **managed clones** (short, hash-based directory names) under the data directory. A local path is cloned from, with `origin` set to the real remote.
- Planning reads from managed clones through git objects. Implementation worktrees are created from them. The user's own working copies are never touched.
- **No footprint in target repos** unless the user runs an explicit export (ADR-0007) or an Implementation step that opens a PR.
- Repo-specific settings default to cog-side config keyed by repo. An in-repo `.cog/config.toml` is an optional, read-only layer.

## Alternatives considered

- **Operate on the user's working copy:** fewer clones, but risks stepping on uncommitted work and makes parallel runs unsafe.
- **Per-repo install or GitHub Action first:** wrong for a personal command center, though a GitHub trigger may come later as another interface.

## Consequences

- Extra disk for clones, mitigated by treeless or partial clone options. **(verify)** at build time.
- Short clone paths also help Windows path-length limits.

## Revisit when

A hosted or team deployment needs per-repo automation.

## Addendum (2026-10-15, Round 4a): read-only working-copy checks

"The user's own working copies are never touched" means never **modified**. When `cog plan` runs inside a working copy of a registered repo, `preflight` may run read-only queries there (`git --no-optional-locks status`, which does not refresh the index) to warn that Planning reads only what the remote has: uncommitted changes, a branch with no upstream, or commits not yet pushed ([Planning reference](../workflows/planning/reference.md), node 1). Planning still reads only the managed clone.
