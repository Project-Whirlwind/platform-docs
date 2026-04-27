# ADR-005: TigerBeetle for revenue tracking

**Date:** 2026-04-25
**Status:** Accepted

## Context

The platform needs to track revenue attributable to individual users across products, with the long-term goal of enabling profit-sharing and co-op-style distribution. This is a financial ledger problem, not a general data storage problem.

Options considered: Postgres with custom accounting tables, TigerBeetle, a managed accounting API (e.g., Lago, Metronome).

A PostgreSQL approach was considered — double-entry accounting can be implemented in Postgres. However, correctness guarantees require careful schema design and disciplined query patterns that are easy to violate. TigerBeetle enforces double-entry invariants at the database level.

## Decision

TigerBeetle as the backing store for `revenue-ledger`. TigerBeetle is a purpose-built financial accounting database that enforces double-entry bookkeeping at the storage level — debits always balance credits, balances cannot go negative unless explicitly allowed, and all of this is enforced without application-layer logic.

`revenue-ledger` wraps TigerBeetle behind a standard HTTP API so no other service ever touches TigerBeetle directly.

**Initial scope:** Proof of concept. Revenue attribution logic runs in the background without affecting the critical path of any product. No profit distribution in v1.

## Consequences

**Easier:**
- Financial invariants are enforced by the database, not application code
- Pending transfers (two-phase commit) support held/reserved revenue
- Audit trail is inherent — every transfer is permanent and immutable
- High throughput if revenue tracking ever needs to scale

**Harder:**
- TigerBeetle requires `--privileged` on Docker Desktop (macOS) due to `fallocate`
- Pre-formatted data file required before first start — handled by `infra-local`'s `make tb-init`
- No SQL query interface — data access only through TigerBeetle's narrow API
- Elixir client ecosystem is early-stage

**Not used for:**
General application data, user records, message storage, or anything outside financial accounting. Those use Postgres.
