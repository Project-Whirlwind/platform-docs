# ADR-001: Polyrepo over monorepo

**Date:** 2026-04-25
**Status:** Accepted

## Context

Project Whirlwind is building shared platform services (`comm-gateway`, `ai-gateway`, `revenue-ledger`) that will be consumed by multiple independent products over time. We needed to decide whether to put all code in a single repository (monorepo) or separate repositories (polyrepo).

Monorepo was considered because it allows atomic changes across service boundaries and simplifies local tooling. However:

- Services have different release cadences and independent deployment requirements
- Future products should not have access to other products' codebases
- The shared services are infrastructure, not components — they communicate over HTTP, not function calls
- The team is small; per-repo context switching is manageable
- Dokploy (the deployment target) maps naturally to one repo per deployed service

## Decision

Polyrepo. Each service and each product is an independent git repository under the `project-whirlwind` GitHub org.

Shared infrastructure for local development lives in `infra-local`. API contracts live in `api-contracts`. These are the only repos that span service boundaries — and they contain no implementation code.

## Consequences

**Easier:**
- Independent versioning and deployment per service
- Clear ownership boundaries
- Future products have no visibility into other products' internals
- Each repo can have its own CI pipeline, dependency updates, and access controls

**Harder:**
- Cross-service changes require coordinated PRs across repos
- Local development requires running multiple repos — `infra-local` mitigates this
- No atomic commits across service boundaries — changes to a service API and its consumers must be coordinated via contract versioning
