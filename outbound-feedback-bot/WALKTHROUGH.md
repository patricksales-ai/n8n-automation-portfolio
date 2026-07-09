# Build Walkthrough — node by node

How the Outbound Feedback Bot works, explained so you can read and present it. There are
**two n8n workflows**: the **Dialer** places the calls, and the **Logger** turns each
finished call into data. The voice agent itself ("Alex") lives in **Vapi** — see
[`prompts/assistant-config.md`](prompts/assistant-config.md).

> n8n note: `{{ ... }}` is an **expression** n8n evaluates at run time.
> `$('Node Name')` reaches back to another node's output. A **Filter** node passes through
> only the items whose condition is true and drops the rest.

---

## Workflow 1 — Outbound Dialer

Reads the call list, removes anyone on the Do-Not-Call list, and places a call for each
remaining lead.

```
Manual Trigger ─▶ Get DNC ─▶ Get row(s) in sheet ─▶ Not on DNC ─▶ HTTP Request (place call)
                (block list)   (leads, Execute Once)   (filter)      POST api.vapi.ai/call
```

**1. When clicking 'Execute workflow'** (Manual Trigger) — kicks off a dialing run. In
production this would be swapped for a Schedule trigger (e.g. "every weekday 10am").

**2. Get DNC** (Google Sheets → Get rows) — reads the **Do Not Call** tab (`phone | reason |
date`). This runs **first**, on purpose (see the gotcha below), so the block list is loaded
before any lead is evaluated.

**3. Get row(s) in sheet** (Google Sheets → Get rows) — reads the **Customers to Call** tab
(`name | business | phone | email`). One item per lead.
- **Setting: "Execute Once" is ON.** Because this node is chained *after* `Get DNC` (which
  can return many rows), without "Execute Once" it would run once per DNC row and multiply
  the leads. "Execute Once" makes it run a single time and return the full lead list.

**4. Not on DNC** (Filter) — drops any lead whose number is on the block list. Condition:

```
true  is equal to  {{ !$('Get DNC').all()
      .map(i => String(i.json.phone).replace(/\D/g,''))
      .includes(String($json.phone).replace(/\D/g,'')) }}
```
Read it as: *"keep this lead only if their number is **not** found in the DNC list."*
`.replace(/\D/g,'')` strips `+`, spaces and dashes from **both** sides, so the match works
regardless of how the numbers are formatted.

**5. HTTP Request** (POST `https://api.vapi.ai/call`) — places the call. Auth is a **Header
Auth** credential holding the Vapi private key (`Authorization: Bearer …`). The body is a
single JSON expression, evaluated **once per surviving lead**:

```js
{{ JSON.stringify({
  assistantId:   "YOUR_ASSISTANT_ID",
  phoneNumberId: "YOUR_PHONE_NUMBER_ID",           // the Twilio number, imported into Vapi
  customer: { number: String($json.phone).startsWith('+')
                        ? String($json.phone)
                        : '+' + String($json.phone) },
  assistantOverrides: {
    variableValues: { customerName: $json.name, business: $json.business }
  }
}) }}
```
- `assistantOverrides.variableValues` is how the lead's **name** reaches the greeting — Vapi
  substitutes `{{customerName}}` in the first message, so each customer is greeted personally.
- The `startsWith('+')` guard re-adds the leading `+` that Google Sheets strips when it reads
  a phone number as a number instead of text.

> ⚠️ **Gotcha — parallel branches don't guarantee order.** The first version fanned the
> trigger out to `Get DNC` and `Get leads` in *parallel*, then referenced `$('Get DNC')` in
> the filter. It failed with **"Node 'Get DNC' hasn't been executed"** because n8n ran the
> leads branch first. Fix: **chain** `Get DNC → leads` so the block list is guaranteed to
> load first — then add "Execute Once" on the leads node to stop the row multiplication.

> ⚠️ **Gotcha — `phoneNumberId` must support the destination.** A *free* Vapi number can't
> place **international** calls (you'll get `400 "Free Vapi numbers do not support
> international calls"`). Importing a Twilio number (BYO) fixes it. For a same-country
> deployment (e.g. an Australian number calling Australian customers) this never comes up.

---

## Workflow 2 — Outbound Call Logger

Vapi calls this workflow's webhook when a call ends. It fetches the full call, flattens it,
and fans it out three ways: **log every call**, **email qualified leads**, **DNC opt-outs**.

```
Webhook ─▶ Wait ─▶ Filter ─▶ HTTP Request ─▶ Normalizer ─┬─▶ Append row in sheet   (all calls)
                (90s)  (end-of-call)  (GET call)          ├─▶ If Qualified ─▶ Send a message (email)
                                                          └─▶ If Opt-Out  ─▶ Add to DNC
```

**1. Webhook** (POST, path `/outbound-logger`) — the URL you put in the Vapi assistant's
**Server URL**. Vapi posts several server messages per call; this catches them all and the
Filter sorts them out.

**2. Wait** (90 seconds) — Vapi computes the structured analysis **asynchronously, a few
seconds after the call ends**. If you fetch the call immediately the analysis is still empty,
so the workflow waits before fetching.

**3. Filter** — passes only the message we care about:
```
{{ $json.body.message.type }}  equals  end-of-call-report
```
The other messages Vapi sends (`speech-update`, `conversation-update`, `status-update`) are
dropped here.

**4. HTTP Request** (GET `https://api.vapi.ai/call/{{ $json.headers['x-call-id'] }}`) —
fetches the complete call object (transcript, recording, cost, structured outputs). Vapi
conveniently sends the call id as an **`x-call-id` header** on the webhook, so we read it
from there. (Fetching a call is free — only *placing* a call costs.)

**5. Normalizer** (Code) — flattens the big Vapi call object into one flat row of ~21 fields.
The key line finds the structured analysis:
```js
const so = call.artifact?.structuredOutputs
        || call.structuredOutputs
        || call.analysis?.structuredOutputs || {};
const a  = Object.values(so)[0]?.result || {};   // the 14 call_analysis fields
```
It then returns `call_id, ended_at, ended_reason, customer_number, recording_url,
transcript, cost` **plus** the 14 analysis fields (sentiment, needs, opportunity_*,
prior_rep, rep_follow_up, outcome_tag, opt_out, summary).

> ⚠️ **Gotcha — where the analysis actually lives.** For web/phone calls the structured
> outputs are at **`call.artifact.structuredOutputs`**, *not* `call.analysis` (which returns
> `{}`) and *not* top-level `call.structuredOutputs`. Reading the wrong path logs every
> analysis field blank while the metadata still fills — a confusing half-empty row. The
> `artifact` path is checked first above. Each output is keyed by a UUID, so we grab
> `Object.values(so)[0].result` rather than hard-coding the key.

**6. Append row in sheet** (Google Sheets → Append) — writes **every** call to the
**Outbound Call Log**. The header row matches the Normalizer's field names exactly, so the
node uses **"Map Automatically"** (no manual column mapping).

**7. If Qualified** (Filter) — a second branch off the Normalizer. Passes only real
opportunities:
```
true  is equal to  {{ ['hot','warm','future'].includes($json.outcome_tag) }}
```
`feedback-only`, `not-interested` and `do-not-call` calls are already logged (step 6) and
stop here — so the rep's inbox only gets genuine leads.

**8. Send a message** (Gmail) — sends the **HTML lead-summary email** to the rep: an
outcome-colored card with the opportunity table, prior rep, summary, a "Play recording"
button and the full transcript. `On Error → Continue`, so an email hiccup can never break the
logging.

**9. If Opt-Out** (Filter) — a third branch off the Normalizer:
```
{{ $json.opt_out === true }}  is true
```

**10. Add to DNC** (Google Sheets → Append) — when a call ended in an opt-out, appends that
number to the **Do Not Call** sheet: `phone = {{ $json.customer_number }}`,
`reason = opt-out during call`, `date = {{ $now.toFormat('yyyy-LL-dd') }}`. Next dialing run,
Workflow 1's **Not on DNC** filter skips them automatically. The loop is closed.

---

## Gotchas & lessons (quick reference)

- **Structured outputs are async** — wait ~90s after the call, then read them from
  `call.artifact.structuredOutputs`, not `call.analysis`.
- **Parallel node references are unreliable** — if a filter needs another node's output,
  chain that node upstream so it's guaranteed to have run.
- **Google Sheets drops the leading `+`** on phone numbers — re-add it in the expression, or
  format the phone column as plain text.
- **Filter direction is easy to invert** — put the boolean expression on one side and the
  literal `true` on the other, and read the whole condition out loud before trusting it.
- **Log everything, notify selectively** — every call goes to the sheet; only qualified leads
  email the rep. Two branches off one Normalizer.
