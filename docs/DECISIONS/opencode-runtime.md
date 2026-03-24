# ADR: OpenCode as Agent Runtime

**Status:** Accepted

**Date:** 2026-03-24

## Context

NanoClaw relies on the Anthropic Agent SDK (Claude Code CLI), which is tied to the Anthropic Messages API. ManaClaw must support multiple model providers and avoid hard vendor coupling at the agent runtime layer.

## Decision

Adopt **OpenCode** as the primary agent runtime inside containers. Operate in a hybrid model:

- **Cold path (default):** `opencode run --attach` for on-demand execution without a long-lived process.
- **Warm path:** `opencode serve` for keep-alive sessions where active workspaces benefit from lower startup latency.

## Consequences

**Positive**

- Provider independence through OpenCode’s broad provider ecosystem (75+ providers).
- Reuse of existing MCP, session, and agent capabilities aligned with OpenCode’s model.

**Negative**

- Loss of NanoClaw’s hook system; mitigated by a **PolicyEngine** plus structured output parsing for governance and automation.
- Dependency on the OpenCode project’s roadmap, release cadence, and long-term stability.

**Neutral**

- Operational complexity splits between cold and warm modes; operators must choose the appropriate mode per deployment or workspace policy.
