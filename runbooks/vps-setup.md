# Runbook: VPS Setup — Hostinger + Dokploy

This runbook covers provisioning the Project Whirlwind VPS and deploying
the first two services (mindblossom + comm-gateway).

**Provider:** Hostinger KVM
**Stack:** Ubuntu 24.04 LTS, Dokploy, Docker, Traefik, Let's Encrypt

---

## Prerequisites

Before starting, have these ready:

- [ ] Hostinger account
- [ ] Domain name decided and registrar access (to set DNS)
- [ ] SSH public key (`~/.ssh/id_ed25519.pub` or similar)
- [ ] Twilio Account SID + Auth Token
- [ ] Mailgun API key + webhook signing key (once domain is sorted)
- [ ] GitHub personal access token with `repo` scope (for Dokploy to pull repos)

---

## Step 1 — Provision the VPS

In Hostinger hPanel → VPS → Create new:

| Setting | Value |
|---|---|
| Plan | **KVM 2** (2 vCPU, 8GB RAM, 100GB NVMe) |
| OS | **OS with Panel → Dokploy** (ships with Ubuntu 24.04 LTS) |
| Datacenter | Closest to primary users |
| SSH key | Paste your public key |

> **Why KVM 2?** Comfortable headroom for mindblossom + comm-gateway + postgres
> + redis + TigerBeetle + Dokploy/Traefik now, with room to add ai-gateway
> and the blogging app without migrating.

> **Why 24.04 LTS and not 25.10?** Non-LTS Ubuntu releases get 9 months of
> security support. 24.04 LTS is supported until April 2029. Always use LTS
> on servers.

---

## Step 2 — DNS

As soon as Hostinger provides the VPS IP, create these DNS records at your registrar:

| Type | Name | Value | TTL |
|---|---|---|---|
| A | `@` | `YOUR_VPS_IP` | 300 |
| A | `app` | `YOUR_VPS_IP` | 300 — mindblossom |
| A | `comms` | `YOUR_VPS_IP` | 300 — comm-gateway |
| A | `in` | `YOUR_VPS_IP` | 300 — Mailgun inbound email |

DNS propagation takes 5–30 minutes. Traefik can't issue TLS certificates
until it resolves, so set DNS before configuring Dokploy.

---

## Step 3 — First SSH login

```bash
ssh root@YOUR_VPS_IP

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

# Firewall
ufw allow 22 && ufw allow 80 && ufw allow 443 && ufw allow 3000
ufw enable

# Verify Dokploy is running (pre-installed by Hostinger)
systemctl status dokploy
```

Dokploy UI is at `http://YOUR_VPS_IP:3000` — first visit creates your admin account.

---

## Step 4 — Dokploy initial configuration

1. Open `http://YOUR_VPS_IP:3000` and create the admin account
2. **Settings → Git Providers → GitHub** — connect your GitHub account or add a deploy key for the `project-whirlwind` org
3. **Settings → Domain** — set your Dokploy UI domain (e.g., `dokploy.yourdomain.com`) so the panel gets HTTPS
4. **Settings → Let's Encrypt** — add your email for cert notifications

---

## Step 5 — Deploy comm-gateway

comm-gateway must be deployed first — mindblossom's inbound webhook receiver depends on it being reachable.

In Dokploy → **Create Application**:

| Field | Value |
|---|---|
| Name | `comm-gateway` |
| Repository | `github.com/Project-Whirlwind/comm-gateway` |
| Branch | `main` |
| Build type | Docker Compose |

**Environment variables** (set in Dokploy UI, never in git):

```
MIX_ENV=prod
PORT=4001
PHX_SERVER=true
DATABASE_URL=ecto://comm_gateway:STRONG_PASSWORD@postgres:5432/comm_gateway_prod
SECRET_KEY_BASE=<generate: mix phx.gen.secret>
SERVICE_TOKEN=<generate: mix phx.gen.secret | head -c 32>
OUTBOUND_SERVICE_TOKEN=<same token — must match COMM_GATEWAY_SERVICE_TOKEN in mindblossom>
TWILIO_ACCOUNT_SID=AC...
TWILIO_AUTH_TOKEN=...
MAILGUN_API_KEY=...
MAILGUN_DOMAIN=...
MAILGUN_WEBHOOK_SIGNING_KEY=...
SMS_EVENT_SUBSCRIBER_URL=http://mindblossom:4000/v1/internal/sms_received
EMAIL_EVENT_SUBSCRIBER_URL=http://mindblossom:4000/v1/internal/email_received
```

> comm-gateway's `docker-compose.yml` needs Traefik labels added before first
> deploy — see Step 7 below.

---

## Step 6 — Deploy mindblossom

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
PHX_HOST=app.yourdomain.com
DATABASE_URL=ecto://mindblossom:STRONG_PASSWORD@postgres:5432/mindblossom_prod
SECRET_KEY_BASE=<generate: mix phx.gen.secret>
GUARDIAN_SECRET_KEY=<generate: mix phx.gen.secret | head -c 64>
COMM_GATEWAY_URL=http://comm-gateway:4001
COMM_GATEWAY_SERVICE_TOKEN=<same as OUTBOUND_SERVICE_TOKEN in comm-gateway>
INBOUND_EMAIL_DOMAIN=in.yourdomain.com
```

---

## Step 7 — Add Traefik labels to service docker-compose files

Before deploying, each service needs Traefik routing labels added to its
`docker-compose.yml`. This is a one-time code change — commit and push, then
trigger the deploy in Dokploy.

**comm-gateway:**
```yaml
services:
  app:
    build:
      context: .
      target: prod
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.comm-gateway.rule=Host(`comms.yourdomain.com`)"
      - "traefik.http.routers.comm-gateway.tls.certresolver=letsencrypt"
      - "traefik.http.services.comm-gateway.loadbalancer.server.port=4001"
    # No ports: section — Traefik handles external routing
```

**mindblossom:**
```yaml
services:
  app:
    build:
      context: .
      target: prod
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.mindblossom.rule=Host(`app.yourdomain.com`)"
      - "traefik.http.routers.mindblossom.tls.certresolver=letsencrypt"
      - "traefik.http.services.mindblossom.loadbalancer.server.port=4000"
```

> **Important:** Database services (postgres, redis) must have **no `ports:`
> section** in production compose files. They're internal-only. Only Phoenix
> app services get Traefik labels and external exposure.

---

## Step 8 — Run migrations

After first deploy, run migrations via Dokploy's terminal or SSH:

```bash
# In Dokploy → comm-gateway → Terminal
mix ecto.migrate

# In Dokploy → mindblossom → Terminal
mix ecto.migrate
```

---

## Step 9 — Update Twilio webhook to VPS URL

Once comm-gateway is deployed and healthy:

```
# In a Claude session with the Twilio MCP loaded:
# "Update the Twilio webhook for +15109747342 to https://comms.yourdomain.com/v1/webhooks/twilio/sms"
```

This replaces the ngrok URL permanently. ngrok is no longer needed for daily use.

---

## Step 10 — Smoke test

```bash
# comm-gateway health
curl https://comms.yourdomain.com/health

# mindblossom health
curl https://app.yourdomain.com/health

# Send a real SMS to +15109747342 and confirm it appears in the feed
```

---

## Passwords and secrets

Generate all production secrets before deploying. Never reuse dev values:

```bash
# In any Elixir environment (or locally if mix is installed)
mix phx.gen.secret          # SECRET_KEY_BASE (64+ chars)
mix phx.gen.secret | head -c 32   # shorter tokens

# Or use openssl
openssl rand -hex 32        # service tokens
openssl rand -base64 64     # secret key base
```

Store the values in a password manager (1Password, Bitwarden) — not in any file
that could be committed. Dokploy's environment variable UI is the only place
they should live.

---

## Architecture notes

- Services communicate internally using Docker service names (`http://comm-gateway:4001`)
- Traefik handles all public HTTPS routing and certificate renewal
- PostgreSQL and Redis are internal-only — no public ports
- TigerBeetle requires `--privileged` in the Dokploy service config (set at service level)
- Each service deploys independently — deploying mindblossom never touches comm-gateway

See [ADR-006](../decisions/ADR-006-dokploy.md) for the full deployment rationale.
