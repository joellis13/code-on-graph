# Code-OG Documentation

**Status:** Rounds 1 to 4 and the review errata delivered; awaiting owner sign-off ([ADR-0021](decisions/0021-sign-off-and-living-specification.md), proposed) · **Last updated:** 2026-10-15

Code-OG (`code-on-graph`, command `cog`) is a standalone command center for AI-assisted software work, built on Python and LangGraph. The first workflow is **Planning**: turn a GitHub issue into a validated, human-approved implementation plan.

**The project is documentation-first.** No code is written until the documentation is signed off ([ADR-0015](decisions/0015-documentation-first-development.md)). Sign-off is accepting [ADR-0021](decisions/0021-sign-off-and-living-specification.md), which ends the documentation rounds and makes the docs a living specification that changes with the code.

---

## Reading order

1. [How cog differs](explanation/how-cog-differs.md): what sets it apart from typical agent tools, and where those tools are stronger.
2. [Architecture](explanation/architecture.md): what the system is, its layers, and its boundaries.
3. [LangGraph primer](explanation/langgraph-primer.md): the vocabulary, including the Redux comparison.
4. [Providers](explanation/providers.md): how models and agent tools plug in.
5. [Configuration and data](explanation/configuration-and-data.md): config layers, the data directory, logging.
6. [Interaction model](explanation/interaction-model.md): how you use it.
7. [Platforms and environments](explanation/platforms-and-environments.md): where `cog` runs on Windows, Linux, and macOS, and what runs in containers.
8. [Repo layout](reference/repo-layout.md): the planned file tree.
9. [Indexing](explanation/indexing.md), [Policy and sandbox](explanation/policy-and-sandbox.md), [Testing and evaluation](explanation/testing-and-evaluation.md), [Documentation strategy](explanation/documentation-strategy.md): the supporting subsystems.
10. [Planning workflow reference](workflows/planning/reference.md): the first workflow, node by node. [Implementation outline](workflows/implementation/outline.md): scoped only.
11. [Roadmap](roadmap.md): build order, exit criteria, and the verification log.
12. [ADR index](decisions/README.md): every decision and why.
13. [Glossary](reference/glossary.md): terms.

---

## Document map and status

Sorted by Diátaxis type.

| Type              | Document                                                                                                          | Status                                         |
| ----------------- | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| Explanation       | [How cog differs](explanation/how-cog-differs.md)                                                                 | Review errata (new)                            |
| Explanation       | [Architecture](explanation/architecture.md)                                                                       | Round 1 ✔                                     |
| Explanation       | [LangGraph primer](explanation/langgraph-primer.md)                                                               | Round 1 ✔                                     |
| Explanation       | [Providers](explanation/providers.md)                                                                             | Round 1 ✔                                     |
| Explanation       | [Configuration and data](explanation/configuration-and-data.md)                                                   | Round 1 ✔                                     |
| Explanation       | [Interaction model](explanation/interaction-model.md)                                                             | Round 1 ✔                                     |
| Explanation       | [Indexing](explanation/indexing.md)                                                                               | Round 3 ✔                                     |
| Explanation       | [Policy and sandbox](explanation/policy-and-sandbox.md)                                                           | Round 3 ✔                                     |
| Explanation       | [Testing and evaluation](explanation/testing-and-evaluation.md)                                                   | Round 3 ✔                                     |
| Explanation       | [Documentation strategy](explanation/documentation-strategy.md)                                                   | Round 3 ✔                                     |
| Explanation       | [Platforms and environments](explanation/platforms-and-environments.md)                                           | Round 4 ✔                                     |
| Workflow          | [Planning reference (v3)](workflows/planning/reference.md) and [diagram (v3)](workflows/planning/workflow.drawio) | Round 2 ✔                                     |
| Workflow          | [Implementation outline](workflows/implementation/outline.md)                                                     | Round 3 ✔ (outline only)                      |
| Decisions         | [ADRs 0001–0021](decisions/README.md)                                                                             | Rounds 1–4 ✔; 0021 proposed                   |
| Reference         | [Glossary](reference/glossary.md)                                                                                 | Round 1 ✔ (moved from the Planning reference) |
| Reference         | [Repo layout](reference/repo-layout.md)                                                                           | Round 1 ✔                                     |
| Reference         | CLI, config, schemas                                                                                              | Generated once code exists                     |
| Diagrams          | [Architecture](diagrams/architecture.drawio), [Providers](diagrams/providers.drawio)                              | Round 1 ✔                                     |
| How-to, Tutorials | Not yet                                                                                                           | After the CLI exists                           |

---

## Conventions

- **American English.**
- Every page opens with **Type, Status, and Last updated**.
- **`(verify)`** marks any claim that depends on external behavior. All are tracked in the [roadmap's verification log](roadmap.md#4-verification-log).
- **Illustrative snippets** (config, signatures) show intent; they are not project code.
- **Diagrams:** draw.io for large ones, one file each (green deterministic, orange LLM, blue human, yellow decision, gray start/end/persisted); Mermaid inline for small teaching diagrams.
- **ADRs** are never edited after acceptance; supersede them instead.
- **Docs are written as a build specification:** stable node ids, contracts in code blocks, a status marker per node.
