# Git Workflow

## Branch model

```
main          ← always deployable, protected
  └── feat/short-description
  └── fix/short-description
  └── chore/short-description
  └── docs/short-description
```

- `main` is the only permanent branch
- All work happens in short-lived branches off `main`
- No `develop`, `staging`, or `release` branches — keep it simple

---

## Branch naming

```
feat/sms-inbound-routing
fix/twilio-signature-validation
chore/update-plug-cowboy
docs/add-api-contract-for-comm-gateway
adr/add-redis-rate-limiting
```

Lowercase, hyphen-separated, no slashes beyond the prefix.

---

## Commit messages

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer]
```

**Types:**

| Type | When |
|------|------|
| `feat` | New feature |
| `fix` | Bug fix |
| `chore` | Dependency updates, tooling, no logic change |
| `docs` | Documentation only |
| `refactor` | Code change with no behavior change |
| `test` | Adding or fixing tests |
| `perf` | Performance improvement |
| `revert` | Reverting a previous commit |

**Examples:**
```
feat(comm-gateway): add Mailgun inbound webhook handler
fix(auth): correct JWT expiry calculation
chore: bump phoenix from 1.7.10 to 1.7.14
docs(adr): add ADR-007 comm-gateway pattern
```

---

## Pull requests

**Title:** Same format as a commit message — `type(scope): description`

**Description template** (add `.github/pull_request_template.md` to each repo):

```markdown
## What
Brief description of the change.

## Why
Why is this change needed? Link to issue or ADR if applicable.

## How
Key implementation decisions. Anything a reviewer should know.

## Testing
How was this tested? What should the reviewer check?

## Checklist
- [ ] Tests added or updated
- [ ] API contract updated if interface changed
- [ ] No secrets committed
- [ ] Health check still passes
```

**Rules:**
- PRs target `main`
- At least one approval required before merge (when team > 1)
- Squash merge — keeps `main` history clean and bisectable
- Delete branch after merge
- CI must pass before merge

---

## Release tagging

Tag releases on `main` using semantic versioning:

```bash
git tag v1.2.3
git push origin v1.2.3
```

- **Patch** (`v1.2.3 → v1.2.4`): bug fix, no API change
- **Minor** (`v1.2.3 → v1.3.0`): new feature, backwards-compatible
- **Major** (`v1.2.3 → v2.0.0`): breaking API change

Tag on `api-contracts` triggers a notice to all consuming services.

---

## What never goes in a commit

- Secrets, API keys, tokens (use environment variables)
- `.env` files (`.env.example` is fine)
- Generated files that can be reproduced (`_build/`, `deps/`, `node_modules/`)
- Binary files or large assets (use object storage)
