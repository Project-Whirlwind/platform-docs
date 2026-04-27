# Runbook: Create a new platform service

Follow these steps when adding a new shared service to Project Whirlwind.

---

## Before writing any code

### 1. Write an ADR

Document the decision to create this service in `platform-docs/decisions/ADR-NNN-{service-name}.md`.

Answer:
- What problem does this service solve?
- Why can't it live inside an existing service?
- What are the tradeoffs of a separate service?

If you can't answer these convincingly, reconsider whether a new service is the right boundary.

### 2. Write the API contract first

Add the OpenAPI spec to `api-contracts` before writing implementation code:

```
api-contracts/{service-name}/v1/openapi.yml
```

The spec is the acceptance criterion. Implementation is done when it satisfies the spec.

Get the contract reviewed before building. Changing a contract after implementation is possible but painful.

### 3. Add the service to platform-docs

Update:
- `architecture/service-map.md` — add the service entry
- `README.md` — add to the org structure section

---

## Set up the repo

### 4. Create the GitHub repo

```
github.com/project-whirlwind/{service-name}
```

Settings:
- Default branch: `main`
- Branch protection on `main`: require PR, require CI
- No direct push to `main`

### 5. Bootstrap the Phoenix app

```bash
# On your local machine (not in Docker)
mix phx.new {service_name} --app {service_name} --no-html --no-assets
cd {service_name}
```

If the service is API-only (no HTML), use `--no-html --no-assets`.

### 6. Add to infra-local

Add the service to `infra-local/docker-compose.yml`:

```yaml
services:
  {service-name}:
    build: ../{service-name}
    ports:
      - "{port}:{port}"
    environment:
      - DATABASE_URL=${...}
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:{port}/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - playground
```

---

## Required setup checklist

```
□ ADR written and accepted
□ API contract in api-contracts repo
□ GitHub repo created with branch protection
□ Phoenix app bootstrapped
□ .env.example with all variables documented
□ Makefile with standard targets (make help, init, up, down, test)
□ docker-compose.yml for local dev
□ Dockerfile for production build
□ GET /health endpoint implemented
□ Startup validation for required env vars (runtime.exs)
□ Service added to infra-local
□ Service added to platform-docs service-map
□ README.md with quickstart
```

---

## Naming conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Repo name | kebab-case | `comm-gateway` |
| Elixir app name | snake_case | `comm_gateway` |
| Docker image | `whirlwind/{service-name}` | `whirlwind/comm-gateway` |
| Internal hostname | service name | `comm-gateway` |
| Default port | assigned per service | see service map |

## Port assignments

| Service | Port |
|---------|------|
| `mindblossom` | 4000 |
| `comm-gateway` | 4001 |
| `ai-gateway` | 4002 |
| `revenue-ledger` | 4003 |
| PostgreSQL | 5432 |
| TigerBeetle | 3001 |
| Redis | 6379 |
