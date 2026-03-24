# ManaClaw Layered Memory

This document describes ManaClaw’s layered, pluggable memory system: how modes behave, how data is structured, and how configuration and optional adapters fit together.

---

## 1. Overview

ManaClaw uses a **layered, pluggable memory** model. Operators choose how much automation, storage, and retrieval cost they want by setting a single mode in `config.toml`:

```toml
[memory]
mode = "off"    # or "lite" | "full"
```

- **Off** — no automated memory pipeline; lowest overhead.
- **Lite** — keyword-oriented retrieval (BM25 over SQLite FTS5) with lightweight fact extraction after each agent turn.
- **Full** — semantic retrieval via embeddings, LLM-based fact extraction, and compaction-oriented maintenance.

Higher modes build on lower layers: raw message history and summaries are relevant across modes; embeddings and external adapters apply only where configured.

---

## 2. Memory Modes

### Off

- No automated memory processing (no scheduled extraction, no FTS5 or embedding writes for facts).
- Context is whatever the agent sees in the **workspace folder** and the current session (e.g. open files, `MEMORY.md`).
- **`MEMORY.md`** in the workspace is the primary **human-visible** artifact; operators or users maintain it manually if desired.
- Simplest operational profile: minimal database churn and no extra model calls for memory.

### Lite

- **Retrieval:** BM25-style **keyword search** over **SQLite FTS5** (or equivalent indexed text search for the configured database) against stored facts.
- **After each agent turn:** extract candidate **facts** using **simple heuristics** (regular expressions and light rules) for names, dates, preferences, action items, and similar patterns — **no LLM call** required for extraction.
- **Storage:** rows in `memory_facts` with an **FTS5** (or compatible) index for search.
- **On a new user/agent message:** run an FTS5 query, rank hits, and **inject relevant facts** into the system (or context) prompt.
- **Rolling summaries:** periodically (every **N** messages or on a schedule), call an LLM to produce **rolling summaries** per workspace or period, stored for reuse in context assembly.

### Full

- **Retrieval:** **Embedding-based semantic search**. Facts (and optionally summaries) carry vector representations.
- **After each turn:** extract facts via an **LLM** using `extraction_model` (structured output with fields such as content, category, confidence, sender when applicable).
- **Embeddings:** generated through `embedding_provider` (and associated model/configuration); facts reference embeddings (e.g. via `embedding_id` or side store, depending on implementation).
- **On a new message:** embed the query (or composed retrieval string), run **cosine similarity** (or provider-native similarity) over stored vectors, take **top-K**, inject into the prompt.
- **Memory compaction:** merge near-duplicate or redundant facts, expire stale rows, optionally **rebuild** or refresh derived state from raw messages. Keeps the fact store bounded and coherent at scale.

---

## 3. Memory Architecture

Memory is organized in conceptual **layers**. Not every layer is active in every mode.

| Layer | Description | Typical modes |
|-------|-------------|----------------|
| **1** | **Raw messages** persisted in SQLite or PostgreSQL | Always (when conversation persistence is enabled) |
| **2** | **Rolling summaries** per workspace and time period | Lite, Full |
| **3** | **Extracted facts** with sender attribution, confidence, categories | Lite, Full |
| **4** | **Embedding index** for semantic search | Full |
| **5** | **External adapters** (Mem0, Graphiti, others) | Optional; can augment or replace parts of 3–4 |

Layer 1 is the audit trail and source for rebuilds. Layers 2–3 are the main retrieval surface for the built-in engine. Layer 4 extends Layer 3 with vectors. Layer 5 integrates third-party memory products without breaking workspace isolation guarantees enforced elsewhere in ManaClaw.

---

## 4. Database Tables

### `memory_facts`

Stores discrete, searchable memory units scoped to a workspace.

| Column | Description |
|--------|-------------|
| `id` | Primary key |
| `workspace_folder` | Workspace scope (folder or logical workspace identifier) |
| `sender` | Attribution (user id, agent id, or channel identity) |
| `content` | Fact text |
| `category` | Taxonomy label (e.g. preference, decision, todo) |
| `confidence` | Numeric score (implementation-defined scale) |
| `embedding_id` | Optional link to embedding row or blob store (Full mode) |
| `created_at` | Creation timestamp |
| `expires_at` | Optional TTL for automatic pruning |

**Lite:** `content` (and related columns) participate in **FTS5** (or equivalent full-text index).  
**Full:** same row may additionally be linked to vectors for similarity search.

### `memory_summaries`

Stores rolling or periodic narrative summaries.

| Column | Description |
|--------|-------------|
| `id` | Primary key |
| `workspace_folder` | Workspace scope |
| `period_start` | Summary window start |
| `period_end` | Summary window end |
| `content` | Summary text |
| `message_count` | Number of messages covered (for auditing and regen) |
| `created_at` | When the summary was written |

Summaries complement facts: they capture gist across many turns where atomic facts would be noisy.

---

## 5. Fact Extraction

### Lite mode

- **Mechanism:** regex and heuristic extractors over the latest turn (and optionally recent history).
- **Targets:** names and entities, dates and deadlines, stated **preferences**, **todos** / action items, explicit **decisions**.
- **Properties:** fast, deterministic, no extra token cost; lower recall and precision than an LLM extractor.

### Full mode

- **Mechanism:** LLM call with **structured output** (JSON or schema-constrained) aligned with `memory_facts` fields.
- **Per-sender attribution:** facts tied to `sender` for workspace-scoped, multi-actor threads.
- **Confidence scoring:** model or rule-assisted scores stored in `confidence`.
- **Deduplication:** merge or skip near-duplicates before insert (may interact with compaction).

---

## 6. Memory Compaction

Compaction keeps the fact store useful and bounded:

- **Merge** redundant or highly similar facts (especially in Full mode with embedding distance).
- **Expire** facts past `expires_at` or past a policy-based age.
- **Rebuild** or refresh derived facts from **Layer 1** raw messages when corruption or drift is detected.

**Triggers:**

- **HTTP API:** `POST /api/v1/memory/:workspaceId/compact` (see `docs/API.md` for auth and response semantics; implementations may return `202 Accepted` with a job identifier for long-running work).
- **Scheduled maintenance:** operator-defined job (cron, internal scheduler) calling the same compaction pipeline.

Compaction should be safe to run concurrently with reads; writes may be briefly locked or queued depending on implementation.

---

## 7. Per-User Memory in Workspace Contexts

Inspired by approaches such as **Memoh**, ManaClaw treats **sender-attributed facts** as first-class:

- Each fact row can record **who** contributed it (`sender`).
- **Recall** can be **sender-aware**: e.g. prefer facts from the current user, or blend global workspace facts with user-specific preferences.
- This supports shared workspaces (teams) without losing individual preference and accountability.

Policy (how much to weight self vs. others) is a product/runtime concern; the schema supports attribution uniformly.

---

## 8. Adapters

Optional **Layer 5** adapters delegate extraction and/or retrieval to external systems.

### Mem0Adapter

- Connects to the **Mem0** API (SaaS or self-hosted).
- Can provide **automatic fact extraction** and **hybrid** retrieval (keyword + semantic, depending on Mem0 capabilities).
- Configured under `[memory.mem0]` in `config.toml` (see below).

### GraphitiAdapter

- Connects to **Graphiti** (temporal knowledge graph) for **enterprise multi-entity** reasoning and time-aware edges.
- Intended for **phase 2** or later integrations where graph traversal complements flat fact stores.

Adapters should respect workspace boundaries and security policies (mount allowlists, API keys, network egress) documented in `docs/SECURITY.md`.

---

## 9. Configuration Reference (`[memory]`)

Excerpt from `config.example.toml`; copy to `config.toml` and adjust.

```toml
[memory]
mode = "lite"                     # "off" | "lite" | "full"
extraction_model = ""             # model for fact extraction in Full mode; often defaults to [provider].model if empty
embedding_provider = ""           # e.g. "ollama" | "openai" | custom; empty may skip embedding writes

[memory.mem0]
enabled = false
api_key = ""
base_url = ""
```

| Key | Description |
|-----|-------------|
| `mode` | Global memory mode: `off`, `lite`, or `full`. |
| `extraction_model` | Model id for LLM-based extraction (**Full**). Empty typically means “use default chat model from `[provider]`.” |
| `embedding_provider` | Backend used to generate query and document embeddings (**Full**). Empty may disable embedding pipeline while still allowing Lite behavior if mode permits. |
| `memory.mem0.enabled` | When true, Mem0 adapter participates in extraction/retrieval (exact merge semantics depend on implementation). |
| `memory.mem0.api_key` | Credential for Mem0 API. |
| `memory.mem0.base_url` | Mem0 API base URL (required for self-hosted or custom endpoints). |

Additional keys (e.g. summary interval **N**, top-K for injection, TTL defaults) may appear in future config versions; consult the latest `config.example.toml` and release notes.

---

## 10. `MEMORY.md` Files

- A workspace-local **`MEMORY.md`** is a **human-readable** artifact: notes, decisions, checklists, or hand-maintained context.
- ManaClaw **may** append or refresh summaries there for operator visibility, but the **source of truth** for automated retrieval is the **database** (`memory_facts`, `memory_summaries`, raw messages, and embeddings when enabled).
- In **Off** mode, `MEMORY.md` is especially important because automated fact pipelines are inactive.
- Teams should treat Git-friendly workspace docs as **documentation**, not as a replacement for backed-up DB state for compliance or analytics.

---

## Related Documentation

- `docs/API.md` — memory search, list facts, compaction endpoints.
- `docs/PRD.md` — product requirements for layered memory.
- `docs/REQUIREMENTS.md` — mode semantics and workspace/global memory constraints.
- `config.example.toml` — authoritative example for all `[memory]` keys shipped with the repo.
