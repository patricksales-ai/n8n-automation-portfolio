# SpeedLead — AI Speed-to-Lead Responder (Make + Zapier + Airtable)

**Every inbound lead gets scored and answered in under a minute — before a competitor picks up the phone.**
SpeedLead catches a web-form enquiry the instant it arrives, uses AI to score how "hot" it is,
files it in a CRM, and — for the hot ones — fires an instant personalised reply to the lead plus
a Slack alert to the team. Cold enquiries are tagged for nurture instead of cluttering the pipeline.

Two platforms, split by job: **Make** runs the real-time capture-score-route-respond pipeline;
**Zapier** runs the CRM-logging and team-notification side. Separation of concerns, on purpose.

> 📖 **[WALKTHROUGH.md](WALKTHROUGH.md)** explains every module in the Make scenario, in order.

---

## Why this exists

**The problem —** for trades and field-service businesses, speed *is* the sale. Lead-response
research is brutally consistent: contact a web lead within the first minute and you're many times
more likely to win it than if you reply an hour later. But the owner is up a ladder, not watching
the inbox. Enquiries sit for hours, get answered after the customer has already booked someone
else, and the genuinely urgent jobs look identical to the tyre-kickers until someone reads every one.

**The result —** a form submission becomes, within seconds: a scored CRM record, an instant reply
in the customer's inbox, and a Slack ping to the team with the summary and next action — while the
lead is still on the page. The owner sees *"🔥 Score 10 — urgent aircon repair"*, not a raw inbox.

---

## What it does

- **Captures** any website-form lead via a webhook (name, email, phone, company, message, source).
- **Validates at the door** — a regex-email + message-present filter drops malformed/spam payloads
  before they cost an Airtable write or an AI call.
- **De-duplicates** — a double-submit or platform retry within 10 minutes never creates a second
  record or sends a second reply.
- **Scores** each lead 1–10 with `gpt-4o-mini`, returning strict JSON: `score`, `intent`, `summary`.
- **Routes** on the score — **hot (≥ 7)** → instant reply email + Slack alert + `Status = Hot`;
  **cold** → `Status = Nurture`.
- **Logs & notifies** (Zapier half) — every hot lead is appended to a Google Sheet and the owner
  gets an internal "new hot lead" email.
- **Fails safe** — the record is captured *before* the AI runs, so an AI outage never loses a lead;
  a human gets pinged to score it manually instead.

---

## Architecture

![The full Make scenario — webhook → validate → dedup → capture → score → enrich → route → respond](docs/make-scenario.png)

*Simplified flow:*

```
                          ┌─────────────────────────  MAKE (real-time)  ─────────────────────────┐

  Website form ──▶ Webhook (speedlead-intake)
                     └[filter: valid lead — email regex + message exists]
                   Airtable Search  ──▶  Aggregator          (dedup: same email in last 10 min)
                     └[filter: not a duplicate]
                   Airtable Create   (capture · Status = New)
                   OpenAI gpt-4o-mini (score / intent / summary, JSON)  ⟿ onerror ▶ Slack "AI failed" ▶ Commit
                   Parse JSON
                   Airtable Update   (enrich · write score/intent/summary)
                   Router ──┬── HOT (score ≥ 7): Update=Hot ▶ Slack alert ▶ n8n reply-email ⟿ onerror ▶ Commit
                            └── COLD:            Update=Nurture

                          └───────────────────────────────────────────────────────────────────────┘

                          ┌───────────────────────  ZAPIER (companion)  ─────────────────────────┐
  Airtable "Hot Leads" view ──▶ New Record in View ──▶ Google Sheets log ──▶ Gmail owner alert
                          └───────────────────────────────────────────────────────────────────────┘
```

### The design decisions that matter

| Choice | Why |
|---|---|
| **Capture-first, enrich-second** | The Airtable record is created (`Status=New`) *before* the OpenAI call, then updated with the score. A lead is never lost to an AI outage or a bad JSON parse. |
| **Dedup filters on the aggregated `id` "does not exist"** — never `length()` | A Search → Aggregator returns a **phantom empty element** when it finds nothing (`__IMTAGGLENGTH__ = 1`), so a `length(array)=0` test blocks *every* lead. Testing the aggregated `id` for does-not-exist is the correct idempotency gate. |
| **Score compared as a NUMBER (`≥ 7`)** in the Router | As a string, `"10"` sorts *below* `"7"` and hot leads route cold. The comparison must be numeric. |
| **Slack via direct Web API over HTTP**, not the native connector | Make's OAuth was broken account-wide (`SC424` / `Cannot assign to read only property 'scope'`). Solution: a least-privilege Slack **bot token** (`chat:write`) called straight against `chat.postMessage` — `application/x-www-form-urlencoded`, so quotes/newlines in the AI summary can't break the body. |
| **Reply email handed off to n8n**, not sent from Make | Make can't send Gmail — OAuth is broken *and* it blocks `smtp.gmail.com`. Make POSTs the lead to an existing **n8n** webhook, which sends via an established Gmail connection. Make orchestrates; n8n owns the send. |
| **Validate at the door, but capture before the AI** | Two different robustness concerns: reject junk *before* any spend (front filter), yet still persist a valid lead *before* the fragile AI step (capture-first). |
| **Least-privilege Airtable PAT** | Scoped to the one base, `data.records:read/write` + `schema.bases:read` only. It deliberately **cannot** create schema — so the `Status` select options are pre-defined in Airtable, not invented by the automation. |

---

## Tech stack

- **Make.com** — real-time capture / score / route / respond (free tier, `us2` region)
- **Zapier** — CRM logging + team notification (free tier, native Airtable trigger)
- **Airtable** — the `Leads` CRM base (least-privilege PAT connection)
- **OpenAI `gpt-4o-mini`** — lead scoring, JSON-object output mode
- **Slack** — hot-lead team alerts (Web API + bot token)
- **n8n** (self-hosted) — the Gmail send for the instant reply, over a webhook

---

## Setup

1. **Import the Make blueprint** — Make → *Create a new scenario → ⋯ → Import Blueprint* →
   [`workflows/speedlead.make.blueprint.json`](workflows/speedlead.make.blueprint.json).

2. **Reconnect the placeholders** the sanitizer stripped (see *Security notes*):
   - `YOUR_AIRTABLE_BASE_ID` / `YOUR_AIRTABLE_TABLE_ID` → select your own base/table on each Airtable module.
   - `xoxb-YOUR-SLACK-BOT-TOKEN` → your Slack bot token in the two Slack HTTP modules' `Authorization` header.
   - `https://YOUR-N8N-HOST/webhook/speedlead-reply` → your n8n reply webhook URL in the HTTP reply module.
   - Reconnect the Airtable + OpenAI **connections** (Make blueprints reference connections by id, not key).

3. **Airtable base** — table `Leads` with: Name, Email, Phone, Company, Message, Source,
   Submitted At (date+time), Score (number), Intent, AI Summary (long text), **Status** (single
   select: `New` / `Hot` / `Nurture` — pre-create these options; the least-privilege PAT can't).

4. **The Zapier companion** (no clean export exists) — rebuild from
   [`docs/`](docs/) screenshots: trigger **Airtable → New Record in View** on a filtered
   `Hot Leads` view → **Google Sheets → Create Spreadsheet Row** → **Gmail → Send Email**.
   ⚠️ Prepend an apostrophe to the Phone field in the Sheets step (see *Gotchas*).

---

## Try it without a website

You don't need a form to run the whole pipeline — fire the webhook directly with the fake hot lead
in [`docs/sample-payload.json`](docs/sample-payload.json):

```powershell
Invoke-RestMethod -Method Post -Uri "<your-make-webhook-url>" `
  -ContentType "application/json" `
  -InFile ".\docs\sample-payload.json"
```

That's exactly how this build was developed and proven — precise hot / lukewarm / spam payloads
fired at the webhook, no real form required. The form is just the production front-end; the
webhook is the contract.

Discrimination proven with three leads:

| Lead | Message | Score | Route |
|---|---|---|---|
| Dave / BrightBuild | *"office aircon died, staff in all week, pay a call-out to look today"* | **10** | 🔥 Hot |
| Megan (homeowner, no company) | *"wondering roughly what a new split system might cost sometime"* | **4** | Nurture |
| Spam | *"GROW YOUR BUSINESS 10X click here"* | **1** | Nurture |

---

## Gotchas worth knowing (all hit and fixed during the build)

- **Make's OAuth is broken account-wide** (`SC424` server-side error) — Slack *and* Gmail native
  connectors both fail. Only **token-based** connections work. Hence Slack-over-HTTP and the
  Gmail-via-n8n handoff. A real-world "the vendor's connector is down, ship anyway" story.
- **Airtable Aggregator phantom element** — a Search with 0 results still yields one empty bundle,
  so dedup must gate on `id` *does-not-exist*, not array length.
- **Sheets turns `+61 400 111 222` into `#ERROR!`** — a leading `+`/`=`/`-`/`@` is parsed as a
  formula. Fix on the Zapier side: prepend a single apostrophe to force text (stays hidden in the cell).
- **Numeric vs string score** in the Router — `"10" < "7"` as strings; use a numeric operator.
- **Least-privilege PAT can't create select options** — `[422] Insufficient permissions` unless the
  `Status` options exist in Airtable first. Correct security posture, documented behaviour.

---

## Security notes

- **No secrets in this repo.** The committed blueprint was passed through
  [an auditable sanitizer](../) that replaces the Slack bot token, n8n host, and Airtable
  base/table IDs with placeholders, re-parses to prove valid JSON, then greps the output to prove
  zero secret patterns survive.
- **The sample payload is entirely fake** — a fictional company and an invented phone/email
  (`.example` domain).
- **Lead data lives in Airtable and the CRM, not the workflow** — the exported blueprint carries
  no real contacts. All lead fields are `{{pills}}`, not baked-in values.

---

## See it working

**Hot-lead alerts in Slack** — score, intent, summary, and reply details, plus the AI-failure
handler pinging the team for manual review when scoring can't run:

![Slack hot-lead alerts, including the AI-failure handler](docs/slack-alert.png)

**The owner's internal heads-up email** (distinct from the customer-facing reply):

![Owner alert email](docs/owner-alert.png)

**The Zapier companion** — Airtable trigger → Google Sheets log → Gmail owner alert:

![Zapier companion zap](docs/zap-overview.png)

**Every hot lead appended to Google Sheets** (phone forced to text so a leading `+` isn't parsed
as a formula):

![Google Sheets hot-leads log](docs/sheets-log.png)

**The Airtable CRM** the pipeline captures and scores into:

![Airtable Leads base](docs/airtable-leads.png)

---

## Results & highlights

- **Sub-minute response, automatically** — hot leads get a personalised reply and a team alert
  before the customer leaves the page.
- **The urgent job is visible instantly** — score + intent + summary, not a raw inbox to triage.
- **Never loses a lead** — capture-first means an AI outage pings a human instead of dropping the enquiry.
- **Robust where it counts** — validation filter, idempotent dedup, and two error handlers that
  keep upstream writes when a downstream call fails.
- **Proven end-to-end** — webhook → Airtable Hot + Slack alert + reply email + Zapier Sheets log +
  owner alert, all on free-tier accounts at $0.

---

## Roadmap

- **Source-agnostic intake** — add Facebook / Google Lead Ads as parallel front-ends to the webhook.
- **Reply on all routes** — currently the instant reply fires on the hot route only; extend a
  softer nurture reply to cold leads.
- **Notion logging module** — mirror the Sheets log into Notion for teams that live there.

---

## License

MIT — see repository root.
