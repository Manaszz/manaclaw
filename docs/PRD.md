# ManaClaw — Product Requirements Document

**Version:** 1.0  
**Status:** Draft  
**Reference architecture:** [NanoClaw](https://github.com/qwibitai/nanoclaw) (conceptual baseline; ManaClaw is a distinct product with OpenCode runtime, extended channels, and pluggable persistence.)

---

## 1. Vision and Goals

ManaClaw is a **lightweight, provider-agnostic, always-on AI assistant** for corporate environments and personal productivity. It preserves the NanoClaw model of **strong workspace isolation** (one container and filesystem context per conversation scope) while replacing the Anthropic Agent SDK with **OpenCode** as the primary agent runtime, adding **database abstraction**, **first-class integration surfaces** (HTTP Webhook API, CLI, Web UI), and a **layered memory system** that can scale from minimal footprint to rich retrieval.

**Primary goals**

- Run continuously with low resource use and predictable startup time.
- Route inbound messages from multiple channels into isolated workspaces without coupling to a single LLM vendor.
- Give operators clear control via configuration, policy, and observability (tokens, tasks, health).
- Ship a credible v1 that is deployable on a single host (SQLite) and upgradeable to shared PostgreSQL where teams require it.

**Secondary goals**

- Align terminology and mental model with “workspace-first” operations (not “group” as in NanoClaw).
- Keep the host orchestrator small and maintainable; push agent complexity into OpenCode inside the sandbox where appropriate.

---

## 2. Terminology

| Term | Definition |
|------|------------|
| **Workspace** | A registered unit of conversation context. Each workspace has an **isolated folder**, **dedicated container**, **session state**, **scheduled tasks**, and **memory** scope. Replaces NanoClaw’s “group.” |
| **Main workspace** | The default or primary workspace used when routing does not resolve a more specific workspace, or for system-wide defaults (exact rules are implementation-defined in routing config). |
| **Channel** | An integration surface that can send messages to the orchestrator and receive replies (e.g., HTTP Webhook, CLI, Web UI; external messengers via skills). |
| **OpenCode runtime** | The in-container agent execution environment. Supports **hybrid** operation: **`run`** (one-shot / batch-style invocation) and **`serve`** (long-lived service mode) as required by task type and deployment profile. |
| **PolicyEngine** | A centralized rules layer that enforces **allowed commands**, **mount and filesystem constraints**, **network egress** policy, and related guardrails before work executes inside a workspace container or on the host boundary. |

---

## 3. User Personas

### 3.1 Individual power user

Runs ManaClaw on a workstation or small VPS. Wants fast setup, SQLite by default, CLI and minimal Web UI, and freedom to point models at OpenAI-compatible endpoints or local inference. Values low RAM and quick restarts.

### 3.2 Small team lead

Coordinates a few workspaces (projects or clients). Needs HTTP Webhook for internal tools, identifiable senders, basic audit of what the assistant did, and optional PostgreSQL for durability and backup. Cares about isolation between workspaces and simple operational runbooks.

### 3.3 Corporate admin

Operates ManaClaw in a controlled environment. Prioritizes **security policy**, **identity binding**, **token and cost tracking**, optional **audit logging**, and integration with existing SSO or API gateways (via Webhook and future auth layers). Needs clear boundaries: what is in scope for self-hosted v1 vs. future multi-tenant SaaS.

---

## 4. Core Features (Prioritized)

### P0 — Must ship for v1

| ID | Feature | Notes |
|----|---------|--------|
| P0-1 | **Workspace isolation** | Per-workspace folder, container, queue, and IPC namespace; no cross-workspace leakage of filesystem or credentials. |
| P0-2 | **OpenCode runtime (hybrid run/serve)** | Container runner spawns OpenCode in appropriate mode; host orchestrates lifecycle, config injection, and log/stream handling. |
| P0-3 | **HTTP Webhook API** | Inbound/outbound message contract for automation and integrations; authentication and rate limits as specified in security docs. |
| P0-4 | **CLI** | Operator workflows: status, workspace management, send/receive testing, task inspection. |
| P0-5 | **SQLite persistence (default)** | Messages, routing state, tasks, sessions, registrations; migrations supported. |
| P0-6 | **Message routing** | Deterministic mapping from channel identity to workspace; formatting and sender allowlisting. |
| P0-7 | **Scheduled tasks** | Cron, interval, and one-shot scheduling with per-workspace scoping and safe execution hooks into OpenCode. |
| P0-8 | **IPC** | Filesystem-based (or equivalent) IPC between host and workspace agents for tasks and control messages. |
| P0-9 | **Provider-agnostic configuration** | Model and provider settings decoupled from any single vendor; credentials supplied via config/env patterns suitable for containers. |
| P0-10 | **PolicyEngine** | Enforce mounts, command allow/deny, and egress policy; integrate with container args and OpenCode permissions. |
| P0-11 | **Basic documentation** | Install, configuration, security considerations, Webhook API overview, and operator troubleshooting. |

### P1 — High value shortly after v1

| ID | Feature | Notes |
|----|---------|--------|
| P1-1 | **Layered memory (Off / Lite / Full)** | Operator-selectable modes controlling cost, storage, and retrieval depth. |
| P1-2 | **Fact extraction** | Automated structuring of stable facts from conversations for retrieval (quality and safety constraints apply). |
| P1-3 | **PostgreSQL adapter** | Optional DB backend with same logical schema/abstractions as SQLite. |
| P1-4 | **Identity binding** | Map external identities (e.g., chat IDs, email addresses) to stable internal principals within policy limits. |
| P1-5 | **Token tracking** | Per-workspace and per-channel usage metrics for budgeting and troubleshooting. |
| P1-6 | **Memory compaction** | Bounded storage via summarization, pruning, or archival according to policy and mode. |

### P2 — Roadmap

| ID | Feature | Notes |
|----|---------|--------|
| P2-1 | **Web UI v2 (Vue 3)** | Full SPA: chat, dashboard, config editing, memory browser, token views. |
| P2-2 | **Messaging channel skills** | Optional integrations: Telegram, Slack, WhatsApp, Gmail (via skill modules, not necessarily in core binary). |
| P2-3 | **Deployment profiles** | Presets for dev / single-node / team server (ports, DB, logging, resource limits). |
| P2-4 | **Audit logging** | Append-only security and operator action logs for compliance-oriented deployments. |
| P2-5 | **Container snapshots** | Save/restore workspace container state for backup, migration, or reproducibility (subject to PolicyEngine and storage constraints). |

---

## 5. Non-Functional Requirements

| Area | Requirement |
|------|-------------|
| **Startup** | Cold start to “ready to accept traffic” **under 5 seconds** on a reference small VM (definition of “ready” includes DB migration check and HTTP listener where applicable). |
| **Memory** | Steady-state host process **under 100 MB RAM** excluding container workloads and model servers (measurement methodology documented for releases). |
| **Concurrency** | Support **multiple workspace containers** concurrently with fair queuing; avoid head-of-line blocking across unrelated workspaces. |
| **Security** | Default-deny sensitive operations unless allowed by PolicyEngine; secrets not written to workspace-visible paths; Webhook authentication mandatory in non-dev profiles. |
| **Reliability** | Graceful degradation: channel or workspace failure must not crash the entire orchestrator; tasks retried or surfaced per policy. |
| **Observability** | Structured logs; health endpoint or CLI health command; correlation identifiers for Webhook requests. |

---

## 6. Out of Scope for v1

The following are **explicitly not required** for the first production-capable release:

- **Multi-tenant SaaS** — no shared control plane, billing, or per-tenant isolation productization.
- **Model hosting** — ManaClaw does not ship or manage inference servers; it integrates with external or self-hosted providers.
- **Voice and video** — no real-time A/V pipelines or telephony.
- **Native mobile applications** — Web UI and Webhook/CLI only on the client surface for v1.

---

## 7. Success Metrics

| Metric | Target / instrument |
|--------|---------------------|
| **Time to first message** | Install to successful round-trip on Webhook or CLI within documented bounds. |
| **Resource profile** | Measured RAM and CPU at idle and under moderate multi-workspace load; regression tests in CI where feasible. |
| **Isolation incidents** | Zero critical escapes (cross-workspace file or secret access) in security review scenarios. |
| **Operator satisfaction** | Complete documented workflows without undocumented steps (tracked via doc revisions and feedback). |
| **Integration adoption** | HTTP Webhook used in reference automation example; skills adoption tracked separately in P2. |

---

## 8. Constraints and Assumptions

- **Container runtime:** Docker (or compatible) is available on the host; workspace isolation depends on it.
- **OpenCode** is the **primary** agent runtime; behavior and flags may evolve—integration layer must be version-pinned and documented.
- **SQLite** remains valid for single-node deployments; PostgreSQL is optional and must not fragment application logic (shared abstractions).
- **Corporate networks** may restrict egress; PolicyEngine and channel design must support “no external call” profiles.
- **NanoClaw lineage** informs architecture but **ManaClaw is not required to maintain API compatibility** with NanoClaw unless explicitly decided elsewhere.

---

## 9. Delivery Phases

| Phase | Name | Focus |
|-------|------|--------|
| **0** | Foundation | Repository layout, reference NanoClaw import, build/test harness, baseline docs skeleton, configuration schema. |
| **1** | Runtime swap | Replace in-container agent with OpenCode (`run`/`serve`), credential/config injection, container images, minimal smoke tests. |
| **2** | Channels and API | HTTP Webhook API, CLI parity for core ops, message routing, sender policy, status/health. |
| **3** | Persistence and tasks | SQLite schema/migrations, scheduler, IPC hardening, PolicyEngine MVP, P0 documentation completion. |
| **4** | Hardening and P1 | PostgreSQL adapter, layered memory modes (Lite path first), token tracking, identity binding foundations, compaction design implementation. |
| **5** | Productization and P2 | Web UI v2, channel skills, deployment profiles, audit logging, snapshots—rolled out incrementally with feature flags where appropriate. |

---

## Document control

| Version | Date | Summary |
|---------|------|---------|
| 1.0 | 2026-03-24 | Initial PRD for ManaClaw scope, priorities, and phases. |
