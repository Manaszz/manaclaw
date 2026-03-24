# ADR: Built-in Channels

**Status:** Accepted

**Date:** 2026-03-24

## Context

NanoClaw ships without first-class channels; every integration path depends on skills. Basic product use therefore requires a separate messaging application or custom wiring before the system is usable.

## Decision

Ship **three built-in channels** in ManaClaw core:

1. **HTTP Webhook API** — implemented with Fastify for programmatic ingress and integrations.
2. **CLI client** — for development, scripting, and administration.
3. **Web UI** — user-facing interface (targeted for v2 where roadmap applies).

Third-party **messaging channels** (e.g. Telegram, Slack, WhatsApp, Gmail) remain **skill-based** extensions rather than core dependencies.

## Consequences

**Positive**

- Usable out of the box without installing messaging-specific skills first.
- Universal integration surface via HTTP for arbitrary upstream systems.
- CLI supports fast iteration and operational tasks without a browser.

**Negative**

- Larger core codebase and maintenance surface for channel stacks (HTTP, CLI, UI).

**Neutral**

- Skills continue to own vertical messaging integrations; core stays provider-agnostic for chat products.
