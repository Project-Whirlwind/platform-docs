# price-watcher

**Repo:** `github.com/project-whirlwind/price-watcher`
**Port:** 4004
**Status:** Planned
**ADR:** [ADR-010](../decisions/ADR-010-price-watcher.md)
**API contract:** `api-contracts/price-watcher/v1/openapi.yml` (to be created)

---

## Purpose

Scheduled price-change detection for product URLs. Products submit a watch (a URL plus a subscriber reference). price-watcher fetches the page on a schedule, extracts the price, confirms any change with a second fetch, and emits a webhook when the price drops, crosses a threshold, or the product disappears.

No public host, no Traefik router, no TLS certificate. Internal network only, same posture as ai-gateway.

---

## What it does

### Product-centric model

N watches on the same product URL share one scheduled fetch. The number of HTTP requests to retail sites scales with the number of distinct products, not the number of subscribers. A `products` table holds one row per canonical URL; a `watches` table links subscribers to products.

### Price extraction

Extraction runs in three tiers, in order: (1) JSON-LD `application/ld+json` blocks with `@type: Product`, (2) OpenGraph `product:price:amount` and `product:price:currency` meta tags, (3) HTML microdata `itemprop=price` and `itemprop=priceCurrency`. The first tier that yields a price wins. Canonical URL is read from `link[rel=canonical]` or `og:url` regardless of which extraction tier fires.

### Confirm-before-alert

When a price change is detected, price-watcher re-fetches the page 10 minutes later before emitting any webhook. If the second fetch confirms the new price, observations are written and alerts fire. If the second fetch matches the previous price, the change is treated as a transient error and no webhook is sent.

### Failure ladder

Consecutive fetch failures extend the check interval. Repeated 404/410 responses mark the product as gone, close all active watches for that product, and emit `product_gone` webhooks to each subscriber.

### Watch resolution

A watch is created with status `pending`. Once the first successful price fetch completes, the watch transitions to `active` and a `watch_resolved` webhook fires, delivering the initial product data to the subscriber. Vendors that block automated fetches (Amazon is the primary example) have their watches created with status `unsupported`; this is returned synchronously in the 201 response to `POST /v1/watches` and no webhook follows. In Phase 0, `watch_resolved` is therefore only ever emitted for `active` watches.

### Per-vendor politeness

A `vendors` table tracks one row per registrable domain. Each vendor has a `crawl_delay_s` column (default 30 seconds). The fetch worker atomically claims a vendor slot via a conditional SQL update before fetching, and snoozes (via Oban's `{:snooze, seconds}` return) if the slot is not available.

robots.txt is respected: the `User-agent: *` group's Disallow prefixes are checked before any fetch. A 404 on robots.txt or an empty Disallow is treated as allow-all.

---

## API

Service-token auth required on all `/v1` routes via the `x-whirlwind-service-token` request header, compared with `Plug.Crypto.secure_compare`. The `GET /health` route requires no auth.

```
POST /v1/watches
  Body: { "url": "https://...", "subscriber_ref": "mb:user:UUID",
          "rule": {"type": "any_drop"} }
  rule is optional; defaults to any_drop
  -> 201 { "watch_id": "...", "status": "active"|"pending"|"unsupported",
           "product": {"title","price_cents","currency","image_url"} | null }
  -> 422 { "errors": {...} } on bad url or subscriber_ref

DELETE /v1/watches/:id
  -> 204 (idempotent; 204 even if the watch is already gone)

GET /health
  -> 200 {"status":"healthy","checks":{"database":"ok"}}
```

Alert rule shapes:

| Shape | Meaning |
|-------|---------|
| `{"type":"any_drop"}` | Trigger on a confirmed drop of at least 2% or $1.00, whichever is larger |
| `{"type":"below_cents","cents":25000}` | Trigger when the price falls at or below the given amount |
| `{"type":"percent_drop","percent":10}` | Trigger when the price drops by at least N percent |

---

## Outbound webhooks

price-watcher posts webhooks to the URL configured in `PRICE_EVENT_SUBSCRIBER_URL`. All requests carry the header `x-whirlwind-service-token: <OUTBOUND_SERVICE_TOKEN>`.

**watch_resolved** (fired when the first price fetch completes for a pending watch; in Phase 0 always carries `status: "active"` -- unsupported watches are signaled in the 201 response to `POST /v1/watches`, not here):
```json
{
  "type": "watch_resolved",
  "watch_id": "...",
  "subscriber_ref": "mb:user:UUID",
  "status": "active",
  "product": {"title","url","image_url","price_cents","currency"}
}
```

**price_changed** (fired after a confirmed price change that triggers the watch rule):
```json
{
  "type": "price_changed",
  "watch_id": "...",
  "subscriber_ref": "mb:user:UUID",
  "product": {"title","url","image_url"},
  "old_price_cents": 34800,
  "new_price_cents": 27800,
  "currency": "USD",
  "in_stock": true
}
```

**product_gone** (fired when consecutive 404/410 responses confirm the product URL is dead):
```json
{
  "type": "product_gone",
  "watch_id": "...",
  "subscriber_ref": "mb:user:UUID",
  "product": {"title","url"}
}
```

---

## Env vars

All variables are read in `runtime.exs`. In production, each uses `System.fetch_env!` and will crash on startup if missing.

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `DATABASE_URL` | yes | none | Postgres connection string |
| `SECRET_KEY_BASE` | yes | none | Phoenix secret key (64+ chars; generate with `mix phx.gen.secret`) |
| `SERVICE_TOKEN` | yes | none | Token callers must send in `x-whirlwind-service-token` |
| `OUTBOUND_SERVICE_TOKEN` | yes | none | Token sent in outbound webhook requests |
| `PRICE_EVENT_SUBSCRIBER_URL` | yes | none | URL where price events are posted (e.g. `http://mindblossom:4000/v1/internal/price_changed`) |
| `PORT` | no | `4004` | HTTP port |

Callers that integrate with price-watcher need two additional variables on their side:

| Variable | Description |
|----------|-------------|
| `PRICE_WATCHER_URL` | Base URL of the price-watcher service (e.g. `http://price-watcher:4004`) |
| `PRICE_WATCHER_SERVICE_TOKEN` | Shared token used for calls to price-watcher and to validate inbound price-watcher webhooks (same single-secret-per-pair scheme as comm-gateway) |

---

## Integration

The price-watcher repo README is the integration guide and the Dokploy deployment runbook. It covers:
- Quick start commands and the full env var table
- The complete API contract with example curl calls
- Step-by-step integration for any Whirlwind product (configure two env vars, call `POST /v1/watches`, implement a webhook endpoint)
- The mindblossom integration as the worked example (`Mindblossom.PriceWatcher.Client` behaviour with HTTP and Stub implementations)
- Dokploy deployment steps (create app, attach `dokploy-network`, provision Postgres, set env vars, run migrations via release command)

Point new integrators at the repo README rather than duplicating those steps here.

---

## Runbook

_To be written when the service is implemented._
