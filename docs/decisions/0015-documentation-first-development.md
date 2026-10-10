# ADR-0015: Documentation first, then code

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

The design spans many decisions that interact (providers, storage, platform, workflows). The owner wants the whole foundation analyzed and written down before building.

## Decision

- **No code until the documentation rounds are complete and signed off.** Round 1: foundation documents, diagrams, and ADRs. Round 2: Planning v3 (reference and diagram). Round 3: remaining explanation documents and the Implementation outline.
- Foundation and Planning are fully designed. Implementation is **scoped and outlined only**, to be designed after Planning is working.
- Open `(verify)` items are tracked in the roadmap's verification log and settled during the build, because some can only be answered by building (recall-eval results, provider quirks).
- Docs are written so coding agents can build from them: stable node ids, contracts in code blocks, a status marker per node.

## Alternatives considered

- **Code-first spike:** faster to feedback, but risks locking in storage and provider shapes before they are understood.

## Consequences

- Illustrative snippets in docs are allowed; they are not code in the project.

## Revisit when

Documentation sign-off is given.
