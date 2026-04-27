# Project Whirlwind — Platform Docs

**Rendered docs:** https://project-whirlwind.github.io/platform-docs/

This repo is the engineering source of truth for Project Whirlwind. It documents architecture decisions, service contracts, development standards, and operational runbooks.

**MindBlossom v2** is the first product built on this platform. Future products inherit the same infrastructure and shared services.

---

## What this repo is

- The answer to "why did we build it this way"
- The starting point for every new engineer, service, or product
- A living document — decisions get superseded, services get added, runbooks get refined

## What this repo is not

- Implementation code
- A ticket tracker
- Marketing copy

---

## Navigation

### Architecture
- [Overview](architecture/overview.md) — platform diagram, principles, service topology
- [Service Map](architecture/service-map.md) — every service, its repo, its role

### Decisions (ADRs)
- [How to write an ADR](decisions/README.md)
- [ADR-001 — Polyrepo structure](decisions/ADR-001-polyrepo.md)
- [ADR-002 — Phoenix/Elixir as standard backend](decisions/ADR-002-phoenix-elixir.md)
- [ADR-003 — PostgreSQL as primary database](decisions/ADR-003-postgresql.md)
- [ADR-004 — Oban for background jobs](decisions/ADR-004-oban.md)
- [ADR-005 — TigerBeetle for revenue tracking](decisions/ADR-005-tigerbeetle.md)
- [ADR-006 — Dokploy for deployment](decisions/ADR-006-dokploy.md)
- [ADR-007 — comm-gateway abstraction](decisions/ADR-007-comm-gateway.md)
- [ADR-008 — ai-gateway abstraction](decisions/ADR-008-ai-gateway.md)
- [ADR-009 — api-contracts as interface source of truth](decisions/ADR-009-api-contracts.md)

### Standards
- [API Design](standards/api-design.md) — REST conventions, versioning, error format
- [Git Workflow](standards/git-workflow.md) — branching, commits, PR process
- [Repo Structure](standards/repo-structure.md) — what every repo must have

### Services
- [comm-gateway](services/comm-gateway.md) — SMS and email integrations
- [ai-gateway](services/ai-gateway.md) — AI provider abstraction and conversation management
- [revenue-ledger](services/revenue-ledger.md) — TigerBeetle revenue tracking

### Runbooks
- [Create a new service](runbooks/new-service.md)
- [Create a new product](runbooks/new-product.md)

---

## Org structure

```
github.com/project-whirlwind/

  # Platform
  platform-docs          ← this repo
  api-contracts          ← OpenAPI specs and event schemas
  infra-local            ← local dev environment
  comm-gateway           ← SMS/email integration service
  ai-gateway             ← AI provider proxy and conversation layer
  revenue-ledger         ← TigerBeetle financial tracking service

  # Products
  mindblossom            ← MindBlossom v2 (Phoenix full-stack, LiveView)
```
