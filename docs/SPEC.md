# ManaClaw Technical Specification

A multi-channel, workspace-isolated AI assistant with an OpenCode-based agent runtime, pluggable persistence (SQLite or PostgreSQL), layered memory, scheduled tasks, and policy-governed container execution.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Architecture: Channel System](#architecture-channel-system)
3. [Folder Structure](#folder-structure)
4. [Configuration](#configuration)
5. [Memory System](#memory-system)
6. [Session Management](#session-management)
7. [Message Flow](#message-flow)
8. [Commands](#commands)
9. [Scheduled Tasks](#scheduled-tasks)
10. [MCP Servers](#mcp-servers)
11. [Deployment](#deployment)
12. [Security Considerations](#security-considerations)

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  HOST PROCESS (Node.js 20+)                                                  │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  HTTP Server (Fastify)                                                  │ │
│  │  REST/Webhook API, health, optional Web UI static assets                │ │
│  └───────────────────────────────┬───────────────────────────────────────┘ │
│                                    │                                          │
│  ┌─────────────────────────────────▼───────────────────────────────────────┐ │
│  │  Orchestrator                                                            │ │
│  │  ┌──────────────┐ ┌────────────────┐ ┌───────────────┐ ┌──────────────┐ │ │
│  │  │MessageRouter │ │ WorkspaceQueue │ │ TaskScheduler │ │  IPCWatcher  │ │ │
│  │  │ (inbound/out)│ │ (per-workspace │ │ (cron/interval│ │ (filesystem  │ │ │
│  │  │  formatting) │ │  fair queue)   │ │  /once, due)  │ │  host<->ctr) │ │ │
│  │  └──────┬───────┘ └───────┬────────┘ └───────┬───────┘ └──────┬───────┘ │ │
│  │         │                 │                  │                │         │ │
│  │         └─────────────────┼──────────────────┼────────────────┘         │ │
│  │                           │                  │                          │ │
│  │         ┌─────────────────▼──────────────────▼─────────────────┐       │ │
│  │         │  DatabaseAdapter (SQLite / PostgreSQL)                 │       │ │
│  │         │  messages, workspaces, sessions, tasks, router state   │       │ │
│  │         └────────────────────────────────────────────────────────┘       │ │
│  │                           ▲                                                │ │
│  │  ┌────────────────────────┴──────────────┐  ┌─────────────────────────┐   │ │
│  │  │  MemoryService                        │  │  PolicyEngine             │   │ │
│  │  │  (Off / Lite / Full, layers)          │  │  mounts, commands,      │   │ │
│  │  │  hooks on ingest / agent context      │  │  egress, OpenCode perms │   │ │
│  │  └───────────────────────────────────────┘  └─────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                                    │ spawn / lifecycle                           │
│  ┌─────────────────────────────────▼──────────────────────────────────────────┐ │
│  │  Channels (built-in + skill-registered)  ──► persist / poll ◄──► DB        │ │
│  └──────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      │ per-workspace container
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  CONTAINER (Docker / Podman)                                                 │
│  ┌───────────────────────────────────────────────────────────────────────────┐ │
│  │  OpenCodeRuntime                                                          │ │
│  │  Working directory: /workspace/workspace (host: groups/{folder}/)          │ │
│  │  Session / project config: .opencode/, opencode.json                      │ │
│  │  MCP: manaclaw (stdio) — schedule_task, send_message, task management     │ │
│  │  Tools: bash (sandboxed), files, web, browser (optional), custom MCP     │ │
│  └───────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|------------|---------|
| HTTP API | Fastify | Webhook and REST surface for integrations |
| Persistence | SQLite (default) or PostgreSQL | Messages, workspaces, sessions, tasks, routing state |
| Container runtime | Docker or Podman | Per-workspace isolation for agent execution |
| Agent runtime | OpenCode | Provider-agnostic agent loop, tools, MCP inside container |
| Host runtime | Node.js 20+ | Orchestration, routing, scheduling, policy, DB adapter |

Throughout this document, **NanoClaw “group” corresponds to ManaClaw “workspace.”** Database tables or legacy field names may still use `group` where migration compatibility matters; operator-facing docs use **workspace**.

---

## Architecture: Channel System

ManaClaw ships **built-in** channels that do not require a skill branch: **HTTP Webhook** (Fastify API) and **CLI** (operator commands and piping). Optional **Web UI** (minimal in v1, fuller SPA in later phases) may share the same HTTP server.

**Skill-based** channels follow the same **self-registration** pattern as NanoClaw, adapted for workspace terminology:

- Telegram  
- Slack  
- WhatsApp  
- Gmail  

Each skill adds a `src/channels/<name>.ts` module that calls `registerChannel()` at load time. The barrel `src/channels/index.ts` imports these modules so registration runs before the orchestrator connects channels.

### Self-Registration Pattern (workspace terminology)

1. Each skill implements a **channel factory** that receives `ChannelOpts` (callbacks for `onMessage`, optional metadata sync, and **registered workspaces**).
2. The factory returns a `Channel` instance when credentials are present, or **`null`** if configuration is missing (the host logs a warning and skips that channel).
3. At startup, the orchestrator iterates `getRegisteredChannelNames()`, instantiates each channel, and calls `connect()`.

### Channel Interface (conceptual)

Every channel implements the same logical contract as NanoClaw’s `Channel` interface: `name`, `connect`, `sendMessage`, `isConnected`, `ownsJid` (or equivalent workspace/chat identifier), `disconnect`, and optional `setTyping` / `syncWorkspaces`.

### Adding a New Channel via Skill

A skill under `.opencode/skills/add-<name>/` (or the project’s documented skills location) should:

1. Add `src/channels/<name>.ts` implementing the channel and calling `registerChannel('<name>', factory)`.
2. Add a barrel import in `src/channels/index.ts`.
3. Return `null` from the factory when secrets or auth files are absent.
4. Document required environment variables or config keys.

See the **Feature** skills `/add-telegram`, `/add-slack`, `/add-whatsapp`, and `/add-gmail` for worked examples.

---

## Folder Structure

```
manaclaw/
├── docs/
│   ├── SPEC.md                 # This specification
│   ├── PRD.md
│   ├── API.md
│   ├── DEPLOYMENT.md
│   ├── SKILLS.md
│   └── DEBUG_CHECKLIST.md
├── src/
│   ├── index.ts                # Host entry: HTTP server + orchestrator wiring
│   ├── channels/
│   │   ├── registry.ts         # Channel factory registry
│   │   └── index.ts            # Barrel imports (self-registration)
│   ├── db.ts                   # DatabaseAdapter: schema, migrations, queries
│   ├── router.ts               # MessageRouter: format, trigger, workspace resolution
│   ├── workspace-queue.ts      # Per-workspace queue + global concurrency
│   ├── task-scheduler.ts       # TaskScheduler: cron / interval / once
│   ├── ipc.ts                  # IPCWatcher: filesystem IPC with containers
│   ├── container-runner.ts     # Spawn OpenCode in container
│   ├── memory-service.ts       # MemoryService: modes and layers
│   ├── policy-engine.ts        # PolicyEngine: mounts, commands, egress
│   ├── config.ts               # Load config.toml + env overrides
│   ├── logger.ts
│   └── types.ts
├── container/
│   ├── Dockerfile              # manaclaw-agent image
│   ├── build.sh                # Optional image build helper
│   └── agent-bundle/           # In-container bootstrap, MCP stdio server, etc.
├── bin/
│   └── manaclaw                # Optional CLI entry (package bin)
├── groups/                     # Per-workspace filesystem roots (legacy path name)
│   ├── MEMORY.md               # Global memory visible to all workspaces (policy)
│   ├── {channel}_{workspace}/  # e.g. webhook_main, slack_acme-corp
│   │   ├── MEMORY.md           # Workspace-scoped memory
│   │   └── logs/
│   └── ...
├── reference/                  # Upstream reference (e.g. NanoClaw)
├── store/                      # Local state (gitignored)
│   ├── manaclaw.db             # SQLite when provider = sqlite
│   └── auth/                   # Channel auth artifacts (e.g. WhatsApp)
├── data/                       # Runtime state (gitignored)
│   ├── sessions/               # OpenCode session files per workspace
│   ├── env/                    # Sanitized env file for container mount
│   └── ipc/                    # IPC directories (messages, tasks)
├── logs/
│   ├── manaclaw.log
│   └── manaclaw.error.log
├── config.toml                 # Operator configuration (from config.example.toml)
├── config.example.toml
├── package.json
└── tsconfig.json
```

**Note:** The directory name `groups/` is retained for familiarity with NanoClaw layouts; each subdirectory represents one **workspace** in ManaClaw terms.

---

## Configuration

ManaClaw reads **`config.toml`** at the project root (copy from `config.example.toml`). Environment variables may override sensitive fields (for example database password).

### Schema Reference (`config.toml`)

| Section | Keys | Description |
|---------|------|-------------|
| `[general]` | `assistant_name`, `trigger_pattern`, `poll_interval_ms`, `scheduler_poll_interval_ms`, `max_concurrent_containers`, `container_timeout_ms`, `idle_timeout_ms` | Core orchestration timing and concurrency |
| `[server]` | `host`, `port`, `api_key` | Fastify bind address and optional Bearer token for API auth |
| `[database]` | `provider` | `sqlite` or `postgres` |
| `[database.sqlite]` | `path` | Filesystem path to SQLite file (often under `store/`) |
| `[database.postgres]` | `host`, `port`, `database`, `user`, `password` | PostgreSQL connection; prefer env for `password` |
| `[container]` | `image`, `runtime` | Agent image name and `docker` or `podman` |
| `[provider]` | `name`, `base_url`, `api_key`, `model`, `fallback_model` | Default LLM provider profile for OpenCode |
| `[memory]` | `mode`, `extraction_model`, `embedding_provider` | Memory mode and optional extraction/embedding routing |
| `[memory.mem0]` | `enabled`, `api_key`, `base_url` | Optional Mem0 adapter |
| `[security]` | `mount_allowlist_path`, `blocked_patterns` | PolicyEngine mount allowlist and path deny patterns |

Paths used in volume mounts must be **absolute** where the container runtime requires them; the implementation resolves relative paths from the project root where appropriate.

### Workspace Container Overrides

Per-workspace JSON (stored in the database alongside registration) may include additional mounts, timeouts, and labels. The PolicyEngine validates each mount against `mount_allowlist_path` and blocked patterns before spawn.

---

## Memory System

ManaClaw supports operator-selectable **memory modes** and a **layered** model for what is stored and retrieved.

### Modes

| Mode | Behavior |
|------|----------|
| **off** | No structured memory service; optional markdown files only (`MEMORY.md`) for human-readable notes |
| **lite** | Lightweight retrieval (for example keyword / BM25 over stored messages and summaries in SQLite) without mandatory embedding infrastructure |
| **full** | Embeddings + optional external memory backends (for example Mem0); deeper retrieval and fact layers enabled |

### Layers (logical)

| Layer | Typical content | Notes |
|-------|-----------------|-------|
| L1 | Raw messages | Durable chat history in DB |
| L2 | Rolling summaries | Bounded context for long threads |
| L3 | Extracted facts | Structured facts with safety and deduplication policy |
| L4 | Vector retrieval | Embedding index (local or API) when mode is `full` |
| L5 | External graph / memory services | Optional Mem0, Graphiti, or similar adapters |

The MemoryService composes layers based on `memory.mode` and feeds the OpenCode runtime with **injected context** according to policy (token budgets, workspace isolation).

---

## Session Management

OpenCode owns the **authoritative session state** inside each workspace container (files under `data/sessions/{workspaceFolder}/` and `.opencode/` as implemented).

ManaClaw **mirrors** a stable session identifier in the database (for example keyed by workspace folder or internal workspace id) so that:

- Restarts can **resume** the correct OpenCode session.
- Multiple channels mapping to the same workspace share continuity.
- Operators can inspect or reset sessions via CLI/API where exposed.

Session migration or reset operations must update both the **DB mirror** and the **on-disk OpenCode session** consistently.

---

## Message Flow

### Incoming Message (conceptual end-to-end)

```
1. User or system sends a message (Webhook, CLI, or skill-based channel)
        │
        ▼
2. Channel adapter normalizes payload → internal message record
        │
        ▼
3. DatabaseAdapter persists message (and chat/workspace metadata as needed)
        │
        ▼
4. Host poll loop OR push hook picks up new messages (per general.poll_interval_ms)
        │
        ▼
5. MessageRouter:
        ├── Resolves workspace (registered workspace, trigger, sender policy)
        ├── Applies trigger pattern and allowlists
        └── Builds conversation context since last agent turn
        │
        ▼
6. WorkspaceQueue enqueues work for that workspace (respects max concurrent containers)
        │
        ▼
7. Container runner starts or reuses a workspace container
        │
        ▼
8. OpenCodeRuntime executes agent turn (tools, MCP, memory-injected context)
        │
        ▼
9. Assistant output returned via IPC/stream parsing to host
        │
        ▼
10. Router sends response via owning channel (HTTP poll response, CLI, Telegram, etc.)
        │
        ▼
11. DB updated: last agent timestamp, session id mirror, optional token usage
```

Built-in HTTP clients typically **POST** a message and **GET** poll for replies; messengers use push/send on the channel implementation.

---

## Commands

### Workspace Commands (main or privileged workspace)

| Command | Example | Effect |
|---------|---------|--------|
| Add workspace | `@ManaClaw add workspace "Acme Support"` | Register a new workspace and folder mapping |
| Remove workspace | `@ManaClaw remove workspace "Acme Support"` | Unregister and policy-clean resources |
| List workspaces | `@ManaClaw list workspaces` | Show registered workspaces |

Exact natural-language forms may be delegated to the agent; CLI/API equivalents are recommended for automation.

### Task Commands (any workspace, with policy)

| Command | Example | Effect |
|---------|---------|--------|
| List tasks | `@ManaClaw list my scheduled tasks` | List this workspace’s tasks |
| Pause / resume / cancel | `@ManaClaw pause task <id>` | Task lifecycle |

Main workspace (if configured) may list or schedule tasks for other workspaces when policy allows.

---

## Scheduled Tasks

The **TaskScheduler** evaluates due tasks on `scheduler_poll_interval_ms`. Tasks run **in workspace context** inside a container with OpenCode and the same tool policy as interactive messages.

### Schedule Types

| Type | Value format | Example |
|------|--------------|---------|
| `cron` | Standard cron expression | `0 9 * * 1` (Mondays 09:00) |
| `interval` | Milliseconds | `3600000` (hourly) |
| `once` | ISO 8601 timestamp | `2026-12-25T09:00:00Z` |

### MCP Tools

Scheduled task creation and management are exposed to the agent via the **manaclaw** MCP server inside the container (for example `schedule_task`, `list_tasks`, `update_task`, `pause_task`, `resume_task`, `cancel_task`, `send_message`). The host enforces authorization: a task may only target its owning workspace unless elevated policy allows otherwise.

---

## MCP Servers

### `manaclaw` MCP (in-container)

The container runs a stdio (or configured) MCP server that proxies structured actions to the host via IPC:

| Tool | Purpose |
|------|---------|
| `schedule_task` | Create cron / interval / once task |
| `list_tasks` | List tasks visible to this workspace |
| `get_task` | Task detail and run history |
| `update_task` | Change prompt or schedule |
| `pause_task` / `resume_task` / `cancel_task` | Lifecycle |
| `send_message` | Send outbound message to workspace’s channel |

Additional tools may be added for memory compaction or status; keep the surface minimal and auditable.

---

## Deployment

Typical layouts:

- **Development:** Single host, SQLite, local Ollama or remote OpenAI-compatible API, Docker for workspace containers only.
- **Production:** Managed PostgreSQL, secrets via environment or secret store, TLS termination in front of Fastify, resource limits on agent containers, log aggregation.

See [DEPLOYMENT.md](./DEPLOYMENT.md) for Docker images, Compose profiles, and platform notes.

---

## Security Considerations

### Container Isolation

- Agent code and shell run **inside** the workspace container, not on the host.
- Mounts are allowlisted and validated by **PolicyEngine**.
- Containers should run as a **non-root** user with minimal capabilities.

### API and Webhook Exposure

- When `server.api_key` is set, require `Authorization: Bearer`.
- Use TLS in production; never expose an unauthenticated Webhook to the public internet.

### Prompt Injection

Untrusted chat content may attempt to override instructions. Mitigations include workspace-scoped tools, PolicyEngine limits, channel allowlists, and review of scheduled task prompts.

### Secrets

- Do not mount host `.env` wholesale; pass a **sanitized** env file into the container (see `data/env/` pattern).
- Keep provider API keys out of workspace-readable paths unless explicitly required.

### Data Protection

- Restrict file permissions on `groups/` and `store/`.
- Back up SQLite/PostgreSQL and `data/sessions/` according to retention policy.

---

## Troubleshooting

For operator diagnostics, see [DEBUG_CHECKLIST.md](./DEBUG_CHECKLIST.md).
