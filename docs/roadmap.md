# Roadmap

**Type:** Explanation · **Status:** Draft v1 · **Last updated:** 2026-10-15

Build order, exit criteria, the verification log, and risks. Nothing here starts until documentation sign-off ([ADR-0015](decisions/0015-documentation-first-development.md)).

---

## 1. Documentation rounds

| Round | Contents                                                                                                                    | Status                    |
| ----- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| 1     | Foundation documents, diagrams, ADRs, glossary, repo layout                                                                 | **Delivered**             |
| 2     | Planning v3: reference and diagram                                                                                          | **Delivered**             |
| 3     | Indexing, policy and sandbox, testing and evaluation, documentation strategy, Implementation outline, ADR-0016 and ADR-0017 | **Delivered**             |
| Gate  | Owner review and sign-off; the verification log reviewed                                                                    | **Next** (before Phase 1) |

---

## 2. Step 0: the repository

1. ~~Create the `code-on-graph` repository (private to start).~~ **Done (2026-10-10).**
2. ~~Commit this `docs/` tree, `README.md`, and `AGENTS.md` as the first commit.~~ **Done (2026-10-10).**
3. Decide the license before the repo is public.
4. ~~Run `gh api rate_limit` and record the output.~~ **Done (2026-10-06):** `code_search` is limited to 10 per minute, matching the observed ~9 calls per minute; recorded in an addendum to ADR-0012.
5. Check `where cog` again on any new machine.

---

## 3. Build phases

Each phase updates the docs it touches. **Exit criteria** are what must be true to move on.

| Phase                                         | Scope                                                                                                                                                                                                                                                                                                                    | Exit criteria                                                                                                                                            |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Walking skeleton**                       | uv project; CI matrix (Windows, Linux, macOS best-effort); config layering; event store and logging; node decorator and registry; all nodes of Planning as stubs; `FakeLLM`; SQLite checkpointer; interrupt and resume through the CLI; generated Mermaid graph with a drift test; docs-site spike; minimal `cog doctor` | A full fake run completes from the CLI; pause and resume works across process restarts; checkpoint round-trip test passes; CI green on Windows and Linux |
| **2. Front half**                             | Stores; GitHub client (GraphQL, mirror, cache, rate limits); managed clones; nodes 1–5 with fixtures                                                                                                                                                                                                                     | Nodes 1–5 pass fixture tests; `--offline` replays from the mirror; API use per run is recorded                                                           |
| **3. Index, retrievers, packer, recall eval** | Index for Java, TypeScript, Python; retrievers; context packer; golden set; `cog eval run`                                                                                                                                                                                                                               | Recall eval runs end to end and produces a baseline; each retriever measured separately                                                                  |
| **4. LLM and planning**                       | `LLMClient`, tier router, OpenAI-compatible adapter (Fireworks, OpenRouter), record/replay; planner, derive batches, validator and repair loop; optional nodes 11, 13, 19; plan-vs-PR overlap eval                                                                                                                       | Invalid canned plans exercise the repair loop; real calls work on both providers; cost recorded per node                                                 |
| **5. Human review and persistence**           | `ask_human`, `hitl_review`, render, persist, export                                                                                                                                                                                                                                                                      | Review round trip works through the CLI; plans are immutable and exportable                                                                              |
| **6. Implementation (design first)**          | Docs-first Implementation design; `AgentRunner`, `NativeRunner`; live contract tests for Copilot and Antigravity; WSL2 or container `ExecutionEnvironment`                                                                                                                                                               | Design reviewed; contract tests pass on the owner's machine; sandbox decision recorded in ADR-0011                                                       |

Copilot's BYOK mode may be the fastest stopgap for early Implementation work, while `NativeRunner` matures (see [Providers](explanation/providers.md)).

---

## 4. Verification log

Every **(verify)** item, collected. Status is updated as items are settled.

| #   | Item                                                                                                                             | Source doc                           | Settle in     |
| --- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ | ------------- |
| V1  | LangGraph async SQLite checkpointer package and class names                                                                      | Primer, ADR-0002, Planning reference | Phase 1       |
| V2  | Whether Pydantic state schemas validate node outputs                                                                             | Primer, ADR-0003                     | Phase 1       |
| V3  | Runtime context versus closures for dependency injection                                                                         | Primer                               | Phase 1       |
| V4  | Interrupted node re-runs from its first line                                                                                     | Primer                               | Phase 1       |
| V5  | Node retry API                                                                                                                   | Primer                               | Phase 1       |
| V6  | Local Studio or dev server worth adding                                                                                          | Primer                               | Phase 1       |
| V7  | Import-boundary checker choice                                                                                                   | Architecture, repo layout            | Phase 1       |
| V8  | Docs tool: Zensical's API-reference support versus Material or Sphinx                                                            | ADR-0010                             | Phase 1 spike |
| V9  | Partial or treeless clones for managed clones                                                                                    | ADR-0008, config                     | Phase 2       |
| V10 | ~~Source of the owner's ~9 calls/min GitHub limit~~ **Settled:** the code-search bucket (10 per minute)                          | ADR-0012                             | Done          |
| V11 | Keyring behavior on Windows, macOS, Linux                                                                                        | Config                               | Phase 1       |
| V12 | Fireworks structured-output behavior per chosen model                                                                            | Providers                            | Phase 4       |
| V13 | OpenRouter provider preferences and data-collection options                                                                      | Providers, Policy                    | Phase 4       |
| V14 | Copilot SDK status, package names, Python support                                                                                | Providers                            | Phase 6       |
| V15 | Copilot BYOK login requirement (test on the owner's account)                                                                     | Providers                            | Phase 6       |
| V16 | Copilot unattended runs from Git Bash on Windows                                                                                 | Providers, ADR-0011                  | Phase 6       |
| V17 | Antigravity headless flags and output format                                                                                     | Providers                            | Phase 6       |
| V18 | Antigravity subscription terms for automation (no official answer found)                                                         | Providers                            | Phase 6       |
| V19 | Antigravity model access for the owner's plan tier                                                                               | Providers                            | Phase 6       |
| V20 | Anthropic terms, only if a Claude runner is ever used                                                                            | Providers                            | Before use    |
| V22 | `GET /rate_limit` is not counted against the limit                                                                               | Planning reference, node 1           | Phase 2       |
| V23 | GraphQL field availability: sub-issues, `closedByPullRequestsReferences`, timeline items                                         | Planning reference, node 2           | Phase 2       |
| V24 | Cost and bounds of the first issue/PR mirror sync on a large repo                                                                | Planning reference, node 8           | Phase 2       |
| V25 | Commit-to-PR mapping for ambiguous cases: GraphQL batch versus REST endpoint                                                     | Planning reference, node 10          | Phase 2       |
| V26 | tree-sitter packaging: `tree-sitter-language-pack` versus core package plus pinned grammars; Windows wheels; on-demand downloads | Indexing, ADR-0017                   | Phase 3 spike |
| V27 | SQLite FTS5 and trigram tokenizer availability on every platform tier                                                            | Indexing                             | Phase 1       |
| V28 | Exact-match strategy: trigram, bounded scan, or ripgrep                                                                          | Indexing                             | Phase 3       |
| V29 | Windows path-trap list (reserved names, alternate streams, 8.3 names, `\\?\` prefixes)                                           | Policy and sandbox                   | Phase 1       |
| V30 | Redaction rule provenance and licenses of any seeded open-source rules                                                           | Policy, ADR-0016                     | Phase 1       |
| V32 | Native Windows containment options (job objects, Windows Sandbox)                                                                | Policy                               | Phase 6       |
| V33 | Reproducible fixture-repo commits across platforms and autocrlf settings                                                         | Testing                              | Phase 1       |
| V34 | Timeline fields needed to replay an issue as it was before the fix (eval leakage prevention)                                     | Testing                              | Phase 3       |
| V35 | Docs tooling: CLI reference generation, executed examples, changelog tool, draw.io export in CI, spell and style checks          | Documentation strategy               | Phase 1       |

V21 and V31 were duplicates of V1 and V13 and were removed. Ids are not reused.

---

## 5. Risks

| Risk                                           | Impact                | Mitigation                                                                            |
| ---------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------- |
| Vendor CLI or SDK behavior changes             | Runner breaks         | Pinned versions; live contract tests; runners are optional                            |
| Subscription terms change or prove restrictive | Cheap path disappears | Direct providers are the base; runners can be disabled by config                      |
| LangGraph API changes                          | Upgrade pain          | Version pin; checkpoint round-trip test; thin wrapper conventions                     |
| Index recall is the ceiling on plan quality    | Weak plans            | Recall eval before tuning; per-retriever measurement                                  |
| Windows-specific process and path issues       | Flaky Implementation  | `ExecutionEnvironment`; WSL2 move; Windows and Linux CI                               |
| GitHub rate limits                             | Slow or failed runs   | Local clones and mirror; GraphQL for fan-out; conditional requests; budget accounting |
| Prompt injection through issue text            | Unsafe actions        | Untrusted-text delimiting; read-only tools; deny-list; sandbox                        |
| Scope growth before value                      | Never ships           | Documentation gates; walking skeleton first; Implementation designed later            |

---

## 6. Bootstrap checklist

- [x] Documentation Round 2 delivered
- [ ] Documentation Round 2 reviewed
- [x] Documentation Round 3 delivered
- [ ] Documentation Round 3 reviewed
- [ ] Owner sign-off; verification log reviewed
- [x] Repository created; docs committed as the first commit
- [ ] License decided
- [x] `gh api rate_limit` output recorded (see ADR-0012 addendum)
- [ ] Phase 1 started
