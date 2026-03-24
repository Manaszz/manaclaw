# ManaClaw

A lightweight, provider-agnostic AI assistant with container isolation, pluggable memory, and built-in HTTP/CLI/Web channels.

## What Is ManaClaw?

ManaClaw is an always-on AI assistant designed for corporate environments and personal productivity. Each registered conversation context (called a **workspace**) runs in its own isolated Docker container with its own memory, session, and scheduled tasks.

Based on [NanoClaw](https://github.com/qwibitai/nanoclaw) architecture, ManaClaw replaces the Anthropic-specific execution layer with [OpenCode](https://opencode.ai) as a provider-agnostic agent runtime, adds database abstraction (SQLite / PostgreSQL), built-in HTTP and CLI channels, and a layered memory system.

## Key Differences from NanoClaw

| Aspect | NanoClaw | ManaClaw |
|--------|----------|----------|
| Agent runtime | Anthropic Agent SDK (Claude Code) | OpenCode (75+ providers) |
| Model support | Anthropic only | OpenAI-compatible (Ollama, vLLM, OpenAI, OpenRouter, ...) |
| Database | SQLite only | SQLite (default) or PostgreSQL |
| Built-in channels | None (all via skills) | HTTP Webhook API + CLI + Web UI (v2) |
| Memory | CLAUDE.md files only | Layered: Off / Lite (BM25) / Full (embeddings) + Mem0/Graphiti adapters |
| Terminology | "group" | "workspace" |

## Quick Start

```bash
git clone https://github.com/Manaszz/manaclaw.git
cd manaclaw
cp config.example.toml config.toml
# Edit config.toml -- set your provider (Ollama by default)
npm install
npm run build
npm start
```

ManaClaw starts an HTTP server on port 8080. Send a message:

```bash
curl -X POST http://localhost:8080/api/v1/messages \
  -H "Content-Type: application/json" \
  -d '{"workspaceId": "main", "content": "Hello, ManaClaw!"}'
```

Or use the CLI:

```bash
npx manaclaw chat main
```

## Architecture

```
Host Process (Node.js)
├── HTTP Server (Fastify)          -- Webhook API + Web UI
├── CLI Channel                    -- Interactive terminal client
├── Orchestrator
│   ├── MessageRouter              -- Routes messages to workspaces
│   ├── WorkspaceQueue             -- Per-workspace concurrency
│   ├── TaskScheduler              -- Cron / interval / once
│   └── IPCWatcher                 -- Container communication
├── OpenCode Runtime               -- Hybrid run/serve agent execution
├── Memory Service                 -- Off / Lite / Full modes
├── PolicyEngine                   -- Mount / command / egress policies
└── Database Adapter               -- SQLite or PostgreSQL
```

Each workspace runs in a Docker container with OpenCode inside:

```
Docker Container (per workspace)
├── OpenCode agent runtime
├── ManaClaw MCP server (IPC tools)
├── Mounted workspace folder (read-write)
└── Mounted project root (read-only)
```

## Configuration

See [config.example.toml](config.example.toml) for all options. Key sections:

- `[general]` -- assistant name, trigger pattern, timeouts
- `[server]` -- HTTP host, port, API key
- `[database]` -- SQLite or PostgreSQL
- `[provider]` -- LLM provider (Ollama, vLLM, OpenAI, etc.)
- `[memory]` -- Off / Lite / Full mode
- `[security]` -- mount allowlist, blocked patterns

## Documentation

- [PRD](docs/PRD.md) -- Product requirements
- [SPEC](docs/SPEC.md) -- Technical specification
- [REQUIREMENTS](docs/REQUIREMENTS.md) -- Philosophy and architecture decisions
- [API](docs/API.md) -- HTTP Webhook API reference
- [SECURITY](docs/SECURITY.md) -- Security model
- [OPENCODE_INTEGRATION](docs/OPENCODE_INTEGRATION.md) -- OpenCode runtime integration
- [MEMORY](docs/MEMORY.md) -- Layered memory system
- [DEPLOYMENT](docs/DEPLOYMENT.md) -- Deployment guide
- [SKILLS](docs/SKILLS.md) -- Skill system
- [DEBUG_CHECKLIST](docs/DEBUG_CHECKLIST.md) -- Troubleshooting
- [Architecture Decisions](docs/DECISIONS/) -- ADR directory

## Skills

ManaClaw supports messaging channels via skills (code modifications to your fork):

- `/add-telegram` -- Telegram bot channel
- `/add-slack` -- Slack workspace channel
- `/add-whatsapp` -- WhatsApp channel
- `/add-gmail` -- Gmail channel

See [docs/SKILLS.md](docs/SKILLS.md) for details.

## Requirements

- Node.js 20+
- Docker (for container isolation)
- An LLM provider (Ollama recommended for local, any OpenAI-compatible API)

## License

MIT
