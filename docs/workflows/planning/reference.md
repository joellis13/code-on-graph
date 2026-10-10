# Planning Workflow — Reference (v3)

**Type:** Workflow reference · **Status:** Draft v3 · **Last updated:** 2026-10-08
Companion to [`workflow.drawio`](workflow.drawio). Node numbers here match the diagram.

**Revision v3 (2026-10-06):** aligns the workflow with the foundation documents and decisions.

1. **Storage** rewritten for the data directory and immutable plan files, with no footprint in target repos ([ADR-0007](../../decisions/0007-plans-as-immutable-json.md), [ADR-0008](../../decisions/0008-standalone-command-center-managed-clones.md)).
2. **LangGraph notes** trimmed to planning-specific patterns and linked to the [primer](../../explanation/langgraph-primer.md). Added the node contract decorator, dependency injection, async nodes, the `RetrieverResult` contract, and the prepare / review / apply rule for interrupt nodes.
3. **LLM nodes (11, 13, 15, 19)** name their `LLMClient` role, tier, structured-output handling, and caching ([Providers](../../explanation/providers.md)).
4. **Principle 9:** repo conventions outrank harness preferences ([ADR-0014](../../decisions/0014-repo-convention-precedence.md)).
5. **GitHub access** (nodes 1, 2, 7, 8, 10) uses GraphQL, a local issue/PR mirror, and API budget accounting ([ADR-0012](../../decisions/0012-github-integration.md)). The pipeline never calls the code-search API.
6. **Security wiring:** path deny-list applied at fetch time, secret redaction in node 12, rendered-HTML hardening in node 20.
7. **Node registry metadata** added (Section 4) so reference tables can be generated.
8. **Open decisions (Section 10)** resolved or moved; **Section 11** keeps the Planning-specific rejected alternatives and links the ADRs.
9. The **glossary** moved to the [shared glossary](../../reference/glossary.md).

**Revision v2.1:** added principle 8, a positioning note, the `QueryPlan` contract for node 11, and Section 11 (decisions and rejected alternatives).

**Status of the design:** proposal for the Code-OG (`cog`) Planning workflow, built on Python and LangGraph. Items marked **(verify)** depend on external APIs or library behavior that must be checked against current docs before implementation; all are tracked in the [roadmap's verification log](../../roadmap.md#verification-log).

**Related:** [Architecture](../../explanation/architecture.md) · [Providers](../../explanation/providers.md) · [Configuration and data](../../explanation/configuration-and-data.md) · [Interaction model](../../explanation/interaction-model.md) · [ADR index](../../decisions/README.md)

---

## 1. Purpose and principles

The planning workflow takes a GitHub issue ID and produces a **validated, human-approved implementation plan** that a small, fast model can execute task by task.

**Positioning.** Deterministic-first is one established approach, not the only one. Many widely used coding agents rely mostly on LLM-driven search loops, and others (Agentless, Aider's repo map) lean on fixed pipelines. This design is deliberately hybrid: a deterministic pipeline up to the planner, bounded LLM steps where judgment is needed, and code that checks every LLM output. Its benefits (reproducibility, cost, debuggability) should be measured, not assumed; see Section 9.

Principles:

1. **Deterministic where checkable, LLM where judgment is required, deterministic on both sides of every LLM call.** Context building happens before the LLM; validation happens after.
2. **Fetch generously, pack selectively.** Store everything retrieved. Send the LLM a budgeted slice plus a manifest of what else exists.
3. **Every node has a typed contract.** Input schema, output schema, and an explicit failure status. Nodes never raise for expected conditions (e.g., "nothing found"); they return a status.
4. **The LLM never asserts structure that code can derive.** Batches, dependency order, and parallelism are computed, not trusted.
5. **Every loop is bounded.** No unbounded LLM-driven cycles. After the limit, escalate to the human.
6. **The structured plan (JSON) is the source of truth.** HTML is only a view.
7. **Measure before adding complexity.** Embeddings, extra retrievers, and LLM review are added when evals show a gap.
8. **LLM outputs are proposals, not facts.** Anything an LLM names (paths, symbols, commands, queries) is checked by code before anything depends on it. In the retrieval stage the LLM never emits file paths at all; paths come from extractors and are verified in node 5. In the planner, paths and commands are verified by node 18.

9. **Repo conventions outrank harness preferences.** The harness has opinions about process (typed contracts, bounded loops, validated LLM output) and none about code style. Precedence, highest first: explicit repo rules (AGENTS.md, CONTRIBUTING, lint and format configs, repo-config style notes); patterns observed in nearby code; nothing. See [ADR-0014](../../decisions/0014-repo-convention-precedence.md).

### Legend (matches diagram colors)

| Color | Meaning |
|---|---|
| Green | Deterministic node |
| Green, dashed orange border | Deterministic with an optional, bounded, validated LLM step |
| Orange | LLM node |
| Blue | Human-in-the-loop |
| Yellow diamond | Decision / conditional edge |
| Gray | Start, end, persisted artifact |

---

## 2. Flow at a glance

```
START (cog plan) → 1 preflight → 2 fetch_issue → 3 normalize_issue
      → 4 extract_references → 5 validate_references
      → STAGE 1 (parallel): 6 code_retriever | 7 linked_prs | 8 related_issues | 9 repo_conventions
      → STAGE 2 (parallel, needs code hits): 10 related_prs | 11 docs_retriever
      → 12 context_packer (join)
      → 13 sufficient?  ── No → 14 ask_human (HITL) ─┐
                        └─ Yes ───────────────────────┤
      → 15 planner (LLM)  ◄───────────────────────────┘
      → 16a context_gaps? ── Yes → 16b gap_resolver → back to 15
                          └─ No
      → 17 derive_batches → 18 plan_validator → valid?
            No → back to 15 (≤ 3 retries, then HITL with errors)
            Yes → 19 llm_plan_review (optional) → 20 render_plan → 21 hitl_review
                  Approve → 22 persist_plan → END (handoff to Implementation)
                  Edit → back to 17 (re-derive, re-validate)
                  Reject + feedback → back to 15
```

**Pause points (shown by the CLI as `paused:<reason>`):** node 14 (`clarify`), node 21 (`review`), node 18 after retries (`validation_failed`), and any node on a budget breach (`budget`). Continue with `cog resume <run_id>`.

### Why two retrieval stages

`related_prs` and `docs_retriever` both benefit from `code_retriever`'s output:

- `related_prs` runs `git log` on the candidate files the code retriever found.
- `docs_retriever` harvests vocabulary (module and function names) from code hits, because issue text rarely uses the same words as docs.

`linked_prs`, `related_issues`, and `repo_conventions` have no such dependency and run in Stage 1 alongside the code retriever.

---

## 3. Core schemas (sketch)

Use Pydantic v2 models for all state values (the state container itself is a TypedDict; see Section 7). These are starting points, not final.

```python
from typing import Literal, Optional
from pydantic import BaseModel, Field

class NormalizedIssue(BaseModel):
    repo: str
    number: int
    title: str
    body_md: str
    comments: list["Comment"]
    labels: list[str]
    state: Literal["open", "closed"]
    linked_issue_numbers: list[int] = []
    updated_at: str                     # cache key component
    sections: dict[str, str] = {}       # parsed issue-template sections, if any

class Reference(BaseModel):
    id: str
    type: Literal["issue_ref", "pr_ref", "url", "path", "stack_frame",
                  "identifier", "error_string", "version", "label_area"]
    raw: str                            # exact text found
    source: str                         # where in the issue (body, comment id, title)
    status: Literal["unchecked", "exists", "renamed", "not_found"] = "unchecked"
    resolved: Optional[str] = None      # e.g., new path after rename

class ContextItem(BaseModel):
    id: str                             # stable, citable by tasks
    kind: Literal["code", "doc", "pr", "issue", "convention", "git_history", "exemplar"]
    path: Optional[str] = None
    symbol: Optional[str] = None
    line_range: Optional[tuple[int, int]] = None
    snippet: str
    reason: str                         # why it was retrieved
    retriever: str                      # which node produced it
    query: Optional[str] = None
    tier: Literal["explicit", "symbol", "lexical", "neighbor", "semantic", "gap_fill"]
    score: float
    tokens: int

class ContextGap(BaseModel):
    kind: Literal["file", "symbol", "search", "doc", "human_question"]
    query: str
    why_blocking: str                   # forces the LLM to justify the request

class RetrieverResult(BaseModel):
    retriever: str                      # e.g., "code", "linked_prs", "gap_resolver"
    status: Literal["ok", "empty", "skipped", "error"]
    items: list[ContextItem] = []
    diagnostics: dict = {}              # queries run, counts, timings, API calls used
    error: Optional[str] = None
    warnings: list[str] = []

class AcceptanceCheck(BaseModel):
    kind: Literal["command"]
    cmd: str
    expect: str                         # e.g., "exit 0"

class Task(BaseModel):
    id: str
    title: str
    description: str                    # complete enough for a small model to execute
    depends_on: list[str] = []
    files_to_modify: list[str] = []
    files_to_create: list[str] = []
    context_refs: list[str]             # ContextItem ids this task relies on
    acceptance: list[AcceptanceCheck]
    stop_conditions: list[str] = []     # when the executor should stop and report
    executor_tier: Literal["small", "medium", "large"] = "small"

class Plan(BaseModel):
    schema_version: int = 1             # persisted plans stay readable across versions
    issue: int
    summary: str
    requirements: list[str]             # extracted from the issue; used for traceability
    tasks: list[Task]
    assumptions: list[str] = []
    open_questions: list[str] = []
    context_gaps: list[ContextGap] = []
    # NOT set by the LLM; filled by derive_batches:
    batches: list[list[str]] = []
    graph_errors: list[str] = []
```

State-level keys that merge across parallel branches (see Section 7):

- `retriever_results: Annotated[list[RetrieverResult], operator.add]`. Every retriever, and `gap_resolver`, writes one result; the packer flattens them into the `PreloadedContext` and derives statuses from them.
- `warnings: Annotated[list[str], operator.add]`

State holds ids and hashes for large payloads (the raw issue, the context snapshot), not the payloads themselves.

---

## 4. Node reference

Each entry lists type, purpose, inputs and outputs, internal workflow, failure handling, and notes. Every node module in the code carries the same four headings (Purpose, Inputs and outputs, Internal workflow, Failure handling) in its docstring, so this reference can be generated from the code later.

### 4.0 Node registry

One row per node. This table is the model for the registry the code will hold; reference tables and the diagram drift check are generated from it ([ADR-0003](../../decisions/0003-langgraph-orchestration-and-state.md)).

**Fields.** *Kind:* `terminal`, `deterministic`, `hybrid` (deterministic with an optional bounded LLM step), `llm`, `hitl`, `decision`. *Role:* the `LLMClient` role resolved to a tier by config (see Section 7). *Idempotent:* safe to re-run (matters for retries and resume); *interrupt-first* nodes pause before doing anything with side effects. *Status:* build status (`spec`, `stub`, `implemented`, `tested`).

| # | Id | Kind | LLM role | Idempotent | Side effects | Planned module | Status |
|---|---|---|---|---|---|---|---|
| START | `start` | terminal | none | n/a | Creates `run_id`, run record | `graph.py` / CLI | spec |
| 1 | `preflight` | deterministic | none | yes | Fetches managed clone; writes run record | `nodes/preflight.py` | spec |
| 2 | `fetch_issue` | deterministic | none | yes (cached) | API calls; raw blob; mirror update | `nodes/fetch_issue.py` | spec |
| 3 | `normalize_issue` | deterministic, pure | none | yes | None | `nodes/normalize_issue.py` | spec |
| 4 | `extract_references` | deterministic, pure registry | none | yes | None | `nodes/extract_references.py` | spec |
| 5 | `validate_references` | deterministic | none | yes | Git reads | `nodes/validate_references.py` | spec |
| 6 | `code_retriever` | deterministic | none | yes | Index build (cached by `base_sha`) | `retrieval/code.py` | spec |
| 7 | `linked_prs` | deterministic | none | yes | API calls (cached) | `retrieval/linked_prs.py` | spec |
| 8 | `related_issues` | deterministic | none | yes | Incremental mirror sync | `retrieval/related_issues.py` | spec |
| 9 | `repo_conventions` | deterministic | none | yes | None (reads clone) | `retrieval/conventions.py` | spec |
| 10 | `related_prs` | deterministic | none | yes | `git log`; some API calls | `retrieval/related_prs.py` | spec |
| 11 | `docs_retriever` | hybrid | `query_gen` (optional) | yes (LLM cached by input hash) | LLM call when enabled | `retrieval/docs.py` | spec |
| 12 | `context_packer` | deterministic | none | yes | Writes context snapshot blob | `retrieval/packer.py` | spec |
| 13 | `sufficient?` | decision | `verdict` (optional) | yes | Optional LLM call | `nodes/sufficient.py` | spec |
| 14 | `ask_human` | hitl | none | **interrupt-first** | Pause; stores answers | `nodes/ask_human.py` | spec |
| 15 | `planner` | llm | `planner` | no (each run is a new, logged call) | LLM and read-only tool calls | `nodes/planner.py` | spec |
| 16a | `context_gaps?` | decision | none | yes | None | `routing.py` | spec |
| 16b | `gap_resolver` | deterministic | none | yes | Reads; docs search | `nodes/gap_resolver.py` | spec |
| 17 | `derive_batches` | deterministic | none | yes | None | `nodes/derive_batches.py` | spec |
| 18 | `plan_validator` | deterministic | none | yes | Git and index reads | `nodes/plan_validator.py` | spec |
| 19 | `llm_plan_review` | hybrid (optional) | `reviewer` | yes | LLM call | `nodes/llm_plan_review.py` | spec |
| 20 | `render_plan` | deterministic | none | yes | Writes derived HTML | `nodes/render_plan.py` | spec |
| 21 | `hitl_review` | hitl | none | **interrupt-first** | Pause | `nodes/hitl_review.py` | spec |
| 22 | `persist_plan` | deterministic | none | yes (by plan hash) | Plan and approval files; metadata rows | `nodes/persist_plan.py` | spec |
| END | `end` | terminal | none | n/a | None | `graph.py` | spec |

---

### START — `cog plan <issue>`

**Type:** terminal

**Input:** an issue reference, either `owner/repo#123` or a bare `123` inside a directory that matches a known managed repo. The repo must already be registered with `cog repo add`; if it is not, stop with that instruction. The CLI creates a `run_id`, which is also the LangGraph `thread_id`, and writes the initial run record.

**Flags that matter here:** `--base <ref>` (plan against a specific commit or branch instead of the default branch head), `--offline` (replay from the local mirror and raw store only), `--non-interactive` (pauses exit with a distinct code instead of prompting). See the [Interaction model](../../explanation/interaction-model.md).

---

### 1. `preflight`

**Type:** deterministic

**Purpose:** Fail fast and cheaply before spending time, API budget, or tokens.

**Internal workflow:**
1. Resolve the repo (explicit argument, else match the current directory's git remote to a registered managed repo).
2. Resolve credentials (`GITHUB_TOKEN`, then `gh auth token`) and verify them with a trivial authenticated call.
3. Check **API budget per bucket** with `GET /rate_limit` (core, GraphQL, search, code search; this call is not counted against the limit, **(verify)**). Warn or abort if core or GraphQL headroom is low. The pipeline never uses the search or code-search APIs, so those buckets are recorded for diagnostics only.
4. Confirm the issue exists. If the number resolves to a **pull request**, stop with a clear message (a PR is not a planning input).
5. Record issue state (open/closed). A closed issue is allowed, but add a warning that HITL will display.
6. **Sync the managed clone** (`git fetch`) and record `base_sha` (the default branch head, or `--base`). No working tree is needed; every read uses git objects at `base_sha`.
7. **Check providers:** every tier the workflow will use exists in config, its endpoint is reachable, and it declares the capabilities required (structured output, tool calling for the planner). Verify the egress policy allows this repo to be sent to each provider.
8. **Load policy and budgets:** resolve the layered config, persist it in the run record, set per-run and per-node token and dollar caps, and note any repo-supplied commands not yet approved (needed before Implementation, not for Planning).

**Outputs:** `base_sha`, `run_id`, `warnings`, a reference to the resolved config, and the budgets.

**Failure handling:** Hard-stop only on unrecoverable conditions (no auth, not found, is a PR, required capability missing). Each hard-stop names the exact fix, such as the environment variable to set. Everything else becomes a warning.

---

### 2. `fetch_issue`

**Type:** deterministic

**Purpose:** Retrieve everything GitHub knows about the issue and store it raw. Nothing is dropped here.

**Fetch list:**
- Issue body, title, author, timestamps, state, labels, assignees, milestone
- All comments (paginate)
- Timeline events (cross-references, linked PRs, label changes)
- Parent issue and sub-issues, if used
- Attachments: for images and logs, store the URL and metadata; download text logs if small
- Linked PRs through the timeline, or GraphQL `closedByPullRequestsReferences` **(verify current API)**

**Transport:** one GraphQL query through the `VCSHost` interface returns the fields above (comments paginated by cursor). REST is used only for pieces GraphQL lacks. **(verify)** field availability, including sub-issues. See [ADR-0012](../../decisions/0012-github-integration.md).

**Internal workflow:**
1. Run a minimal GraphQL query for `updatedAt`. If `(repo, issue_number, updated_at)` matches a cached raw record, return it. In `--offline` mode only the cache and mirror are consulted, and a miss is a hard stop.
2. Fetch the pieces, with retry and backoff on transient errors. Honor `Retry-After`, and treat primary and secondary limits as different conditions. If the budget is exhausted, stop with `failed: rate_limited` and the reset time.
3. Write the raw JSON to the blob store (unmodified), and update the issue/PR mirror.
4. Return a pointer plus metadata, not the full blob, in graph state.
5. Record API calls used, by bucket, in the event store.

**Outputs:** `raw_issue_ref`.

**Notes:** Over-fetching is cheap; the packer decides what reaches the LLM. Keep the raw record so you can replay a run offline.

---

### 3. `normalize_issue`

**Type:** deterministic, pure function

**Purpose:** Convert raw GitHub JSON into the stable `NormalizedIssue` schema so downstream nodes never touch GitHub's shapes.

**Internal workflow:**
1. Map fields; convert timestamps to ISO-8601 UTC.
2. Normalize Markdown: strip tracking pixels, collapse excessive whitespace, but **preserve code blocks and backticks exactly** (they feed identifier extraction).
3. If the repo uses issue templates, parse sections by heading (e.g., "Steps to reproduce", "Expected behavior") into `sections`.
4. Order comments chronologically; tag bot comments.
5. Validate against the schema.

**Failure handling:** Schema validation errors indicate a bug or an API change; fail loudly with the offending payload saved.

---

### 4. `extract_references`

**Type:** deterministic, a registry of pure extractors

**Purpose:** Find explicit, checkable entities in the issue text. High precision, no judgment.

**Design:** Each extractor is a pure function `(NormalizedIssue) -> list[Reference]`, registered by name. The node runs all registered extractors and concatenates results. Adding a type means adding one function and one test; the graph doesn't change.

| Extractor | Finds | Method | Caveat |
|---|---|---|---|
| `issue_pr_refs` | `#123`, `owner/repo#123`, GitHub issue/PR URLs | regex | Ignore matches inside code blocks if they're not real refs (e.g., `#define`) |
| `urls` | external links, docs links | regex / URL parser | Classify by host (docs vs. paste sites vs. other) |
| `file_paths` | `src/foo/bar.py`, `./config.yaml` | regex against known extensions plus repo-root-relative patterns | Paths may be stale; validated in node 5 |
| `stack_frames` | Python `File "x.py", line N`; JS `at fn (x.js:L:C)`; Java frames | per-language regex | Keep file, line, function; keep the full trace text |
| `identifiers` | backticked tokens, CamelCase, snake_case, dotted names | tokenization of code spans and prose | Many false positives in prose; validated in node 5 |
| `error_strings` | quoted error messages, exception names, error codes | regex for `Error:`/`Exception`, quoted strings in logs | Useful as literal search queries |
| `versions` | semver, commit SHAs, tags | regex | Commit SHAs can be checked with `git cat-file -e` |
| `label_area` | labels → code areas | lookup table in repo config | Table is hand-maintained; start small |

**Outputs:** `refs: list[Reference]` with `status="unchecked"`.

**Brittleness warning:** Regex is reliable for structured tokens (URLs, `#123`, stack frames) and unreliable for prose. Don't try to extract "meaning" here. Extract tokens and let node 5 and the retrievers verify them.

---

### 5. `validate_references`

**Type:** deterministic

**Purpose:** Check each reference against reality so downstream nodes work with verified facts and the planner can see what was real versus guessed.

**Internal workflow:**
1. **Paths:** does the file exist at `base_sha` (looked up in the managed clone's git tree; no working tree needed)? If not, check `git log --follow --diff-filter=R` for a rename and set `status="renamed"` with `resolved`. Otherwise `not_found`.
2. **Identifiers:** look up in the symbol index (node 6's index; build it here if absent). Exists → `exists`; otherwise `not_found`.
3. **Stack frames:** verify the file exists and the line is within bounds; capture a ±N-line snippet at `base_sha`. (Line numbers drift; flag if the file changed since the issue was created, using `updated_at` versus the file's last commit.)
4. **Commit SHAs:** `git cat-file -e`.
5. **Issue/PR refs:** confirm they exist (cheap API call, cached).
6. Drop nothing. Keep `not_found` refs, since "the issue mentions a symbol that doesn't exist" is itself useful signal for the planner and the human.

**Outputs:** `refs` with statuses.

**Notes:**
- Issues are often written on case-insensitive systems. If an exact path match fails but a case-insensitive match exists, resolve it with `status="renamed"` and a warning.
- References to paths on the policy deny-list are kept as references (the planner and human should see them) but their contents are never read or emitted.
- Paths are normalized to POSIX style before comparison and storage.

---

### Stage 1 — parallel, independent retrievers

All four read `refs` and `NormalizedIssue`, and each writes one `RetrieverResult` into the reducer-merged `retriever_results` key. They share one contract:

- **`status`:** `ok`; `empty` (searched, found nothing); `skipped` (a prerequisite is missing, for example no mirror yet); or `error`.
- **`items`:** `ContextItem`s with provenance.
- **`diagnostics`:** queries run, counts, timings, and API calls used.

Each retriever wraps its body so that expected failures **return a status** instead of raising. An `error` or `skipped` result becomes a warning, not a crash. Because retrievers are interchangeable, each lives in its own module, is unit-tested alone, and has its recall measured separately (Section 9).

---

### 6. `code_retriever`

**Type:** deterministic (embeddings optional)

**Purpose:** Find the code the planner needs. This is the most important retriever and the least standardized; the tiers below are a synthesis of established techniques, not a standard.

**Index prerequisites (built once per `base_sha` from the managed clone's git objects, cached):**
- Symbol index: definitions and references (tree-sitter, or `ast` for Python, or an LSP; ctags as a simple fallback)
- Lexical index: SQLite FTS5 over code chunks, with identifiers split at index time (see the [glossary](../../reference/glossary.md): *identifier splitting*), plus ripgrep for exact matches
- Import/dependency graph
- Optional: embeddings table
- Repo map (ranked signatures of important files/symbols, in the style of Aider)

**Tiered cascade (highest confidence first):**

| Tier | Source | Confidence |
|---|---|---|
| 0 `explicit` | Paths and stack frames named in the issue: read those files/regions | highest |
| 1 `symbol` | Symbols named in the issue: definitions, plus callers/references | high |
| 2 `lexical` | BM25 (FTS5) and ripgrep over error strings, validated identifiers, and key phrases | medium |
| 3 `neighbor` | Tests covering the files above, files they import / that import them, CODEOWNERS owners, sibling files in the same package or module (**exemplars**, emitted with `kind="exemplar"` so the planner can imitate local patterns), **co-change history** (files that historically change in the same commits) | medium |
| 4 `semantic` | Embedding search over code chunks (optional; add only if evals justify) | lowest |
| always | A compact repo map for orientation | n/a |

**Internal workflow:**
1. Collect tier-0 items from `refs` where `status in {exists, renamed}`.
2. For each tier-1 symbol, pull the definition body and a bounded number of references.
3. Run tier-2 searches with queries built from `error_strings`, validated `identifiers`, and (optionally) the issue title; fuse rankings with RRF if more than one query/index is used.
4. Expand with tier-3 neighbors of the top-K tier 0–2 files (tests first; co-change from `git log`).
5. Emit **symbol-level snippets**, not whole files: `{path, symbol, line_range, snippet, reason, tier, score}`.
6. Return `empty` if nothing was found at any tier, an explicit signal that the planner needs gap-filling or human help.
7. Never read or emit paths on the policy deny-list; denied hits are dropped and counted in `diagnostics`.

**Co-change note:** deterministic, cheap, and strongly predictive. Compute from `git log --name-only` over a recent window and cache a file-pair co-occurrence table.

**Languages:** Java, TypeScript, and Python come first, using tree-sitter grammars behind the `Index` interface (a language server can be added later). Repo size varies, so builds are cached by commit and designed for large repos. Details: [Indexing](../../explanation/indexing.md) and [ADR-0017](../../decisions/0017-index-design.md).

---

### 7. `linked_prs`

**Type:** deterministic

**Purpose:** Fetch PRs explicitly tied to the issue.

**Internal workflow:**
1. Collect PR numbers from timeline cross-reference events, closing references, and `pr_ref` references.
2. Fetch all of them in **one batched GraphQL query**: metadata, description, review comments, changed-file list, diff stats. Fetch a diff only if it is small (or only for files that overlap code hits). Cache by `(repo, pr, updated_at)`.
3. Emit `ContextItem(kind="pr", tier="explicit")`.

**Notes:** If a linked PR is **merged**, the issue may already be solved or partially solved, so add a warning for HITL.

---

### 8. `related_issues`

**Type:** deterministic

**Purpose:** Surface prior discussion and duplicates.

**Internal workflow:**
1. Explicitly linked issues (`linked_issue_numbers`, `issue_ref` refs): fetch title, state, and key comments in one batched GraphQL query.
2. Run FTS5 over the **local issue/PR mirror** using validated identifiers and error strings. The mirror is synced incrementally by `updatedAt`; the first sync pages through GraphQL and is bounded by config (for example, a time window) **(verify cost on a large repo)**.
3. If no mirror exists yet for the repo, return `skipped` with a warning. The GitHub search API (30 requests per minute) is **not** used by default.
4. Emit items with provenance; mark duplicates and "closed as completed" prominently.

---

### 9. `repo_conventions`

**Type:** deterministic

**Purpose:** Give the planner (and executors) the repo's own rules, so the plan follows the repo and not the harness's taste (principle 9).

**Internal workflow:**
1. Read, from the managed clone at `base_sha`, `AGENTS.md`, `CONTRIBUTING.md`, `CODEOWNERS`, PR/issue templates, any assistant-instruction files the repo carries, and the optional `.cog/config.toml`.
2. Read **style and tooling configs** as rules: for example `.editorconfig`, Checkstyle, Spotless, ESLint, Prettier, `tsconfig`, `ruff`, `pyproject.toml`, `tox.ini`. They are reported, not interpreted.
3. Discover how to test, lint, format, type-check, and build: parse `pom.xml` and Gradle files with their wrappers (`mvnw`, `gradlew`), `package.json` scripts, `Makefile`, `pyproject.toml`/`tox.ini`, and CI workflow files.
4. Emit a `conventions` item including a **resolved command table** (`test`, `lint`, `format`, `typecheck`, `build`). The plan validator later uses this table to check acceptance commands. Commands that come from repo-supplied config and are not yet approved are marked **untrusted**.
5. Include any free-text style notes from repo config, verbatim, for the planner.

**Notes:**
- **Exemplars** (sibling code to imitate) depend on code hits, so `code_retriever`'s neighbor tier selects them (node 6). This node supplies the rules and tool configs.
- Read through the deny-list. Wrapper scripts are resolved explicitly on Windows (`mvnw.cmd`, `gradlew.bat`, `npm.cmd`).

---

### Stage 2 — parallel, depends on `code_retriever`

---

### 10. `related_prs`

**Type:** deterministic

**Purpose:** Find *past* PRs that touched the same code, which are often the best exemplars for how changes are made here.

**Internal workflow:**
1. Take candidate files from `code_retriever` (tiers 0–2, top-K).
2. Run `git log --follow -n N -- <path>` per file at `base_sha` (history on `main`).
3. Map commits to PRs:
   - **Squash merges:** GitHub's default commit subject ends with `(#123)`; parse it.
   - **Merge commits:** subject `Merge pull request #123 from ...`.
   - **Rebase merges / anything ambiguous:** ask GitHub, batched in one GraphQL query where possible (a commit's associated pull requests) or via the REST commits-to-pulls endpoint **(verify)**. Each call is budget-accounted.
4. Rank PRs by number of overlapping files and recency; drop bot-only and revert/merge-noise commits.
5. For the top few, fetch title, description, and changed-file list.
6. If history yields too little, fall back to full-text search over the **local PR mirror**, not the GitHub search API (30 requests per minute).

---

### 11. `docs_retriever`

**Type:** hybrid: deterministic by default, with an optional bounded LLM query-generation step

**Purpose:** Find relevant documentation despite the vocabulary gap between issue text and docs.

**Internal workflow:**
1. **Build queries (in order of preference):**
   a. **Validated identifiers** from node 5 (`status=exists`), literal and high-confidence.
   b. **Vocabulary bridging:** harvest module, class, and function names from the top `code_retriever` hits (this is why docs runs after code).
   c. The issue title and body as a whole-text query for hybrid search.
   d. **Optional LLM query generation:** `LLMClient`, role `query_gen`, small tier; structured output validated by Pydantic (at most N queries, each tagged with its target index); temperature 0; cached by input hash through the record/replay layer. Its input is the issue text plus the **validated** identifiers and the vocabulary harvested in (b), so it works from real names. Its output is search queries only: no file paths, no symbols presented as fact. See the contract below.
2. **Filter** LLM queries deterministically: drop any query returning zero hits; cap the total.
3. **Search** with hybrid retrieval: BM25 (FTS5) plus embeddings if available, fused with RRF. Start with FTS5 alone and measure.
4. **Chunk-level results** with heading path as metadata (so the packer can show "Guide > Auth > Tokens").
5. Emit `ContextItem(kind="doc")` with the query that found each item.

**Sources:** documentation in the repo at `base_sha` (README, `docs/`, other Markdown), read from the managed clone through the deny-list. The GitHub wiki, if a repo uses one, is a separate git repository and would be an additional source later; it is out of scope for v1.

**Optional LLM step: contract**

```python
class SearchQuery(BaseModel):
    text: str
    target: Literal["code", "docs", "issues"]
    rationale: str

class QueryPlan(BaseModel):
    queries: list[SearchQuery] = Field(max_length=8)   # cap enforced by the schema
```

After the call: drop queries that return zero hits, cache by input hash, run at temperature 0. Enable this step by default only if the context-recall eval (Section 9) shows lexical queries built from validated identifiers are missing relevant docs. Because some JSON-schema modes do not show the model the schema, state the schema in the prompt as well ([Providers](../../explanation/providers.md), Section 2).

**Notes:** Do not use TF-IDF/RAKE/YAKE-style keyword extractors on issue text. Issues are short and noisy, so they perform poorly there.

---

### 12. `context_packer` (join)

**Type:** deterministic

**Purpose:** Turn everything retrieved into the `PreloadedContext` the LLM will see, within a token budget, with provenance. This is also the checkpoint where content is made safe to leave the machine.

**Internal workflow:**
1. Wait for all retrievers (LangGraph waits on a list-form edge).
2. Merge `retriever_results` into one item list, statuses, and warnings.
3. **Apply the path deny-list** again (defense in depth) and **redact secrets**: detected tokens, keys, and credentials are replaced with `[REDACTED:<kind>]`. Items that are secrets in their entirety are dropped. Redaction counts become a warning and a metric.
4. **Dedupe:** same path + overlapping line ranges merge; the same PR or issue from multiple retrievers keeps all provenance.
5. **Rank** by tier first (`explicit` > `symbol` > `lexical` > `neighbor` > `semantic`), then score.
6. **Budget:** fill up to the token budget, reserving fixed shares (e.g., issue text, conventions, code, docs, PRs). Truncate at symbol/section boundaries, never mid-snippet.
7. **Build a manifest** of items that exist but were not included (id, kind, one-line summary, token cost), so the planner can request them through tools or `context_gaps`.
8. Attach to every included item: `retriever`, `query`, `tier`, `score`, `tokens`.
9. Include retriever statuses and warnings in the packed context (e.g., "docs_retriever: empty").

**Output:** `PreloadedContext` (items + manifest + statuses + warnings), stored in the blob store so the exact context is reproducible. State holds only its hash.

**Note:** The packer is where "the LLM sees only a slice" is enforced, and where redaction runs. Tune budgets with evals (Section 9). Tool results returned to the planner pass through the same redaction.

---

### 13. `sufficient?` (decision)

**Type:** deterministic checks first; optional LLM verdict

**Purpose:** Catch issues that can't be planned well before spending a planner call.

**Deterministic checks (any failure marks the issue "insufficient"):**
- Body shorter than a threshold or empty
- Bug issue (by label/template) with no reproduction steps or expected behavior
- Template sections left unfilled
- Zero resolved references **and** zero code hits above tier 3

**Optional LLM verdict:** only if deterministic checks are ambiguous. Structured output `{sufficient: bool, missing: [..], questions: [..]}`. A small model is fine (`LLMClient`, role `verdict`, small tier; off by default until evals justify it). **This node also builds the clarifying questions** that node 14 presents, so node 14 can be interrupt-only.

**Branches:** Yes → node 15. No → node 14.

---

### 14. `ask_human` (HITL)

**Type:** human-in-the-loop (LangGraph `interrupt`)

**Purpose:** Get clarifying answers cheaply, before planning.

**Interrupt-first rule:** this node does nothing before `interrupt()`, because a resumed node re-runs from its first line ([primer, Section 5](../../explanation/langgraph-primer.md)). **Node 13 builds the questions**; this node only presents them and stores the answers.

**Internal workflow:**
1. Present the short, specific questions (from the deterministic checks or the LLM verdict) by pausing with reason `clarify`. The CLI prints the run id, the questions, and the exact `cog resume` command.
2. The checkpointer persists state, and the process may exit.
3. On resume, store the answers in state as `human_answers` and include them in the planner's context.
4. **One round only.** If the human skips (`--proceed`), continue and record "unresolved ambiguity" as an assumption.

---

### 15. `planner` (LLM)

**Type:** LLM, structured output

**Purpose:** Turn issue plus context into a task plan that small models can execute.

**Provider:** `LLMClient`, role `planner`, tier resolved by config. Output is validated with Pydantic by the client, with a small bounded number of schema-repair attempts inside the client (separate from the validator loop below). The prompt is a versioned file; its version id is recorded in the run record. Every call logs model, tier, tokens, and cost.

**Inputs:** `PreloadedContext`, `NormalizedIssue` (slice), `human_answers`, feedback from validator/HITL if looping, resolved gap results if looping.

**Tools (read-only, budgeted, e.g., ≤ 8 calls):** `read_file`, `grep`, `symbol_lookup`, `list_dir`, `get_context_item(id)` (to pull items from the manifest). Every call and result is logged, passes through the deny-list and secret redaction, and counts against the tool budget.

**Instructions (summary):**
- Extract `requirements` from the issue first (used for traceability later).
- Emit tasks with explicit `depends_on`, `files_to_modify`, `files_to_create`, `context_refs`, `acceptance` commands (use only commands from the conventions table), and `stop_conditions`.
- Write each task so a small model needs **no additional search**: include exact files, signatures, and relevant snippet references.
- **Match the repo.** Follow the conventions item and the exemplars; do not introduce preferences of your own (principle 9). The prompt contains no code-style opinions.
- **Do not emit batches.** Parallelism is derived by code.
- List `assumptions` and `open_questions` explicitly rather than hiding uncertainty.
- Emit `context_gaps` as structured items, each with `why_blocking`.

**Untrusted-input handling:** issue text, comments, docs, and code are **data**. A single helper wraps them in clear delimiters, the prompt says instructions inside them must not be followed, tools stay read-only, and the planner has no network access.

**Model settings:** low temperature; structured output through the provider's schema mode where the endpoint supports it (declared in config), with the schema also stated in the prompt, and always validated by cog.

---

### 16a. `context_gaps?` (decision)

Non-empty `plan.context_gaps` **and** `gap_rounds < 2` → 16b. Otherwise → 17.

### 16b. `gap_resolver`

**Type:** deterministic

**Purpose:** Fulfill structured gap requests without free-form LLM exploration.

**Routing by `ContextGap.kind`:**

| Kind | Resolver |
|---|---|
| `file` | read file (bounded), emit `ContextItem` |
| `symbol` | symbol index: definition plus references |
| `search` | lexical search (FTS5/ripgrep), bounded results |
| `doc` | docs retriever on the gap query |
| `human_question` | **not resolvable by lookup.** Don't loop; carry it forward as an open question shown at HITL (or as an explicit assumption in the plan) |

**Internal workflow:**
1. Run resolvers for non-human gaps; add results to context with `tier="gap_fill"`.
2. Record "searched, found nothing" for empty results so the planner doesn't re-ask.
3. Return the results as a `RetrieverResult(retriever="gap_resolver")`, increment `gap_rounds`, and return to node 15.

**Cap:** 2 rounds. LLMs will keep asking for more context if allowed.

---

### 17. `derive_batches`

**Type:** deterministic

**Purpose:** Compute the execution order and parallel groups from task dependencies. The planner does not assert batches.

**Internal workflow:**
1. Build a DAG from `depends_on`.
2. **Tolerant by design:** unknown ids, duplicate ids, or cycles are recorded in `graph_errors`; the node does not raise (the validator reports them and triggers repair).
3. Layer tasks by longest-path depth (Kahn's algorithm with levels); tasks at the same level form a batch.
4. **Write-set check:** within a batch, `files_to_modify ∪ files_to_create` must be disjoint. On overlap, move one task to the next batch (an implicit dependency) and note it. Common hot spots: lockfiles, barrel/index files, generated files, shared config.
5. Store `batches` and `graph_errors`.

---

### 18. `plan_validator`

**Type:** deterministic

**Purpose:** Reject bad plans mechanically, with precise, actionable errors, before a human spends time on them.

**Checks:**
1. Schema conformance (Pydantic)
2. Task ids unique; every `depends_on` id exists; no cycles (`graph_errors` empty)
3. Every `files_to_modify` path exists at `base_sha`; every `files_to_create` has an existing parent directory and doesn't already exist
4. Every referenced symbol resolves in the symbol index
5. Every `acceptance.cmd` is resolvable via the conventions command table (or an explicit allow-list); commands from repo-supplied config you have not yet approved are flagged
6. Every task cites at least one `context_refs` id that exists in the context
7. Every `requirement` is covered by at least one task (simple id/keyword mapping, with the LLM review optionally checking semantic coverage)
8. Write-sets within batches are disjoint (after node 17's adjustments)
9. Deny-list: no task touches protected paths (CI workflows, secrets, lockfiles unless explicitly allowed), using the shared policy module with paths POSIX-normalized and compared case-folded
10. Size limits: maximum tasks and per-task description size (keeps executor prompts small)
11. **Convention check (advisory, warning only):** if the conventions table defines lint or format commands, tasks that modify source files should include them in `acceptance`, or the plan should say why not. The repo's own tooling, not the harness's taste, is the style gate (principle 9).

**Output:** `ValidationReport {passed: bool, errors: [{check, task_id, message}]}`.

**Branches:** passed → node 19; failed → back to node 15 with the error list, **up to 3 retries**. After that, go to HITL with the validation failures displayed (don't loop forever).

---

### 19. `llm_plan_review` (optional)

**Type:** hybrid: runs after all deterministic checks pass

**Purpose:** Catch what code can't: does the plan actually address the issue? (This is the "rubber duck" step, placed last on purpose.)

**Internal workflow:**
1. Use a **different model** than the planner where possible (`LLMClient`, role `reviewer`; config warns when reviewer and planner resolve to the same model).
2. Input: issue requirements, plan, validation report. Not the full context.
3. Structured output: `[{severity: info|warn|critical, task_id?, finding}]` covering requirement→task coverage, risky changes, missing tests, ordering concerns.
4. Findings are **advisory** and attached to the HITL view. Only `critical` findings send the plan back to node 15, once.

**Rationale:** LLM review is the least reliable validator, so it runs after deterministic checks and only for semantic judgment.

---

### 20. `render_plan`

**Type:** deterministic

**Purpose:** Produce the human-readable view. Pure function of the Plan JSON plus context metadata.

**Content:** summary; requirements→tasks traceability table; batches with tasks and acceptance commands; assumptions and open questions (including carried-over `human_question` gaps); validation report; LLM review findings; warnings (e.g., "docs_retriever empty", "linked PR already merged"); a provenance view showing which context items each task relied on and their tiers (so a reviewer can see what was explicit vs. guessed).

**Implementation:** Jinja2 template → HTML (optionally also Markdown). The HTML is never parsed back and is stored as a derived file beside the plan JSON. **Hardening:** the page renders issue-derived and LLM-derived text, so the template auto-escapes everything, never injects raw HTML, and carries a restrictive content-security policy.

---

### 21. `hitl_review`

**Type:** human-in-the-loop (LangGraph `interrupt`)

**Interrupt-first rule:** all preparation (validation, rendering) is finished in nodes 18–20; this node contains only the interrupt and reads the decision. Applying the decision happens in the routing that follows and in node 22.

**Interrupt payload (pause reason `review`):** `plan_id`, content hash, rendered artifact path, summary of warnings and findings. The CLI prints a short summary and the HTML path, then exits.

**Resume payload:** `{decision: "approve" | "edit" | "reject", edited_plan?: Plan, feedback?: str}`, supplied with `cog resume <run_id> --decision ...`. Responses are validated before the run continues.

| Decision | Behavior |
|---|---|
| **Approve** | → node 22 |
| **Edit** | The human edits the plan **JSON** (`--decision edit --file edited.json`). The edit becomes a new immutable plan version with the previous hash as its parent. → node 17 (re-derive batches, since dependencies may have changed) → node 18 (re-validate; hand edits are never trusted blindly) → back here |
| **Reject + feedback** | Feedback is appended to planner input → node 15. Unbounded, since it is human-driven |

The review HTML is only a view ([ADR-0007](../../decisions/0007-plans-as-immutable-json.md)).

---

### 22. `persist_plan`

**Type:** deterministic; terminal artifact

**Purpose:** Produce the handoff artifact for the Implementation workflow. Writes only into cog's **data directory**; nothing is written into the target repo.

**Internal workflow:**
1. Write `plans/<owner>__<name>/<issue>/<plan_id>/v<N>/plan.json`: the plan with `schema_version`, `base_sha`, its SHA-256, and a pointer to the stored context snapshot (blob hash) and resolved config, so execution and audits are reproducible.
2. Write `approval.json` beside it: the approved plan hash, approver, timestamp, and any edits.
3. Write the derived `plan.html` beside the JSON for reviewers (disposable; regenerable).
4. Insert or update the plan metadata and run record in `cog.db`.
5. The step is idempotent by plan hash: re-running it with the same plan changes nothing.

**Export:** `cog plan export <plan_id>` copies the plan into a target repo (`.cog/plans/...`), optionally as a commit or PR. It is the only path from the data directory into a repo, and it runs only on request.

**Staleness check (for the Implementation workflow):** compare `base_sha` with the current default branch head; if planned files changed since, warn or re-validate.

---

### END → Implementation

The Implementation workflow takes the approved plan, executes batches sequentially and tasks within a batch in parallel (each task in its own git worktree), gates each batch on its acceptance commands, and escalates to a stronger model on repeated failure. That workflow is out of scope here; a scoped outline is in [the Implementation outline](../implementation/outline.md).

---

## 5. Loops and limits

| Loop | Limit | On exhaustion |
|---|---|---|
| Clarify (14) | 1 round | Proceed; record assumption |
| Gap resolution (16) | 2 rounds | Proceed; unresolved gaps shown at HITL |
| Validator repair (18 → 15) | 3 retries | Escalate to HITL with errors displayed |
| LLM review (19) | 1 pass; blocks only on critical | Findings shown at HITL |
| HITL reject (21 → 15) | Unbounded (human-driven) | n/a |
| Schema repair inside `LLMClient` (15, 11, 13, 19) | A small fixed number of attempts per call | Return a status; the node handles it (for node 15, counts toward the validator loop) |
| Budget breach (any node) | Per-run and per-node token and dollar caps | Pause with reason `budget`; the human raises the cap, continues, or aborts |

---

## 6. Storage

Everything lives in cog's data directory; target repos are untouched. Full layout: [Configuration and data](../../explanation/configuration-and-data.md). Decisions: [ADR-0006](../../decisions/0006-local-first-storage-no-logging-service.md), [ADR-0007](../../decisions/0007-plans-as-immutable-json.md), [ADR-0008](../../decisions/0008-standalone-command-center-managed-clones.md).

| Data | Where | Why |
|---|---|---|
| Approved plans, approvals | Files: `plans/<repo>/<issue>/<plan_id>/vN/` (`plan.json`, `approval.json`) | **System of record.** Immutable versions with parent hashes; diffable; survive database loss |
| Rendered HTML | Beside the plan JSON (`plan.html`) | Derived view; disposable |
| Plan metadata for queries | `cog.db` (SQLite, WAL) | Index only; rebuildable by rescanning `plans/` |
| Raw fetched issue data, context snapshots, prompts and responses | `blobs/` (content-addressed files) | Replay and audit |
| Issue and PR mirror | `mirrors/<repo>/` (SQLite + FTS5) | `related_issues`, offline replay, avoids the search API |
| Code index (FTS5, symbols, import graph, co-change, optional embeddings) | `indexes/<repo>/<base_sha>/` | Rebuildable cache keyed by commit |
| Run history, events, metrics | `cog.db`, `runs/<run_id>/events.jsonl` | Queryable; cost is `NULL` when unknown |
| Graph checkpoints | `checkpoints.db` via the LangGraph SQLite checkpointer **(verify package and class names)** | Interrupt/resume and replay |
| Target repo | Nothing, unless `cog plan export` is run | No footprint by default |

Rules:
- Never make a database the only record of an approval.
- Enable WAL mode; keep write transactions short; guard clone fetches and index builds with per-repo locks.
- State holds ids and hashes for large payloads, not the payloads.
- Never put the data directory in a cloud-synced folder.

---

## 7. LangGraph implementation notes

General LangGraph concepts are in the [primer](../../explanation/langgraph-primer.md); this section covers what is specific to Planning. Code is illustrative, not project code.

### Conventions this workflow follows

- **Async nodes** ([ADR-0002](../../decisions/0002-async-first.md)) registered through the node decorator, which enforces the node contract from principle 3 ([ADR-0003](../../decisions/0003-langgraph-orchestration-and-state.md)).
- **Injected dependencies.** `build_planning_graph(deps, checkpointer)` receives the GitHub client, index, stores, provider router, and clock. Nodes never import adapters or SDKs.
- **Small state.** State holds ids and hashes; large payloads (raw issue, context snapshot) live in the blob store.
- **Statuses, not exceptions,** for expected conditions.
- **`thread_id = run_id`.**

### LLM roles and tiers

Nodes ask for a **role**; config resolves roles to tiers, and tiers to `(adapter, model)` pairs ([Providers](../../explanation/providers.md), Section 4). This keeps model choices out of the workflow.

| Role | Used by | Default tier | Notes |
|---|---|---|---|
| `planner` | Node 15 | large | Structured output; read-only tool budget |
| `query_gen` | Node 11 (optional) | small | Cached by input hash |
| `verdict` | Node 13 (optional) | small | Off by default until evals justify it |
| `reviewer` | Node 19 (optional) | medium, or an explicit different model | Config warns if it resolves to the same model as `planner` |

`executor_tier` in a task (small, medium, large) uses the same tier names, and the escalation ladder is an ordered list of tiers.

### Parallel fan-out and join

```python
import operator
from typing import Annotated, TypedDict

class State(TypedDict):
    issue_ref: str                                              # blob hash of raw issue
    issue: NormalizedIssue
    refs: list[Reference]
    retriever_results: Annotated[list[RetrieverResult], operator.add]   # reducer merges parallel writes
    warnings: Annotated[list[str], operator.add]

@node(id="code_retriever", kind="deterministic")
async def code_retriever(state: State, deps: Deps) -> dict:
    result = await deps.retrieval.code(state["refs"], state["issue"])   # returns RetrieverResult
    return {"retriever_results": [result],
            "warnings": result.warnings}                          # status is data, not an exception

g.add_edge("validate_references", "code_retriever")
g.add_edge("validate_references", "linked_prs")
g.add_edge("validate_references", "related_issues")
g.add_edge("validate_references", "repo_conventions")

g.add_edge("code_retriever", "related_prs")       # stage 2 depends on code hits
g.add_edge("code_retriever", "docs_retriever")

# list form waits for ALL listed nodes before running the join
g.add_edge(
    ["linked_prs", "related_issues", "repo_conventions", "related_prs", "docs_retriever"],
    "context_packer",
)
```

Code hits from `code_retriever` reach the packer through the same `retriever_results` key, so the packer never depends on a separate list. Use `Send` only when the number of branches is dynamic (for example, one lookup per extracted reference). Each retriever lives in its own module (or subgraph if it has its own multi-step flow) so it can be tested alone.

### Human-in-the-loop: prepare, review, apply

A resumed node re-runs from its first line, so pausing nodes contain **only** the interrupt:

```python
from langgraph.types import interrupt

@node(id="hitl_review", kind="hitl")
async def hitl_review(state: State, deps: Deps) -> dict:
    decision = interrupt({"plan_id": state["plan_id"], "html": state["html_path"],
                          "warnings": state["warnings"]})   # first and only action
    return {"hitl": decision}                                # resumed with Command(resume={...})
```

Nodes 14 and 21 follow this rule. The preparation happens in the nodes before them (13 builds questions; 18–20 validate and render), and the application happens after (routing, node 22). **(Verify against the current LangGraph docs; APIs have changed between versions.)**

### Conditional edges

Use `add_conditional_edges` with small pure routing functions (`route_sufficient`, `route_gaps`, `route_validation`, `route_hitl`). Keep routing functions free of side effects so they are unit-testable.

### Budgets and pauses

A per-run or per-node budget breach is not a silent stop: it raises a pause with reason `budget`, and the human chooses to raise the cap, continue, or abort. Pause reasons for this workflow are `clarify` (14), `review` (21), `validation_failed` (18 after retries), and `budget` (any node). See the [Interaction model](../../explanation/interaction-model.md), Section 4.

### Testing

- Unit-test each extractor, retriever, routing function, and the policy checks with fixture issues (golden inputs → expected `Reference`, `ContextItem`, and `RetrieverResult` values).
- Run whole-graph tests with `FakeLLM`, including deliberately invalid plans (to exercise the repair loop) and interrupts (pause and resume across a process restart).
- Test the checkpointer round trip: a state containing every model type survives save and load.
- A drift test compares the compiled graph's generated Mermaid, the registry in Section 4.0, and the draw.io node names.
- Details: [Testing and evaluation](../../explanation/testing-and-evaluation.md).

---

## 8. Brittleness watchlist

| Risk | Mitigation |
|---|---|
| Regex extraction on prose | Extract only structured tokens; validate against the index; keep statuses |
| Stale paths, line numbers, symbol names in issues | `validate_references` with rename detection; flag drift |
| One retriever failing breaks the run | Retrievers return statuses; the packer proceeds with partial context and warns |
| LLM invents files, symbols, or commands | `plan_validator` checks existence against the index and the conventions table |
| LLM asserts parallel batches that conflict | Batches derived in code; write-sets checked |
| Unbounded LLM loops | Every loop capped; escalate to HITL |
| Prompt injection via issue/comments/docs | Treat as data, delimited; read-only tools; no network; path deny-list; secret redaction on context and tool results ([Policy and sandbox](../../explanation/policy-and-sandbox.md)) |
| Parallel executor edits collide | Per-task git worktrees; deterministic merge; batch gates (Implementation workflow) |
| Embedding maintenance (chunking, staleness) | Start FTS5-only; add embeddings only if recall evals show a gap |
| Index staleness | Key all indexes by `base_sha`; rebuild or incrementally update |
| Rubber-duck LLM review treated as truth | Runs last, advisory, different model |
| Secrets in code or issues sent to a provider | Redaction in node 12 and on tool results; egress policy checked in `preflight`; keys never in logs |
| GitHub rate limits (code search 10/min, search 30/min, core and GraphQL 5,000/hour) | No search or code-search API; managed clones and local mirror; GraphQL batching; ETag caching on REST; per-run API accounting |
| Windows path, case, and line-ending differences | POSIX-normalized paths; case-insensitive fallback in node 5; reads through git blobs; Windows and Linux CI |
| Provider structured-output quirks (per model and per endpoint) | Capabilities declared in config and checked in `preflight`; schema also in the prompt; always validate and repair |
| Harness taste overriding a repo's style | Principle 9; no style opinions in prompts; exemplars; repo lint and format commands as the gate |

---

## 9. Evaluation and metrics

**Primary eval, context recall (no LLM required):**
Take historical issues that have a merged PR. Run nodes 1–12 as of the PR's base commit. Measure what fraction of the files (and, ideally, symbols) the real PR touched appeared in the packed context, overall and by tier. Use this to tune retrievers, tiers, and budgets, and to decide whether embeddings are worth adding.

**Secondary metrics:**
- Context precision (share of packed items the plan actually cites)
- Validator first-pass rate and repair iterations per plan
- Gap rounds per plan; how often `human_question` gaps occur
- Human edit rate and reject rate at HITL
- Tokens, cost, and wall-clock per plan, by node
- GitHub API calls per run, by bucket
- Secret redactions per run
- Convention adherence (optional): whether plans (and later, changes) match the repo's observed patterns
- Downstream (Implementation workflow): first-pass task success per executor, escalation rate, batch conflict rate

**Set up early:** a "golden set" of 20–50 historical issues, stored with their PR file lists, rerun on every retriever change. It runs as `cog eval run`, replaying recorded GitHub responses and, where LLM steps are involved, the record/replay layer, so evals are repeatable and do not re-spend tokens. Details: [Testing and evaluation](../../explanation/testing-and-evaluation.md).

---

## 10. Open decisions

| # | Decision | Status |
|---|---|---|
| 1 | Primary language(s) of target repos | **Resolved:** Java, TypeScript, Python (tree-sitter first, behind the `Index` interface) |
| 2 | Typical repo size | **Resolved for design:** varies, so design for large repos (cached, commit-keyed builds). Whether embeddings are worth their cost remains **data-driven** (below) |
| 3 | Local mirror of issues/PRs | **Resolved:** yes ([ADR-0012](../../decisions/0012-github-integration.md)) |
| 4 | Token budget split between issue, code, docs, PRs, and conventions | **Open:** tune with the recall eval |
| 5 | Whether node 13's LLM verdict is needed | **Open:** default off; add if deterministic checks prove insufficient |
| 6 | Model tiers for `executor_tier` and the escalation ladder | **Resolved:** a config concern ([ADR-0004](../../decisions/0004-two-provider-interfaces.md)); defaults in [Providers](../../explanation/providers.md) |
| 7 | Whether to add embeddings | **Open, data-driven:** only if the recall eval shows a gap lexical and structural search cannot close |
| 8 | Bounds for the first issue/PR mirror sync (time window, count) | **New, open:** decide when the mirror is built; check cost on a large repo |
| 9 | Bare issue number outside a known clone: error, or prompt to choose a repo | **New, open** (Interaction model, Section 8) |
| 10 | Wiki documentation as an additional `docs_retriever` source | **New, deferred:** docs live in the repos for now |
| 11 | Secret-redaction rules and false-positive handling | **Resolved in design:** [ADR-0016](../../decisions/0016-secret-redaction.md) and [Policy and sandbox](../../explanation/policy-and-sandbox.md), Section 7. Tuning against real repos remains open |

---

## 11. Decisions and rejected alternatives

Recorded so the reasoning survives. Each row says when to revisit.

| Alternative | Decision | Why | Revisit when |
|---|---|---|---|
| LLM-first extraction (an LLM emits `search_keywords` and `mentioned_files` as the first step) | Rejected | Paths and symbols produced by an LLM can look authoritative and be wrong (e.g., `src/auth.py` instead of `src/auth/token.py`), and nothing downstream checks them. It can also reformat stack frames and error strings, breaking exact-match search, and it puts a model call (on untrusted text) at the start of every run. Regex plus an existence check is high-precision and testable | Never for structured tokens. For *queries*, see the next row |
| LLM generates search queries | Optional, bounded (node 11) | Vocabulary choice is a judgment task. Constrained, tagged, capped, filtered by code | Recall eval shows validated-identifier queries miss relevant docs |
| TF-IDF / RAKE / YAKE keyword extraction on issue text | Rejected | Poor on short, noisy, code-heavy text | Unlikely |
| Vector database from day one | Rejected | FTS5 plus brute-force numpy handles tens of thousands of chunks; embeddings add chunking and staleness work | Recall eval shows a gap lexical search can't close, or corpus grows beyond brute-force range |
| LLM-asserted batches | Rejected | Code derives them from `depends_on`; write-sets are checked | Never |
| LLM review as the main validator | Rejected | Least reliable validator; runs last and is advisory (node 19) | Never as primary |
| Open-ended agentic loop for context gathering | Rejected for v1 | Costly and hard to reproduce. Replaced by a read-only tool budget plus structured `context_gaps` (nodes 15, 16) | Eval shows frequent unresolved gaps, or recall is poor on ambiguous issues |
| Purely deterministic preload with no planner tools or gaps | Rejected | Fails on vague issues; the planner can't ask for what the pipeline missed | n/a |

### Decisions recorded as ADRs

Platform and project-wide decisions live in the [ADR index](../../decisions/README.md) rather than here. The ones that shape this workflow:

| ADR | Effect on Planning |
|---|---|
| [0003](../../decisions/0003-langgraph-orchestration-and-state.md) | Node contract, typed state, injected dependencies, registry |
| [0004](../../decisions/0004-two-provider-interfaces.md) | Planning uses `LLMClient` only; roles and tiers |
| [0005](../../decisions/0005-direct-providers-first.md) | Fireworks and OpenRouter first; no agent CLI needed |
| [0007](../../decisions/0007-plans-as-immutable-json.md) | Plans are immutable files; HTML derived; no repo footprint |
| [0008](../../decisions/0008-standalone-command-center-managed-clones.md) | Managed clones; reads through git objects |
| [0012](../../decisions/0012-github-integration.md) | GraphQL, local mirror, API budget; no code-search API |
| [0014](../../decisions/0014-repo-convention-precedence.md) | Principle 9; conventions and exemplars |
| [0016](../../decisions/0016-secret-redaction.md) | Redaction in node 12, tool results, adapters, and logs |
| [0017](../../decisions/0017-index-design.md) | Commit-keyed index, parse cache, tree-sitter; used by nodes 5, 6, 9, 10, 11, 16b, 18 |

---

## Glossary

Terms used in this document are defined in the [shared glossary](../../reference/glossary.md). Terms specific to this workflow appear there under *Search and retrieval*, *LangGraph and orchestration*, and the other original sections.
