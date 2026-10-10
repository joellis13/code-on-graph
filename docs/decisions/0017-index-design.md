# ADR-0017: Index design: commit-keyed SQLite, tree-sitter, git objects, parse cache

- **Status:** Accepted (packaging choice open)
- **Date:** 2026-10-08

## Context

The planner sees only a slice of the repo, so retrieval quality caps plan quality. Repos are Java, TypeScript, and Python, of varying size, on Windows, Linux, and macOS, and cog never touches a user's working copy. Full detail is in [Indexing](../explanation/indexing.md).

## Decision

- **One index per `(repo, base_sha)`**, stored as SQLite in the data directory and published **atomically** (build to a temporary file, then rename).
- **Reads come from git objects** (`git ls-tree`, `git cat-file --batch`), never a working tree.
- **Components:** file inventory, FTS5 lexical index with identifier splitting, symbol index, **name-based** reference table, import graph, test map, co-change table, repo map. Embeddings are deferred until the recall eval shows a gap.
- **Parser:** tree-sitter grammars for Java, TypeScript, and Python behind an `Index` interface, so a language server can be added later.
- **Parse cache keyed by git blob id plus grammar version.** Builds always start from the tree, yet only changed blobs are re-parsed.
- **Policy at build time:** denied, binary, generated, vendored, and oversized files are skipped, with reasons recorded. Secret-bearing files never enter the index.
- **References are candidates with a confidence level, not proofs.**
- **Packaging is decided by a Phase 3 spike:** `tree-sitter-language-pack` versus the core `tree-sitter` package with individually pinned grammars. The winner is recorded as an amendment to this ADR. Grammar versions are pinned and part of the cache key; on-demand parser downloads require review.

## Alternatives considered

- **Language servers first:** more accurate references, but heavy per-language setup and poor fit for historical commits.
- **Embeddings first:** extra cost and egress concerns before evidence they are needed.
- **Ripgrep over a working tree:** conflicts with the managed-clone, no-working-tree model; kept as a fallback only if measurements demand it.
- **Incremental updates in v1:** the parse cache gives most of the benefit with less complexity.

## Consequences

- Reference accuracy is limited; the planner and validator treat references as hints.
- Cache management belongs to `cog gc`.

## Revisit when

The recall eval shows structural gaps, grammar packaging proves unreliable on Windows, or index builds are too slow on large repos.
