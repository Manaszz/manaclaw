# Active Context: ManaClaw

## Current Work Focus

**Phase 0 COMPLETE.** Architecture plan and implementation plan finalized. Repository initialized with full documentation. Ready to begin Phase 1: Core Skeleton + DB + Channels.

## Recent Changes

- 2026-03-24: Initialized git repo, pushed to https://github.com/Manaszz/manaclaw
- 2026-03-24: Created 16 documentation files (PRD, SPEC, REQUIREMENTS, SECURITY, API, MEMORY, OPENCODE_INTEGRATION, DEPLOYMENT, SKILLS, DEBUG_CHECKLIST, 6 ADRs)
- 2026-03-24: Created project scaffolding (package.json, tsconfig.json, config.example.toml, vitest.config.ts, .env.example, LICENSE)
- 2026-03-24: Pushed architecture and implementation plans to GitHub
- 2026-03-24: NanoClaw reference cloned to `reference/nanoclaw/`

## Next Steps

### Phase 1: Core Skeleton + DB + Channels
1. `src/types.ts` -- all TypeScript interfaces (workspace terminology)
2. `src/config.ts` -- TOML config loader with typed interface
3. `src/logger.ts` -- Pino logger
4. `src/db/adapter.ts` -- DatabaseAdapter interface
5. `src/db/sqlite-adapter.ts` -- Full SQLite implementation with FTS5
6. Core modules adapted from NanoClaw: router, workspace-queue, task-scheduler, ipc, sender-allowlist, workspace-folder
7. `src/channels/webhook.ts` -- Fastify HTTP server with all API endpoints
8. `src/channels/cli.ts` -- CLI channel
9. `src/index.ts` -- Orchestrator (with agent runtime stub)
10. `bin/manaclaw.ts` -- CLI entry point

## Active Decisions and Considerations

- **OpenCode runtime mode**: Hybrid run/serve decided. Cold `opencode run --attach` default, warm `opencode serve` for active workspaces.
- **DB abstraction**: Interface-based. SQLite first (Phase 1), PostgreSQL in Phase 4.
- **Workspace terminology**: "group" renamed to "workspace" throughout.
- **Built-in channels**: HTTP Webhook + CLI in Phase 1. Web UI in Phase 5.

## Important Patterns and Preferences

- Developer uses Windows + PowerShell. Use PowerShell-compatible commands.
- Use `Write` tool for file creation, not heredoc/echo.
- Plans stored in `.cursor/plans/` and tracked in git.
- NanoClaw reference in `reference/nanoclaw/` -- use for adapting modules.
- #workspace-terminology -- "group" -> "workspace" everywhere in code and docs.
- #db-abstraction -- always code against `DatabaseAdapter` interface, never direct SQLite calls outside adapter.

## Learnings and Project Insights

- Claude Agent SDK cannot be used without Anthropic provider -- `cli.js` hardcodes Anthropic Messages API format. #sdk-limitation
- NanoClaw's hook system (12 events) has no direct OpenCode equivalent. Compensated by output parsing + PolicyEngine + IPC. #hook-gap
- Memoh's graduated memory modes (Off/Sparse/Dense) are an excellent pattern -- adopted as Off/Lite/Full. #memory-design
- Memoh's per-user memory in group contexts is valuable for corporate scenarios. #per-user-memory
