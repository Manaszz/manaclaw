# ManaClaw Skills

ManaClaw extends core behavior through **skills**: versioned, installable units that can add channels, automation, container packages, or operator workflows. This document describes the skill model, how ManaClaw differs from NanoClaw, built-in slash-style skills for messengers, and how to contribute.

---

## Skill Types

| Type | Description | Typical delivery |
|------|-------------|------------------|
| **Feature** | Adds or extends product behavior (often via a **git branch** or fork merge) | New `src/channels/*.ts`, router rules, config schema updates |
| **Utility** | Developer or operator helpers | Scripts, CLI docs, diagnostics under `bin/` or `docs/` |
| **Operational** | Runbooks, systemd/Compose fragments, monitoring | `docs/`, sample `deploy/` assets |
| **Container** | Changes inside the agent image | `container/Dockerfile`, MCP packages, OpenCode defaults |

Feature skills that touch the host **must** register channels or hooks using the same **self-registration** pattern as NanoClaw (see [SPEC.md](./SPEC.md#architecture-channel-system)).

---

## How ManaClaw Skills Differ from NanoClaw

| Aspect | NanoClaw | ManaClaw |
|--------|----------|----------|
| Agent runtime | Claude Code / Anthropic Agent SDK | **OpenCode** inside per-workspace containers |
| Project agent config | `.claude/` | **`.opencode/`** (`opencode.json`, agents, commands) |
| Skill metadata | Claude Code skills | **OpenCode-oriented** skills and project docs |
| Memory files | `CLAUDE.md` hierarchy | **`MEMORY.md`** (human layer) plus **MemoryService** layers in DB |
| Terminology | Group | **Workspace** |

When porting a NanoClaw skill:

1. Replace Anthropic-specific env handling with **OpenCode provider** configuration.  
2. Move skill instructions from `.claude/skills/` to **`.opencode/skills/`** (and related OpenCode directories).  
3. Rename user-facing “group” strings to **workspace**.  
4. Validate channel code against the ManaClaw `Channel` contract and **PolicyEngine** rules.

---

## Available Feature Skills (Messaging)

These skills add external channels via the registry pattern. Each is typically invoked as a **slash command** in the operator’s OpenCode or documented CLI workflow (for example `/add-telegram`):

| Skill | Purpose |
|-------|---------|
| `/add-telegram` | Telegram Bot API channel: register factory, webhook or long-poll setup per docs |
| `/add-slack` | Slack channel: app tokens, signing secret, event subscription |
| `/add-whatsapp` | WhatsApp channel: session auth under `store/auth/`, Baileys or equivalent stack |
| `/add-gmail` | Gmail integration: OAuth credentials, polling or push |

Each skill ships:

- Channel implementation under `src/channels/<name>.ts`  
- Barrel import line in `src/channels/index.ts`  
- Operator documentation for credentials and environment variables  
- Optional `.opencode/skills/add-<name>/` instructions for repeatable installation

---

## Contributing a Skill

1. **Open an issue or design note** describing the channel or feature, trust boundaries, and data retention.  
2. **Implement self-registration** in `src/channels/` (for channels) without breaking built-in HTTP/CLI.  
3. **Document configuration** in `docs/` (required keys, example `config.toml` fragments).  
4. **Add tests** where feasible (registry load, factory returns null without creds).  
5. **Respect PolicyEngine** (mounts, egress, blocked path patterns).  
6. **Avoid embedding secrets** in repository files; use env vars and `store/` for auth artifacts.

---

## Skill Directory Structure (illustrative)

```
.opencode/
├── skills/
│   ├── add-telegram/
│   │   └── SKILL.md          # Operator steps, env vars, troubleshooting
│   ├── add-slack/
│   │   └── SKILL.md
│   ├── add-whatsapp/
│   │   └── SKILL.md
│   └── add-gmail/
│       └── SKILL.md
├── agents/                     # Optional custom agents for OpenCode
└── commands/                   # Optional command definitions
```

Host-side code added by Feature skills usually lives under:

```
src/channels/<channel>.ts       # registerChannel(...)
src/channels/index.ts           # import './<channel>.js'
```

---

## See Also

- [SPEC.md](./SPEC.md) — architecture, message flow, MCP  
- [DEPLOYMENT.md](./DEPLOYMENT.md) — runtime and Compose  
- [DEBUG_CHECKLIST.md](./DEBUG_CHECKLIST.md) — channel and container diagnostics  
