# Repo Structure Standards

Every service and product repo in the Whirlwind org follows this baseline structure. Consistency means any engineer can orient in any repo within minutes.

---

## Required files in every repo

```
repo-name/
  README.md               ← purpose, quickstart, links
  .env.example            ← all env vars with descriptions, no real values
  .gitignore              ← language-appropriate, never commit secrets or build artifacts
  docker-compose.yml      ← local dev stack (service + its dependencies)
  Dockerfile              ← production image build
  Makefile                ← standard commands (see below)
  CHANGELOG.md            ← version history (keep current)
  docs/
    architecture.md       ← service-specific design decisions
    runbook.md            ← how to operate this service
```

---

## README.md must include

1. **One-sentence description** — what does this service do?
2. **Quick start** — from zero to running in under 5 commands
3. **Key commands** — link to Makefile targets
4. **Environment variables** — link to `.env.example` with explanations
5. **API** — link to the contract in `api-contracts`
6. **Dependencies** — what services this calls and what calls this

---

## Makefile standard targets

Every service Makefile must implement these targets:

| Target | Action |
|--------|--------|
| `make help` | List all targets with descriptions |
| `make init` | First-time setup (copy .env, build, seed) |
| `make up` | Start the service and its dependencies |
| `make down` | Stop (keep data) |
| `make reset` | Stop and wipe data |
| `make test` | Run test suite |
| `make lint` | Run linter/formatter check |
| `make format` | Auto-format code |
| `make logs` | Tail service logs |
| `make ps` | Show running containers |
| `make build` | Build production Docker image |

---

## Phoenix service structure

```
services/{service-name}/
  lib/
    {service_name}/
      application.ex
      router.ex
      controllers/        ← HTTP request handling
      workers/            ← Oban background jobs
      services/           ← business logic, external API calls
      schemas/            ← Ecto schemas
      repo.ex
    {service_name}_web/
      endpoint.ex
      router.ex
  test/
    {service_name}/
    {service_name}_web/
    support/
  config/
    config.exs
    dev.exs
    prod.exs
    runtime.exs           ← runtime env var binding
  priv/
    repo/
      migrations/
  mix.exs
  mix.lock
```

---

## Environment variable conventions

- All required vars documented in `.env.example` with a comment explaining each
- Variables are validated at startup in `runtime.exs` — a missing required variable crashes the app with a clear message, not a runtime error 10 requests later
- No `VITE_`-style client-side exposure of secrets — server-side only for credentials
- Naming: `SCREAMING_SNAKE_CASE`, prefixed with service name for shared vars: `COMM_GATEWAY_SERVICE_TOKEN`

---

## What every service must have

**Health endpoint:** `GET /health` returns `200` with dependency status (see [API Design standards](api-design.md))

**Structured logging:** JSON logs in production, human-readable in dev. Every log line includes `service`, `level`, and `request_id` where applicable.

**Graceful shutdown:** Handle `SIGTERM` — finish in-flight requests, drain Oban queues with a timeout, close DB connections cleanly.

**Startup validation:** Fail fast if required env vars are missing or database is unreachable. Don't start serving traffic in a broken state.
