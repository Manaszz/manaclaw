# Product Context: ManaClaw

## Why This Project Exists

NanoClaw provides excellent container isolation and a small codebase, but is locked to Anthropic's Claude via the Agent SDK. OpenClaw is model-agnostic but has ~500K lines of code, complex configuration, and application-level (not container-level) security. Neither fits the "lightweight + provider-agnostic + secure" sweet spot for corporate use.

ManaClaw fills this gap: NanoClaw's simplicity and isolation model + OpenClaw's provider independence + Memoh's memory engineering ideas, with OpenCode as the agent runtime.

## Problems It Solves

1. **Vendor lock-in**: Organizations want to use local models (Ollama, vLLM) or switch providers without rewriting their assistant infrastructure.
2. **Security**: AI agents executing code need real OS-level isolation, not just permission checks.
3. **Memory**: Simple file-based memory (CLAUDE.md) doesn't scale for long-running corporate assistants that need semantic recall.
4. **Integration**: Need an HTTP API to connect AI assistants to CI/CD, monitoring, workflow tools -- not just messaging apps.
5. **Database flexibility**: SQLite for personal use, PostgreSQL for enterprise -- same codebase.

## How It Should Work

- User sends a message via HTTP API, CLI, or messaging channel (Telegram, Slack, etc.)
- ManaClaw routes the message to the appropriate workspace
- A Docker container is spawned (or reused if warm) with OpenCode inside
- OpenCode processes the message using the configured LLM provider
- Response streams back through the same channel
- Memory service extracts facts, updates summaries, stores in DB
- Scheduled tasks run autonomously in workspace containers

## User Experience Goals

- **Zero-config local start**: `npm install && npm start` with SQLite + Ollama
- **CLI-first for developers**: `manaclaw chat main` for interactive use
- **HTTP-first for integrations**: REST API for everything
- **Understand the full codebase**: Small enough to read and modify
- **Secure by default**: Container isolation is not optional
