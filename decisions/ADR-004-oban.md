# ADR-004: Oban for background job processing

**Date:** 2026-04-25
**Status:** Accepted

## Context

Multiple services require reliable background job processing:
- `comm-gateway`: outbound SMS/email with retry logic
- `mindblossom`: AI response pipeline, link metadata enrichment
- `revenue-ledger`: periodic usage resets, attribution processing

Options considered: Oban (Postgres-backed), Exq (Redis-backed), Broadway (message pipeline), custom GenServer.

## Decision

Oban as the standard background job library across all Phoenix services.

Oban stores jobs in PostgreSQL, which means:
- No additional infrastructure — Postgres is already required
- Jobs survive application restarts
- Job state is queryable with standard SQL
- Transactional job enqueueing (enqueue a job in the same transaction as the data it processes)

Redis-backed alternatives (Exq) were rejected because Redis would become a hard dependency solely for job queuing, and Postgres is already present.

## Consequences

**Easier:**
- Zero additional services needed for background processing
- Transactional enqueueing prevents ghost jobs on failed transactions
- Oban Web (dashboard) provides observability without extra tooling
- Scheduled/cron jobs (e.g., reset daily Twilio usage counters) are first-class

**Harder:**
- High job throughput puts more load on Postgres (mitigated by Oban's queue configuration)
- Very high-volume pipelines (millions of jobs/day) may require Oban Pro or a different approach

**Replaces:**
- `reset_twilio_daily_usage()` Postgres function + external cron → Oban scheduled job
- `enrich-link-metadata` Supabase Edge Function → Oban worker
