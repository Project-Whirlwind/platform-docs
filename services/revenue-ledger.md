# revenue-ledger

**Repo:** `github.com/project-whirlwind/revenue-ledger`
**Port:** 4003
**Status:** Planned (background POC)
**ADR:** [ADR-005](../decisions/ADR-005-tigerbeetle.md)
**API contract:** `api-contracts/revenue-ledger/v1/openapi.yml` (to be created)

---

## Purpose

Double-entry financial ledger for tracking revenue attributable to users across all Whirlwind products. Foundation for future profit-sharing and co-op distribution. Backed by TigerBeetle.

## What it does

- `POST /v1/accounts` — create a revenue account for a user
- `POST /v1/transfers` — record a revenue attribution transfer
- `GET /v1/accounts/:id` — get account with current balance
- `GET /v1/accounts/:id/transfers` — get transfer history

## Key design decisions

- TigerBeetle enforces double-entry invariants — every credit has a matching debit, balances cannot go negative (unless flagged)
- Revenue attribution is append-only — no transfers are deleted, only reversed with a counter-transfer
- `revenue-ledger` is the only service that touches TigerBeetle — no product calls TB directly
- Initial scope is POC: revenue is recorded but no distribution logic yet

## Revenue model

Each user has a TigerBeetle account. When a product records revenue:
1. Debit the "platform revenue pool" account
2. Credit the user's account with their attributed share

Distribution (profit-sharing) will be a separate flow reading from these balances.

## Runbook

_To be written when the service is implemented._
