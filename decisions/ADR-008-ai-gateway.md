# ADR-008: ai-gateway as the AI abstraction layer

**Date:** 2026-04-25
**Status:** Accepted

## Context

MindBlossom v2 requires AI-generated responses to inbound SMS and email. Future products will likely use AI differently (summarization, classification, generation). Calling the Claude API or OpenAI API directly from product code would:
- Couple products to specific AI providers
- Duplicate conversation state management, rate limiting, and retry logic
- Make it hard to track AI costs and usage across products

## Decision

A dedicated `ai-gateway` service that abstracts AI provider access. Products call `ai-gateway`; `ai-gateway` calls the underlying provider.

`ai-gateway` is built on **LiteLLM**, an open-source proxy that provides a unified OpenAI-compatible API across providers (Claude, GPT-4, Gemini, etc.). This means:
- Switching from Claude to GPT-4 is a `ai-gateway` config change
- Products use a single, stable API regardless of what's behind it

**What `ai-gateway` manages:**
- Conversation threading (maintains history keyed by `conversation_id`)
- Tool definitions for operating on user data (passed through to the provider)
- Per-caller rate limiting (prevents one product from exhausting the API budget)
- Cost and usage tracking per caller and per model

## Consequences

**Easier:**
- Provider switches require no product code changes
- Rate limiting and cost visibility are centralized
- New products get AI capabilities by calling one internal API
- Conversation state is managed in one place, not per-product

**Harder:**
- An additional network hop for AI calls (internal, acceptable latency)
- `ai-gateway` must be highly available — it is on the critical path for AI response features
- Tool definitions are passed by the caller — requires trust that callers don't pass malicious tools
- LiteLLM adds an additional container to the service

**Tool use note:**
Products define the tools available to the AI (e.g., `search_links`, `tag_message`) and pass them with each request. `ai-gateway` does not define product-specific tools — it only proxies them to the provider.
