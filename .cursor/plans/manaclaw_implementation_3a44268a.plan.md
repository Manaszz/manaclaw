---
name: ManaClaw Implementation
overview: "PRD, техническая спецификация и пошаговый tasklist для полной реализации ManaClaw (все 6 фаз). Терминология: NanoClaw \"group\" = ManaClaw \"workspace\"."
todos:
  - id: phase-0-repo
    content: "Phase 0.1: Repository setup -- init git, .gitignore, package.json, tsconfig, config.example.toml, LICENSE"
    status: completed
  - id: phase-0-docs
    content: "Phase 0.2: Documentation -- PRD.md, SPEC.md, REQUIREMENTS.md, SECURITY.md, API.md, OPENCODE_INTEGRATION.md, MEMORY.md, DEPLOYMENT.md, SKILLS.md, DEBUG_CHECKLIST.md, 6 ADR files"
    status: completed
  - id: phase-0-readme
    content: "Phase 0.3: README.md + initial commit + push"
    status: completed
  - id: phase-1-types-config
    content: "Phase 1.1: types.ts, config.ts (TOML loader), logger.ts"
    status: pending
  - id: phase-1-db
    content: "Phase 1.2: DatabaseAdapter interface + SQLiteAdapter + tests"
    status: pending
  - id: phase-1-core
    content: "Phase 1.3: router, workspace-queue, task-scheduler, ipc, sender-allowlist, workspace-folder (adapted from NanoClaw)"
    status: pending
  - id: phase-1-webhook
    content: "Phase 1.4: HTTP Webhook channel (Fastify, all API endpoints) + tests"
    status: pending
  - id: phase-1-cli
    content: "Phase 1.5: CLI channel (bin/manaclaw.ts) + tests"
    status: pending
  - id: phase-1-orchestrator
    content: "Phase 1.6: Orchestrator (src/index.ts) + integration test"
    status: pending
  - id: phase-2-opencode
    content: "Phase 2: OpenCode runtime (config generator, output parser, runtime, credential manager, PolicyEngine, container image, e2e test)"
    status: pending
  - id: phase-3-memory
    content: "Phase 3: Memory service (Off/Lite/Full modes, fact extractor, summarizer, Mem0 adapter stub)"
    status: pending
  - id: phase-4-postgres
    content: "Phase 4: PostgreSQL adapter, identity binding, token tracking, Docker Compose"
    status: pending
  - id: phase-5-skills-ui
    content: "Phase 5: Messaging skills, Web UI v2, deployment profiles, hardening"
    status: pending
isProject: false
---

# ManaClaw: PRD + Spec + Implementation Tasklist

## Part 1: Product Requirements Document (PRD)

### 1.1 Vision

ManaClaw -- лёгкий always-on AI assistant для корпоративной среды и личной продуктивности. Провайдер-независимый (OpenAI-compatible), с контейнерной изоляцией инструментов, pluggable memory и built-in HTTP/CLI/Web каналами.

Ключевое отличие от NanoClaw: полная независимость от Anthropic, DB abstraction (SQLite/PostgreSQL), built-in HTTP API и CLI, layered memory system.

### 1.2 Terminology

- **Workspace** (ранее "group" в NanoClaw) -- единица изоляции: зарегистрированный контекст общения с ассистентом. Каждый workspace получает свою папку, контейнер, сессию, задачи, память.
- **Main workspace** -- привилегированный admin workspace (аналог main group в NanoClaw).
- **Channel** -- источник/приёмник сообщений (HTTP Webhook, CLI, Telegram, Slack, etc.).
- **OpenCode Runtime** -- агентный движок внутри контейнера.
- **PolicyEngine** -- система контроля доступа к инструментам, маунтам, сети.

### 1.3 User Personas

- **Individual power user**: запускает ManaClaw локально, использует CLI + Telegram, SQLite, Ollama для local LLM. Хочет приватного always-on ассистента.
- **Small team lead**: запускает ManaClaw на сервере, несколько workspaces для разных проектов/каналов, HTTP Webhook для CI/CD интеграций.
- **Corporate admin**: деплоит ManaClaw с PostgreSQL, multiple workspaces, Slack + Gmail, token tracking, RBAC, audit logging.

### 1.4 Core Features (prioritized)

**P0 -- Must Have (Phase 0-2)**

- Workspace isolation: per-workspace container, folder, session, tasks
- OpenCode runtime inside container (hybrid run/serve)
- HTTP Webhook API (REST + SSE)
- CLI client
- SQLite database (default)
- Message routing and conversation catch-up
- Scheduled tasks (cron/interval/once)
- IPC between host and container
- Provider-agnostic model configuration (OpenAI-compatible)
- PolicyEngine for mounts/commands
- Basic documentation (SPEC, REQUIREMENTS, SECURITY, API)

**P1 -- Should Have (Phase 3-4)**

- Layered memory service (Off/Lite/Full modes)
- LLM-driven fact extraction
- Rolling summaries
- PostgreSQL adapter
- Cross-channel identity binding
- Token usage tracking per workspace/provider
- Memory compaction jobs

**P2 -- Nice to Have (Phase 5)**

- Web UI v2 (Vue 3 SPA: chat, dashboard, config, memory browser)
- Messaging channel skills (Telegram, Slack, WhatsApp, Gmail)
- Deployment profiles (local-dev, corp-onprem, personal-secure)
- Audit logging
- Container snapshots (save/restore workspaces)

### 1.5 Non-Functional Requirements

- **Startup**: < 5s cold start, < 500ms warm message routing
- **Resource**: < 100MB RAM host process (without containers)
- **Concurrency**: configurable max concurrent containers (default 5)
- **DB**: SQLite handles 100K+ messages, PostgreSQL for enterprise scale
- **Security**: no real credentials inside containers, policy-enforced mounts
- **Testing**: unit tests for all adapters, integration test for full message flow

### 1.6 Out of Scope (v1)

- Multi-tenant SaaS deployment
- Built-in model hosting (Ollama/vLLM are external services)
- Voice/video channels
- Mobile app
- Real-time collaborative editing

---

## Part 2: Technical Specification

### 2.1 System Architecture

```
Host Process (Node.js, single process)
├── HTTP Server (Fastify)
│   ├── Webhook API (/api/v1/*)
│   └── Web UI static files (v2)
├── Orchestrator (src/index.ts)
│   ├── MessageRouter -- routes incoming messages to workspaces
│   ├── WorkspaceQueue -- per-workspace concurrency control
│   ├── TaskScheduler -- cron/interval/once task execution
│   └── IPCWatcher -- filesystem-based container communication
├── OpenCodeRuntime (src/runtime/)
│   ├── invoke(workspaceId, prompt, sessionId) -- abstracts run/serve
│   ├── configGenerator -- per-workspace opencode.json
│   └── outputParser -- parse OpenCode stdout JSON stream
├── MemoryService (src/memory/)
│   ├── factExtractor -- LLM-driven after each turn
│   ├── summarizer -- rolling summaries
│   └── adapters/ -- Mem0, Graphiti (optional)
├── PolicyEngine (src/security/)
│   ├── validateMount(hostPath, containerPath, readonly)
│   ├── validateCommand(cmd, workspaceId)
│   └── validateEgress(host, port, workspaceId)
├── DatabaseAdapter (src/db/)
│   ├── SQLiteAdapter (default)
│   └── PostgresAdapter (optional)
└── Channels (src/channels/)
    ├── WebhookChannel (built-in)
    ├── CLIChannel (built-in)
    └── [TelegramChannel, SlackChannel, ...] (via skills)
```

### 2.2 Database Schema

**Existing tables (from NanoClaw, renamed group -> workspace):**

- `chats` (jid TEXT PK, name TEXT, channel TEXT, workspace_folder TEXT, is_main BOOLEAN)
- `messages` (id INTEGER PK, chat_jid TEXT, sender TEXT, content TEXT, timestamp TEXT, message_id TEXT)
- `registered_workspaces` (jid TEXT PK, name TEXT, folder TEXT, trigger TEXT, channel TEXT, is_main BOOLEAN, container_config TEXT JSON, added_at TEXT)
- `sessions` (workspace_folder TEXT PK, session_id TEXT)
- `scheduled_tasks` (id TEXT PK, workspace_folder TEXT, prompt TEXT, schedule_type TEXT, schedule_value TEXT, status TEXT, next_run TEXT, created_at TEXT, max_calls INTEGER, call_count INTEGER)
- `task_run_logs` (id INTEGER PK, task_id TEXT, started_at TEXT, finished_at TEXT, duration_ms INTEGER, result TEXT, error TEXT)
- `router_state` (chat_jid TEXT PK, last_agent_timestamp TEXT, last_processed_message_id TEXT)

**New tables:**

- `memory_facts` (id TEXT PK, workspace_folder TEXT, sender TEXT, content TEXT, category TEXT, confidence REAL, embedding_id TEXT NULL, created_at TEXT, expires_at TEXT NULL)
- `memory_summaries` (id TEXT PK, workspace_folder TEXT, period_start TEXT, period_end TEXT, content TEXT, message_count INTEGER, created_at TEXT)
- `token_usage` (id INTEGER PK, workspace_folder TEXT, provider TEXT, model TEXT, input_tokens INTEGER, output_tokens INTEGER, cost_usd REAL, timestamp TEXT)
- `identity_links` (id INTEGER PK, user_id TEXT, channel TEXT, channel_user_id TEXT, display_name TEXT, linked_at TEXT, UNIQUE(channel, channel_user_id))

### 2.3 HTTP Webhook API

Base URL: `http://localhost:8080/api/v1`

**Endpoints:**

- `POST /messages` -- send message to workspace
  - Body: `{ workspaceId: string, content: string, sender?: string }`
  - Response: `{ messageId: string, status: "queued" }`
- `GET /messages/:workspaceId` -- get responses (polling)
  - Query: `?since=ISO_TIMESTAMP&limit=50`
  - Response: `{ messages: Message[] }`
- `GET /messages/:workspaceId/stream` -- SSE stream of responses
  - Returns: `text/event-stream` with `data: { type, content, ... }`
- `GET /workspaces` -- list registered workspaces
- `POST /workspaces` -- register workspace
  - Body: `{ name, folder, trigger?, channel?, containerConfig? }`
- `DELETE /workspaces/:id` -- remove workspace
- `GET /tasks` -- list tasks (optionally filtered by workspace)
- `POST /tasks` -- create scheduled task
- `PATCH /tasks/:id` -- update task (pause/resume/cancel/modify)
- `GET /tasks/:id` -- get task details + run history
- `GET /memory/:workspaceId/search?q=...` -- search memory facts
- `GET /usage` -- token usage stats
- `GET /health` -- service health

Auth: `Authorization: Bearer <API_KEY>` (configured in config.toml)

### 2.4 OpenCode Runtime Protocol

**Cold invoke (default):**

```
Host spawns:
  docker run --rm -i \
    -v groups/{workspace}:/workspace/current \
    -v data/sessions/{workspace}:/home/node/.opencode/ \
    [-v additional mounts] \
    -e OPENCODE_PROVIDER=... \
    -e OPENCODE_MODEL=... \
    manaclaw-agent:latest \
    opencode run --attach --session {sessionId} \
    < prompt_stdin > output_stdout
```

**Warm invoke (keep-alive, activated after 3+ messages in idle window):**

```
Host sends HTTP POST to running container:
  POST http://container:{OPENCODE_PORT}/api/send
  Body: { content: "user message", session: "{sessionId}" }
  Response: SSE stream of agent events
```

**Per-workspace `opencode.json` generation:**

```json
{
  "provider": { "name": "ollama", "baseUrl": "http://host.docker.internal:11434" },
  "model": "qwen2.5:14b",
  "permissions": { "edit": "auto", "bash": "auto", "webfetch": "auto" },
  "mcpServers": {
    "manaclaw": { "type": "stdio", "command": "node", "args": ["/workspace/mcp/manaclaw-mcp.js"] }
  },
  "agents": {
    "planner": { "description": "Plan complex tasks", "model": "inherit" }
  }
}
```

### 2.5 Config Schema (`config.toml`)

```toml
[general]
assistant_name = "ManaClaw"
trigger_pattern = "@ManaClaw"
poll_interval_ms = 2000
scheduler_poll_interval_ms = 60000
max_concurrent_containers = 5
container_timeout_ms = 1800000
idle_timeout_ms = 300000

[server]
host = "0.0.0.0"
port = 8080
api_key = ""             # empty = no auth required

[database]
provider = "sqlite"      # "sqlite" | "postgres"

[database.sqlite]
path = "store/manaclaw.db"

[database.postgres]
host = "localhost"
port = 5432
database = "manaclaw"
user = "manaclaw"
password = ""

[container]
image = "manaclaw-agent:latest"
runtime = "docker"       # "docker" | "podman"

[provider]
name = "ollama"          # "ollama" | "vllm" | "openai" | "openrouter" | custom
base_url = "http://localhost:11434"
api_key = ""
model = "qwen2.5:14b"
fallback_model = ""

[memory]
mode = "lite"            # "off" | "lite" | "full"
extraction_model = ""    # model for fact extraction (defaults to provider.model)
embedding_provider = ""  # "ollama" | "openai" | custom; empty = skip embeddings

[memory.mem0]
enabled = false
api_key = ""
base_url = ""

[security]
mount_allowlist_path = "~/.config/manaclaw/mount-allowlist.json"
blocked_patterns = [".ssh", ".gnupg", ".aws", ".env", "credentials", "private_key"]
```

### 2.6 Container Image Design

`container/Dockerfile`:

```
FROM node:22-slim
# Install OpenCode CLI
RUN npm install -g opencode
# Create workspace dirs
RUN mkdir -p /workspace/current /workspace/mcp /home/node/.opencode
# Copy ManaClaw MCP server (for scheduler/messaging IPC)
COPY mcp/ /workspace/mcp/
USER node
WORKDIR /workspace/current
ENTRYPOINT ["opencode"]
```

### 2.7 Memory Service Design

Three modes, configurable per-instance:

**Off**: No memory processing. Markdown files in workspace folder only. Simplest.

**Lite**: BM25 keyword search over SQLite FTS5.

- After each agent turn: extract facts via simple heuristics (no LLM call)
- Store in `memory_facts` table with FTS5 index
- On new message: query FTS5 for relevant facts, inject into system prompt
- Rolling summaries via LLM every N messages

**Full**: Embedding-based semantic search.

- After each agent turn: extract facts via LLM call to extraction_model
- Generate embeddings via embedding_provider
- Store facts + embeddings in `memory_facts`
- On new message: embed query, cosine similarity search, inject top-K into prompt
- Memory compaction job: merge similar facts, expire stale ones

### 2.8 Security Model

Layers (from outermost to innermost):

1. **API Authentication**: Bearer token on HTTP Webhook
2. **Workspace isolation**: separate container, folder, session per workspace
3. **Mount security**: allowlist validated before container spawn, blocked patterns, symlink resolution
4. **PolicyEngine**: per-workspace rules for commands, mounts, network egress
5. **Credential manager**: real credentials never enter containers; injected via opencode.json env or proxy
6. **OpenCode permissions**: `{ "edit": "auto", "bash": "auto" }` inside container
7. **Container boundary**: Docker/Podman isolation, non-root user, ephemeral containers

---

## Part 3: Detailed Step-by-Step Tasklist

### Phase 0: Repository and Documentation Bootstrap

**0.1 Repository setup**

- Clone empty repo `https://github.com/Manaszz/manaclaw` into `D:\codding\ai\manaclaw`
- Download NanoClaw reference: `git clone --depth 1 https://github.com/Manaszz/nanoclaw.git reference/nanoclaw`
- Create `.gitignore` (reference/, node_modules/, dist/, store/, data/, logs/, *.db, .env, config.toml)
- Create `package.json` with dependencies (better-sqlite3, fastify, pino, cron-parser, yaml, zod, typescript, tsx, vitest, eslint, prettier)
- Create `tsconfig.json`
- Create `config.example.toml`
- Create `LICENSE` (MIT)

**0.2 Documentation**

- Write `README.md` -- vision, quick start, architecture diagram, features
- Write `docs/PRD.md` -- full PRD (based on this plan)
- Write `docs/SPEC.md` -- full technical spec
- Write `docs/REQUIREMENTS.md` -- philosophy, product boundaries
- Write `docs/SECURITY.md` -- trust model, container isolation, PolicyEngine
- Write `docs/API.md` -- HTTP Webhook API reference
- Write `docs/OPENCODE_INTEGRATION.md` -- OpenCode modes, config, sessions
- Write `docs/MEMORY.md` -- layered memory system
- Write `docs/DEPLOYMENT.md` -- Docker, compose, config, production
- Write `docs/SKILLS.md` -- skill model for ManaClaw
- Write `docs/DEBUG_CHECKLIST.md` -- diagnostics and troubleshooting
- Write `docs/DECISIONS/opencode-runtime.md`
- Write `docs/DECISIONS/db-abstraction.md`
- Write `docs/DECISIONS/channels.md`
- Write `docs/DECISIONS/provider-abstraction.md`
- Write `docs/DECISIONS/memory-strategy.md`
- Write `docs/DECISIONS/workspace-terminology.md`

**0.3 Initial commit**

- Stage all files, commit "Phase 0: repository and documentation bootstrap"
- Push to GitHub

### Phase 1: Core Skeleton + DB + Channels

**1.1 Types and config**

- Create `src/types.ts` -- adapt NanoClaw types.ts, rename group -> workspace (Channel, Message, RegisteredWorkspace, ScheduledTask, WorkspaceConfig, RouterState, MemoryFact, MemorySummary, TokenUsageRecord, IdentityLink)
- Create `src/config.ts` -- TOML config loader using `@iarna/toml` or `smol-toml`, typed config interface, env var overrides
- Create `src/logger.ts` -- pino logger (from NanoClaw)

**1.2 Database layer**

- Create `src/db/adapter.ts` -- DatabaseAdapter interface (all methods)
- Create `src/db/sqlite-adapter.ts` -- implement all methods using better-sqlite3
  - Schema creation with all tables (existing + new)
  - Migration logic (version table, incremental migrations)
  - FTS5 index for memory_facts (Lite mode)
- Create `src/db/index.ts` -- factory function, reads config, returns adapter
- Write tests: `src/db/__tests__/sqlite-adapter.test.ts` -- CRUD for all entities

**1.3 Core modules (from NanoClaw, adapted)**

- Create `src/router.ts` -- adapt NanoClaw router.ts, rename group -> workspace
- Create `src/workspace-queue.ts` -- adapt NanoClaw group-queue.ts
- Create `src/task-scheduler.ts` -- adapt NanoClaw task-scheduler.ts
- Create `src/ipc.ts` -- adapt NanoClaw ipc.ts, rename group -> workspace
- Create `src/sender-allowlist.ts` -- from NanoClaw
- Create `src/workspace-folder.ts` -- adapt NanoClaw group-folder.ts
- Create `src/channels/registry.ts` -- from NanoClaw channels/registry.ts

**1.4 HTTP Webhook channel**

- Create `src/channels/webhook.ts` -- Fastify server, all API endpoints
  - POST /api/v1/messages
  - GET /api/v1/messages/:workspaceId
  - GET /api/v1/messages/:workspaceId/stream (SSE)
  - CRUD /api/v1/workspaces
  - CRUD /api/v1/tasks
  - GET /api/v1/health
  - Bearer token auth middleware
- Write tests: `src/channels/__tests__/webhook.test.ts`

**1.5 CLI channel**

- Create `bin/manaclaw.ts` -- CLI entry using commander or yargs
  - `manaclaw chat [workspace]` -- interactive chat (readline + HTTP client)
  - `manaclaw send <workspaceId> "message"` -- one-shot
  - `manaclaw workspaces list/add/remove`
  - `manaclaw tasks list/create/pause/resume/cancel`
  - `manaclaw status` -- health check
  - `manaclaw memory search <query>`
- Create `src/channels/cli.ts` -- CLIChannel implements Channel interface

**1.6 Orchestrator**

- Create `src/index.ts` -- main orchestrator
  - Initialize config, logger, database
  - Start HTTP server (Webhook channel)
  - Load registered workspaces from DB
  - Connect all available channels
  - Start scheduler loop
  - Start IPC watcher
  - Start message polling loop
  - Agent invocation stub (returns placeholder until Phase 2)
  - Graceful shutdown (SIGTERM/SIGINT)
- Create `src/channels/index.ts` -- barrel imports

**1.7 Integration test**

- Write `tests/integration/message-flow.test.ts` -- send message via HTTP, verify it's stored in DB, verify stub response comes back

**1.8 Commit**

- Commit "Phase 1: core skeleton with DB abstraction and HTTP/CLI channels"

### Phase 2: OpenCode Runtime Integration

**2.1 OpenCode config generator**

- Create `src/runtime/opencode-config.ts`
  - generateConfig(workspace, providerConfig) -> opencode.json content
  - Per-workspace provider overrides
  - MCP server config for ManaClaw IPC tools
  - Agent definitions (default + workspace-specific)
- Write tests

**2.2 Output parser**

- Create `src/runtime/output-parser.ts`
  - Parse OpenCode stdout JSON-lines stream
  - Extract: assistant text, tool calls, tool results, errors, session info, token usage
  - Map to ManaClaw internal message types
- Write tests with sample OpenCode output fixtures

**2.3 OpenCode runtime**

- Create `src/runtime/opencode-runtime.ts`
  - `invoke(workspaceId, prompt, sessionId): AsyncGenerator<RuntimeEvent>`
  - Cold mode: spawn `docker run ... opencode run --attach --session`
  - Warm mode: HTTP POST to running container's opencode serve
  - Per-workspace state tracking (cold/warm/draining)
  - Transition logic (3+ messages in idle window -> warm)
  - Timeout handling (container timeout, idle timeout)
  - Session ID extraction and DB mirroring
  - Token usage extraction and DB recording
- Write tests with mocked Docker

**2.4 Credential manager**

- Create `src/security/credential-manager.ts`
  - Build env vars for container from config.toml provider section
  - Never expose raw API keys in container env; use opencode.json or proxy
  - Support multiple providers per workspace
- Write tests

**2.5 PolicyEngine**

- Create `src/security/policy-engine.ts`
  - validateMount(hostPath, containerPath, readonly) -- allowlist check, blocked patterns, symlink resolution
  - validateCommand(cmd, workspaceId) -- optional command blocklist
  - validateEgress(host, port, workspaceId) -- optional network policy
  - Default policy: read-only project mount, writeable workspace, no mounts to sensitive dirs
- Create `src/security/mount-security.ts` -- adapt from NanoClaw
- Write tests

**2.6 Container image**

- Create `container/Dockerfile` -- Node 22 slim + OpenCode CLI + ManaClaw MCP server
- Create `container/mcp/manaclaw-mcp.js` -- MCP server for IPC tools (schedule_task, send_message, list_tasks, etc.)
- Create `container/build.sh` -- build script

**2.7 Wire it all together**

- Update `src/index.ts` -- replace agent stub with OpenCodeRuntime.invoke()
- Update message flow: prompt construction, runtime invocation, response parsing, memory hooks
- Update task scheduler: spawn OpenCode via runtime for scheduled tasks

**2.8 End-to-end test**

- Write `tests/integration/e2e-opencode.test.ts` -- message -> HTTP -> orchestrator -> container -> OpenCode -> response
- Manual test with Ollama locally

**2.9 Commit**

- Commit "Phase 2: OpenCode runtime integration with PolicyEngine"

### Phase 3: Memory Service

**3.1 Memory service core**

- Create `src/memory/memory-service.ts`
  - Constructor takes mode (off/lite/full) and DB adapter
  - `onTurnComplete(workspaceId, messages)` -- hook called after each agent turn
  - `getRelevantContext(workspaceId, query, limit)` -- returns facts/summaries for prompt injection
  - `compact(workspaceId)` -- merge redundant facts
  - `rebuild(workspaceId)` -- rebuild from raw messages
  - Mode switching at runtime

**3.2 Fact extractor**

- Create `src/memory/fact-extractor.ts`
  - Lite mode: regex/heuristic extraction (names, dates, preferences, action items)
  - Full mode: LLM call with extraction prompt, structured output
  - Per-sender fact attribution
  - Confidence scoring
  - Deduplication against existing facts

**3.3 Summarizer**

- Create `src/memory/summarizer.ts`
  - Rolling summary: every N messages or T hours, summarize conversation
  - Store in memory_summaries table
  - Incremental: new summary builds on previous
  - Per-workspace and optionally per-sender

**3.4 Memory adapters**

- Create `src/memory/adapters/mem0-adapter.ts` -- Mem0 API client (stub, real implementation when Mem0 is configured)
- Create `src/memory/adapters/graphiti-adapter.ts` -- stub interface for future

**3.5 Integration**

- Wire memory service into orchestrator: call `onTurnComplete` after each agent response
- Wire memory service into prompt construction: call `getRelevantContext` before agent invocation
- Add memory compaction to scheduled maintenance (daily or configurable)

**3.6 Tests**

- Write `src/memory/__tests__/memory-service.test.ts`
- Write `src/memory/__tests__/fact-extractor.test.ts`
- Write `src/memory/__tests__/summarizer.test.ts`

**3.7 Commit**

- Commit "Phase 3: layered memory service (Off/Lite/Full modes)"

### Phase 4: PostgreSQL + Identity + Token Tracking

**4.1 PostgreSQL adapter**

- Create `src/db/postgres-adapter.ts` -- implement DatabaseAdapter using `pg`
  - All same methods as SQLite adapter
  - Connection pooling
  - Schema creation and migrations (SQL files in `db/migrations/`)
- Write tests (testcontainers or docker PostgreSQL)

**4.2 Identity binding**

- Create `src/identity/identity-service.ts`
  - linkIdentity(userId, channel, channelUserId, displayName)
  - resolveIdentity(channel, channelUserId) -> unified userId
  - Auto-link on first message (create new userId if no match)
  - Manual link via API/CLI
- Add to Webhook API: `POST /api/v1/identities/link`, `GET /api/v1/identities/:userId`
- Wire into memory service: per-sender facts use unified userId

**4.3 Token usage tracking**

- Create `src/tracking/usage-tracker.ts`
  - recordUsage(workspaceId, provider, model, input, output, cost)
  - getUsage(filters: { workspace?, provider?, since?, until? })
  - Daily/weekly/monthly aggregation
- Add to Webhook API: `GET /api/v1/usage`, `GET /api/v1/usage/:workspaceId`
- Wire into OpenCode runtime: extract token counts from output, record

**4.4 Docker Compose**

- Create `docker-compose.yml` with profiles:
  - Core: manaclaw server
  - `postgres` profile: PostgreSQL container
  - `ollama` profile: Ollama container

**4.5 Tests and docs**

- Integration test: full flow with PostgreSQL
- Update `docs/DEPLOYMENT.md` with Docker Compose instructions
- Write `docs/DECISIONS/identity-binding.md`

**4.6 Commit**

- Commit "Phase 4: PostgreSQL adapter, identity binding, token tracking"

### Phase 5: Messaging Skills + Web UI v2

**5.1 Skill system**

- Create `skills/` directory structure (adapted from NanoClaw skills-as-branches)
- Adapt `/add-telegram` skill for ManaClaw (OpenCode agents instead of Claude skills)
- Adapt `/add-slack` skill
- Adapt `/add-whatsapp` skill
- Adapt `/add-gmail` skill
- Write `docs/SKILLS.md` updates with ManaClaw-specific instructions

**5.2 Web UI v2**

- Create `packages/web/` -- Vue 3 + Tailwind CSS + Vite
- Chat interface with streaming (SSE from Webhook API)
- Dashboard: workspaces list, active containers, task scheduler
- Configuration UI: providers, workspaces, mounts, policies
- Memory browser: search facts, view summaries, manage compaction
- Token usage charts (per workspace, per provider, over time)
- Dark/light theme
- Build and serve from Fastify static handler

**5.3 Deployment profiles**

- `config.local-dev.toml` -- SQLite, localhost Ollama, no auth
- `config.corp-onprem.toml` -- PostgreSQL, vLLM, API key auth, full memory
- `config.personal-secure.toml` -- SQLite, Ollama, full memory, strict policy

**5.4 Hardening**

- Audit logging (all tool calls, API requests, config changes)
- Container snapshot support (save/restore workspace containers)
- Graceful upgrade path (rolling restart without losing active sessions)
- Windows WSL2 setup documentation

**5.5 Final commit**

- Commit "Phase 5: messaging skills, Web UI v2, deployment profiles"

---

## File Creation Order (When Execution Starts)

Phase 0 first batch (project scaffolding):

1. `.gitignore`
2. `package.json`
3. `tsconfig.json`
4. `config.example.toml`
5. `LICENSE`
6. `README.md`

Phase 0 second batch (documentation):
7. `docs/PRD.md`
8. `docs/SPEC.md`
9. `docs/REQUIREMENTS.md`
10. `docs/SECURITY.md`
11. `docs/API.md`
12. `docs/OPENCODE_INTEGRATION.md`
13. `docs/MEMORY.md`
14. `docs/DEPLOYMENT.md`
15. `docs/SKILLS.md`
16. `docs/DEBUG_CHECKLIST.md`
17. `docs/DECISIONS/*.md` (6 ADR files)

Phase 1 (source code, in dependency order):
18. `src/types.ts`
19. `src/config.ts`
20. `src/logger.ts`
21. `src/db/adapter.ts`
22. `src/db/sqlite-adapter.ts`
23. `src/db/index.ts`
24. `src/channels/registry.ts`
25. `src/router.ts`
26. `src/workspace-queue.ts`
27. `src/workspace-folder.ts`
28. `src/sender-allowlist.ts`
29. `src/task-scheduler.ts`
30. `src/ipc.ts`
31. `src/channels/webhook.ts`
32. `src/channels/cli.ts`
33. `src/channels/index.ts`
34. `src/index.ts`
35. `bin/manaclaw.ts`