# Progress: ManaClaw

> **Changelog Reference**: See git log for commit history.

## What Works

- Repository initialized and pushed to GitHub (2 commits)
- Full documentation set: 16 docs covering PRD, spec, security, API, memory, deployment, skills, debug, 6 ADRs
- Project scaffolding: package.json, tsconfig.json, config.example.toml, vitest.config.ts, .env.example
- Architecture plan and implementation plan finalized in `.cursor/plans/`
- NanoClaw reference downloaded for code adaptation
- Memory Bank initialized

## What's Left to Build

### Phase 1: Core Skeleton + DB + Channels (NEXT)
- [ ] Types and config (types.ts, config.ts, logger.ts)
- [ ] DatabaseAdapter interface + SQLiteAdapter + tests
- [ ] Core modules from NanoClaw (router, workspace-queue, task-scheduler, ipc, etc.)
- [ ] HTTP Webhook channel (Fastify) + tests
- [ ] CLI channel + tests
- [ ] Orchestrator + integration test

### Phase 2: OpenCode Runtime
- [ ] OpenCode config generator
- [ ] Output parser
- [ ] OpenCode runtime (hybrid run/serve)
- [ ] Credential manager
- [ ] PolicyEngine
- [ ] Container image (Dockerfile)
- [ ] End-to-end test

### Phase 3: Memory Service
- [ ] Memory service core (Off/Lite/Full modes)
- [ ] Fact extractor (heuristic + LLM)
- [ ] Summarizer
- [ ] Mem0 adapter stub

### Phase 4: PostgreSQL + Identity + Tracking
- [ ] PostgreSQL adapter
- [ ] Cross-channel identity binding
- [ ] Token usage tracking
- [ ] Docker Compose

### Phase 5: Skills + Web UI v2
- [ ] Messaging channel skills (Telegram, Slack, WhatsApp, Gmail)
- [ ] Web UI v2 (Vue 3 SPA)
- [ ] Deployment profiles
- [ ] Audit logging
- [ ] Container snapshots

## Current Status

| Phase | Status | Commit |
|-------|--------|--------|
| Phase 0: Repo + Docs | COMPLETE | `30586d0`, `3e81966` |
| Phase 1: Skeleton + DB + Channels | NOT STARTED | -- |
| Phase 2: OpenCode Runtime | NOT STARTED | -- |
| Phase 3: Memory Service | NOT STARTED | -- |
| Phase 4: PostgreSQL + Identity | NOT STARTED | -- |
| Phase 5: Skills + Web UI | NOT STARTED | -- |

## Known Issues

- None yet (no code to have bugs in).

## Evolution of Project Decisions

1. **Initial concept** (2026-03-23): Fork NanoClaw, replace Anthropic with OpenAI-compatible.
2. **OpenClaw study** (2026-03-23): Studied OpenClaw architecture. Decided NOT to copy its gateway model -- too complex.
3. **OpenCode decision** (2026-03-23): User proposed OpenCode as agent runtime. Confirmed as primary runtime inside sandbox.
4. **Memoh study** (2026-03-24): Studied Memoh. Adopted: graduated memory modes, per-user memory, identity binding, token tracking, container snapshots concept.
5. **DB abstraction** (2026-03-24): User requested SQLite + PostgreSQL support. Designed DatabaseAdapter interface.
6. **Built-in channels** (2026-03-24): User requested HTTP Webhook, CLI, Web UI. Added as built-in (not skill-based).
7. **Workspace terminology** (2026-03-24): Renamed "group" to "workspace" per user choice.
8. **Full scope PRD** (2026-03-24): User chose full 6-phase scope for PRD/spec, not just MVP.
