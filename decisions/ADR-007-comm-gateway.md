# ADR-007: comm-gateway as the communication abstraction layer

**Date:** 2026-04-25
**Status:** Accepted

## Context

MindBlossom v2 (and future products) need to send and receive SMS and email. In MindBlossom v1, Twilio and Mailgun were called directly from Supabase Edge Functions within the product codebase.

This created several problems:

1. **Direct provider coupling** — adding a new product duplicated provider integration logic; switching providers required changes across all products.

2. **Outbound responses never reliably worked** — the v1 webhook handlers attempted to generate and send AI responses synchronously within the webhook request lifecycle. Twilio requires a response within 15 seconds or it retries the webhook. AI response generation (5–30s) consistently exceeded this window, causing Twilio to retry and the response to the user to either never arrive or arrive multiple times.

3. **No inbound safety limits** — a single email containing ~300 links imported all of them unexpectedly, with no per-message cap enforced anywhere.

## Decision

A dedicated `comm-gateway` service is the single integration point for all communication.

**Webhook acknowledgment is always immediate.** The webhook handler validates the signature, applies safety checks, enqueues an Oban job, and responds 200 — all within milliseconds. Any downstream processing (product notification, AI response generation, outbound reply) happens asynchronously out of band. This is the fix for the v1 outbound reliability failure.

```
Twilio → POST /v1/webhooks/twilio (comm-gateway)
  1. Validate Twilio signature
  2. Apply safety limits (size, rate, link count)
  3. Enqueue delivery job
  4. Respond 200 immediately ← Twilio marks delivered

  (async, Oban job)
  5. POST sms_received event → subscribing product
  6. Product enqueues ProcessInboundSms worker
  7. Worker calls ai-gateway → gets response
  8. Worker calls POST /v1/messages/sms (comm-gateway)
  9. comm-gateway → Twilio API → user's phone
```

**Safety limits are enforced in `comm-gateway` before any event is forwarded.** These are technical limits (size, rate, link count), not business logic. All limits are configurable via environment variables. When a limit is exceeded, the event is either rejected entirely or truncated with `truncated: true` in the forwarded event so products can surface the information to users.

**Provider adapters implement a shared behaviour.** Switching from Mailgun to Postmark is a `comm-gateway` config change. No product changes.

## Consequences

**Easier:**
- Outbound replies are decoupled from webhook latency — they work reliably regardless of AI response time
- Safety limits are defined and enforced once, not per-product
- Adding a new product requires only a config registration — no provider integration work
- Provider switches are contained to one service
- Webhook signature verification, retry logic, and delivery tracking are implemented once

**Harder:**
- `comm-gateway` is a critical dependency — its unavailability blocks all inbound and outbound communication across all products
- Events are delivered with at-least-once semantics — products must handle duplicate `provider_message_id` values idempotently
- An additional network hop for all communication (internal, sub-millisecond on the same network)

**First provider set:** Twilio (SMS), Mailgun (inbound + outbound email)

## Test coverage requirements

Given the v1 failures, the following must be covered by the test suite before the service ships:

- Valid provider signature → accepted and forwarded
- Invalid provider signature → 403, nothing forwarded
- Email body > `MAX_EMAIL_BODY_BYTES` → 413, not forwarded
- Email with > `MAX_LINKS_PER_EMAIL` links → truncated, `truncated: true` in event
- Email to multiple recipients → rejected (prevents bulk email import)
- Sender exceeds rate limit → 429
- Outbound SMS: provider returns error → Oban retries with backoff
- Outbound SMS: max retries exceeded → error logged, delivery marked failed
- Provider config swap → correct adapter called (Twilio vs Telnyx)
