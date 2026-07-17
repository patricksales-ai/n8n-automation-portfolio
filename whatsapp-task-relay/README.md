# TaskRelay — WhatsApp → AI → Slack + Monday.com

**Client requests arrive as WhatsApp messages. They should end up as tasks on a board.**
TaskRelay closes that gap: it reads every inbound WhatsApp message, uses AI to decide whether
it's actually a request, extracts a structured task from the free-form text, and files it —
Slack alert to the team, item on the Monday.com board — in seconds.

Chit-chat never makes it through. "thanks", "ok", "got it" are filtered out before they
become noise.

> 📖 **[WALKTHROUGH.md](WALKTHROUGH.md)** explains every node in the workflow, line by line.

---

## Why this exists

**The problem —** agencies run client comms on WhatsApp because that's where clients already
are. But WhatsApp is a chat app, not a work tracker. Requests arrive mid-conversation, buried
between small talk and photos, and someone has to read every thread and manually re-type the
real ones into the project tool. Things get missed. The board is always a day behind the chat.

**The result —** the message *is* the task. A client texts "get the new roofing campaign live
by Friday, budget $2k" and it lands on the board — titled, attributed to the right client,
prioritised, summarised with the budget and deadline intact — before anyone has opened the app.

---

## What it does

- **Listens** for inbound WhatsApp messages (WhatsApp Business Cloud API).
- **Normalizes** any message type — text, or a photo/video with the task in the **caption**.
- **Decides** whether the message is a real request at all (`is_task`), so acknowledgements
  never reach the board.
- **Extracts** a structured task: title, client, type, priority, and a summary that preserves
  budget / deadline / targeting detail.
- **Alerts** the team on **Slack** instantly.
- **Files** the task on a **Monday.com** board via GraphQL, mapped to real column IDs, with
  priority landing on Monday's native label.

---

## Architecture

```
WhatsApp Trigger (client texts a task)
   → Edit Fields          (normalize: client_phone · client_name · msg_type · text)
   → Basic LLM Chain      (gpt-4o-mini extracts a structured task)
        └─ OpenAI Chat Model   (Response Format = JSON Object)
   → Normalize            (Code: force a consistent { output: {...} } shape)
   → IF is_task
        ├─ Slack               (alert the team channel)
        └─ Create Monday Item  (GraphQL create_item on the board)
```

Both outputs hang off the IF's `true` branch: the task is **announced and filed**. The `false`
branch is deliberately unconnected — not a task, nothing happens.

### The design decisions that matter

| Choice | Why |
|---|---|
| **Basic LLM Chain**, not an AI Agent | Stateless one-shot extraction — no tools, no cross-message memory. An Agent adds overhead and failure modes for nothing. |
| **JSON Object** response format + a Normalize node | The Structured Output Parser throws *"Failed to parse agent steps"* on self-hosted n8n even for valid JSON. JSON-mode + Normalize is the portable pattern. |
| Priority emitted as **`High`/`Medium`/`Low`** | Matches Monday's native labels *exactly*. Its labels are case-sensitive — a lowercase `high` silently creates a duplicate label instead of matching. Pin the vocabulary in the prompt; don't map downstream. |
| **HTTP + GraphQL**, not the native Monday node | One Header Auth credential, no node-version roulette, and `create_item` stays explicit and portable. |
| Column IDs **introspected, never guessed** | Monday auto-generates them. Status is `project_status`, *not* `status`. Guessing fails silently. |

---

## Tech stack

- **n8n** (self-hosted) — orchestration
- **OpenAI `gpt-4o-mini`** — task extraction
- **WhatsApp Business Cloud API** (Meta) — the inbound channel
- **Slack** — team alerts
- **Monday.com** — the board, via the GraphQL API

---

## Setup

1. **Import the workflow** into n8n (`Workflows → ⋯ → Import from File`):
   [`workflows/whatsapp-task-relay.json`](workflows/whatsapp-task-relay.json).

2. **Create the Monday board** — columns: Status, Priority, Notes, Client, Type, Link.
   Then **introspect the real column IDs** (they won't match the display names):
   ```graphql
   query { boards(ids: YOUR_BOARD_ID) { columns { id title type } } }
   ```
   Map them into the `Create Monday Item` node's `column_values`, and set your board ID in
   place of `YOUR_MONDAY_BOARD_ID`.

3. **Create credentials** and select them on each node:
   - **OpenAI** — API key
   - **Monday** — n8n **Header Auth**: name `Authorization`, value = your API token **raw**
     (Monday does *not* want a `Bearer` prefix)
   - **Slack** — bot token; set the channel (`YOUR_SLACK_CHANNEL_ID`)
   - **WhatsApp Trigger** — Client ID = Meta **App ID**, Client Secret = Meta **App Secret**

4. **WhatsApp webhook** — with the OAuth credential, n8n **auto-registers the webhook when you
   Publish**. Don't configure a callback URL by hand in Meta. Note WhatsApp allows **one
   webhook per Meta app**; if Publish reports *"App ID already has a webhook subscription"*,
   clear it:
   ```
   GET    https://graph.facebook.com/v25.0/{app-id}/subscriptions?access_token={app-id}|{app-secret}
   DELETE https://graph.facebook.com/v25.0/{app-id}/subscriptions?object=whatsapp_business_account
   ```

---

## Try it without a Meta account

You don't need WhatsApp set up to run the whole pipeline. Paste
[`docs/sample-payload.json`](docs/sample-payload.json) into the **WhatsApp Trigger → set mock
data**, then execute. Everything downstream behaves exactly as it does on a live message.

That's how this build was made: the AI, Slack, and Monday nodes were all built and proven
against that mock **before** a single Meta credential existed — the fiddly external
integration was saved for last. When the first real WhatsApp message finally arrived, the
payload matched the mock exactly and **no node needed changing**.

Sample input:

```
Hey can you get the new roofing ad campaign live by Friday? Budget is $2k, target homeowners in Dallas.
```

Extracted:

```json
{ "is_task": true,
  "title": "Launch roofing ad campaign by Friday",
  "client": "Apex Roofing",
  "type": "task",
  "priority": "High",
  "summary": "Get the new roofing ad campaign live by Friday with a budget of $2k, targeting homeowners in Dallas." }
```

---

## How WhatsApp data actually gets out

The mechanism, since it's the part people hand-wave:

- The channel is the **WhatsApp Business Cloud API** (Meta Graph API). Inbound messages arrive
  as a **webhook** — subscribe the `messages` field.
- **Text** is right in the payload: `messages[0].text.body`, with the sender at
  `messages[0].from` and their profile name at `contacts[0].profile.name`.
- **Media** (photo/video/doc) gives you a **media ID, not the file**. Fetching it is two Graph
  calls: `GET /{media-id}` → a temporary URL, then `GET {url}` with the token → the binary,
  which you'd store (e.g. Drive) and pass on as a link. *TaskRelay reads the media **caption**
  today — where clients type the actual request — and leaves the download for a later pass.*
- The first question worth asking any client: **Cloud API / a BSP (Twilio, 360dialog), or the
  plain WhatsApp Business app?** The Business app has **no official API** and needs a BSP
  bridge — that determines whether any of this is possible.

---

## Security notes

- **No secrets in this repo.** n8n exports *reference* credentials by name — never keys. The
  Monday board ID and Slack channel are placeholders.
- **The sample payload is entirely fake** — invented phone numbers, a fictional company.
- **Client messages live in WhatsApp and Monday, not the workflow** — the exported JSON
  reveals no real conversations or contacts.

---

## Results & highlights

- **The message is the task** — no one re-types a client request into the board again.
- **Noise filtered at the source** — an `is_task` gate keeps "thanks" and "ok" off the board,
  which is what keeps the team trusting the channel.
- **Detail survives** — budgets, deadlines, and targeting land in the summary instead of being
  lost in a chat thread.
- **Proven end-to-end on a real message** — phone → Meta Cloud API → n8n → AI → Slack + Monday,
  on a free Meta test number at $0, with the app unpublished.

---

## Roadmap

- **Media pipeline** — resolve the media ID via the Graph API, store the file, attach a link to
  the board's Link column. (Caption text already works today.)
- **Threading** — group follow-up messages onto the existing item rather than creating a new one.
- **Reply confirmation** — send the client a "got it, logged as *X*" acknowledgement.

---

## License

MIT — see `LICENSE` (add your preferred license file).
