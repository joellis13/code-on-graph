# Roadmap

**Type:** Explanation · **Status:** Draft v1 · **Last updated:** 2026-10-15

Build order, exit criteria, the verification log, and risks. Nothing here starts until documentation sign-off ([ADR-0015](decisions/0015-documentation-first-development.md)).

---

## 1. Documentation rounds

| Round | Contents                                                                                                                                                             | Status                    |
| ----- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| 1     | Foundation documents, diagrams, ADRs, glossary, repo layout                                                                                                          | **Delivered**             |
| 2     | Planning v3: reference and diagram                                                                                                                                   | **Delivered**             |
| 3     | Indexing, policy and sandbox, testing and evaluation, documentation strategy, Implementation outline, ADR-0016 and ADR-0017                                          | **Delivered**             |
| 4     | Owner-review decisions: ADR-0018 to ADR-0020, Platforms and environments, Planning updates (requirement anchoring, repo-fit checks, command table), this build order | **Delivered**             |
| Gate  | Owner review and sign-off; the verification log reviewed                                                                                                             | **Next** (before Phase 1) |

---

## 2. Step 0: the repository

1. ~~Create the `code-on-graph` repository (private to start).~~ **Done (2026-10-10).**
2. ~~Commit this `docs/` tree, `README.md`, and `AGENTS.md` as the first commit.~~ **Done (2026-10-10).**
3. Decide the license before the repo is public.
4. ~~Run `gh api rate_limit` and record the output.~~ **Done (2026-10-06):** `code_search` is limited to 10 per minute, matching the observed ~9 calls per minute; recorded in an addendum to ADR-0012.
5. Check `which cog` (`where cog` on native Windows) again on any new machine.

---

## 3. Build phases

Each phase updates the docs it touches. **Exit criteria** are what must be true to move on.

| Phase                                         | Scope                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Exit criteria                                                                                                                                                                                                                                                                                                                                |
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **1. Walking skeleton (thin slice, fakes)**   | uv project; CI (Linux primary, Windows smoke job, macOS best-effort); config layering; event store and logging; node decorator and registry (every node present as a `spec` row); the slice's nodes: 1, 1b, 2–5, 9 (with command-table discovery), 12, 15, 17, 18, 20–22, on a thin `Index` (`git ls-tree` and `git grep` at `base_sha`, which also stays as the lexical-only fallback); `FakeLLM`; SQLite checkpointer; pause and resume through a minimal CLI (`plan`, `resume`, `doctor`); generated Mermaid graph with a drift test | A full fake run of the slice completes from the CLI; pause and resume works across process restarts; checkpoint round-trip test passes; CI green on Linux and the Windows smoke job                                                                                                                                                          |
| **2. First real plan**                        | GitHub client (GraphQL, cache, rate limits); managed clones; `cog repo add` with command discovery and `cog repo commands`; `LLMClient`, tier router, OpenAI-compatible adapter (Fireworks, OpenRouter), record/replay, budgets; requirement anchoring in the validator; basic approve and reject; WSL2 setup verified on the owner's machine                                                                                                                                                                                           | A real `cog plan` on a real issue in one of the owner's repos produces a validated plan that is reviewed and approved through the CLI; cost recorded per node; command discovery meets the bar on the owner's repos (ADR-0019) or overrides are declared; notes recorded on what the plan missed and whether retrieval or planning caused it |
| **3. Index, retrievers, packer, recall eval** | Real index for Java, TypeScript, Python behind `Index`; code retriever tiers; context packer tuning; blast-radius check; golden set where history exists; `cog eval run` (recall and plan-vs-PR overlap); docs-site spike                                                                                                                                                                                                                                                                                                               | Recall eval runs end to end and compares against the thin-retriever baseline; each retriever measured separately; blast-radius check passes its fixtures                                                                                                                                                                                     |
| **4. Remaining Planning nodes**               | Nodes 7, 8 (with the first mirror sync at `cog repo add`), 10, 11, 13, 14, 16a and 16b, 19 (reviewer from another model family); repair loop hardened; issue-coverage warning                                                                                                                                                                                                                                                                                                                                                           | Invalid canned plans exercise the repair loop; real calls work on both providers; the reviewer's catch rate on planted flaws is measured; cost recorded per node                                                                                                                                                                             |
| **5. Review and persistence polish**          | Edit decision (re-derive and re-validate), clarify flow, export, `cog backup`, `cog gc`                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Review round trip including edit works through the CLI; plans are immutable, exportable, and backed up                                                                                                                                                                                                                                       |
| **6. Implementation (design first)**          | Docs-first Implementation design; `AgentRunner`, `NativeRunner` (the intended runner); live contract tests for Copilot and Antigravity as baselines; container `ExecutionEnvironment`                                                                                                                                                                                                                                                                                                                                                   | Design reviewed; contract tests pass on the owner's machine; container design decisions recorded in a new ADR                                                                                                                                                                                                                                |

**Why this order.** A real plan is in your hands in Phase 2, so retrieval and planning problems show up early. The thin `Index` stays as the lexical-only fallback, so nothing is thrown away: later phases deepen the same nodes and add the others.

Copilot's BYOK mode may be the fastest stopgap for early Implementation work, while `NativeRunner`, the intended runner and the point of the project, matures (see [Providers](explanation/providers.md)).

---

## 4. Verification log

Every **(verify)** item, collected. Status is updated as items are settled.

| #   | Item                                                                                                                                                                  | Source doc                           | Settle in     |
| --- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ | ------------- |
| V1  | LangGraph async SQLite checkpointer package and class names                                                                                                           | Primer, ADR-0002, Planning reference | Phase 1       |
| V2  | Whether Pydantic state schemas validate node outputs                                                                                                                  | Primer, ADR-0003                     | Phase 1       |
| V3  | Runtime context versus closures for dependency injection                                                                                                              | Primer                               | Phase 1       |
| V4  | Interrupted node re-runs from its first line                                                                                                                          | Primer                               | Phase 1       |
| V5  | Node retry API                                                                                                                                                        | Primer                               | Phase 1       |
| V6  | Local Studio or dev server worth adding                                                                                                                               | Primer                               | Phase 1       |
| V7  | Import-boundary checker choice                                                                                                                                        | Architecture, repo layout            | Phase 1       |
| V8  | Docs tool: Zensical's API-reference support versus Material or Sphinx                                                                                                 | ADR-0010                             | Phase 1 spike |
| V9  | Partial or treeless clones for managed clones                                                                                                                         | ADR-0008, config                     | Phase 2       |
| V10 | ~~Source of the owner's ~9 calls/min GitHub limit~~ **Settled:** the code-search bucket (10 per minute)                                                               | ADR-0012                             | Done          |
| V11 | Keyring behavior on Windows, macOS, Linux                                                                                                                             | Config                               | Phase 1       |
| V12 | Fireworks structured-output behavior per chosen model                                                                                                                 | Providers                            | Phase 4       |
| V13 | OpenRouter provider preferences and data-collection options                                                                                                           | Providers, Policy                    | Phase 4       |
| V14 | Copilot SDK status, package names, Python support                                                                                                                     | Providers                            | Phase 6       |
| V15 | Copilot BYOK login requirement (test on the owner's account)                                                                                                          | Providers                            | Phase 6       |
| V16 | Copilot unattended runs from inside WSL2                                                                                                                              | Providers, ADR-0011                  | Phase 6       |
| V17 | Antigravity headless flags and output format                                                                                                                          | Providers                            | Phase 6       |
| V18 | Antigravity subscription terms for automation (no official answer found)                                                                                              | Providers                            | Phase 6       |
| V19 | Antigravity model access for the owner's plan tier                                                                                                                    | Providers                            | Phase 6       |
| V20 | Anthropic terms, only if a Claude runner is ever used                                                                                                                 | Providers                            | Before use    |
| V22 | `GET /rate_limit` is not counted against the limit                                                                                                                    | Planning reference, node 1           | Phase 2       |
| V23 | GraphQL field availability: sub-issues, `closedByPullRequestsReferences`, timeline items                                                                              | Planning reference, node 2           | Phase 2       |
| V24 | Cost and bounds of the first issue/PR mirror sync on a large repo                                                                                                     | Planning reference, node 8           | Phase 2       |
| V25 | Commit-to-PR mapping for ambiguous cases: GraphQL batch versus REST endpoint                                                                                          | Planning reference, node 10          | Phase 2       |
| V26 | tree-sitter packaging: `tree-sitter-language-pack` versus core package plus pinned grammars; Linux and macOS wheels (native Windows best-effort); on-demand downloads | Indexing, ADR-0017                   | Phase 3 spike |
| V27 | SQLite FTS5 and trigram tokenizer availability on every platform tier                                                                                                 | Indexing                             | Phase 1       |
| V28 | Exact-match strategy: trigram, bounded scan, or ripgrep                                                                                                               | Indexing                             | Phase 3       |
| V29 | Windows path-trap list (reserved names, alternate streams, 8.3 names, `\\?\` prefixes)                                                                                | Policy and sandbox                   | Phase 1       |
| V30 | Redaction rule provenance and licenses of any seeded open-source rules                                                                                                | Policy, ADR-0016                     | Phase 1       |
| V32 | ~~Native Windows containment options~~ **Dropped:** `cog` runs in WSL2 and Implementation uses containers (ADR-0018)                                                  | Policy                               | Phase 6       |
| V33 | Reproducible fixture-repo commits across platforms and autocrlf settings                                                                                              | Testing                              | Phase 1       |
| V34 | Timeline fields needed to replay an issue as it was before the fix (eval leakage prevention)                                                                          | Testing                              | Phase 3       |
| V35 | Docs tooling: CLI reference generation, executed examples, changelog tool, draw.io export in CI, spell and style checks                                               | Documentation strategy               | Phase 1       |
| V36 | Path form for opening a file that lives inside WSL2 from Windows (network share or opener helper)                                                                     | Platforms                            | Phase 1       |
| V37 | SQLite write-ahead mode reliability on `/mnt/*`, the reason the data directory stays on the distro's disk                                                             | Platforms, Config                    | Phase 1       |
| V38 | `/etc/wsl.conf` automount and interop settings for an optionally hardened distro                                                                                      | Platforms, Policy                    | Phase 6       |
| V39 | WSL2 default resource limits, `.wslconfig` keys, and virtual-disk shrink behavior                                                                                     | Platforms                            | Phase 1       |
| V40 | Creating a dedicated WSL distro instance on the installed WSL version; `wsl --export` for backups                                                                     | Platforms                            | Phase 1       |
| V41 | Docker Desktop licensing versus Docker Engine inside WSL2; macOS container runtime options                                                                            | Platforms, ADR-0018                  | Phase 6       |
| V42 | Command discovery quality on the owner's repos: Gradle convention plugins, npm test runners, Playwright, multi-module builds                                          | ADR-0019, Planning reference node 9  | Phase 2       |
| V43 | Current Ubuntu LTS release available for WSL                                                                                                                          | Platforms                            | Phase 1       |

V21 and V31 were duplicates of V1 and V13 and were removed. Ids are not reused.

---

## 5. Risks

| Risk                                           | Impact                       | Mitigation                                                                                                                                                                |
| ---------------------------------------------- | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Vendor CLI or SDK behavior changes             | Runner breaks                | Pinned versions; live contract tests; runners are optional                                                                                                                |
| Subscription terms change or prove restrictive | Cheap path disappears        | Direct providers are the base; runners can be disabled by config                                                                                                          |
| LangGraph API changes                          | Upgrade pain                 | Version pin; checkpoint round-trip test; thin wrapper conventions                                                                                                         |
| Index recall is the ceiling on plan quality    | Weak plans                   | Recall eval before tuning; per-retriever measurement                                                                                                                      |
| Windows-specific process and path issues       | Flaky Implementation         | `cog` runs in WSL2 on Windows; containers for Implementation; Linux CI plus a Windows smoke job                                                                           |
| WSL2 data loss or resource starvation          | Lost plans; slow machine     | Opt-in backup directory, `cog backup`, `wsl --export`, `.wslconfig` limits, `cog doctor` checks ([Platforms and environments](explanation/platforms-and-environments.md)) |
| Command discovery misses a repo's commands     | Plans lack acceptance checks | Config overrides; provenance and confidence on every entry; Phase 2 measurement on the owner's repos; gated `custom_argv` (ADR-0019)                                      |
| Undeclared file edits during Implementation    | Failed tasks or conflicts    | Detect-and-review scope rule (ADR-0020)                                                                                                                                   |
| GitHub rate limits                             | Slow or failed runs          | Local clones and mirror; GraphQL for fan-out; conditional requests; budget accounting                                                                                     |
| Prompt injection through issue text            | Unsafe actions               | Untrusted-text delimiting; read-only tools; deny-list; sandbox                                                                                                            |
| Scope growth before value                      | Never ships                  | Documentation gates; walking skeleton first; Implementation designed later                                                                                                |

---

## 6. Bootstrap checklist

- [x] Documentation Round 2 delivered
- [ ] Documentation Round 2 reviewed
- [x] Documentation Round 3 delivered
- [ ] Documentation Round 3 reviewed
- [x] Documentation Round 4 delivered
- [ ] Documentation Round 4 reviewed
- [ ] Owner sign-off; verification log reviewed
- [x] Repository created; docs committed as the first commit
- [ ] License decided
- [x] `gh api rate_limit` output recorded (see ADR-0012 addendum)
- [ ] Phase 1 started
