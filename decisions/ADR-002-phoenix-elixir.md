# ADR-002: Phoenix/Elixir as the standard backend

**Date:** 2026-04-25
**Status:** Accepted

## Context

We needed a backend language and framework that would serve as the standard across all platform services and products. Options considered: Node.js (Fastify/Express), Go, Python (FastAPI), Ruby on Rails, Elixir/Phoenix.

Requirements:
- Excellent concurrency for webhook processing and AI response pipelines
- Strong support for background jobs and long-running processes
- Mature ecosystem for HTTP APIs, WebSockets, and database access
- Operable by a small team
- Good fit for the specific workload: inbound webhooks → process → store → respond

## Decision

Elixir with Phoenix for all backend services.

The BEAM virtual machine's process model is a direct fit for the core workload: each inbound SMS or email maps naturally to a supervised OTP process. Failures are isolated and supervised — a crashed AI job doesn't affect the web server. Phoenix Channels replace the need for a separate WebSocket service. Oban (background jobs on Postgres) removes the need for a separate queue service.

The language is functional and immutable by default, which aligns with the domain model already designed in MindBlossom v1 (TypeScript DDD with immutable entities).

## Consequences

**Easier:**
- Webhook processing, background jobs, real-time features — all first-class in Phoenix/OTP
- Single language across all services reduces context switching
- Oban eliminates Redis as a hard dependency for job queues
- Phoenix.PubSub eliminates the need for a separate pub/sub broker on a single node

**Harder:**
- Elixir has a smaller talent pool than Node or Python
- Functional programming requires a mental model shift for engineers from OO backgrounds
- Some libraries are less mature than their Node/Python equivalents
- Frontend engineers cannot contribute to backend code without learning Elixir
