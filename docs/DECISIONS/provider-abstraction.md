# ADR: OpenAI-Compatible Provider Contract

**Status:** Accepted

**Date:** 2026-03-24

## Context

NanoClaw is Anthropic-centric. ManaClaw must interoperate with Ollama, vLLM, OpenAI, OpenRouter, and other hosts without bespoke client code per vendor.

## Decision

Standardize on the **OpenAI-compatible API** (chat/completions style) as the **transport contract** between ManaClaw’s orchestration layer and model backends. **OpenCode** performs provider routing and compatibility handling. **Per-workspace model configuration** lives in `opencode.json` (or the project’s canonical OpenCode config), so tenants can pin models and endpoints per workspace.

## Consequences

**Positive**

- Broad compatibility with a large set of providers and gateways (on the order of 75+ in the OpenCode ecosystem).
- First-class support for local inference stacks (Ollama, vLLM) alongside cloud APIs.

**Negative**

- Vendor-specific capabilities outside the OpenAI-compatible surface may be unavailable or lossy.
- Tool-calling behavior and reliability vary by model; quality is not uniform across providers.

**Neutral**

- ManaClaw focuses on contract and policy; provider quirks are concentrated in OpenCode and configuration rather than scattered adapters in application code.
