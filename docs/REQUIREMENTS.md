# ManaClaw Requirements

Original requirements and design decisions from the project. This document defines philosophy, boundaries, and non-negotiable architectural choices.

---

## Why ManaClaw Exists

ManaClaw is a lightweight, secure alternative to heavier agent stacks that accumulate gateways, opaque configuration, and weak execution boundaries. Large systems often run agents without true process isolation, rely on application-level permission hacks, and grow too large to audit or trust.

ManaClaw keeps the surface area small: understandable code, **container-isolated** agent execution, and **provider-agnostic** model access via OpenAI-compatible APIs. It adds first-class **HTTP webhook** and **CLI** channels alongside optional messaging integrations, so it fits **personal workflows and small teams** without becoming a generic platform.

---

## Philosophy

### Small Enough to Understand

The codebase should remain readable end-to-end. Prefer a single coherent runtime and clear modules over microservices, opaque middleware, and unnecessary abstraction.

### Security Through Isolation

Agents run in **Linux containers** with explicit mounts. Isolation is an OS boundary, not a policy layer inside the app. Shell and tool use are constrained by what the container can see; the host is not the agent’s default filesystem.

### Provider-Agnostic Models

ManaClaw talks to **OpenAI-compatible** HTTP APIs. The project does not hard-code a single vendor SDK as the only path to intelligence. Swap endpoints and keys; behavior stays consistent at the integration boundary.

### Database Flexibility

Persistence uses a **small abstraction** over SQL: **SQLite by default** for simplicity and local use; **PostgreSQL optional** for durability, concurrency, and deployment scenarios that need a shared server database.

### Built for Individual and Team Productivity

ManaClaw targets **solo operators and small corporate or team** setups: shared workspaces, clear boundaries between contexts, and channels that fit automation (webhooks, CLI) as well as human chat where applicable.

### Customization = Code Changes

Avoid configuration sprawl. Environment and a few toggles cover deployment; behavior changes belong in code the operator can read and own. A small codebase makes that practical.

### Layered Memory

Memory is not a single global file. ManaClaw uses a **layered model** with **Off / Lite / Full** modes so operators can trade context richness for cost, latency, and privacy. Details belong in architecture docs; the requirement is **explicit modes**, not implicit “always load everything.”

### Skills Over Features

Contributions should prefer **skills** that reshape or extend the codebase (e.g. add a channel, swap DB backend) over piling optional features into one binary that tries to satisfy every deployment at once.

---

## Architecture Decisions

### Message Routing

- Inbound events arrive through defined **channels** (e.g. HTTP webhook, CLI; messaging adapters where present).
- Routing associates each event with a **workspace** and applies trigger rules, authentication, and rate limits as documented per channel.
- Unregistered or unauthorized sources are ignored or rejected at the boundary.

### Workspace Isolation

- **Workspace** replaces the older “group” concept: a named context with its own data directory, policies, and schedules.
- Each workspace has a dedicated folder under the workspace root; mounts are explicit for container runs.
- Cross-workspace access is denied unless handled through a designated **main** (admin) workspace with documented privileges.

### Memory System

- **Off**: No persistent memory beyond the current run (or minimal session-only state).
- **Lite**: Lightweight context (summaries, small artifacts) with bounded size and clear eviction rules.
- **Full**: Richer retention and recall appropriate for longer-running assistance, still subject to workspace boundaries and operator controls.
- Workspace-level and global layers are composed according to mode; global writable memory is restricted to privileged paths (analogous to a main workspace).

### Session Management

- **OpenCode** is the **agent runtime**: sessions, tool loops, and compaction behavior follow OpenCode’s model rather than a vendor-specific agent SDK.
- Long context is managed via compaction or summarization so sessions remain usable without unbounded cost.

### Container Isolation

- Every agent invocation runs inside a container with mounted workspace (and optional) paths only.
- Browser automation, shell, and file tools execute **inside** the container image; Chromium or equivalent tooling lives there, not on arbitrary host paths by default.

### Scheduled Tasks

- Users may schedule recurring or one-time jobs in a workspace context.
- Tasks run as full agent (or tool) invocations inside containers, with the same isolation rules as interactive use.
- Task definitions and run history persist via the configured database; scheduler runs on the host and dispatches work.
- From the main workspace: schedule or manage tasks for any workspace; from other workspaces: only that workspace’s tasks unless policy says otherwise.

### Workspace Management

- New workspaces are registered explicitly (CLI, API, or privileged channel), not inferred from every incoming message.
- Registration is stored in the database; each workspace gets a filesystem subtree and optional extra mounts in `containerConfig`.
- Renaming or retiring workspaces is an explicit operation to avoid orphaned data or schedules.

### Channels

- **HTTP Webhook API**: machine-first ingress for integrations, automation, and corporate glue.
- **CLI**: local and scripted control without a separate UI.
- Additional channels (e.g. messaging) are optional adapters behind the same routing and workspace model.

---

## Integration Points

### OpenCode (Agent Runtime)

- Drives tool use, session lifecycle, and agent behavior.
- Replaces a single-vendor “agent SDK” as the core execution engine while keeping the stack swappable at the model HTTP boundary.

### Model Providers (OpenAI-Compatible)

- Configuration points to base URL, API key, and model names compatible with the OpenAI Chat/Completions (or documented) schema.
- No requirement to use a specific cloud; self-hosted or alternate gateways are valid if they honor the contract.

### Database

- Abstract access layer with **SQLite** default and **PostgreSQL** optional, selected by configuration.
- Schema migrations apply to both backends where features overlap.

### Webhook and CLI

- Webhook: authenticated HTTP endpoints for inbound events and optional callbacks.
- CLI: workspace-scoped commands for send, status, admin, and maintenance operations.

### Scheduler

- Host-side loop loads due tasks from the database and spawns containerized runs.
- In-container MCP or tools expose schedule CRUD where applicable; delivery back to channels uses a shared `send_message` (or equivalent) abstraction.

### Web Access and Browser Automation

- Web search and fetch tools as configured for the runtime.
- Browser automation via a containerized browser CLI with snapshot-style interaction, consistent with isolation goals.

---

## RFS (Request for Skills)

Skills we would welcome from contributors:

### Channels and Ingress

- `/add-telegram`, `/add-slack`, `/add-discord` — optional messaging channels behind the workspace router
- `/harden-webhook` — auth schemes, signing, IP allowlists, idempotency patterns

### Runtime and Platform

- `/convert-to-apple-container` — macOS-oriented container runtime swap (where applicable)
- `/setup-linux` / `/setup-windows` — documented full stack on Linux or Windows (e.g. WSL2 + container engine)

### Data and Memory

- `/migrate-to-postgres` — operational playbook and config for PostgreSQL-only or hybrid deployments
- `/tune-memory-modes` — presets for Off/Lite/Full per workload (personal vs team)

### Provider and Observability

- `/add-provider-preset` — documented configs for common OpenAI-compatible hosts
- `/add-metrics` — minimal metrics export without turning the project into a microservice mesh

---

## Vision

A **small, auditable** agent assistant that:

- Uses **OpenCode** as the agent runtime and **OpenAI-compatible** APIs for models
- Enforces **container isolation** for every agent run
- Persists state through a **SQL abstraction** (SQLite default, PostgreSQL optional)
- Exposes **HTTP webhooks** and **CLI** as first-class channels, with optional messaging adapters
- Organizes work in **workspaces** with **layered memory** (Off / Lite / Full)
- Ships under the **MIT license** and stays understandable as a whole codebase

**Planned evolution:**

- **v2**: **Web UI** for monitoring, workspace administration, and light operator workflows—without replacing the core preference for code-level customization.

---

## Setup & Customization

### Approach

- Few environment variables and one obvious config path; no wizard required for advanced users.
- Setup and deep customization are expected to involve reading and editing code or running documented skills.
- AI-assisted development (e.g. IDE agents) is a first-class way to deploy and adapt ManaClaw.

### Skills (Examples)

- `/setup` — dependencies, database init, container image build, channel smoke tests
- `/customize` — add channels, change routing, extend tools, adjust memory defaults
- `/update` — merge upstream changes, run migrations, reconcile local forks

### Deployment

- Single primary service process (plus container engine and optional reverse proxy for webhooks)
- Suitable for a dedicated VM, internal server, or power-user laptop; scale-out is not a day-one goal

---

## Project Name and License

**ManaClaw** — a compact, isolated, provider-flexible claw on the same problem space as larger “claw”-style assistants, deliberately smaller and clearer.

Licensed under the **MIT License**.
