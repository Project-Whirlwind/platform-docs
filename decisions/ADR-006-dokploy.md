# ADR-006: Dokploy for deployment

**Date:** 2026-04-25
**Status:** Accepted

## Context

The platform needs a deployment target that:
- Runs on a self-managed VPS (not locked into a cloud provider)
- Supports Docker Compose natively (matching the local dev model)
- Provides a management UI for environment variables, logs, and deployments
- Is operable by a small team without DevOps specialization
- Is cost-effective at early stage

Options considered: bare Docker Compose on VPS, Fly.io, Railway, Render, Dokploy, Coolify, Kamal.

Managed platforms (Fly, Railway, Render) were rejected due to vendor lock-in and cost at scale. Bare Docker Compose was rejected because it lacks a management layer. Kamal was considered but adds Ruby toolchain complexity.

## Decision

Dokploy running on a self-managed VPS. Dokploy:
- Deploys Docker Compose applications directly from Git repositories
- Manages environment variables and secrets via its UI
- Uses Traefik as a reverse proxy with automatic TLS
- Provides deployment logs and basic service management

Each service is a separate Dokploy application pointing at its own repo. This maps directly to the polyrepo structure.

## Consequences

**Easier:**
- Local Docker Compose and production Docker Compose are the same file
- No cloud-specific configuration or tooling
- Traefik handles TLS automatically — no manual certificate management
- Per-service deployment: deploying `comm-gateway` doesn't touch `mindblossom`

**Harder:**
- VPS provisioning and OS maintenance are our responsibility
- No autoscaling — horizontal scaling requires manual Dokploy configuration
- TCP services (Postgres, TigerBeetle) are not routed through Traefik — need internal network access only
- `--privileged` for TigerBeetle must be set at the Dokploy service level

**Key constraint:**
Services that should not be publicly accessible (Postgres, TigerBeetle, Redis) must have no public port bindings in the production compose file. Only `comm-gateway`, `ai-gateway`, `revenue-ledger`, and product APIs expose HTTP ports (via Traefik labels).
