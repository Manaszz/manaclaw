# ADR: Workspace Terminology

**Status:** Accepted

**Date:** 2026-03-24

## Context

NanoClaw uses the term **“group”** for multiple concepts at once: a chat grouping, an isolation boundary, and the on-disk workspace folder. That overload harms documentation, APIs, and mental models—especially when comparing messaging “groups” to security and filesystem isolation.

## Decision

Rename **“group”** to **“workspace”** consistently across ManaClaw. A **workspace** is a registered conversation context that owns:

- An isolated folder on disk
- Container and runtime association
- Session, tasks, and memory scoped to that unit

The **main workspace** denotes a **privileged administrative context** (system or operator scope), not an arbitrary user chat.

## Consequences

**Positive**

- Clearer documentation and API language aligned with isolation and filesystem layout.
- Reduced confusion between messaging “group” semantics and ManaClaw’s unit of work.

**Negative**

- No longer a one-to-one naming map to NanoClaw when cross-referencing code, issues, or migrations.

**Neutral**

- Migration guides and changelogs should call out the terminology change explicitly for upgraders from NanoClaw-aligned forks or docs.
