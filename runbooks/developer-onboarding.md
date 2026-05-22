# Runbook: Developer Onboarding

This runbook covers getting a new developer set up on Project Whirlwind — local dev environment first, production access second (only for devs who need it).

---

## Prerequisites

- Added to the `project-whirlwind` GitHub org (ask the team lead)
- Docker Desktop installed locally
- SSH key generated (`~/.ssh/id_ed25519` or similar)

---

## Track 1 — Local Dev (everyone)

### Step 1 — Clone the repos

The local stack uses Docker Compose with sibling directories. Clone them all into the same parent folder:

```bash
git clone git@github.com:project-whirlwind/infra-local
git clone git@github.com:project-whirlwind/mindblossom
git clone git@github.com:project-whirlwind/comm_gateway
```

Your directory layout should look like:

```
~/Projects/
  infra-local/
  mindblossom/
  comm_gateway/
```

### Step 2 — First-time setup

```bash
cd infra-local
make init    # copies .env.example → .env, builds images, formats TigerBeetle data file
make up      # starts postgres, redis, tigerbeetle, comm-gateway, mindblossom
make ps      # verify all services are healthy
```

Confirm everything is running:

```bash
curl http://localhost:4001/health   # comm-gateway
curl http://localhost:4000/health   # mindblossom
```

Both should return `{"status":"healthy",...}`.

### Step 3 — Day-to-day workflow

```bash
make logs service=mindblossom   # tail logs for one service
make shell service=mindblossom  # open IEx shell
make migrate                    # run Ecto migrations after schema changes
make restart service=mindblossom  # restart without rebuild (Phoenix hot-reloads automatically)
```

Run `make help` for the full command list.

---

## Track 2 — Webhook Testing (optional)

If you need Twilio or Mailgun webhooks to reach your local machine, you need your own ngrok account and static domain. The ngrok service is already wired into `docker-compose.yml` — you just need to supply credentials.

1. Sign up at [ngrok.com](https://ngrok.com) (free tier is sufficient)
2. Go to ngrok dashboard → Domains → claim a free static domain
3. Add to your `infra-local/.env`:

```
NGROK_AUTHTOKEN=your-auth-token
NGROK_DOMAIN=your-domain.ngrok-free.app
```

4. Start ngrok alongside the stack:

```bash
make up        # core services
docker compose up ngrok -d   # tunnel
```

The ngrok web inspector runs at `http://localhost:4040`.

> **Note:** The Twilio phone number can only point to one webhook URL at a time. Coordinate with the team on who owns the webhook when testing SMS locally. See [vps-setup.md](vps-setup.md) for the production webhook URL.

---

## Track 3 — Production Access (deployers only)

Production access requires Tailscale (VPN) and a Dokploy account. Only request this if you need to deploy or debug production services.

### Step 1 — Join the Tailscale tailnet

Dokploy's admin panel runs on port 3000 and is only reachable via Tailscale — it is not exposed to the internet.

1. Install Tailscale: `brew install tailscale` (Mac) or [tailscale.com/download](https://tailscale.com/download)
2. Ask the team lead to invite your email via the Tailscale admin console
3. Accept the invite and run `tailscale up`
4. Confirm connectivity: `tailscale status` — you should see the `whirlwind-prod` server listed

### Step 2 — Access Dokploy

Once on the tailnet:

```
http://<whirlwind-prod-tailscale-ip>:3000
```

Ask the team lead for the server's Tailscale IP and to create your Dokploy account (Settings → Users → Invite User).

### Step 3 — Production secrets

Production environment variables are managed in the Dokploy UI only — they are never stored in git or sent over Slack/email. The team lead will share them via the team password manager (1Password/Bitwarden).

For the full list of required env vars per service, see [vps-setup.md](vps-setup.md), Steps 7–8.

### Step 4 — SSH access (direct server access)

Only needed if you're doing OS-level work on the VPS. Provide your SSH public key to the team lead:

```bash
cat ~/.ssh/id_ed25519.pub
```

They'll add it to `/home/deploy/.ssh/authorized_keys`. You then SSH as:

```bash
ssh deploy@<whirlwind-prod-tailscale-ip>
```

Never SSH as root — the root login is disabled.

---

## Checklist

| Step | Who | Required for |
|------|-----|-------------|
| Added to GitHub org | Team lead | Everyone |
| Clone repos + `make init && make up` | You | Local dev |
| ngrok token + static domain | You | Webhook testing |
| Tailscale invite accepted | Team lead + you | Prod access |
| Dokploy account created | Team lead | Prod access |
| Prod secrets shared via password manager | Team lead | Prod access |
| SSH key added to server | Team lead | Direct server access |

---

## Related docs

- [vps-setup.md](vps-setup.md) — full production infrastructure setup
- [ADR-006-dokploy.md](../decisions/ADR-006-dokploy.md) — why Dokploy was chosen
- [ADR-001-polyrepo.md](../decisions/ADR-001-polyrepo.md) — repo structure rationale
