# Developer Onboarding

## Which track do you need?

| I need to... | Track |
|---|---|
| Deploy to production or manage env vars | Track 1 |
| Write code and run the stack locally | Track 2 |
| Test Twilio/Mailgun webhooks locally | Track 2 + 3 |

---

## Track 1 — Production Access

Needed to deploy services, view production logs, or manage environment variables. Production is locked behind Tailscale (a private VPN) — the Dokploy admin panel is not exposed to the internet.

### Step 1 — Join the Tailscale network

1. Create a Tailscale account at [tailscale.com](https://tailscale.com) — use your PWW Github org email
2. Install: `brew install --cask tailscale` (Mac) or [tailscale.com/download](https://tailscale.com/download)
3. Sign in to the Tailscale app
4. **Tell whoever manages the tailnet your Tailscale account email** — they'll invite you from [login.tailscale.com/admin/users](https://login.tailscale.com/admin/users)
5. Accept the invite email
6. Confirm you're on the network: `tailscale status` — you should see `whirlwind-prod`

### Step 2 — Access Dokploy

Open in your browser:

```
http://whirlwind-prod:3000
```

Your Dokploy account will be created by whoever manages the instance (Settings → Users → Create User). Credentials come via the team password manager — not email.

### Step 3 — Production secrets

Env vars are managed in the Dokploy UI only — never stored in git or shared over Slack. They'll be shared with you via the team password manager (1Password/Bitwarden). See [vps-setup.md](vps-setup.md) Steps 7–8 for the full list per service.

### Step 4 — SSH access (rarely needed)

Only for direct OS-level work on the server. Share your public key with whoever manages the VPS:

```bash
cat ~/.ssh/id_ed25519.pub
```

Once added: `ssh deploy@<whirlwind-prod-tailscale-ip>` — never SSH as root.

---

## Track 2 — Local Dev

**Before you start:**
- Accept the GitHub org invite for `project-whirlwind` (someone on the team sends this)
- Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
- Generate an SSH key if you don't have one: `ssh-keygen -t ed25519`

**Clone the repos into the same parent folder:**

```bash
git clone git@github.com:project-whirlwind/infra-local
git clone git@github.com:project-whirlwind/mindblossom
git clone git@github.com:project-whirlwind/comm_gateway
```

**Start the stack:**

```bash
cd infra-local
make init   # copies .env.example → .env, builds images
make up     # starts all services
```

**Verify everything is running:**

```bash
curl http://localhost:4001/health   # comm-gateway → {"status":"healthy",...}
curl http://localhost:4000/health   # mindblossom  → {"status":"healthy",...}
```

**Day-to-day:**

```bash
make logs service=mindblossom    # tail logs
make shell service=mindblossom   # IEx shell
make migrate                     # run migrations
make help                        # full command list
```

---

## Track 3 — Webhook Testing (optional)

Needed only if you're testing Twilio/Mailgun webhooks locally. You'll need a free [ngrok](https://ngrok.com) account with a static domain.

1. Sign up at ngrok.com → Domains → claim a free static domain
2. Add to `infra-local/.env`:

```
NGROK_AUTHTOKEN=your-auth-token
NGROK_DOMAIN=your-domain.ngrok-free.app
```

3. Run: `docker compose up ngrok -d`

Inspector runs at `http://localhost:4040`.

> The Twilio number can only point to one webhook at a time — coordinate with the team before redirecting it.

---

## Related docs

- [vps-setup.md](vps-setup.md) — production infrastructure
- [ADR-006-dokploy.md](../decisions/ADR-006-dokploy.md) — why Dokploy
