# Testing and Evaluation

**Type:** Explanation · **Status:** Draft v1 · **Last updated:** 2026-10-06
Items marked **(verify)** are tracked in the [roadmap's verification log](../roadmap.md#verification-log).

The Planning workflow's first twelve nodes are deterministic, so most of the system is testable **without an LLM and without a network**. This page defines how.

---

## 1. Principles

1. **Offline by default.** The default `pytest` run needs no network, no credentials, and no provider.
2. **Fakes, not mocks of internals.** Tests inject fake implementations of the real interfaces (`FakeLLM`, `FakeRunner`, a fake `VCSHost`) through the same dependency container production uses.
3. **Every contract has a test.** Every node, retriever, routing function, and policy rule is covered by a test written against its documented contract, not by a coverage percentage.
4. **Determinism.** Inject the clock, sort before comparing, fix random seeds, and make fixture repos reproducible.
5. **Windows first, Linux always.** The suite runs on both in CI; macOS is best-effort.
6. **Live is opt-in.** Anything that touches a real provider, CLI, or GitHub is marked `live`, skipped by default, and run on purpose.

---

## 2. Test tiers

| Tier            | Directory            | Needs                                                               | Speed             | Runs in CI                         |
| --------------- | -------------------- | ------------------------------------------------------------------- | ----------------- | ---------------------------------- |
| **Unit**        | `tests/unit/`        | Nothing                                                             | Fast              | Every push                         |
| **Integration** | `tests/integration/` | Real SQLite, real `git` on fixture repos, recorded GitHub responses | Medium            | Every push                         |
| **Graph**       | `tests/graph/`       | `FakeLLM`, fixture repos, temp data directory                       | Medium            | Every push                         |
| **Eval**        | `tests/eval/`        | Golden set, recorded data, managed clones at fixed commits          | Slow              | Smoke subset on push; full nightly |
| **Live**        | `tests/live/`        | Real providers, CLIs, GitHub; credentials                           | Slow, costs money | **Never automatically**            |
| **Snapshots**   | `tests/snapshots/`   | Nothing                                                             | Fast              | Every push                         |

Markers (`live`, `eval`, `slow`) select tiers. A plain `pytest` runs unit, integration, graph, and snapshots.

---

## 3. Unit tests

- **Extractors** (node 4): golden inputs to expected `Reference` lists, including false-positive traps (`#define`, `#` in code blocks).
- **Routing functions:** pure functions tested on tables of states.
- **`derive_batches`:** property-based tests (Hypothesis) assert invariants: no task precedes its dependencies, batches are layers of the DAG, write-sets within a batch are disjoint, and cycles or unknown ids produce `graph_errors`, never exceptions.
- **Plan validator:** one test per check, plus combined failures.
- **Path normalization, identifier splitting, redaction, deny-list:** property and corpus tests ([Policy and sandbox](policy-and-sandbox.md)).
- **Config layering:** precedence, tighten-only enforcement, and `--explain` output.
- **Event shapes:** costs are `None` when unknown, never `0`.

---

## 4. Fakes and recorded data

### `FakeLLM`

Scripted by call sequence or by matcher. It can return:

- valid structured outputs;
- **deliberately invalid** ones (wrong schema, invented paths, cyclic dependencies) to exercise the repair loop and the validator;
- injected errors (rate limit, timeout, content filter, malformed JSON);
- usage numbers, so budget and cost accounting are testable.

### Record/replay for LLM calls

Modes: `live`, `record`, `replay`, `cache-only`. A recorded call is stored as one JSON line keyed by a hash of model, prompt version, rendered messages, and parameters, under `tests/fixtures/llm/`. Rules:

- Recordings pass through the redactor before being written.
- A changed prompt changes the key, so stale recordings are detected, not silently reused.
- Re-recording is a deliberate command (`--record`), reviewed in the diff.

### Recorded GitHub responses

- The `GitHubClient` is tested at the **HTTP level** (a mock transport such as respx) with recorded GraphQL and REST responses.
- A recorder script captures real responses from a chosen repo, **scrubs tokens and personal data**, and writes fixtures with the query that produced them.
- Query text is snapshot-tested so an accidental change to a GraphQL query is visible.
- Rate-limit headers, conditional-request 304 responses, and secondary-limit errors have dedicated fixtures.

### Fixture repos

Small synthetic repos, built by a script, one per ecosystem: **Java** (Maven wrapper layout), **TypeScript** (npm), **Python** (uv). Each has issues-worth of history: renames, co-changing files, squash-merge-style commit subjects, tests alongside sources.

- Built with `git` using fixed author, committer, and date environment variables so commit ids are **reproducible** **(verify across platforms, including autocrlf settings)**.
- Include Windows cases: CRLF files, names differing only by case, Unicode names, deep paths.
- Synthetic content only: no secrets, no third-party code with awkward licenses.

---

## 5. Integration tests

- Index builds on fixture repos: symbols, imports, test map, co-change, repo map, parse-cache reuse, atomic publish, skipped-file reasons.
- Retrievers end to end against the index and recorded GitHub data, including `skipped` (no mirror) and `error` paths.
- Stores: plan files, approvals, metadata rebuild from files, migrations on a copy of an old schema, blob-store integrity.
- `git` helpers on Windows paths and CRLF.
- The issue/PR mirror: incremental sync against recorded pages.

---

## 6. Graph tests

Whole-graph runs with fakes, in a temporary data directory:

| Scenario                                      | Asserts                                                                          |
| --------------------------------------------- | -------------------------------------------------------------------------------- |
| Happy path                                    | All 22 nodes run; plan persisted; events recorded; costs `None` or consistent    |
| Retriever failure                             | One retriever returns `error`; run completes with a warning                      |
| Insufficient issue                            | Pause `clarify`; resume with answer; one round only                              |
| Invalid plans                                 | Repair loop runs up to 3 times, then pauses `validation_failed`                  |
| Gap resolution                                | Up to 2 rounds; `human_question` gaps carried to review                          |
| Review decisions                              | Approve, edit (re-derive and re-validate; new version), reject (back to planner) |
| **Pause and resume across a process restart** | State from the file-based checkpointer survives a new process                    |
| Budget breach                                 | Pause `budget`; resume with a raised cap                                         |
| Cancel                                        | Run stops at the next safe point; no partial plan                                |
| Interrupt-first                               | Nodes 14 and 21 perform no side effects before `interrupt()`                     |
| Checkpoint round trip                         | A state containing every model type survives save and load                       |
| **Drift**                                     | Generated Mermaid, the node registry, and the draw.io node names agree           |

---

## 7. Snapshot tests

- CLI `--help` for every command (and later the generated reference).
- JSON Schemas for `Plan`, `Task`, and other handoff contracts. A change without a `schema_version` bump fails ([Documentation strategy](documentation-strategy.md)).
- Rendered review HTML for a fixture plan, including **adversarial text** (script tags, markdown injection) proving escaping.
- Event-store schema and migrations.

---

## 8. Live contract tests

Opt-in, run on your machine, never in CI, each with a spending cap and a cost report.

| Target                                          | Questions answered                                                                                                                      |
| ----------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| Each LLM provider/model (Fireworks, OpenRouter) | Does structured output work with the declared mode? Are capabilities as declared? Is usage reported?                                    |
| Each runner (Copilot, Antigravity, others)      | Does an **unattended** run behave like an interactive one: shell, permission mode, output format, exit codes, process-tree termination? |
| GitHub                                          | Do the real GraphQL queries still return the fields we depend on? Rate-limit headers as expected?                                       |

Results are recorded with tool versions, and a generated "verified versions" table can be published in the docs. A failing live test does not fail CI; it flags an adapter for attention. This is how the verification items for providers get settled.

---

## 9. Evaluation: context recall

Tests check that code matches its contract. **Evaluation** checks that the system is _good_. The primary eval needs no LLM.

### What it measures

Take historical issues that have a merged closing PR. Run nodes 1–12 as of the PR's base commit. Measure what fraction of the files (and, ideally, symbols) the real PR touched appeared in the packed context.

| Metric                         | Definition                                                                                  |
| ------------------------------ | ------------------------------------------------------------------------------------------- |
| **Recall (packed)**            | Touched files present in the packed context ÷ touched files                                 |
| **Recall by tier**             | The same, split by which tier surfaced each file                                            |
| **Candidate recall and rank**  | Recall in the pre-budget candidate list, and mean reciprocal rank of the first touched file |
| **Precision**                  | Packed items the PR touched (or the later plan cites) ÷ packed items                        |
| **Per-retriever contribution** | **Ablation:** rerun with one retriever removed and measure the drop                         |
| **Cost**                       | Tokens in the packed context, time per node, GitHub API calls                               |

### Building the golden set

A script (`scripts/build_golden_set.py`, planned) mines merged PRs that close issues, and writes one record per case: repo, issue number, base commit, the PR's files, and optionally symbols. Selection rules:

- The PR touches source files, not only docs or lockfiles.
- The issue has enough text to plan from.
- Exclude bot-authored PRs and very large PRs (cap the file count).
- Balance across languages and repos (Java, TypeScript, Python; back end, front end, test automation).
- Target **20–50 cases to start**, grown over time.

**Prevent leakage.** An issue's later comments may name the files the fixing PR changed, which would make the eval trivially easy. Replay the issue **as it was before the fix began**: drop comments and edits after the PR's first commit or open time, and use the PR's base commit, not the default branch head **(verify the timeline fields available to enforce this)**.

**Avoid overfitting.** Split the set into a development part (used while tuning) and a held-out part (checked occasionally, never tuned against).

### Running it

`cog eval run` replays recorded issue data and uses managed clones at the recorded commits, so it is repeatable and offline. It writes a JSON result and a Markdown summary table, keeps history under `runs/`, and compares against the stored baseline. A change that lowers recall beyond a threshold fails the nightly job; thresholds are set after the first baseline.

Use it to tune retrievers, tiers, and budgets, and to decide whether embeddings are worth adding.

---

## 10. Evaluation: later stages

- **Planner quality.** Metrics that do not need an LLM judge: validator first-pass rate, repair iterations, gap rounds, human edit and reject rates, cost per plan, and traceability coverage. LLM-judged scores are advisory only, since LLM review is the least reliable validator. Use recorded context snapshots so prompt or model changes are compared on **identical inputs**.
- **Prompt changes** are treated like code changes: snapshot tests plus an eval run before merge when they could affect recall or plan quality.
- **Convention adherence** (optional): whether plans, and later changes, match the repo's observed patterns ([ADR-0014](../decisions/0014-repo-convention-precedence.md)).
- **Implementation:** first-pass task success per runner and tier, escalation rate, batch conflict rate, time and cost. Designed with the Implementation workflow ([outline](../workflows/implementation/outline.md)).

---

## 11. Continuous integration

| Job            | Runs                                                         | Contents                                                            |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------------- |
| Lint and types | Every push                                                   | ruff, pyright, import-boundary check                                |
| Tests          | Every push, **Windows and Linux** matrix (macOS best-effort) | Unit, integration, graph, snapshots                                 |
| Docs           | Every push                                                   | Strict docs build, link check, executed examples, schema diff check |
| Eval smoke     | Every push                                                   | Small subset, replay only                                           |
| Eval full      | Nightly                                                      | Whole golden set, baseline comparison                               |
| Live           | Manual                                                       | Contract tests on your machine                                      |

Keep the default job fast. Targets are set from measurements in Phase 1.

---

## 12. Hygiene

- **Flaky tests** are quarantined with a marker and an issue; retries never hide them.
- **Windows file locking:** SQLite connections are closed before temporary directories are removed; long-path and CRLF cases are tested deliberately.
- **Secrets:** fixtures and recordings are scanned by the redactor in CI; a hit fails the build.
- **Time and randomness** are injected, never read globally.
- **Test data ownership:** fixture repos are generated, so they stay small and license-clean.

---

## 13. Open questions

- Initial thresholds for eval regressions and CI time budgets (set from the first baseline).
- Whether the golden set draws from your own repos, public repos, or both, and the licensing implications of recording public data.
- How much of the mirror is replayed in evals versus rebuilt.
- Whether Hypothesis-style property tests extend to path normalization on real Windows file systems in CI.
