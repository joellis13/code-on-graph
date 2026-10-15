# ADR-0021: Sign-off ends the documentation rounds; the docs become a living specification

- **Status:** Proposed. Accepting this ADR is the owner's sign-off. On acceptance, [ADR-0015](0015-documentation-first-development.md)'s status line becomes "Superseded by ADR-0021", and `AGENTS.md` "Current phase" is updated.
- **Date:** 2026-10-15
- **Supersedes, on acceptance:** [ADR-0015](0015-documentation-first-development.md)'s gate ("no code until the documentation rounds are complete and signed off"). Its other decisions stay in force: Implementation gets a documentation-first design pass before it is built, and docs are written so coding agents can build from them.

## Context

ADR-0015 planned three documentation rounds before code. Four rounds were delivered: about 45,000 words, 20 ADRs, and 43 verification items, with no code. An outside review found drift between pages: an outdated state model in the primer, import rules that contradicted the design, three event sinks with no stated system of record, stale ADR context after ADR-0018, and verification items scheduled after the phase that needs them. These were fixed in the review errata.

The drift is a symptom, not the problem. Many open questions can only be answered by building: retrieval recall, which retrievers earn their cost, command-discovery quality, and whether plans are executable by small models. Specifying further adds pages that must be kept consistent but cannot be validated, so each round costs more and teaches less.

## Decision

1. **The documentation rounds end with Round 4 and its review errata.** There is no Round 5. Accepting this ADR is the sign-off, and Phase 1 starts.
2. **Detail is added just in time.** A page gains detail when the roadmap phase that builds it starts, not before. Implementation still gets its documentation-first design pass at the start of Phase 6, sized to what Phase 6 needs.
3. **Docs follow the code** under "document on merge" ([Documentation strategy](../explanation/documentation-strategy.md), Section 8): a pull request that makes a page true or false changes the page in the same pull request.
4. **Docs shrink as code grows.** When code can generate a reference (contract schemas, the node registry table, the command table, the CLI and config references), the hand-written copy is replaced by a link to the generated page in the pull request that adds the generator.
5. **One home per concept, enforced.** Contributors follow the consistency checklist in `AGENTS.md`. The first Phase 1 deliverable is a docs check run in CI (headers, internal links, the ADR index, verification markers against the log, and retired terms).
6. **Soft size budget.** An explanation page that grows past about 500 lines is reviewed for splitting or trimming. ADRs record decisions, not explanations, and stay short.

## Alternatives considered

- **Keep rounds until every open question is closed:** many questions cannot close without code, so the gate would never open.
- **Drop documentation-first entirely:** loses the value already built (contracts, node ids, decisions) and the design pass Implementation still needs.

## Consequences

- Code starts. The roadmap's Phase 1 is re-scoped to reach a real plan early.
- The specification may be wrong in places; that is found by building and fixed under rule 3.
- Doc maintenance moves from rounds to pull requests, with the checklist and the docs check catching drift.

## Revisit when

The docs check or reviews show the specification drifting from the code faster than pull requests fix it, or a collaborator needs a frozen specification.
