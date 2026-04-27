# Runbook: Create a new product

Follow these steps when building a new product on Project Whirlwind.

---

## Before writing any code

### 1. Write a PRD

A Product Requirements Document defines what the product does and for whom. It lives outside this repo (notion, linear, etc.) but is linked from the product repo's README.

A PRD for a Whirlwind product must include:
- Problem statement and target user
- Core user flows (not features — flows)
- Which platform services it will use (`comm-gateway`, `ai-gateway`, `revenue-ledger`)
- Data model sketch (what does this product's Postgres schema look like?)
- Non-goals (what this product explicitly will not do)

### 2. Define the product's API contract

If the product exposes an API consumed by other services or a frontend, add it to `api-contracts`:

```
api-contracts/{product-name}/v1/openapi.yml
```

### 3. Add to platform-docs

Update `architecture/service-map.md` with the new product repos.

---

## Set up the repos

A product gets two repos minimum:

| Repo | Contents |
|------|---------|
| `{product-name}-api` | Phoenix backend |
| `{product-name}-web` | Frontend (React or LiveView) |

### 4. Bootstrap the Phoenix backend

```bash
mix phx.new {product_name}_api --app {product_name}_api --no-html --no-assets
```

Install baseline dependencies in `mix.exs`:

```elixir
defp deps do
  [
    {:phoenix, "~> 1.7"},
    {:phoenix_ecto, "~> 4.4"},
    {:ecto_sql, "~> 3.11"},
    {:postgrex, "~> 0.18"},
    {:oban, "~> 2.18"},        # background jobs
    {:guardian, "~> 2.3"},     # JWT auth
    {:pow, "~> 1.0"},          # auth flows
    {:plug_cowboy, "~> 2.7"},
    {:jason, "~> 1.4"},
    {:req, "~> 0.5"},          # HTTP client for calling platform services
    {:finch, "~> 0.18"},
  ]
end
```

### 5. Configure platform service connections

In `runtime.exs`, bind platform service URLs from environment:

```elixir
config :my_app,
  comm_gateway_url: System.fetch_env!("COMM_GATEWAY_URL"),
  comm_gateway_token: System.fetch_env!("COMM_GATEWAY_SERVICE_TOKEN"),
  ai_gateway_url: System.fetch_env!("AI_GATEWAY_URL"),
  ai_gateway_token: System.fetch_env!("AI_GATEWAY_SERVICE_TOKEN"),
  revenue_ledger_url: System.fetch_env!("REVENUE_LEDGER_URL"),
  revenue_ledger_token: System.fetch_env!("REVENUE_LEDGER_SERVICE_TOKEN")
```

### 6. Add to infra-local

Add the product's API and any product-specific dependencies to `infra-local/docker-compose.yml`. Platform services (`comm-gateway`, etc.) are already in the playground stack.

---

## Required setup checklist

```
□ PRD written and linked from repo README
□ API contract in api-contracts (if applicable)
□ GitHub repos created with branch protection
□ Phoenix backend bootstrapped with baseline deps
□ Auth configured (Guardian + Pow)
□ Oban configured
□ Platform service clients configured (comm-gateway, ai-gateway, revenue-ledger)
□ .env.example complete
□ Makefile with standard targets
□ docker-compose.yml for local dev
□ Dockerfile for production build
□ GET /health endpoint
□ Product added to platform-docs service-map
□ Product added to infra-local
```

---

## Platform service contracts

When calling platform services, always:
1. Reference the OpenAPI spec in `api-contracts` for the exact request/response shape
2. Handle `503` responses — platform services can be temporarily unavailable
3. Use Oban to make calls async where latency is acceptable (AI responses, non-critical comms)
4. Set timeouts on all outbound HTTP calls — never wait indefinitely
