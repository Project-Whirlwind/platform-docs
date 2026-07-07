# ADR-010: price-watcher as the price-change detection service

**Date:** 2026-07-06
**Status:** Accepted

## Context

MindBlossom v2 lets users save product links from SMS and email. Links tagged "shopping" point to product pages on retail sites, and users want to know when a saved product drops in price, falls below a threshold, or disappears. price-watcher monitors those pages on a schedule and notifies the subscriber (via webhook) when that happens. Without a dedicated crawler, detecting price changes would mean crawling retailer pages directly from product code. The work involves:
- Crawling third-party retail pages on a schedule
- Respecting per-vendor crawl politeness (rate limits, robots.txt, crawl delays)
- Tiered price extraction across structured data formats (JSON-LD, OG tags, microdata)
- Confirming a price drop with a second fetch before alerting, to avoid false positives from transient pricing errors

This workload cannot live inside mindblossom. Crawler traffic patterns are fundamentally different from web-request handling. Scheduled bulk fetches with retries, back-off logic, and per-vendor politeness windows have their own failure modes. A bad crawl loop, a vendor temporarily blocking the crawler's IP, or a parse bug should not affect mindblossom's ability to receive and process SMS. Keeping the crawl workload isolated means its blast radius is contained to price-watcher. Price tracking is also useful beyond mindblossom: any Whirlwind product that handles product links can call price-watcher's API to create a watch, and embedding this in mindblossom would make that reuse awkward.

The tradeoff of a separate service is operational overhead: one more service to deploy, operate, and monitor (a Dockerfile, a Dokploy app, and a dedicated Postgres database). Debugging a missed alert crosses two service boundaries (price-watcher logs and mindblossom logs), and webhook delivery requires both services to be healthy for the full notification loop to work.

## Decision

A dedicated `price-watcher` service built on Elixir/Phoenix and Oban, with its own Postgres database. It runs internal-only on port 4004 with no public host, no Traefik router, and no TLS certificate.

The design uses a product-centric model: N watches on the same product URL share one scheduled fetch, not one fetch per watch. This keeps crawl volume proportional to the number of distinct products, not the number of subscribers.

**What price-watcher manages:**
- URL normalization and canonical product identity (one product row per distinct URL)
- Vendor registry with per-vendor crawl delay and robots.txt compliance
- Scheduled price checks via Oban workers (SweepDueProducts cron, CheckPrice, ConfirmPriceChange)
- Confirm-before-alert: a 10-minute re-fetch confirms a price change before any webhook fires
- Failure ladder: consecutive fetch errors extend the check interval; repeated 404/410 responses mark a product as gone and close its watches
- Outbound webhooks to `PRICE_EVENT_SUBSCRIBER_URL` for watch resolution, price changes, and product-gone events
- Service-token auth on all `/v1` routes via `x-whirlwind-service-token`

Money is stored as `price_cents :integer` + `currency :string` (ISO 4217). No floats anywhere.

## Consequences

**Easier:**
- Crawl politeness, rate limits, and retry logic are isolated from mindblossom and any other product
- Any Whirlwind product can track prices by calling `POST /v1/watches`
- Scaling the crawl workload (more products, more frequent checks) does not affect mindblossom response times
- Crawl bugs and vendor blocks are contained to price-watcher

**Harder:**
- One more service to deploy and monitor (Dockerfile, Dokploy app, Postgres DB)
- Webhook delivery requires both price-watcher and mindblossom to be healthy for a full notification round-trip
- Debugging a missed price alert requires checking logs in both services
- Amazon and other vendors that block automated fetches are deferred (watches for unsupported vendors are created with status `unsupported` rather than failing loudly)
