# iOS blueprint plan (MindBlossom first, Shelves next)

Status: Draft for review. Author: Fable, 2026-07-07.
Tracking: platform-docs#2. Do not create the `ios-blueprint` repo until this is approved.

This plan is executable by Opus/Sonnet subagents: each milestone is small,
independently buildable, and has acceptance criteria. Section 1 is a decision,
not an assumption, and must be settled before any code.

---

## 0. What we are building

Two things, in order:

1. **`ios-blueprint`** — a scaffold repo in the Project-Whirlwind org that any
   Whirlwind product can fork/instantiate. It owns the shared shell: auth
   against a Phoenix backend, secure token storage, a typed API client,
   networking/error conventions, a share-sheet extension pattern, the
   push-or-poll story, project structure, and CI. It does **not** own product
   screens or branding.
2. **`mindblossom-ios`** — the first instantiation. Screens and branding for
   MindBlossom: auth, feed read, note capture, and share-a-link.

Second consumer, later: **Shelves** (Phoenix 1.8 + LiveView book marketplace,
currently local-only; will need map + photo capture). Section 5 covers what it
swaps.

**Blueprint vs product boundary (the rule the implementer follows):**

| Blueprint owns | Product owns |
|----------------|--------------|
| Auth flow + token storage (Keychain) | Login/branding screens' copy and art |
| API client (base URL injected, typed requests, error/retry) | Product endpoints + models |
| Share extension skeleton | What the share target does with the payload |
| Push registration / polling scheduler | Which events matter |
| App scaffold, config, CI, lint | Feature screens, navigation, icons |

---

## 1. Technology decision (settle first)

Evaluate three options against our real constraints. Recommendation below, but
the implementer should confirm it still holds at build time.

### Option A — Native SwiftUI + async/await (recommended)

- **Fit with two Phoenix backends:** neutral. Talks to the JSON API we already
  need to build (mindblossom#65) over `URLSession`. No server coupling.
- **Agent-implementability:** highest. SwiftUI + `URLSession` + `Codable` is
  the most heavily documented iOS stack; Opus/Sonnet write it reliably. No
  hidden bridging layers.
- **Share extension:** first-class. App Extensions are native; the share sheet
  is exactly what we need for "share a link into MindBlossom."
- **Contacts / push:** first-class (`Contacts`, `UserNotifications`).
- **App Store friction:** lowest. No third-party runtime to justify.
- **Maintenance for a solo owner:** low. One language, one toolchain, no JS
  build. The cost is more code per product, which is cheap for an agent.
- **Exit path if wrong:** screens are isolated; a future cross-platform need
  can wrap or replace the view layer without touching the API client.

### Option B — LiveView Native

- **Fit:** superficially attractive — both products are Phoenix/LiveView, so
  server-driven UI could reuse existing LiveViews.
- **Risk (the deciding factor):** the ecosystem is young and moves fast;
  native share extensions, offline capture, and background push do not map
  cleanly onto a server-driven model, and those are core to MindBlossom's
  value (capture a link even with no signal, then sync). Betting the blueprint
  on it exposes every future product to that immaturity.
- **Verdict:** not for the blueprint. Revisit for a read-only companion screen
  later if it matures.

### Option C — React Native / Expo

- **Fit:** mature cross-platform, large ecosystem.
- **Cost:** adds a JS/Node toolchain to an otherwise Elixir + Swift shop, and
  still needs native modules for the share extension and Contacts. That is the
  worst of both worlds for a solo maintainer, and more moving parts for an
  agent to get wrong.
- **Verdict:** only reconsider if an Android build becomes a near-term
  requirement. It is not today.

**Recommendation: Option A (native SwiftUI).** Boring, well-documented,
first-class access to the exact OS features MindBlossom depends on, lowest
maintenance for one person. Record the decision as an ADR in platform-docs when
approved.

---

## 2. API prerequisites (blocking)

The app cannot start until the backend exposes:

- **mindblossom#65** — personal API tokens + read endpoints (feed export,
  single entry) + create-note endpoint. List the exact routes and payloads
  there; the client in Milestone 2 is generated against them.
- **mindblossom#63** — contacts import endpoint (for the Contacts sync
  milestone; not needed for v1 capture, so it only blocks the later milestone).

Auth model: Guardian bearer token minted on the API page, stored in Keychain.
No OAuth in v1.

---

## 3. Milestones (each independently buildable, with acceptance)

### Blueprint milestones

**M1 — Repo scaffold + CI.** SwiftUI app target, config for injected API base
URL (scheme/plist), SwiftLint, GitHub Actions building the app and running unit
tests on a simulator.
_Acceptance:_ CI green on an empty app that launches to a placeholder screen.

**M2 — API client.** Typed `APIClient` over `URLSession`: base URL from config,
bearer-token header, `Codable` request/response, uniform error type, one retry
on transient failure. Unit-tested against a stub `URLProtocol`.
_Acceptance:_ a documented GET and POST round-trip through the client with a
mocked server; errors surface as typed values.

**M3 — Auth + Keychain.** Paste-token (or login) flow that stores the bearer in
Keychain, injects it into `APIClient`, and clears on sign-out.
_Acceptance:_ token persists across app restart; sign-out removes it; requests
carry it.

**M4 — Share extension skeleton.** App Extension that accepts a URL/text from
the system share sheet and hands it to a product-defined handler via the shared
app group; queues locally if offline.
_Acceptance:_ sharing a Safari link shows MindBlossom in the sheet and enqueues
the URL even in airplane mode.

### MindBlossom product milestones

**M5 — Feed read.** List the user's entries from the export endpoint (paginated)
with pull-to-refresh. Read-only.
_Acceptance:_ entries render, pagination works, empty and error states handled.

**M6 — Note capture.** Compose + POST a note via the create endpoint; optimistic
insert into the feed.
_Acceptance:_ a note created on device appears server-side and in the feed.

**M7 — Share target wired.** The M4 extension posts the shared link as a note
(or link entry) via the API client; the offline queue drains on next launch.
_Acceptance:_ a link shared offline lands as an entry once connectivity
returns.

**M8 — Contacts sync (needs mindblossom#63).** Read device contacts (with
permission) and bulk-import via the contacts endpoint.
_Acceptance:_ granting permission imports contacts idempotently; re-run does
not duplicate.

Push vs poll: v1 uses pull-to-refresh + share-queue drain (no server push).
APNs is a later milestone once there is a reason to push.

---

## 4. Positioning / App Store metadata

Align with the ownership message. MindBlossom is user-owned; contrast with
PE-owned assistants (Perplexity, Poke) and PayPal-owned Honey. Revenue-share /
co-op intent (revenue-ledger is the platform seam) is stated as **roadmap**, not
a v1 feature claim — do not imply payouts exist. Keep store copy about the
product (capture from anywhere, your AI second brain, your data exportable);
put the ownership philosophy in the longer description and the site, not in
feature bullets that could read as promises.

---

## 5. Instantiating the blueprint for Shelves (later)

What Shelves swaps when it forks `ios-blueprint`:
- **API base URL + models** — points at the Shelves Phoenix backend; its own
  endpoints and `Codable` types.
- **Auth** — Shelves uses `phx.gen.auth` (email + password + display name);
  the blueprint's paste-token flow may need a real login screen variant. Note
  this as the one place the blueprint's auth abstraction must stay flexible.
- **Map + photo** — Shelves needs `MapKit` (book pins) and camera/photo capture
  for listings; these are product modules, not blueprint. The blueprint should
  not assume no-camera.
- **Share extension** — likely unused by Shelves v1; the skeleton stays dormant.

Keep the blueprint's assumptions minimal enough that Shelves does not fight it:
inject the base URL, keep auth swappable, do not hardcode "text/link capture"
as the only content type.

---

## 6. Open questions for the reviewer

- Confirm native SwiftUI over LiveView Native (Section 1). If you want to pilot
  LiveView Native anyway, scope it as a throwaway read-only spike, not the
  blueprint.
- Token paste vs full in-app login for v1 auth (M3). Paste is faster to ship;
  login is friendlier. Recommend paste for v1, login when the API page supports
  a mobile-friendly flow.
- Android horizon: if it is within a year, revisit Option C now rather than
  later.
