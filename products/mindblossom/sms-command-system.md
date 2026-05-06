# SMS Command System — Technical Design

The SMS command system allows users to trigger behaviors by including special symbol-prefixed tokens in any SMS message. The architecture is extensible: new symbols map to new handler modules without modifying the parser or dispatcher.

---

## Processing pipeline

Every inbound SMS passes through this pipeline in order:

```
inbound SMS text
      │
      ▼
  1. Tokenizer
     Splits message into typed tokens:
     URL | Hashtag | SlashCommand | PlainText
      │
      ▼
  2. CommandParser
     Extracts command tokens from the token list.
     Separates content tokens (URLs, plain text) from command tokens.
      │
      ├──► Content tokens → normal capture flow (SmsMessage + Link records)
      │
      └──► Command tokens → CommandDispatcher
                                │
                                ▼
                         3. SymbolRegistry
                            Looks up symbol → handler module
                                │
                                ▼
                         4. Handler.execute/3
                            Each handler runs in a transaction with the
                            newly created SmsMessage as context
```

A message with only command tokens and no content (e.g. `#research` sent alone) applies commands retroactively to the most recent captured item and is stored with `is_hidden: true` so it doesn't appear in the feed.

---

## Tokenizer

`Mindblossom.Messaging.Tokenizer` parses raw SMS text into a list of typed tokens.

```elixir
@type token ::
  {:url, String.t()}
  | {:hashtag, String.t()}        # "#research" → {:hashtag, "research"}
  | {:slash_command, String.t()}  # "/new Reading List" → {:slash_command, "new Reading List"}
  | {:text, String.t()}

@spec tokenize(String.t()) :: [token()]
```

**URL detection:** Uses a regex anchored to `http(s)://` or bare domain patterns. URL fragments (`#section`) are part of the URL token and are never parsed as hashtags.

**Hashtag detection:** A `#` followed by one or more word characters (`\w`) or hyphens, not immediately preceded by `://`. Normalized to lowercase.

**Slash command detection:** `/` at the start of a token followed by a known command verb. Unknown verbs are passed through as `:text` with a warning log.

---

## Symbol registry

The registry maps symbols to handler modules. It is defined in application config and consulted at dispatch time.

```elixir
# config/config.exs
config :mindblossom, :command_handlers, %{
  "#" => Mindblossom.Commands.TagHandler,
  "/" => Mindblossom.Commands.SystemCommandHandler
}
```

Adding a new symbol requires only a new config entry and a module implementing `CommandHandler`. No changes to the parser or dispatcher.

**Future: per-user symbol assignments**

A `symbol_assignments` table will allow users to reassign symbols to different behaviors (e.g. map `@` to a "send to contact" handler):

```
symbol_assignments
  id            uuid PK
  user_id       uuid FK → users
  symbol        text        (single character or short prefix)
  handler       text        (module name as string)
  config        jsonb       (handler-specific options)
  created_at    utc_datetime_usec
  updated_at    utc_datetime_usec
```

At dispatch time, the registry first checks for a user-level override, then falls back to the global config.

---

## CommandHandler behaviour

Every handler implements this behaviour:

```elixir
defmodule Mindblossom.Commands.CommandHandler do
  @type item :: %{type: :sms_message | :email | :note, id: Ecto.UUID.t()}
  @type context :: %{user: User.t(), message: SmsMessage.t() | nil, retroactive_item: item() | nil}

  @callback execute(payload :: String.t(), context :: context()) ::
              :ok
              | {:ok, reply :: String.t()}  # send SMS reply back to user
              | {:error, reason :: term()}
end
```

- `payload` is the token content after the symbol (e.g. `"research"` for `#research`, `"new Reading List"` for `/new Reading List`)
- `context.message` is the SmsMessage just created (nil if retroactive)
- `context.retroactive_item` is the most recent item if the message was command-only
- Returning `{:ok, reply}` causes the dispatcher to enqueue a reply SMS via comm-gateway

---

## TagHandler

`Mindblossom.Commands.TagHandler` implements the `#` symbol.

**Algorithm:**
1. Normalize payload to lowercase slug: `"Reading List"` → `"reading-list"`
2. Find or create a `Tag` for the user with that slug
3. Determine target item:
   - If `context.message` has content tokens → tag that message
   - If retroactive → tag `context.retroactive_item`
4. Insert into the appropriate join table (`sms_message_tags`, `email_tags`, or `note_tags`) with `source: "sms_command"`
5. Return `:ok` (no SMS reply for tag commands)

**New tag color:** When auto-creating a tag from SMS, color defaults to `"gray"`. The user can update the color in the UI.

---

## SystemCommandHandler

`Mindblossom.Commands.SystemCommandHandler` implements the `/` symbol.

Dispatches to sub-handlers based on the verb:

| Command | Action |
|---------|--------|
| `/new <name>` | Creates a collection; replies with confirmation + slug |
| `/lists` | Replies with collection list (name + count, up to 10) |
| `/help` | Replies with a brief command reference |
| Unknown | Replies with: `Unknown command. Text /help for a list of commands.` |

---

## Data model additions

### Tags table

Add `slug` column (derived from `name`, stored for fast SMS lookup):

```sql
ALTER TABLE tags ADD COLUMN slug text;
CREATE UNIQUE INDEX tags_user_id_slug_index ON tags (user_id, slug) WHERE user_id IS NOT NULL;
```

Slug derivation: lowercase, strip non-alphanumeric except hyphens, collapse multiple hyphens. Applied in `Tag.changeset/2` via a `put_change` before validation.

### Tag join tables

Add `source` column to all three join tables:

```sql
ALTER TABLE sms_message_tags ADD COLUMN source text NOT NULL DEFAULT 'manual';
ALTER TABLE email_tags        ADD COLUMN source text NOT NULL DEFAULT 'manual';
ALTER TABLE note_tags         ADD COLUMN source text NOT NULL DEFAULT 'manual';
```

Valid values: `manual` | `sms_command` | `ai_suggested`

### Collections

```sql
CREATE TABLE collections (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  name        text NOT NULL,
  slug        text NOT NULL,
  description text,
  created_at  utc_datetime_usec NOT NULL,
  updated_at  utc_datetime_usec NOT NULL
);

CREATE UNIQUE INDEX collections_user_id_slug_index ON collections (user_id, slug);
CREATE INDEX collections_user_id_index ON collections (user_id);
```

**Collection memberships** — separate join tables per item type (avoids polymorphic FK complications):

```sql
CREATE TABLE collection_sms_messages (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  collection_id   uuid NOT NULL REFERENCES collections(id) ON DELETE CASCADE,
  sms_message_id  uuid NOT NULL REFERENCES sms_messages(id) ON DELETE CASCADE,
  added_by        text NOT NULL DEFAULT 'user',  -- 'user' | 'sms_command'
  added_at        utc_datetime_usec NOT NULL
);
CREATE UNIQUE INDEX ON collection_sms_messages (collection_id, sms_message_id);

-- mirror tables for emails and notes:
-- collection_emails  (collection_id, email_id)
-- collection_notes   (collection_id, note_id)
```

### AI tag suggestions

```sql
CREATE TABLE tag_suggestions (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  item_type   text NOT NULL,   -- 'sms_message' | 'email' | 'note'
  item_id     uuid NOT NULL,
  tag_name    text NOT NULL,
  status      text NOT NULL DEFAULT 'pending',  -- 'pending' | 'accepted' | 'rejected'
  created_at  utc_datetime_usec NOT NULL,
  updated_at  utc_datetime_usec NOT NULL
);
CREATE INDEX tag_suggestions_item_index ON tag_suggestions (item_type, item_id);
CREATE INDEX tag_suggestions_user_status ON tag_suggestions (user_id, status);
```

---

## AI tagging pipeline

> **Status: blocked** — requires ai-gateway, which is not yet built.

Once ai-gateway is available, the pipeline is:

```
Link enrichment completes (FetchLinkMetadata worker)
      │
      ▼
Enqueue SuggestTags worker (if user has no manual tags on this item)
      │
      ▼
SuggestTags worker
  Builds prompt: item title + description (or note body)
  POST /v1/chat → ai-gateway with structured output schema
  Receives [{tag_name, confidence}] list
  Inserts into tag_suggestions (status: pending, top 3 by confidence)
      │
      ▼
UI shows suggestions as ghost pills on the item card
User taps → accepted: creates Tag (if new) + join record (source: ai_suggested)
           → dismisses: status → rejected (never shown again for that item)
```

The `SuggestTags` worker respects per-user limits (future): if a user is on a free tier with a monthly AI call budget, the worker checks before calling ai-gateway and discards gracefully if over budget.

---

## Extensibility summary

To add a new symbol behavior (e.g. `@contact` to forward an item):

1. Write a module implementing `CommandHandler` behaviour
2. Add one line to `config/config.exs`:
   ```elixir
   "@" => Mindblossom.Commands.ForwardHandler
   ```
3. The tokenizer, dispatcher, and pipeline require no changes

For per-user customization (future), a user can override any symbol via the `symbol_assignments` table — the dispatcher checks user overrides before falling back to global config.
