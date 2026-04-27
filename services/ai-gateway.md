# ai-gateway

**Repo:** `github.com/project-whirlwind/ai-gateway`
**Port:** 4002
**Status:** Planned
**ADR:** [ADR-008](../decisions/ADR-008-ai-gateway.md)
**API contract:** `api-contracts/ai-gateway/v1/openapi.yml` (to be created)

---

## Purpose

Provider-agnostic AI completion layer. Products call `ai-gateway`; it routes to Claude, GPT-4, or other providers via LiteLLM. Manages conversation threading, rate limits, and cost tracking.

## What it does

- `POST /v1/chat` — request a completion. Accepts messages array, optional tools, optional `conversation_id` for threading
- `GET /v1/conversations/:id` — retrieve conversation history
- `DELETE /v1/conversations/:id` — clear a conversation thread
- Per-caller rate limiting (by `X-Whirlwind-Service-Token`)
- Usage and cost tracking per caller per model

## Key design decisions

- LiteLLM runs as a sidecar container — `ai-gateway` calls it on `localhost:11434` (internal)
- `ai-gateway` owns conversation state in Postgres — products do not store conversation history
- Tool definitions are passed by the caller per request — `ai-gateway` does not define product tools
- Responses are synchronous for now; streaming can be added when needed

## Runbook

_To be written when the service is implemented._
