# ADR-003: PostgreSQL as the primary database

**Date:** 2026-04-25
**Status:** Accepted

## Context

Every product and most platform services need a relational data store. We needed a default choice that all services could use without justification.

MindBlossom v1 used Supabase (managed Postgres). The v2 migration moves to self-managed Postgres on the VPS.

## Decision

PostgreSQL 16 as the standard database for all services. Each product and stateful service runs its own isolated Postgres instance (separate database, or separate container). Services do not share a database.

Extensions enabled by default:
- `uuid-ossp` — UUID generation
- `pgcrypto` — cryptographic functions
- `pg_stat_statements` — query performance monitoring

Extensions added when needed:
- `pgvector` — vector embeddings for AI semantic search

## Consequences

**Easier:**
- Ecto (Elixir's database library) has deep Postgres integration
- JSONB columns handle semi-structured data without a separate document store
- Oban uses Postgres for job queuing — no separate broker needed
- Full SQL expressiveness: window functions, CTEs, JSONB queries, full-text search

**Harder:**
- Self-managed Postgres requires backup strategy (no Supabase safety net)
- No built-in auth layer (Supabase Auth replaced by application-layer auth)
- Row Level Security was used in v1 — must be replaced with application-layer authorization
- Each new service needs its own Postgres provisioning

**Migration note (MindBlossom v1 → v2):**
Supabase Auth's `auth.users` table and JWT handling must be replaced. Guardian + Pow is the standard choice. RLS policies move to Ecto query scoping.
