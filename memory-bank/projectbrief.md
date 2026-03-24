# Project Brief: ManaClaw

> **Changelog**
> - 2026-03-24: Initial creation. Phase 0 complete.

## Project Name
ManaClaw

## Repository
https://github.com/Manaszz/manaclaw

## One-Line Summary
Lightweight, provider-agnostic AI assistant with container isolation, pluggable memory, and built-in HTTP/CLI/Web channels.

## Core Requirements

1. **Provider Independence**: OpenAI-compatible model interface. Works with Ollama, vLLM, OpenAI, OpenRouter, any compatible endpoint. No Anthropic lock-in.
2. **Agent Runtime**: OpenCode as primary agent runtime inside Docker containers (replaces NanoClaw's Anthropic Agent SDK).
3. **Container Isolation**: Each workspace runs in its own Docker container with filesystem, process, and network isolation.
4. **Database Abstraction**: SQLite (default, zero-config) or PostgreSQL (optional, for enterprise).
5. **Built-in Channels**: HTTP Webhook API + CLI + Web UI (v2). Messaging channels (Telegram, Slack, WhatsApp, Gmail) via skills.
6. **Layered Memory**: Off / Lite (BM25/FTS5) / Full (embeddings) modes. Optional Mem0/Graphiti adapters.
7. **Workspace Model**: Each registered conversation context ("workspace", renamed from NanoClaw "group") has isolated folder, container, session, tasks, memory.
8. **Corporate + Personal**: Designed for both corporate environments and personal productivity.

## Goals
- Maintain NanoClaw's lightweight feel (~3-5K LoC host process)
- Stronger provider independence than NanoClaw
- Better memory extensibility than file-based CLAUDE.md
- Production-ready for corporate deployment (PostgreSQL, token tracking, audit)
- MIT license

## Non-Goals (v1)
- Multi-tenant SaaS
- Built-in model hosting
- Voice/video channels
- Mobile app

## Origin
Based on NanoClaw (https://github.com/qwibitai/nanoclaw) architecture. NanoClaw reference downloaded in `reference/nanoclaw/` (gitignored). Influenced by Memoh (multi-provider memory, identity binding, token tracking) and OpenClaw (model abstraction).
