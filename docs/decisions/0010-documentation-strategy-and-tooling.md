# ADR-0010: Documentation strategy: Diátaxis, generated reference, tool spike

- **Status:** Accepted
- **Date:** 2026-10-06

## Context

Documentation must cover technical and product/usage topics, be mostly automated, and not rot. The project will also be built partly by coding agents that use the docs as a specification.

## Decision

- Organize under **Diátaxis**: tutorials, how-to guides, reference, and explanation. Diátaxis is a way of sorting documents, not a storage system. Files live in the repo under `docs/`.
- **Generate** whatever can be derived: API reference from docstrings, config reference from the settings model, CLI reference from the Typer app, JSON Schemas for handoff contracts (committed, with a version-bump check), node reference tables from the node registry, and graph diagrams from compiled graphs.
- A **generation script of our own** emits plain Markdown, so the site generator is swappable.
- **Site tool:** spike Zensical first. Material for MkDocs is in maintenance mode and its team now works on Zensical, which is still pre-1.0 and has preliminary mkdocstrings support. Fall back to Material + mkdocstrings (pinned) or Sphinx + MyST. **(verify)** at spike time.
- Hand-drawn diagrams use **draw.io** (one file per diagram). Small teaching diagrams use **Mermaid** inline.
- Rot prevention in CI: strict docs build, link checking, executed examples, CLI help snapshots.
- **ADRs** record decisions. Open items carry **(verify)** markers tracked in the roadmap's verification log.

## Alternatives considered

- **Docusaurus:** capable but adds a Node toolchain and has no native Python API docs.
- **Hand-written reference:** drifts.

## Consequences

- Generated pages are not committed except contract schemas.

## Revisit when

The spike shows Zensical's API-reference story is inadequate.
