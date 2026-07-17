# Build Walkthrough — node by node

How TaskRelay works, explained so you can read and present it. A client texts a task
to WhatsApp; an AI extracts a structured task from the free-form message; the team gets
a Slack alert and a Monday.com board item — with chit-chat filtered out.

> n8n note: `{{ ... }}` is an **expression** n8n evaluates at run time. The sample
> WhatsApp payload in [`docs/sample-payload.json`](docs/sample-payload.json) lets you run
> the whole pipeline with no Meta account — paste it into the trigger's *set mock data*.

```
WhatsApp Trigger → Edit Fields → Basic LLM Chain → Normalize → If(is_task) ─┬─ Slack
                                      └ OpenAI gpt-4o-mini                  └─ Create Monday Item
```

---

**1. WhatsApp Trigger** — fires on every inbound `messages` event from the WhatsApp
Business Cloud API. With n8n's WhatsApp OAuth credential (Client ID = Meta **App ID**,
Client Secret = Meta **App Secret**), n8n **auto-registers the webhook when you Publish** —
don't configure a callback URL by hand in Meta. WhatsApp allows **one webhook per Meta app**.

The trigger emits Meta's `value` object, so the payload reads `messages[0]`, `contacts[0]`
at the top level.

**2. Edit Fields** — normalizes any inbound message into four fields:

| Field | Expression |
|---|---|
| `client_phone` | `{{ $json.messages[0].from }}` |
| `client_name` | `{{ $json.contacts[0].profile.name }}` |
| `msg_type` | `{{ $json.messages[0].type }}` |
| `text` | `{{ $json.messages[0].text?.body ?? $json.messages[0].image?.caption ?? $json.messages[0].video?.caption ?? '' }}` |

The `text` expression is the interesting one. A media message has **no** `text` object —
`messages[0].text.body` would throw. Optional chaining (`?.`) plus null-coalescing (`??`)
walks a fallback chain: message body → image caption → video caption → empty string. Since
clients typically type the actual task in a photo's **caption**, this captures the request
from a media message without any of the media-download machinery.

**3. Basic LLM Chain** (+ **OpenAI Chat Model**) — the brain. `gpt-4o-mini` turns free-form
chat into structured data. Prompt in [`prompts/task-extractor.txt`](prompts/task-extractor.txt).

Two deliberate choices:

- **Basic LLM Chain, not an AI Agent.** This is a stateless one-shot extraction — no tools,
  no memory to carry between messages. An Agent would add overhead and an extra failure mode
  for nothing.
- **Response Format = `JSON Object`, not a Structured Output Parser.** On self-hosted n8n the
  Structured Output Parser throws *"Failed to parse agent steps"* even on valid,
  schema-matching JSON. Instead: set the model's **Options → Response Format → JSON Object**
  (the API requires the word "json" in the prompt — it's there) and write the exact JSON shape
  into the prompt.

Output: `{ is_task, title, client, type, priority, summary }`

`priority` is emitted as **`High` / `Medium` / `Low`** — matching Monday's built-in Priority
labels *exactly*. Monday's labels are case-sensitive, so a lowercase `high` doesn't match
`High`; it silently creates a **duplicate label**. Pinning the vocabulary in the prompt is
cheaper and cleaner than mapping it downstream.

**4. Normalize** (Code) — forces one consistent shape regardless of how the model or the n8n
version hands back the JSON:

```javascript
const j = $input.item.json;
let o = typeof j.text === 'string'
  ? JSON.parse(j.text.replace(/```json\s*/gi, '').replace(/```/g, '').trim())
  : j;
if (!o.output) o = { output: o };
return { json: o };
```

If the chain returns a JSON **string** (in `text`), parse it — stripping any ``` fence the
model added. If n8n already parsed it, pass it through. Either way, wrap it so everything
downstream reads `$json.output.*`. This is what makes the workflow portable across n8n
versions instead of breaking on an upgrade.

**5. If** — `{{ $json.output.is_task }}` **is true**.

The noise filter. Real client threads are full of "thanks", "ok", "got it" — without this
gate, every acknowledgement becomes a Slack ping and a board item, and the team stops
trusting the channel. The `false` branch is intentionally left unconnected: not a task,
nothing happens.

**6. Send a message** (Slack) — on `true`, alerts the team channel:

```
🔧 *New task from {{ $('Normalize').item.json.output.client }}*  ({{ $('Edit Fields').item.json.client_phone }})

*{{ $('Normalize').item.json.output.title }}*  ·  priority: {{ ... .priority }}  ·  type: {{ ... .type }}

{{ $('Normalize').item.json.output.summary }}
```

Nodes are referenced explicitly (`$('Normalize')`, `$('Edit Fields')`) rather than `$json`,
so the message doesn't depend on what the IF happens to pass through.

**7. Create Monday Item** (HTTP Request) — also on `true`. Files the task on the board via
GraphQL. `POST https://api.monday.com/v2`, Generic **Header Auth** (`Authorization` = the raw
token — Monday does **not** want a `Bearer` prefix).

The body is an n8n **expression that returns an object**:

```javascript
{{ {
  "query": "mutation ($board: ID!, $name: String!, $cols: JSON!) { create_item (board_id: $board, item_name: $name, column_values: $cols, create_labels_if_missing: true) { id name } }",
  "variables": {
    "board": "YOUR_MONDAY_BOARD_ID",
    "name": $('Normalize').item.json.output.title,
    "cols": JSON.stringify({
      "project_status": { "label": "New" },
      "priority":       { "label": $('Normalize').item.json.output.priority },
      "text":           $('Normalize').item.json.output.summary,
      "text_mm5avj0":   $('Normalize').item.json.output.client,
      "text_mm5ava7h":  $('Normalize').item.json.output.type
    })
  }
} }}
```

Four things are load-bearing here:

- **Returning an object, not a string.** n8n serializes it and handles every escaping problem
  for free. Hand-writing JSON with `{{ }}` holes in it breaks the moment an AI-generated title
  contains an apostrophe or a quote.
- **GraphQL variables** (`$board`, `$name`, `$cols`) keep values out of the query string —
  no injection, no escaping.
- **`column_values` must be a JSON-*string***, not an object. That's a Monday API quirk, hence
  the inner `JSON.stringify(...)`.
- **Column IDs are auto-generated and must be introspected**, never guessed. On this board
  Status is `project_status` — *not* `status` — and the custom columns are `text_mm5avj0` /
  `text_mm5ava7h`. Get yours with:
  ```graphql
  query { boards(ids: YOUR_BOARD_ID) { columns { id title type } } }
  ```

---

## Gotchas & lessons

- **Introspect the board before writing the mutation.** Monday's column IDs don't match their
  display names, and Status being `project_status` rather than `status` would have failed
  silently. One throwaway GraphQL query removes all guesswork.
- **`create_labels_if_missing: true` is a footgun as much as a fix.** It let a lowercase
  `high` succeed — by inventing a *duplicate* label next to the native `High`. The write
  "worked" and the board was subtly wrong. Prefer matching a tool's native vocabulary in the
  prompt over letting the API invent new values.
- **Vague prompt rules are non-deterministic.** "use the client name; extract the company if
  the message names one" returned `Apex Roofing` on one run and `Mike - Apex Roofing` on the
  next — same input. Pinning an explicit example in the rule fixed it.
- **The Structured Output Parser is unreliable on self-host.** JSON-mode + a Normalize node is
  the portable pattern.
- **Meta's Development mode does *not* block inbound webhooks for a test number.** The
  Webhooks page warns that unpublished apps receive no production data — that's boilerplate.
  A real phone → test number message delivers fine, unpublished, at $0. No business
  verification needed for a demo.
- **"App ID already has a webhook subscription" on Publish** — WhatsApp allows one webhook per
  app and Meta's own dashboard test-webhook viewer can occupy the slot. Meta's UI may show
  nothing while a subscription exists; trust the Graph API instead:
  ```
  GET  https://graph.facebook.com/v25.0/{app-id}/subscriptions?access_token={app-id}|{app-secret}
  DELETE .../subscriptions?object=whatsapp_business_account
  ```
- **Build the brain on pinned data; save the Meta setup for last.** The entire pipeline was
  built and proven against [`docs/sample-payload.json`](docs/sample-payload.json) before a
  single Meta credential existed. When the real message finally arrived, the payload matched
  the mock exactly and **no node needed changing**.
