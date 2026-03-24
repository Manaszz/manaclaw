# ADR: Layered Pluggable Memory

**Status:** Accepted

**Date:** 2026-03-24

## Context

NanoClaw relies on `CLAUDE.md`-style files without search or embeddings. Corporate deployments need stronger recall, retrieval, and optional integration with dedicated memory products—without mandating heavy infrastructure for every install.

## Decision

Adopt a **layered, pluggable memory** model with three core modes:

| Mode | Behavior |
|------|----------|
| **Off** | File-only context (closest to NanoClaw-style usage). |
| **Lite** | BM25 / full-text search (FTS5) backed by SQLite for lightweight retrieval. |
| **Full** | Embeddings-based semantic retrieval. |

Support **external adapters** for enterprise systems such as **Mem0** and **Graphiti**. The graduated design lets small teams stay minimal while large orgs opt into richer memory stacks.

## Consequences

**Positive**

- Zero extra infrastructure for Off and Lite modes in typical SQLite deployments.
- Semantic search in Full mode when embeddings are enabled and configured.
- Optional Mem0/Graphiti paths for organizations that standardize on those platforms.

**Negative**

- Full mode adds an extraction/indexing pipeline that increases latency and operational surface.
- Full mode requires an embedding-capable provider or service; it is not free by default.

**Neutral**

- Operators choose the tier that matches compliance, cost, and recall requirements per environment or workspace.
