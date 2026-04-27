# ADR-009: api-contracts as the interface source of truth

**Date:** 2026-04-25
**Status:** Accepted

## Context

In a polyrepo environment, services communicate over HTTP. Without a shared definition of what each service exposes, there are two failure modes:
1. A service changes its API and breaks callers silently
2. A caller builds against an undocumented API and gets surprised by changes

We needed a single place where all inter-service contracts are defined, versioned, and independent of any implementation.

## Decision

A dedicated `api-contracts` repo containing OpenAPI 3.x specs for every service API and AsyncAPI schemas for events. This repo is the source of truth — if a behavior is not in the spec, it is not a contract.

**Rules:**
- A service spec is written in `api-contracts` **before** the service is implemented
- A service implementation must conform to its spec — the spec is the acceptance criterion
- Breaking changes to a spec require a new version (`/v2/`) and a migration plan
- Products reference the spec version they consume — they are not automatically upgraded

**What lives in `api-contracts`:**
- `{service}/v{n}/openapi.yml` — request/response schemas
- `events/{event_name}.json` — AsyncAPI event schemas
- `CHANGELOG.md` per service — what changed and when

**What does not live in `api-contracts`:**
- Implementation code, SDKs, or generated clients (those may live in service repos)
- Internal APIs not consumed by other services

## Consequences

**Easier:**
- New engineers can understand what a service does without reading its code
- Breaking changes are caught at the contract level before deployment
- API spec doubles as documentation — no separate doc generation needed
- Future SDK generation is straightforward from OpenAPI specs

**Harder:**
- Contract-first development requires writing the spec before the implementation, which takes discipline
- Keeping the spec in sync with the implementation requires tooling (contract testing, spec validation in CI)
- Small teams may feel the overhead is disproportionate early on — it pays off at the second service and beyond
