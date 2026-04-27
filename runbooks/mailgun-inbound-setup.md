# Runbook: Mailgun Inbound Email Setup

This runbook covers everything needed to receive inbound email in comm-gateway:
domain verification, DNS, Mailgun route creation, local dev with ngrok, and
production smoke testing.

Run this after the VPS is provisioned (see `runbooks/vps-setup.md`), or in
parallel with it — the Mailgun domain and DNS steps can be done before the VPS
is live.

---

## How inbound email works

```
sender → Mailgun MX servers → Mailgun route → comm-gateway webhook
         (real SMTP, your domain)              (HTTP POST, can be ngrok in dev)
```

Mailgun receives the email at your MX records, then fires an HTTP POST to your
configured route URL. That URL is your comm-gateway `/v1/webhooks/mailgun/email`
endpoint — which can be an ngrok tunnel during local development.

---

## Prerequisites

- [ ] Domain with registrar access (to add DNS records)
- [ ] Mailgun account with a sending domain already verified (or create one now)
- [ ] comm-gateway running locally (for ngrok testing) or deployed (for prod)
- [ ] `MAILGUN_WEBHOOK_SIGNING_KEY` from Mailgun dashboard → Webhooks → HTTP webhook signing key

---

## Step 1 — Create the inbound subdomain in Mailgun

Inbound email should use a subdomain (e.g. `in.yourdomain.com`) separate from your
sending domain. This keeps MX records clean and lets you switch inbound providers
independently.

In Mailgun dashboard → **Sending → Domains → Add New Domain**:

| Field | Value |
|---|---|
| Domain | `in.yourdomain.com` |
| Region | US or EU (match your sending domain region) |

---

## Step 2 — Add DNS records

Mailgun will show you the records to add. You need:

| Type | Name | Value |
|---|---|---|
| MX | `in` | `mxa.mailgun.org` (priority 10) |
| MX | `in` | `mxb.mailgun.org` (priority 10) |
| TXT | `in` | `v=spf1 include:mailgun.org ~all` |

> **Note:** The `A` record for `in.yourdomain.com` pointing to your VPS IP was
> already added in the VPS setup runbook (Step 2). Mailgun does not need the
> `A` record — it only uses MX for inbound.

Add these at your registrar. Propagation takes 5–30 minutes. Mailgun's domain
verification UI will show green once it sees the records.

---

## Step 3 — Create the Mailgun inbound route

Mailgun routes match incoming email recipients and forward to your webhook.

In Mailgun dashboard → **Receiving → Create Route**:

| Field | Value |
|---|---|
| Expression type | Match Recipient |
| Recipient filter | `.*@in\.yourdomain\.com` |
| Actions | **Forward** → your webhook URL |
| Description | `comm-gateway inbound` |

**Webhook URLs:**

| Environment | URL |
|---|---|
| Local dev (ngrok) | `https://your-id.ngrok.io/v1/webhooks/mailgun/email` |
| Production | `https://comms.yourdomain.com/v1/webhooks/mailgun/email` |

Create the route with the ngrok URL for local testing, then update it to the
production URL after VPS deploy. You can also create two routes simultaneously
(one for each URL) to test before cutover.

> **Using the Twilio MCP in a Claude session:**
> ```
> Create a Mailgun inbound route matching .*@in.yourdomain.com forwarding to
> https://comms.yourdomain.com/v1/webhooks/mailgun/email
> ```
> The Twilio MCP is for SMS; Mailgun routes must be created via the Mailgun
> dashboard or API directly.

---

## Step 4 — Set comm-gateway environment variables

Both locally (`.env`) and in Dokploy, set:

```bash
MAILGUN_WEBHOOK_SIGNING_KEY=<from Mailgun dashboard → Webhooks → HTTP webhook signing key>
MAILGUN_API_KEY=<your Mailgun private API key>
MAILGUN_DOMAIN=in.yourdomain.com
EMAIL_EVENT_SUBSCRIBER_URL=http://mindblossom:4000/v1/internal/email_received
```

For local dev in `infra-local/.env` or `comm_gateway/.env`:

```bash
MAILGUN_WEBHOOK_SIGNING_KEY=<signing key>
EMAIL_EVENT_SUBSCRIBER_URL=http://host.docker.internal:4000/v1/internal/email_received
```

---

## Step 5 — Local dev with ngrok

Mailgun's MX-to-webhook flow works with ngrok exactly like Twilio SMS. ngrok
handles only the HTTP leg — Mailgun handles email receipt at their servers.

```bash
# Start ngrok pointing at comm-gateway port 4001
ngrok http 4001
```

Take the `https://` forwarding URL and update your Mailgun route to:
```
https://your-id.ngrok.io/v1/webhooks/mailgun/email
```

To test: send an email to any address `@in.yourdomain.com`. The ngrok inspector
at `http://localhost:4040` will show the incoming POST from Mailgun, and your
local comm-gateway logs will show the event being processed.

> **Tip:** Start comm-gateway and mindblossom locally before testing. comm-gateway
> forwards the event to mindblossom — if mindblossom isn't running, comm-gateway's
> `DispatchEvent` worker will retry (Oban, up to 5 attempts, exponential backoff).

---

## Step 6 — Re-derive user email addresses

After confirming the domain is working, run the re-derive task in mindblossom
to update all existing users to the new inbound domain:

```bash
# Dry run first — see what would change
INBOUND_EMAIL_DOMAIN=in.yourdomain.com mix mindblossom.rederive_assigned_emails --dry-run

# Apply
INBOUND_EMAIL_DOMAIN=in.yourdomain.com mix mindblossom.rederive_assigned_emails
```

In production (via Dokploy terminal):
```bash
# INBOUND_EMAIL_DOMAIN is already set as an env var in Dokploy
mix mindblossom.rederive_assigned_emails
```

---

## Step 7 — Production smoke test

Once the VPS is live and the Mailgun route points to the production URL:

```bash
# 1. Check comm-gateway health
curl https://comms.yourdomain.com/health

# 2. Send a test email to your assigned address
#    (find it in mindblossom — it's the assigned_email field on your user record)
#    e.g.: mb-a1b2c3d4@in.yourdomain.com

# 3. Watch comm-gateway logs in Dokploy — you should see:
#    - incoming POST to /v1/webhooks/mailgun/email
#    - Oban job enqueued for DispatchEvent
#    - HTTP POST to mindblossom /v1/internal/email_received → 200

# 4. Check mindblossom — the email should appear in the feed
```

---

## Troubleshooting

**Mailgun shows "delivered" but comm-gateway never receives the POST:**
- Check the route's webhook URL — it must be the full `https://` URL
- Test the route in Mailgun dashboard → Receiving → your route → Send Test
- Check that comm-gateway is healthy and the ngrok/Traefik tunnel is active

**comm-gateway returns 403 on the webhook:**
- `MAILGUN_WEBHOOK_SIGNING_KEY` is wrong — double-check it in Mailgun dashboard
  under **Webhooks → HTTP webhook signing key** (not the API key)

**comm-gateway accepts the event but mindblossom doesn't receive it:**
- Check `EMAIL_EVENT_SUBSCRIBER_URL` in comm-gateway's env — must match where
  mindblossom is running
- Check Oban job status: failed DispatchEvent jobs appear in mindblossom's
  LiveDashboard at `/dev/dashboard`

**User's email not found (email arrives but gets silently dropped):**
- The `To` address doesn't match any `assigned_email` in the users table
- Run `mix mindblossom.rederive_assigned_emails` if the domain recently changed
- Check the user's `assigned_email` directly in the database

---

## Switching domains in the future

1. Add the new domain to Mailgun and verify DNS (Steps 1–2 above)
2. Create a new inbound route pointing at the same webhook URL (Step 3)
3. Update `INBOUND_EMAIL_DOMAIN` in mindblossom's env
4. Run `mix mindblossom.rederive_assigned_emails` — updates all existing users
5. Delete the old Mailgun route once traffic confirms the new one works
6. Zero code changes required
