# Build Walkthrough — node by node

How SmartDispatch works, explained so you can read and present it. Two workflows share one Google
Sheet as their database: **Dispatch** (intake → triage → match → book → notify → escalate) and
**Completion** (tech says "Done" → price → invoice).

> n8n note: `{{ ... }}` is an **expression** n8n evaluates at run time. `$('Node Name')` reads another
> node's output. The brains use `gpt-4o-mini` in **JSON mode** (Response Format = `JSON Object`) plus a
> small Normalize code node — not a Structured Output Parser, which throws *"Failed to parse agent
> steps"* on self-hosted n8n even on valid JSON.

---

## The shared database — one Google Sheet, two tabs

**`Technicians`** (the roster) — one row per tech:

| name | telegram_chat_id | areas | skills | calendar_id | on_call |
|---|---|---|---|---|---|
| Marco Reyes | … | `Makati,Taguig` | `aircon,refrigeration` | *(that tech's calendar)* | FALSE |
| Jenny Cruz | … | `Makati,Pasay` | `heating,ventilation` | … | TRUE |
| Dan Lim | … | `Quezon City` | `aircon,electrical` | … | FALSE |

`areas` and `skills` are comma-separated tags. Each tech's `calendar_id` points at their **own**
Google Calendar, so availability is checked per-technician.

**`Jobs`** (the lifecycle) — one row per request:

`job_id · date · customer · contact · address · issue · urgency · assigned_tech · status · price ·
calendar_event_id · customer_email`

`status` walks `BOOKED → DONE → INVOICED`, with `ESCALATED` as the no-tech branch.

---

# Workflow A — Dispatch

```
Chat ─────────────┐
Gmail  → Parse email ─┤
Telegram → Parse telegram ─┴→ Basic LLM Chain → Normalize → [Technicians] → matcher
     → Get many events → Pick first free → Tech available?
         ├ true  → Switch(urgency) → Create an event → Append row → (customer + ops + Telegram reply)
         └ false → Append row (escalated) → ops alert
```

## Intake — three channels, one field

The whole point of the front end: no matter which channel a request arrives on, it is normalised into
a single field called **`chatInput`**, so all three feed the *same* brain with no branching.

**1. When chat message received** *(Chat Trigger)* — n8n's built-in chat widget. It emits `chatInput`
natively, so it wires straight into the brain.

**2. Gmail Trigger** *(polls a labelled inbox)* — fires on new mail carrying the Gmail label
`SmartDispatch-Intake` (label ID `YOUR_GMAIL_LABEL_ID`). **Why a dedicated label and not the whole
inbox:** this same workflow *sends* email to the owner (confirmations, ops alerts), so a whole-inbox
trigger would ingest its own outbound as fake customer requests. Watching one label breaks that loop.
**Simplify is OFF** — with Simplify ON, a `multipart/alternative` email returns only a truncated
`snippet` (it cuts off before the customer's contact block). OFF, n8n runs a mail parser and gives
clean top-level `text` / `from` / `subject` fields.

**3. Parse email** *(Code)* — folds the parsed email into `chatInput`:

```js
const from = j.from?.text || j.from?.value?.[0]?.address || '';
const subject = j.subject || '';
const body = (j.text || '').trim();
const chatInput = `From: ${from}\nSubject: ${subject}\n\n${body}`.trim();
```

Prepending the `From:` line lets the brain pick up `customer_email` from the header even if the body
doesn't mention it.

**4. Telegram Trigger** *(webhook)* — fires on any message to the bot. Unlike Gmail's poll, this is a
real webhook, so the workflow must be **Active** on a public host for it to fire live.

**5. Parse telegram** *(Code)* — same `chatInput` shape, and carries the `telegram_chat_id` so we can
reply back later:

```js
const m = item.json.message || {};
const sender = [`${m.from?.first_name ?? ''} ${m.from?.last_name ?? ''}`.trim(),
                m.from?.username ? `@${m.from.username}` : ''].filter(Boolean).join(' ');
const chatInput = `From: ${sender} (Telegram)\n${m.text || ''}`.trim();
return { json: { chatInput, telegram_chat_id: m.chat?.id } };
```

## The brain

**6. Basic LLM Chain** (+ **OpenAI Chat Model**, `gpt-4o-mini`) — the triage brain. Full system prompt
in [`prompts/triage-brain.txt`](prompts/triage-brain.txt). Two settings make the multi-channel design
work:

- **Source for Prompt = `Define below`**, User Message = `{{ $json.chatInput }}`. This is what lets all
  three triggers share one brain — each emits `chatInput`, and the brain reads that one field.
- **Response Format = `JSON Object`.** The prompt spells out the exact output shape.

It returns `customer_name`, `contact`, `customer_email`, `address`, `appliance`, `skill_required`,
`issue_summary`, `urgency`, `urgency_reasoning`, and a `missing_info[]` list of anything the customer
didn't provide.

**7. Normalize** *(Code)* — strips any ```` ```json ```` fences and parses the model's text into a real
object, always shaped `{ output: {...} }`:

```js
let o = typeof j.text === 'string'
  ? JSON.parse(j.text.replace(/```json\s*/gi, '').replace(/```/g, '').trim())
  : j;
if (!o.output) o = { output: o };
```

## Matching

**8. [Technicians]** *(Google Sheets — read)* — reads the whole roster.

**9. matcher** *(Code)* — the heart of dispatch. It scores every tech on **area** and **skill**, keeps
only those who match both, and sorts by score. Area is a substring test (so `"123 Rizal St, Makati"`
matches the tag `makati`); skill is an **exact** match on the canonical `skill_required`:

```js
const jobArea  = String(job.address ?? '').toLowerCase();
const jobSkill = String(job.skill_required ?? '').toLowerCase();
// per tech:
const areaMatch  = areas.some(a => jobArea.includes(a));
const skillMatch = skills.includes(jobSkill);
```

If nobody matches, it doesn't return an empty list (an empty list would stall the workflow — see
Gotchas). It emits a single marker item instead:

```js
if (candidates.length === 0) {
  return [{ json: { _noMatch: true, job, calendar_id: 'you@example.com' } }];
}
```

**10. Get many events** *(Google Calendar)* — for each candidate, fetches events on **that tech's**
`calendar_id` in the `now → now+2h` window. **Always Output Data is ON**, so a free tech (zero events)
still emits a passthrough item instead of vanishing.

**11. Pick first free** *(Code)* — maps the calendar results back to candidates via `pairedItem`, and
returns exactly one tagged item for the next node to branch on:

- matcher's `_noMatch` marker → `{ _escalate: true, _reason: 'no_match' }`
- first candidate with no conflicting event → `{ ...candidate, _escalate: false }`
- candidates existed but all are busy → `{ _escalate: true, _reason: 'all_busy' }`

## Routing

**12. Tech available?** *(If)* — condition `{{ $json._escalate }}` **is false**.
`true` → the booking path; `false` → the escalation path.

### Booking path (true)

**13. Switch** *(on urgency)* — three rules, `Emergency` / `Within-24h` / `This-week`. They currently
converge on one booking node — the structure is in place for per-tier handling (e.g. emergencies
booking a sooner slot) as a later enhancement.

**14. Create an event** *(Google Calendar)* — books on `{{ $json.calendar_id }}` (the matched tech's
calendar). Summary reads `appliance - customer_name (urgency)`; description carries the issue, contact,
and address. The returned event `id` doubles as the `job_id`.

**15. Append row in sheet** *(Google Sheets)* — writes the `Jobs` row: `status = BOOKED`,
`assigned_tech` = the picked tech's name, `customer_email` from the job, `calendar_event_id` = the event
id. Fields are pulled from `$('Pick first free')` (a node whose output shape is proven) rather than off
the previous node, so an upstream change can't silently break the mapping.

Three nodes fan out from the Append in **parallel**:

**16. customer confirmation** *(Gmail)* — to `{{ $('Pick first free').first().json.job.customer_email
|| 'you@example.com' }}` (falls back to the owner inbox when no email was given). Plain-text greeting
with the tech, issue, address, and ETA.

**17. ops alert** *(Gmail)* — internal notification with the full job detail. The subject uses an
Emergency ternary so urgent jobs are visually flagged.

**18. From Telegram?** *(If)* → **Send a text message** *(Telegram)* — this is how a Telegram customer
gets a reply *back on Telegram*. The catch: the Basic LLM Chain **drops input passthrough**, so
`telegram_chat_id` can't ride the item downstream — we have to reference `$('Parse telegram')`
directly. But on a chat/Gmail run that node never executed, and referencing an unexecuted node throws.
The guard is n8n's `.isExecuted`:

- **If** condition: `{{ $('Parse telegram').isExecuted }}` **is true**
- **Send** Chat ID: `{{ $('Parse telegram').first().json.telegram_chat_id }}`

So it fires only on Telegram-originated bookings; chat and Gmail runs skip it.

### Escalation path (false)

**19. Append row in sheet (escalated)** *(Google Sheets)* — logs the job `status = ESCALATED`,
`assigned_tech = UNASSIGNED`, `job_id = ESC-{{ $now.toMillis() }}`, no calendar event.

**20. Send a message** *(Gmail)* — ops alert titled `NO TECH AVAILABLE - <customer> (<urgency>)`, with
a Reason line that reads `no_match` vs `all_busy` off `$('Tech available?').first().json._reason`, so a
human can pick it up.

---

# Workflow B — Completion

```
Chat/"Done" → Basic LLM Chain → Normalize → Get row(s) [Jobs, BOOKED]
   → Price calc → mark DONE + price → invoice email → mark INVOICED
```

**1. When chat message received** — the tech's "Done" message. (In production this is the same Telegram
bot; chat is the test harness.)

**2. Basic LLM Chain** (+ `gpt-4o-mini`, JSON mode) — parses the free-text completion into
`{ customer_name, completion_notes, parts_used, parts_cost }`. Prompt in
[`prompts/completion-parser.txt`](prompts/completion-parser.txt). `parts_cost` is forced to a bare
number, and the model is told never to invent a price.

**3. Normalize** *(Code)* — same fence-strip + parse into `{ output: {...} }`.

**4. Get row(s) in sheet** *(Google Sheets)* — looks up the open job: **`customer` =
`{{ $('Normalize').first().json.output.customer_name }}` AND `status` = `BOOKED`** (combine = AND).
Filtering on `BOOKED` means stray `ESCALATED` rows are ignored automatically.

**5. Price calc** *(Code)* — the rate card:

```js
const BASE = 800;
const URGENCY_PREMIUM = { 'Emergency': 1500, 'Within-24h': 500, 'This-week': 0 };
const total = BASE + (URGENCY_PREMIUM[job.urgency] ?? 0) + (Number(comp.parts_cost) || 0);
```

So an Emergency aircon job with a ₱1,500 capacitor totals `800 + 1500 + 1500 = ₱3,800`. It staples the
job row, completion notes, and the line-item breakdown together for the invoice. (Rates are PHP
placeholders — tune to your market.)

**6. mark DONE + price** *(Google Sheets — update)* — matches on `job_id`, sets `status = DONE` and
`price = total`.

**7. Send a message** *(Gmail — invoice)* — to `{{ $('Price calc').first().json.customer_email ||
'you@example.com' }}`. **Email Type = HTML**, not plain text: the invoice has an aligned line-item
table (call-out / urgency premium / parts / **Total due**), and Gmail collapses the whitespace of a
plain-text table into an unreadable blob. HTML with a real `<table>` renders clean.

**8. mark INVOICED** *(Google Sheets — update)* — matches on `job_id`, sets `status = INVOICED`. The row
has now completed its lifecycle.

---

## Gotchas & lessons

- **The LLM maps free text → a fixed vocabulary; code does the exact match.** The first version matched
  the customer's raw `appliance` string against the roster's `skills` tags. A real email said "air
  conditioner"; the tag was `aircon`; neither string contained the other, so a perfectly serviceable
  job **escalated** as "no tech." The fix wasn't a bigger synonym list in code — it was adding a
  canonical `skill_required` enum to the brain (which is *good* at fuzzy→canonical mapping) and having
  the matcher do a clean `skills.includes(jobSkill)`. Chat tests with tidy wording had hidden the bug;
  real email intake exposed it.

- **An empty item list can't drive a branch.** In n8n a node that receives zero items simply doesn't
  run — so returning `[]` from the matcher or the availability check doesn't *trigger* an escalation
  branch, it just stalls the chain. Both decision nodes therefore always emit exactly **one** item
  carrying an `_escalate` flag, and an `If` routes on that. This is the single most important pattern in
  the build.

- **The brain drops input passthrough.** The Basic LLM Chain outputs only the model's response; fields
  that entered with the message (like `telegram_chat_id`) don't survive to downstream nodes. Anything
  you need after the brain must either be re-derived, referenced from an upstream node via
  `$('Node')`, or carried inside the `job` object the matcher builds.

- **`$('Node').isExecuted` is the clean multi-channel guard.** Referencing an upstream node that didn't
  run in the current execution throws. When a shared downstream branch needs data from one *optional*
  channel, gate it on `.isExecuted` so the reference only happens when that channel actually fired.

- **Gmail Simplify hides the body on multipart mail.** Simplify ON returns a truncated `snippet`; turn
  it OFF to get the mail-parsed `text` / `from` / `subject`.

- **A trigger that watches its own outbound will loop.** Because Dispatch emails the owner, the Gmail
  trigger watches a dedicated **label**, never the whole inbox.

- **Columnar invoices must be HTML.** Gmail collapses plain-text whitespace, so any email with aligned
  columns needs a real HTML `<table>`.

- **Validate written data, not just that a node ran green.** An early version wrote explanatory prose
  into a sheet cell's value; every row looked "successful" because the node ran, but the *contents* were
  junk. Green means the node executed — it doesn't mean it wrote the right thing.
