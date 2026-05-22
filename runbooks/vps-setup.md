# Runbook: VPS Setup — Hetzner + Cloudflare + Tailscale + Dokploy

This runbook covers provisioning the Project Whirlwind VPS and deploying
the first two services (mindblossom + comm-gateway).

**Provider:** Hetzner Cloud
**DNS/CDN:** Cloudflare
**Admin access:** Tailscale
**Stack:** Ubuntu 24.04 LTS, Dokploy, Docker, Traefik, Let's Encrypt

---

## Prerequisites

Before starting, have these ready:

- [ ] Hetzner Cloud account (cloud.hetzner.com)
- [ ] Cloudflare account with domain registered or transferred
- [ ] Tailscale account (tailscale.com — free for 3 users)
- [ ] SSH public key (`~/.ssh/id_ed25519.pub` or similar)
- [ ] Twilio Account SID + Auth Token
- [ ] Mailgun API key + webhook signing key (once domain is sorted)
- [ ] GitHub personal access token with `repo` scope (for Dokploy to pull repos)

---

## Step 1 — Provision the VPS

In Hetzner Cloud Console → Projects → Add Server:

| Setting | Value |
|---|---|
| Location | **Ashburn, VA** (`ash`) for US-based users |
| Image | **Ubuntu 24.04** |
| Type | **CPX31** (4 vCPU x86, 8GB RAM, 160GB NVMe — $24.99/mo) |
| SSH keys | Add your public key |
| Firewall | Create new — see Step 3 |

> **Why CPX31?** 8GB gives comfortable headroom for the full platform stack (Postgres,
> Redis, TigerBeetle, comm-gateway, mindblossom). CPX31 is the smallest x86 option with
> 8GB RAM available in US datacenters. If your users are EU-based, use CAX21 in `hel1`
> (ARM, €9.49/mo) — Docker and Elixir run fine on ARM.
>
> **Why Ubuntu 24.04 LTS?** Supported until April 2029. Never use non-LTS on servers —
> interim releases (25.10 etc.) get 9 months of support then go EOL.

**As of 2026-04-28:** Server `whirlwind-prod` (CPX31, Ashburn) is provisioned at `178.156.186.8`.

---

## Step 2 — DNS (Cloudflare)

As soon as Hetzner provides the VPS IP, create these records in Cloudflare:

| Type | Name | Value | Proxy | Notes |
|---|---|---|---|---|
| A | `@` | `178.156.186.8` | DNS only | Root domain |
| A | `app` | `178.156.186.8` | ✅ Proxied | mindblossom — CDN on |
| A | `comms` | `178.156.186.8` | DNS only | comm-gateway webhooks — proxy off, Twilio posts directly |
| A | `in` | `178.156.186.8` | DNS only | Mailgun inbound email |

> **Why proxy off for `comms.`?** Twilio and Mailgun POST webhooks directly to your
> server. Cloudflare proxy adds unnecessary hops and can interfere with request
> headers used in webhook signature validation.

DNS propagation takes 5–30 minutes. Traefik can't issue TLS certs until it resolves.

---

## Step 3 — Hetzner firewall

In Hetzner Cloud → Firewalls → Create Firewall, allow inbound:

| Protocol | Port | Source | Purpose |
|---|---|---|---|
| TCP | 22 | Your IP only (or Tailscale range) | SSH |
| TCP | 80 | Any | HTTP (Traefik redirects to HTTPS) |
| TCP | 443 | Any | HTTPS |

**Do NOT open port 3000.** Dokploy admin is Tailscale-only (Step 4).

Apply this firewall to the server.

---

## Step 4 — Tailscale (lock down admin access)

SSH in as root, then install Tailscale before anything else:

```bash
ssh root@178.156.186.8

# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up

# Note your Tailscale IP — looks like 100.x.x.x
tailscale ip
```

On your dev machine, also run `tailscale up` if not already installed (`brew install tailscale`).

From this point, access the server via its **Tailscale IP** for all admin work.
Dokploy runs at `http://100.x.x.x:3000` — visible only on your Tailnet, invisible to the internet.

---

## Step 5 — First login and server hardening

```bash
# Create a non-root deploy user
adduser deploy
usermod -aG sudo deploy
mkdir -p /home/deploy/.ssh
cp ~/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh

# Disable root SSH login and password auth
sed -i 's/PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config
systemctl restart ssh

# Install Dokploy (installs Docker as a dependency — ~30 seconds)
curl -sSL https://dokploy.com/install.sh | sh
```

Dokploy is now running. Access it at `http://YOUR_TAILSCALE_IP:3000`.

---

## Step 6 — Dokploy initial configuration

1. Open `http://YOUR_TAILSCALE_IP:3000` — first visit creates your admin account
2. **Settings → Git Providers → GitHub** — connect the `project-whirlwind` org
3. **Settings → Let's Encrypt** — add your email for cert notifications

---

## Step 7 — Deploy comm-gateway

comm-gateway must be deployed first — mindblossom's inbound webhook receiver depends on it.

In Dokploy → **Create Application**:

| Field | Value |
|---|---|
| Name | `comm-gateway` |
| Repository | `github.com/Project-Whirlwind/comm-gateway` |
| Branch | `main` |
| Build type | Docker Compose |

**Environment variables** (set in Dokploy UI — never in git):

```
MIX_ENV=prod
PORT=4001
PHX_SERVER=true
DATABASE_URL=ecto://comm_gateway:STRONG_PASSWORD@postgres:5432/comm_gateway_prod
SECRET_KEY_BASE=<generate: openssl rand -base64 64>
SERVICE_TOKEN=<generate: openssl rand -hex 32>
OUTBOUND_SERVICE_TOKEN=<same token — must match COMM_GATEWAY_SERVICE_TOKEN in mindblossom>
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
MAILGUN_API_KEY=...
MAILGUN_DOMAIN=...
MAILGUN_WEBHOOK_SIGNING_KEY=...
SMS_EVENT_SUBSCRIBER_URL=http://mindblossom:4000/v1/internal/sms_received
EMAIL_EVENT_SUBSCRIBER_URL=http://mindblossom:4000/v1/internal/email_received
TRAEFIK_ENABLE=true
COMM_GATEWAY_HOST=comms.mindblossom.app
```

---

## Step 8 — Deploy mindblossom

In Dokploy → **Create Application**:

| Field | Value |
|---|---|
| Name | `mindblossom` |
| Repository | `github.com/Project-Whirlwind/mindblossom` |
| Branch | `main` |
| Build type | Docker Compose |

**Environment variables:**

```
MIX_ENV=prod
PORT=4000
PHX_SERVER=true
PHX_HOST=app.mindblossom.app
DATABASE_URL=ecto://mindblossom:STRONG_PASSWORD@postgres:5432/mindblossom_prod
SECRET_KEY_BASE=<generate: openssl rand -base64 64>
GUARDIAN_SECRET_KEY=<generate: openssl rand -base64 64>
COMM_GATEWAY_URL=http://comm-gateway:4001
COMM_GATEWAY_SERVICE_TOKEN=<same as OUTBOUND_SERVICE_TOKEN in comm-gateway>
TRAEFIK_ENABLE=true
MINDBLOSSOM_HOST=app.mindblossom.app
```

> **Traefik labels are already in the docker-compose.yml** — activated by setting
> `TRAEFIK_ENABLE=true` and the `*_HOST` variable. No code changes needed.

---

## Step 9 — Run migrations

In Dokploy → each application → **Terminal**:

```bash
# comm-gateway
mix ecto.migrate

# mindblossom
mix ecto.migrate
```

---

## Step 10 — Update Twilio webhook

Once comm-gateway is deployed and healthy at `https://comms.mindblossom.app/health`:

```
# In a Claude session with the Twilio MCP loaded:
# "Update the Twilio webhook for +15109747342 to https://comms.mindblossom.app/v1/webhooks/twilio/sms"
```

ngrok is now retired for daily use.

---

## Step 11 — Smoke test

```bash
curl https://comms.mindblossom.app/health   # {"status":"healthy",...}
curl https://app.mindblossom.app/health     # {"status":"healthy",...}
# Send an SMS to +15109747342 — should appear in feed within seconds
```

---

## Secrets management

Generate all production secrets before deploying. Never reuse dev values:

```bash
openssl rand -base64 64   # SECRET_KEY_BASE, GUARDIAN_SECRET_KEY
openssl rand -hex 32      # service tokens
```

Store in a password manager (1Password, Bitwarden). Dokploy's env UI is the
only place production secrets should exist — not in any file, not in git.

---

## Architecture notes

- Services communicate internally via Docker service names (`http://comm-gateway:4001`)
- Traefik handles all public HTTPS routing and certificate renewal — activated per-service by `TRAEFIK_ENABLE=true`
- PostgreSQL and Redis: internal only, no public ports
- TigerBeetle: requires `--privileged` in Dokploy service config
- Dokploy admin panel: Tailscale-only, never exposed publicly
- Each service deploys independently — pushing to mindblossom never touches comm-gateway

See [ADR-006](../decisions/ADR-006-dokploy.md) for deployment rationale.
