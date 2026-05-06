# MindBlossom — Product Requirements

MindBlossom is a personal capture-and-retrieval tool. Users send anything — links, notes, thoughts — via SMS or email and the system saves, enriches, and organizes it. The interface is SMS-first: a user should be able to do everything meaningful without opening a browser.

---

## Current state (as of 2026-05-05)

| Feature | Status |
|---------|--------|
| User registration (phone + email) | Shipped |
| Assigned email inbox per user | Shipped |
| Inbound SMS capture | Shipped |
| Inbound email capture | Shipped |
| Link extraction from SMS and email | Shipped |
| OG metadata enrichment (title, description, image) | Shipped |
| Feed UI (chronological, SMS + email + notes) | Shipped |
| Link detail view | Shipped |
| Note detail view | Shipped |
| Tag data model + global/per-user tags | Shipped (schema only — no UI or SMS assignment) |
| Tag assignment via UI | Not started |
| Tag assignment via SMS commands | Not started |
| AI-suggested tags | Blocked (ai-gateway not built) |
| Collections | Not started |
| Note creation via UI | Not started |
| Edit (any item type) | Not started |
| Welcome SMS + email on registration | Not started |

---

## Feature requirements

### 1. Capture

Already working. Defined here for completeness.

**SMS capture:** Any inbound SMS from a registered phone number is saved as an `sms_message`. Links in the body are extracted to `links` and enqueued for OG metadata fetch. Non-link content becomes the message body.

**Email capture:** Any email to the user's assigned address (`{handle}@in.mindblossom.app`) is saved as an `email`. Links in the body are extracted to `email_links`.

**Note (no-link SMS):** An inbound SMS with no URL is a note. It is saved as an `sms_message` and also creates a `note` record with `source: "sms_message"`. This is the current schema — the feed already treats SMS-sourced notes as notes.

---

### 2. SMS Command System

Users can include special symbol-prefixed tokens in any SMS message to trigger behaviors. The system is designed to be extensible: new symbols can be registered to new behaviors without changing the parser.

See [sms-command-system.md](./sms-command-system.md) for full technical design.

**Default symbol assignments:**

| Symbol | Behavior |
|--------|----------|
| `#word` | Apply tag `word` to items in this message (or most recent item if standalone) |
| `/command` | System commands: `/new`, `/lists`, `/help` |

**User-defined symbol reassignment** is a future capability — stored in a `symbol_assignments` table per user, allowing symbols like `@` or `!` to be mapped to custom behaviors.

**Tag via SMS — inline:**
```
https://vercel.com/ai-sdk #research #tooling
```
→ saves the link and applies tags `research` and `tooling`

**Tag via SMS — retroactive (standalone):**
```
User sends: https://example.com        → captured, untagged
User sends: #research                  → applies "research" to the most recent item
```

**System commands:**
```
/new Reading List     → creates a collection named "Reading List"
/lists                → replies with collection names and item counts
/help                 → replies with a brief command reference
```

**Rules:**
- Tags are case-insensitive, normalized to lowercase, hyphens allowed (`#reading-list`)
- `#` fragments in URLs (e.g. `https://docs.com/guide#section`) are excluded from tag parsing
- A standalone `#tag` or `/command` message (no other content) applies retroactively to the most recent captured item and is itself hidden from the feed (`is_hidden: true`)
- Multiple tags in one message all apply
- Unknown `/commands` receive a helpful error reply via SMS

---

### 3. Tags

Tags are user-defined labels applied to any captured item (SMS message, email, note).

**Data model:** Tag schema and join tables already exist. Additions needed:
- `slug` field on `tags` (url-safe lowercase, hyphens — derived from name, used for SMS lookup)
- `source` field on tag join tables: `manual | sms_command | ai_suggested`

**Tag management (UI):**
- Create tag (name + color picker)
- Rename tag
- Delete tag (removes all assignments)
- Merge tags

**Tag assignment (UI):**
- Add/remove tags on any item from the detail view
- Tag pills on feed cards with single-tap add/remove

**Tag assignment (SMS):**
- Via `#tagname` as described above
- If the tag doesn't exist, it is created automatically (no color — shown as neutral gray until user picks one in the UI)

**AI-suggested tags** (blocked on ai-gateway):
- After link enrichment completes, enqueue an AI job to suggest 1–3 tags from the item's title and description
- Suggestions stored in `tag_suggestions` table with `status: pending`
- Shown in the UI as ghost pills the user can accept (one tap) or dismiss
- Accepted suggestions create a real tag assignment with `source: ai_suggested`
- If a user has already manually tagged an item, AI skips suggestions for that item
- On by default; per-user controls and tier limits are a future addition

---

### 4. Collections

A collection is a named, user-curated list of captured items. Items can belong to multiple collections. Collections are distinct from tags: tags are labels, collections are curated lists.

**Data model:** To be built. See [sms-command-system.md](./sms-command-system.md) for schema.

**Creating a collection:**
- Via SMS: `/new Reading List`
  - Reply: `"Reading List" created. Text any message with #reading-list to add it.`
  - System auto-derives a slug (`reading-list`) as the SMS shorthand
- Via UI: "New collection" button, name input

**Adding items to a collection:**
- Via SMS: include `#slug` in the message (the slug matches the collection's slug, not a free tag)
- Via UI: "Add to collection" action on any item's detail view or via context menu on feed cards

**Collection feed:**
- Collections panel in the sidebar (name + item count)
- Selecting a collection filters the main feed to items in that collection

**Collection management (UI only):**
- Rename, delete
- Remove individual items from collection
- No SMS management beyond creation

**Interaction with tags:**
- A collection does not automatically pull in all items with a matching tag
- Items must be explicitly added to a collection (via SMS command or UI)
- The collection slug and tag slug share the same namespace; creating a collection with `/new Reading List` does not create a tag named `reading-list`

---

### 5. Note Creation

**SMS-sourced notes** already work (any SMS with no URL is stored as a note).

**Manual note creation (UI):**
- "New note" button in the feed header
- Simple textarea — title (optional) and body
- Notes appear in the feed alongside SMS and email captures, sorted by created_at

**Future:** voice-to-note via SMS (user sends a voice memo → transcribed and stored as note).

---

### 6. Edit

Edit is UI-only for all item types. No SMS edit flow.

**Editable fields by item type:**

| Item | Editable fields |
|------|----------------|
| SMS message | Body (corrects transcription errors), tags |
| Email | Subject (display title), tags |
| Note | Title, body, tags |
| Link (any source) | Display title (overrides OG title), tags |

**Behavior:**
- Original captured content is preserved in a `raw_*` field or metadata map
- Edits update the display field only — the original is never lost
- Last-edited timestamp shown in detail view

---

## Phasing

### Now (current sprint)
- Welcome SMS + email on registration (see separate plan)
- Tag assignment via UI (completes the existing schema)

### Next
- SMS command parser and `#tag` assignment via SMS
- Collection data model and `/new` command
- Note creation via UI

### Later (blocked or lower priority)
- AI-suggested tags (blocked on ai-gateway)
- Per-user symbol reassignment
- Voice-to-note
- Edit (any item type)
- Tag/collection tier controls
