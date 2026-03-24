# Tech Context: ManaClaw

## Technologies Used

| Technology | Purpose | Version |
|-----------|---------|---------|
| Node.js | Host process runtime | 20+ |
| TypeScript | Primary language | 5.7+ |
| Fastify | HTTP server for Webhook API | 5.x |
| better-sqlite3 | SQLite adapter | 11.x |
| pg (planned) | PostgreSQL adapter | Phase 4 |
| OpenCode | Agent runtime inside containers | latest |
| Docker | Container isolation | 28.x |
| Pino | Structured logging | 9.x |
| Zod | Schema validation | 3.x |
| Vitest | Testing framework | 3.x |
| smol-toml | TOML config parser | 1.x |
| cron-parser | Cron expression parsing | 5.x |
| Commander | CLI argument parsing | 13.x |

## Development Setup

```bash
git clone https://github.com/Manaszz/manaclaw.git
cd manaclaw
cp config.example.toml config.toml
npm install
npm run build
npm start        # production
npm run dev      # development (tsx watch)
npm test         # vitest
```

**Prerequisites**: Node.js 20+, Docker, an LLM provider (Ollama recommended for local dev).

## Technical Constraints

- **Single process**: Host is one Node.js process (like NanoClaw). No microservices.
- **Windows primary dev**: Developer runs on Windows 10 (`win32 10.0.26200`), PowerShell. Docker via Docker Desktop.
- **Container runtime**: Docker required for workspace isolation. Podman support planned.
- **OpenCode dependency**: Agent execution depends on OpenCode CLI being installed in container image.
- **SQLite limitations**: FTS5 required for Lite memory mode. No concurrent write from multiple processes.

## Dependencies

### Production
- `better-sqlite3`, `fastify`, `@fastify/cors`, `pino`, `pino-pretty`
- `cron-parser`, `commander`, `smol-toml`, `yaml`, `zod`

### Development
- `typescript`, `tsx`, `vitest`, `eslint`, `prettier`
- `@types/better-sqlite3`, `@types/node`

### Future (not yet added)
- `pg` (PostgreSQL adapter, Phase 4)
- `openai` (optional, for provider fallback utilities)

## Tool Usage Patterns

- **Config**: TOML (`config.toml`) parsed by `smol-toml`, env var overrides
- **Logging**: Pino with structured JSON, `pino-pretty` for dev
- **Testing**: Vitest with globals, Node environment
- **Build**: `tsc` to `dist/`, `tsx` for dev
- **CLI**: Commander-based, calls Webhook API internally

## Environment Template

```env
# .env.example
OPENAI_API_KEY=
ANTHROPIC_API_KEY=
OPENROUTER_API_KEY=
OLLAMA_BASE_URL=http://localhost:11434
# MANACLAW_DB_PASSWORD=
# MANACLAW_API_KEY=
# MEM0_API_KEY=
```
