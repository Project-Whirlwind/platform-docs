# comm-gateway

**Repo:** `github.com/project-whirlwind/comm-gateway`
**Port:** 4001
**Status:** Planned
**ADR:** [ADR-007](../decisions/ADR-007-comm-gateway.md)
**API contract:** `api-contracts/comm-gateway/v1/openapi.yml` (to be created)

---

## Purpose

Single integration point for all inbound and outbound communication. Abstracts Twilio (SMS) and Mailgun (email) so no product ever calls a provider directly. Enforces safety limits on all inbound content before it reaches any product.

---

## What it does

### Inbound

- Receives Twilio webhooks for inbound SMS
- Receives Mailgun webhooks for inbound email
- Validates webhook signatures — requests with invalid signatures are rejected with 403 before any processing
- Applies configurable safety limits (see below)
- Normalizes payloads into a standard event format
- Forwards normalized events to subscribing products via HTTP POST

**Webhook acknowledgment is always immediate.** `comm-gateway` responds 200 to Twilio and Mailgun within milliseconds — before any product processing occurs. Twilio requires a response within 15 seconds or it retries; never block the webhook handler on downstream work.

### Outbound

- `POST /v1/messages/sms` — send an SMS via the configured provider
- `POST /v1/messages/email` — send an email via the configured provider
- Twilio number assignment (round-robin across active numbers)
- Retry logic for failed sends (Oban, exponential backoff, max 3 attempts)
- Delivery status tracking via provider receipt webhooks

---

## Safety limits

All limits are configurable via environment variables. They apply before an event is forwarded to any product.

| Limit | Env var | Default | Behaviour on exceed |
|---|---|---|---|
| Max email body size | `MAX_EMAIL_BODY_BYTES` | 512KB | Reject with 413, do not forward |
| Max links per email | `MAX_LINKS_PER_EMAIL` | 20 | Truncate; event includes `truncated: true` |
| Max links per SMS | `MAX_LINKS_PER_SMS` | 5 | Truncate; event includes `truncated: true` |
| Max email recipients (To) | `MAX_EMAIL_RECIPIENTS` | 1 | Reject; prevents processing mass/bulk email |
| Rate limit per sender (inbound) | `RATE_LIMIT_PER_SENDER_HOUR` | 30 | Reject with 429 |
| Max attachment size | `MAX_ATTACHMENT_BYTES` | 10MB | Drop attachment; still forward event |

When link truncation occurs, the forwarded event includes the original count so the product can inform the user:

```json
{
  "type": "email_received",
  "links": ["https://...", "..."],
  "link_count": 20,
  "link_count_original": 300,
  "truncated": true
}
```

---

## Event format

Both inbound SMS and email are normalized to a standard event. Products receive the same shape regardless of provider.

**`sms_received`:**
```json
{
  "id": "uuid-v4",
  "type": "sms_received",
  "received_at": "2026-04-26T04:00:00Z",
  "from": "+15556664573",
  "to": "+15557771234",
  "body": "Check this out https://example.com",
  "links": ["https://example.com"],
  "link_count": 1,
  "link_count_original": 1,
  "truncated": false,
  "provider": "twilio",
  "provider_message_id": "SMxxxxxxxxxxxxxxxx"
}
```

**`email_received`:**
```json
{
  "id": "uuid-v4",
  "type": "email_received",
  "received_at": "2026-04-26T04:00:00Z",
  "from": "sender@example.com",
  "to": ["inbox@in.mindblossom.ai"],
  "subject": "Interesting article",
  "body": "...",
  "links": ["https://example.com"],
  "link_count": 1,
  "link_count_original": 1,
  "truncated": false,
  "provider": "mailgun",
  "provider_message_id": "mailgun-message-id"
}
```

Products POST their event endpoint URL to `comm-gateway` on startup. `comm-gateway` delivers events with at-least-once semantics — products must be idempotent on `provider_message_id`.

---

## Provider adapter pattern

Each provider implements a shared behaviour. The active provider is selected via env var — switching providers requires only a config change, zero product code changes.

```elixir
defmodule CommGateway.SMS.Provider do
  @callback send(to: String.t(), from: String.t(), body: String.t()) ::
    {:ok, %{provider_id: String.t()}} | {:error, reason :: term()}

  @callback validate_webhook(conn :: Plug.Conn.t()) ::
    {:ok, params :: map()} | {:error, :invalid_signature}
end
```

**SMS providers:** `SMS_PROVIDER=twilio` (default), `telnyx` (planned)
**Email providers:** `EMAIL_PROVIDER=mailgun` (default), `postmark` (planned)

---

## Key design decisions

- Webhook handlers always respond immediately — downstream processing is always async via Oban
- Safety limits are enforced here, not in products — products trust that events arriving from `comm-gateway` are already within bounds
- Signature validation is the first thing that happens — no limit checks or parsing before auth
- Provider switching requires zero product changes
- Truncation is transparent — events carry `truncated` + `link_count_original` so products can surface this to users

---

## Runbook

_To be written when the service is implemented._
