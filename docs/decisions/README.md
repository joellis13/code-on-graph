# Architecture Decision Records

Each file records one decision: context, decision, alternatives, consequences, and when to revisit. Status is **Accepted** unless noted.

| ADR                                                                       | Title                                                                                                                               |
| ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| [0001](0001-python-and-uv.md)                                             | Python with uv, single package, `src/` layout                                                                                       |
| [0002](0002-async-first.md)                                               | Async-first                                                                                                                         |
| [0003](0003-langgraph-orchestration-and-state.md)                         | LangGraph orchestration, typed state, and a node contract                                                                           |
| [0004](0004-two-provider-interfaces.md)                                   | Two provider interfaces, `LLMClient` and `AgentRunner`                                                                              |
| [0005](0005-direct-providers-first.md)                                    | Direct API providers first; agent CLIs optional                                                                                     |
| [0006](0006-local-first-storage-no-logging-service.md)                    | Local-first storage; no logging service                                                                                             |
| [0007](0007-plans-as-immutable-json.md)                                   | Plans are immutable JSON; SQLite indexes; HTML derived                                                                              |
| [0008](0008-standalone-command-center-managed-clones.md)                  | Standalone command center with managed clones                                                                                       |
| [0009](0009-layered-configuration.md)                                     | Layered configuration                                                                                                               |
| [0010](0010-documentation-strategy-and-tooling.md)                        | Documentation strategy and tooling                                                                                                  |
| [0011](0011-platform-support-and-implementation-execution-environment.md) | Platform support and Implementation execution environment (**partly superseded by 0018**)                                           |
| [0012](0012-github-integration.md)                                        | GitHub integration                                                                                                                  |
| [0013](0013-scale-seams-not-scale-features.md)                            | Scale seams, not scale features                                                                                                     |
| [0014](0014-repo-convention-precedence.md)                                | Repo conventions outrank harness preferences                                                                                        |
| [0015](0015-documentation-first-development.md)                           | Documentation first, then code                                                                                                      |
| [0016](0016-secret-redaction.md)                                          | Secret redaction before egress, in logs, and in storage                                                                             |
| [0017](0017-index-design.md)                                              | Index design (packaging choice open)                                                                                                |
| [0018](0018-wsl2-home-and-container-execution.md)                         | cog runs in WSL2 on Windows; Implementation executes in containers (partly supersedes 0011)                                         |
| [0019](0019-acceptance-checks-reference-the-command-table.md)             | Acceptance checks reference the resolved command table                                                                              |
| [0020](0020-task-file-scope-detect-and-review.md)                         | Task file scope is detect-and-review                                                                                                |
| [0021](0021-sign-off-and-living-specification.md)                         | Sign-off ends the documentation rounds; the docs become a living specification (**Proposed**; supersedes 0015's gate on acceptance) |

**Rejected alternatives for the Planning workflow** (LLM-first extraction, vector database from day one, and so on) are Planning-specific and live in the [Planning reference](../workflows/planning/reference.md), Section 11, which also maps these ADRs to their effect on Planning.

New ADRs use the next number and the same template. Never edit an accepted ADR's decision; supersede it with a new one.
