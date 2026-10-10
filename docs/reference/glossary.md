# Glossary

**Type:** Reference · **Status:** Draft v1 · **Last updated:** 2026-10-08

The shared glossary for all Code-OG documents. The first group of sections was moved here from the Planning workflow reference (v2.1) without changes of meaning; the sections starting at **Platform and runtime** are new. Where a term is tied to a Planning node, the node is noted in the Planning reference.

---

## Search and retrieval

**Lexical search / keyword search.** Matching documents by the literal words (tokens) they contain. Fast, exact, and predictable; weak when the query uses different words than the document. Used in nodes 6, 8, 11.

**BM25.** A classic ranking formula for lexical search. It scores a document higher when it contains the query terms often (term frequency), but with diminishing returns, and when those terms are rare across the whole collection (inverse document frequency); it also normalizes for document length. Think of it as the well-tuned successor to TF-IDF. It is the default ranking in most search engines, including SQLite FTS5.

**TF-IDF.** "Term frequency × inverse document frequency." An older, simpler cousin of BM25 that weights words by how often they appear in a document versus how common they are overall. Not recommended for extracting keywords from short, noisy text like issues.

**RAKE / YAKE.** Statistical algorithms that pull "keywords" out of a single piece of text without a corpus or model. They tend to perform poorly on short, noisy, code-heavy text, which is why this design avoids them.

**Embedding.** A list of numbers (a vector) produced by a model that represents the _meaning_ of a piece of text, so that texts with similar meaning have vectors that are close together. Enables "semantic search": finding relevant text even when it uses different words. Costs: an embedding model, chunking decisions, storage, and keeping vectors fresh when files change.

**Semantic search.** Retrieval based on embedding similarity rather than literal word overlap.

**Cosine similarity.** A measure of how closely two vectors point in the same direction (1 = same direction, 0 = unrelated). The standard way to compare embeddings.

**Brute-force vector search.** Compute cosine similarity between the query vector and _every_ stored vector (typically with numpy), then take the top results. Perfectly fine up to tens of thousands of chunks; no vector database required.

**Vector database.** A server or library that stores embeddings and finds nearest neighbors quickly using specialized indexes (approximate nearest neighbor search). Needed at larger scale; **not** needed to combine BM25 with embeddings at this project's likely size. Examples: `sqlite-vec` (SQLite extension), LanceDB (embedded), or hosted services.

**Hybrid search.** Running lexical search (BM25) and semantic search (embeddings) together and merging their results. Lexical catches exact identifiers and error strings; semantic catches paraphrases.

**RRF (Reciprocal Rank Fusion).** A simple way to merge several ranked lists. Each item's score is the sum of `1 / (k + rank)` over each list it appears in (commonly `k = 60`). It needs almost no tuning and doesn't require the lists' scores to be comparable.

**FTS5.** SQLite's built-in full-text search module. You create a virtual table, insert text, and query with `MATCH`; the `bm25()` function ranks results (in SQLite, lower values rank better). Supports prefix queries, phrase queries, and several tokenizers (including `trigram`, useful for substring matching of identifiers). Runs inside a single database file with no server.

**Tokenizer / tokenization.** How text is split into searchable units. Default tokenizers split on whitespace and punctuation, so `parseHttpHeader` and `parse_http_header` each become a single token and won't match a search for "http header."

**Identifier splitting.** Preprocessing code so identifiers are also indexed as their component words (`parseHttpHeader` → `parse http header`; `snake_case` → `snake case`), usually in an extra column alongside the original. Without it, lexical search over code performs poorly.

**ripgrep (`rg`).** A very fast command-line text search tool that respects `.gitignore`. Good for exact matches over a working tree with no index.

**Chunking.** Splitting files or documents into pieces (by function/class for code, by heading/section for docs) before indexing, so search returns focused snippets rather than whole files.

**Vocabulary mismatch / vocabulary bridging.** The problem that issue authors and codebases describe the same thing in different words. _Bridging_ means using words from one side (e.g., function names found in code) to query the other (docs). Related to _pseudo-relevance feedback_: using top results from a first search to build a better second query.

**Context recall.** The fraction of the "right" context that your retrieval actually surfaced, e.g., the share of files in the real merged PR that appeared in your packed context. The central metric for tuning retrievers (Section 9).

**Context precision.** The fraction of retrieved context that was actually useful.

**Query generation.** Using a (small) LLM to turn an issue into a short list of search queries. A legitimate LLM job (it requires judgment about vocabulary), but it should be constrained and its output validated.

## Code understanding

**Symbol.** A named code element: function, class, method, variable, constant.

**Symbol index.** A lookup table of where each symbol is defined and referenced. Lets you answer "where is `parse_header` defined, and who calls it?" without text search.

**AST (Abstract Syntax Tree).** A tree representation of code structure. Python's built-in `ast` module produces one; it is enough for Python-only repos.

**tree-sitter.** A fast parsing library with grammars for many languages that produces syntax trees. Used to extract symbols and signatures across languages without a full compiler.

**ctags.** A long-standing tool that scans source files and emits a simple index of symbol definitions. Less precise than tree-sitter or an LSP, but easy to set up.

**LSP (Language Server Protocol).** A standard protocol by which editors ask a language-specific server for definitions, references, and diagnostics. Precise, but requires running and configuring a server per language.

**Repo map.** A compact, ranked outline of a repository (important files and the signatures inside them), giving an LLM orientation without reading everything. Popularized by Aider, which builds it with tree-sitter and graph ranking.

**Import graph / dependency graph.** Which files import which. Used to find structural neighbors of a file.

**Co-change history.** Files that tend to be modified together in the same commits. Computed from `git log --name-only`; a cheap, deterministic, and often strong predictor of related files.

**CODEOWNERS.** A file mapping paths to responsible people or teams. Useful as a hint about which area a file belongs to.

**AGENTS.md.** A convention for a repo-level file that gives AI coding tools project instructions (build/test commands, conventions). Other tools use similar files with different names.

**Stack frame / stack trace.** The list of function calls (with file and line) active when an error occurred. A high-value, highly structured piece of evidence in bug reports.

## Git and GitHub

**`base_sha`.** The commit SHA of the repository state the plan was built against. All indexes and validations are keyed to it, so results are reproducible.

**Squash merge.** A merge style that collapses a PR into a single commit on `main`. GitHub's default subject is `PR title (#123)`, so the PR number can be parsed from the commit message.

**Merge commit.** A merge style that keeps the PR's commits and adds a commit titled `Merge pull request #123 from ...`.

**Timeline events (GitHub).** A stream of events on an issue (cross-references, label changes, linked PRs, etc.). The most reliable source for "which PRs reference this issue."

**Git worktree.** An additional working directory attached to the same repository, on its own branch. Lets parallel tasks edit files in isolation, then merge deterministically.

**`git log --follow`.** Shows history for a file across renames.

## LangGraph and orchestration

**Workflow vs. agent.** A _workflow_ follows a path defined in code, with LLM calls at specific nodes. An _agent_ lets the LLM decide the next step in a loop. This design is a workflow with a small, bounded amount of agent-like behavior (the planner's read-only tools).

**Agentic search.** Letting an LLM drive repeated search and file-reading calls (grep, open file, follow references) to find relevant code, instead of a fixed retrieval pipeline. More flexible on vague issues; costlier and harder to reproduce. The main alternative to this design's deterministic preload.

**Graph / node / edge.** In LangGraph, a workflow is a graph. A _node_ is a function that reads and updates state; an _edge_ connects nodes; a _conditional edge_ chooses the next node via a routing function.

**State.** The shared data structure passed through the graph (here, Pydantic models). Nodes return partial updates to it.

**Reducer.** A function that tells LangGraph how to merge multiple writes to the same state key (e.g., `operator.add` to concatenate lists). Required when parallel branches write to the same key.

**Fan-out / fan-in (join).** Fan-out runs several nodes in parallel from one node; fan-in waits for them and combines results in a downstream node.

**`Send`.** A LangGraph mechanism for launching a dynamic number of parallel node invocations (e.g., one per reference). Use static edges when the branches are fixed.

**Subgraph.** A graph used as a node inside another graph. Useful to package each retriever independently.

**Checkpointer.** Component that saves graph state after each step (e.g., to SQLite), enabling pause/resume, replay, and recovery.

**`thread_id`.** The identifier that groups checkpoints for one run; here, the `run_id`.

**`interrupt()`.** LangGraph's way to pause a run to wait for a human; execution resumes later with the human's input (via `Command(resume=...)`).

**HITL (Human-in-the-loop).** A point where a human reviews or decides before the workflow continues.

**Structured output.** Constraining an LLM to return JSON matching a schema, validated by code (here, Pydantic), rather than free text.

**Pydantic.** A Python library for defining data models with validation. Used here for every schema.

**Idempotent.** Safe to run repeatedly with the same result. Fetch and retrieval nodes should be idempotent so caching and retries are safe.

**Cache key.** The combination of inputs that determines whether a stored result is still valid (e.g., `repo + issue + updated_at`).

**Provenance.** A record of where each piece of context came from (which retriever, which query, which source), so reviewers and evals can trace it.

**Confidence tier.** A label indicating how reliable a context item's selection was: explicitly named in the issue (highest) down to semantic guess (lowest).

**Token budget.** The maximum number of tokens allotted to the context sent to the LLM; the context packer enforces it.

**Manifest.** A short list of items that exist but weren't included in the packed context, letting the planner request them by id.

**Context gap.** A structured request from the planner for information it needs but doesn't have.

**Rubber ducking.** Explaining a problem (here, having a model critique output) to catch mistakes. In this design it is the final, advisory validation step.

**Escalation ladder.** A policy of retrying a failed task on progressively stronger models (or involving a human) rather than improvising.

## Planning and execution

**Task packet.** A task described completely enough (files, signatures, context references, acceptance checks, stop conditions) that a small model can execute it without further searching.

**`depends_on`.** The list of task ids that must finish before a task can start.

**DAG (Directed Acyclic Graph).** A graph of dependencies with no cycles. Valid task plans form a DAG.

**Topological sort / Kahn's algorithm.** Ordering nodes of a DAG so every task comes after its dependencies. Computing the "level" of each task (its longest dependency chain) yields batches of tasks that can run in parallel.

**Write-set.** The set of files a task may modify or create. Tasks in the same batch must have disjoint write-sets so parallel edits can't collide.

**Acceptance criteria / acceptance check.** A runnable check (test, linter, type-check command) that proves a task is done. Gates each batch.

**Stop conditions.** Explicit situations in which an executor must stop and report rather than improvise.

**Deny-list.** Paths or actions a plan may not touch without explicit approval (CI workflows, secrets, lockfiles).

**Traceability.** Mapping each issue requirement to the tasks that satisfy it, and each task to the context it relied on.

## Data and storage

**SQLite.** A serverless SQL database stored in a single file. Good for caches, indexes, run history, and LangGraph checkpoints.

**WAL (Write-Ahead Logging).** A SQLite journaling mode (`PRAGMA journal_mode=WAL;`). Writes go to a separate log file first, so readers don't block the writer and the writer doesn't block readers. Still only one writer at a time (set `busy_timeout` so concurrent writers wait instead of erroring). Creates `-wal` and `-shm` side files next to the database.

**Rebuildable cache.** Data that can be regenerated from other sources (e.g., a search index built from the repo). Safe to delete and not worth committing.

**Golden set.** A fixed collection of historical cases with known-good outcomes (e.g., issues paired with their merged PRs' file lists), rerun after every change to detect regressions.

**Content hash (SHA-256).** A fingerprint of data's contents. Used to version plans and detect changes.

## Security

**Prompt injection.** Text in processed data (an issue, comment, doc, or code comment) that tries to give the LLM instructions. Mitigation here: treat all such content as data, give the planner only read-only tools and no network access, and validate outputs against deny-lists.

**Untrusted input.** Anything not authored by you or your tooling, including GitHub issue text, which anyone with access to the repository may write.

---

## Platform and runtime

**Code-OG / `cog`.** The project (`code-on-graph`) and its command. A standalone command center for AI-assisted software work.

**Command center.** The standalone-application model: installed once, operates on many repos, keeps state in its own data directory, leaves no footprint in target repos unless asked.

**Data directory.** Where cog keeps all its state: checkpoints, event store, plans, indexes, mirrors, blobs, and managed clones. Located with `platformdirs`; overridable with `COG_HOME`.

**Managed clone.** A clone of a target repo kept by cog in its data directory, under a short hash-based name. Planning reads from it; Implementation worktrees are created from it. Your own working copies are never touched.

**Run.** One execution of one workflow on one input. Has a `run_id`, which is also the LangGraph `thread_id`.

**Superstep.** One round of execution in LangGraph: every ready node runs (in parallel), then their updates are merged and a checkpoint is saved.

**Walking skeleton.** The first build milestone: every node present as a stub, wired end to end with real checkpointing and interrupts and a fake LLM, so the whole flow runs before any node is made real.

**Async-first.** Nodes, services, and adapters are written as `async` functions (ADR-0002).

**WSL2.** Windows Subsystem for Linux 2: a Linux environment on Windows. The expected execution environment for Implementation (ADR-0011).

**Platform tiers.** Windows (primary), Linux (CI and WSL2), macOS (best-effort).

**Event store.** The SQLite record of what every node, LLM call, tool call, and GitHub call did during runs.

**Blob store.** Content-addressed files on disk holding large payloads (prompts, responses, context packs) referenced by hash from events.

**Content-addressed storage.** Storing a file under the hash of its contents, so identical content is stored once and references can be verified.

**WAL (write-ahead logging).** A SQLite mode that lets readers and a writer work concurrently; used for the event store.

**Layered configuration.** Settings resolved from several sources in a fixed precedence (defaults, user, repo, local override, environment, flags).

**Trust on first use.** Showing a repo-supplied setting (such as a command) once, recording your approval, and asking again only if it changes.

**Seam.** An interface with a simple local implementation now, placed where a scaled implementation could be swapped in later (ADR-0013).

## Providers and execution

**`LLMClient`.** The interface for completions with optional structured output and a bounded read-only tool loop. Used by Planning.

**`AgentRunner`.** The interface for executing a task packet in a working directory under a policy. Used by Implementation.

**`NativeRunner`.** An `AgentRunner` built on cog's own tool-calling loop, as opposed to a vendor CLI.

**Tier.** A named model class (small, medium, large) mapped in config to an adapter and model. Plans refer to tiers as `executor_tier`.

**Adapter.** Code that translates between cog's interface and one provider or product.

**Capabilities.** Flags an adapter declares (structured output, sessions, MCP, cost reporting, takeover, native shell) so callers can check before relying on them.

**OpenAI-compatible endpoint.** A server implementing the OpenAI Chat Completions API shape. Fireworks, OpenRouter, Ollama, and vLLM can all be reached this way.

**Quirks.** Per-provider differences (such as how structured output is requested) recorded in config rather than code.

**BYOK (bring your own key).** Using your own provider credentials with a vendor tool, such as Copilot CLI pointed at Fireworks.

**Record/replay.** Saving LLM requests and responses keyed by a hash, then replaying them in tests and evals without calling the provider.

**Live contract test.** An opt-in test that runs a real provider or CLI to confirm that its behavior (unattended runs, flags, output format) still matches what the adapter expects.

**Takeover.** Handing a task to a human in an interactive agent session when automated escalation is exhausted.

**ExecutionEnvironment.** The interface through which all commands run: `LocalProcess`, `Wsl`, or `Container`.

**Worktree.** A separate checked-out copy of a repo sharing its git history, used to isolate each task's changes.

**Subscription token.** The login credential for a vendor subscription. cog never extracts or reuses these to call vendor APIs.

## Policy and safety

**Policy.** Rules enforced in code about what an agent may read, change, or run.

**Path deny-list.** Patterns for files an agent may never read or modify (for example, `.env*`, `**/*.pem`, `.git/**`, `.github/workflows/**`). Checked inside the tool function, with paths normalized so `..`, symlinks, and Windows case differences cannot bypass it.

**Command allow-list.** The only commands an agent may run, taken from the repo's known test, lint, and build commands rather than arbitrary shell.

**Sandbox.** A physical boundary (worktree, WSL2, container) that limits damage if the rules fail.

**Redaction.** Removing secrets from content before it is sent to a provider or written to logs.

**Egress.** Data leaving your machine toward a provider or service. Governed by policy.

**Convention precedence.** The rule that explicit repo rules beat nearby observed patterns, which beat any harness default (ADR-0014).

**Exemplar.** A sibling file or similar code retrieved so the planner and executors can imitate the repo's existing patterns.

## GitHub

**GraphQL points.** GraphQL's rate limit is a cost-based budget (5,000 points per hour) rather than a request count; one query can cost many points.

**Primary and secondary rate limits.** Primary limits are hourly quotas per API family (REST, search, code search, GraphQL). Secondary limits throttle bursts and are signaled differently; waiting out the primary reset does not clear them.

**Conditional request.** A REST request carrying an ETag so that an unchanged resource returns a cheap 304 response. Not available for GraphQL.

**Issue/PR mirror.** A local, incrementally synced copy of issues and pull requests with full-text search, used instead of the rate-limited search API.

**`VCSHost`.** The interface hiding which version-control host and authentication method is in use.

## Documentation and process

**ADR (Architecture Decision Record).** A short file recording one decision: context, decision, alternatives, consequences, and when to revisit.

**Diátaxis.** A way of sorting documentation into four types: tutorials (learning by doing), how-to guides (completing a task), reference (looking things up), and explanation (understanding why). It sorts documents; it does not store them.

**Documentation-first.** The project rule that documentation rounds are completed and signed off before code is written (ADR-0015).

**`(verify)`.** A marker on any claim that depends on external behavior and must be checked against current sources during the build. Tracked in the roadmap's verification log.

## Planning workflow additions (v3)

**`RetrieverResult`.** The shared return shape of every retriever: a status (`ok`, `empty`, `skipped`, `error`), a list of context items, and diagnostics. Parallel results merge through one reducer key.

**Role (LLM role).** The name a workflow uses to request a model (`planner`, `query_gen`, `verdict`, `reviewer`). Config resolves a role to a tier and the tier to an adapter and model.

**Node registry.** The table of node metadata (id, kind, LLM role, idempotence, side effects, module, build status) from which reference tables and the diagram drift check are generated.

**Pause reason.** Why a run is waiting for a human: `clarify`, `review`, `validation_failed`, `budget`, and, in Implementation, `takeover`.

**Interrupt-first.** The rule that a node containing `interrupt()` does nothing with side effects before it, because a resumed node re-runs from its first line.

**Prepare / review / apply.** The pattern that splits a human pause into a node that prepares, a node that contains only the interrupt, and a step that applies the decision.

**Plan version.** One immutable `plan.json` (`v1`, `v2`, ...). An edit creates a new version whose parent is the previous plan's hash.

**Approval record.** `approval.json`: the approved plan hash, approver, timestamp, and edits. Stored beside the plan; never only in a database.

**Exemplar.** (Extended.) Selected by `code_retriever`'s neighbor tier and emitted with `kind="exemplar"`.

## Round 3 additions

**Parse cache.** Parsed and chunked output stored by git blob id plus grammar version, so a new commit re-parses only changed files.

**Identifier splitting.** Indexing each chunk twice, as written and with identifiers split at case changes, underscores, dots, and digits, so "invoice total" matches `calcInvoiceTotal`.

**Atomic publish.** Building an index into a temporary file and renaming it into place, so no partial index is ever visible.

**Name-based reference.** A usage found by matching an identifier name rather than resolving what it refers to. Treated as a candidate with a confidence level.

**Policy snapshot.** The immutable, resolved policy for one run, saved in the run record and read by every enforcement point.

**Last-line egress filter.** Redaction and egress checks applied to the final outbound payload inside every LLM adapter, so no code path bypasses them.

**Salted per-run hash.** The short suffix in a redaction token, letting repeats be recognized within a run without revealing or enabling guessing of the secret.

**Network profile.** A per-command setting (`none`, `package-registry-only`, `full`) for what network access a command may have.

**Sandbox tiers.** Tier 0 policy only (Planning); 1 separate worktree with policy and scrubbed environment; 2 WSL2; 3 container.

**Golden set.** Historical issues with merged closing PRs, stored with the files the PR touched, used for the context-recall eval.

**Context recall.** The fraction of files a real PR touched that appeared in the packed context.

**Ablation.** Re-running an eval with one component removed to measure its contribution.

**Leakage (eval).** Information from after the fix, such as later comments naming the changed files, that makes an eval trivially easy. Prevented by replaying the issue as it was before the fix.

**Live test.** A test marked `live` that uses real providers, CLIs, or GitHub; skipped by default and never run in CI.

**Executed docs.** Documentation examples and terminal transcripts produced by actually running commands, so they cannot drift from behavior.
