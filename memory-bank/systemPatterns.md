# System Patterns: ManaClaw

## Architecture Overview

```
Host Process (Node.js, single process)
├── HTTP Server (Fastify)
│   ├── Webhook API (/api/v1/*)
│   └── Web UI static files (v2)
├── Orchestrator (src/index.ts)
│   ├── MessageRouter
│   ├── WorkspaceQueue (per-workspace concurrency)
│   ├── TaskScheduler (cron/interval/once)
│   └── IPCWatcher (filesystem-based)
├── OpenCodeRuntime (src/runtime/)
│   ├── Hybrid run/serve mode
│   ├── Per-workspace opencode.json generation
│   └── Output stream parsing
├── MemoryService (src/memory/)
│   ├── Off / Lite (FTS5) / Full (embeddings)
│   └── Mem0/Graphiti adapters
├── PolicyEngine (src/security/)
│   └── Mount / command / egress validation
├── DatabaseAdapter (src/db/)
│   ├── SQLiteAdapter (default)
│   └── PostgresAdapter (optional)
└── Channels (src/channels/)
    ├── WebhookChannel (built-in)
    ├── CLIChannel (built-in)
    └── [Telegram, Slack, ...] (via skills)
```

## Key Technical Decisions

| Decision | Choice | ADR |
|----------|--------|-----|
| Agent runtime | OpenCode (not Claude Agent SDK) | docs/DECISIONS/opencode-runtime.md |
| DB layer | Abstract adapter: SQLite / PostgreSQL | docs/DECISIONS/db-abstraction.md |
| Built-in channels | HTTP Webhook + CLI + Web UI (v2) | docs/DECISIONS/channels.md |
| Model contract | OpenAI-compatible via OpenCode | docs/DECISIONS/provider-abstraction.md |
| Memory | Layered Off/Lite/Full + adapters | docs/DECISIONS/memory-strategy.md |
| Terminology | "workspace" (not "group") | docs/DECISIONS/workspace-terminology.md |

## Design Patterns In Use

- **Self-registration**: Channels register via factory pattern at startup (from NanoClaw)
- **Adapter pattern**: DatabaseAdapter interface with SQLite/PostgreSQL implementations
- **Layered memory**: L1 raw messages -> L2 summaries -> L3 facts -> L4 embeddings -> L5 external
- **Hybrid runtime**: Cold `opencode run` (default) -> Warm `opencode serve` (keep-alive) per workspace
- **Policy enforcement**: Host-side PolicyEngine validates before container spawn
- **IPC via filesystem**: JSON files in `data/ipc/` for container-host communication

## Component Relationships

- Orchestrator -> WorkspaceQueue -> OpenCodeRuntime -> Docker Container
- All channels -> MessageRouter -> DatabaseAdapter (store) -> Orchestrator (poll)
- OpenCodeRuntime -> CredentialManager (inject secrets) + PolicyEngine (validate)
- MemoryService hooks into Orchestrator: pre-invoke (getRelevantContext) + post-invoke (onTurnComplete)

## Critical Implementation Paths

1. **Message flow**: Channel -> DB -> Poll -> Router -> Queue -> Runtime -> Container -> OpenCode -> Response -> Channel
2. **Session continuity**: OpenCode `--session` flag -> session ID extracted from output -> mirrored in DB sessions table
3. **Warm transition**: 3+ messages in idle window -> spawn `opencode serve` inside container -> HTTP POST for subsequent messages
