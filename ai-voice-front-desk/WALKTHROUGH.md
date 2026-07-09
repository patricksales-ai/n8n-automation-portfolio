# Build Walkthrough — node by node

How the AI Voice Front Desk works, explained so you can read and present it. The product is
**four small n8n workflows** that share one "brain." A phone call comes in over **Twilio**,
which talks to n8n in turns using TwiML (`<Say>` to speak, `<Gather input="speech">` to
listen). The same brain also runs **outbound**.

> Twilio note: each webhook returns **TwiML** (XML). `<Say>` speaks text, `<Gather>` records
> the caller's speech and POSTs it to the next webhook. n8n's "Respond to Webhook" node returns
> that XML with `Content-Type: text/xml`.

```
Twilio number ─▶ Inbound Greeting ─▶ AI Voice Agent ─┬─▶ Smart Booking (own webhook)
                                                      └─▶ <Dial> human (transfer)
Lead sheet ─────▶ Outbound Call ─────▶ (same AI Voice Agent brain)
```

---

## Workflow 1 — Inbound Greeting

The number the customer dials points here.

**1. Webhook** (`/voice-inbound`, POST) — Twilio hits this the moment a call connects.

**2. Respond to Webhook** — returns TwiML: a spoken greeting plus a
`<Gather input="speech" action=".../voice-respond">`. The greeting plays, then Twilio records
the caller's first sentence and POSTs it to the AI Voice Agent. Header `Content-Type: text/xml`
is required or Twilio won't parse it.

---

## Workflow 2 — AI Voice Agent (the brain)

Every spoken turn from the caller lands here. This is the workflow that thinks.

**1. Webhook** (`/voice-respond`, POST) — receives the caller's transcribed speech
(`SpeechResult`) and the `CallSid` (a unique per-call id).

**2. AI Agent** — the conversation engine. User prompt = `{{ $json.body.SpeechResult }}`.
Three attached sub-nodes:
- **OpenAI Chat Model** — `gpt-4o-mini`.
- **Simple Memory** — keyed by `{{ $json.body.CallSid }}`, so each call has its own running
  memory and the agent remembers earlier turns *within the same call*.
- **book_appointment** — an **HTTP Request tool** the agent can call. It POSTs to the Smart
  Booking workflow's own webhook (`/vfd-book`) with `name`, `service`, `startTime` (filled by
  the model via `$fromAI(...)`) and `phone` (taken from the Twilio payload). The system message
  tells the agent to **read the details back and wait for a "yes" before booking.**

**3. If** — checks the agent's reply for a transfer signal:
`{{ $json.output }}` **contains** `<<TRANSFER>>`. (The system message says: if the caller asks
for a human, reply with exactly `<<TRANSFER>>`.)
- **true → Respond Transfer** — returns TwiML `<Say>connecting you…</Say><Dial>+…</Dial>`,
  handing the call to a real person.
- **false → Respond to Webhook** — returns `<Say>{{ $json.output }}</Say>` followed by another
  `<Gather>` — i.e. speak the answer, then listen again. This is the conversational loop.

> ⚠️ **Gotcha — why booking runs over HTTP, not a sub-workflow tool.** On n8n 2.x, a **Code
> node inside a sub-workflow invoked by an AI Agent's "Call Sub-Workflow" tool** throws a masked
> task-runner error. The fix was to give Smart Booking its **own webhook** and have the agent
> call it as a plain **HTTP tool** — same result, no task-runner bug.

---

## Workflow 3 — Smart Booking

Books an appointment, spreads work fairly across three staff calendars, and never
double-books. Called by the agent over its webhook.

**1. Booking Webhook** (`/vfd-book`, POST) — receives `name`, `service`, `startTime`, `phone`.

**2. Get Rows** (Google Sheets) — reads the bookings log. The row count acts as the
round-robin counter (the log *is* the counter — survives restarts, unlike in-memory state).

**3. Build Candidates** (Code) — outputs the three stylists in **round-robin order** starting
at `idx = rowCount % 3`, each carrying their calendar id + the requested slot. It also contains
a **date guard**: any past-dated `startTime` (a model hallucination) is rolled forward a year at
a time until it's in the future, so a booking can never land in the past.

**4. Get availability in a calendar** (Google Calendar, once per candidate) — asks each
stylist's calendar whether that slot is free.

**5. Pick First Free** (Code) — zips the candidates with their availability and picks the
**first stylist who's actually free**, returning `booked: true {stylist, calendarId, …}` or
`booked: false {response: "try another time"}`.

**6. If** — `{{ $json.booked }}` is true?
- **true →** **Create an event** (Google Calendar) → **Append row in sheet** (log the booking)
  → **Confirmation** (Set node builds the spoken response) → **Send an SMS** (Twilio, `On Error →
  Continue` so a failed SMS never breaks a booking) → **Respond to Webhook** (returns the
  confirmation text to the agent).
- **false →** **Respond to Webhook1** — returns "try another time" to the agent.

> ⚠️ **Gotcha — round-robin without double-booking.** Rotating across staff isn't enough: two
> callers wanting the same slot would both be assigned by pure rotation. The **availability
> check + "pick first free"** is what guarantees the assigned stylist is genuinely open.

---

## Workflow 4 — Outbound Call (SDR)

Turns the same brain into an outbound dialer.

**1. Manual Trigger** — starts a run (would be a Schedule trigger in production).

**2. Get row(s) in sheet** (Google Sheets) — reads the lead list (`Name | Phone`).

**3. Make a call** (Twilio, once per lead) — dials each lead with **TwiML** that opens with a
personalized pitch (`<Say>Hi {{ $json.Name }}…</Say>`) then `<Gather action=".../voice-respond">`.
The lead's spoken reply funnels into **the same AI Voice Agent** — so qualifying, booking and
transfer all work outbound too, with zero duplicated logic.

> ⚠️ **Gotcha — Google Sheets eats the `+`.** A number like `+1415…` is stored as `1415…`
> (Sheets treats it as a number). The dial expression re-adds the `+`:
> `{{ String($json.Phone).startsWith('+') ? $json.Phone : '+' + $json.Phone }}`.

---

## Gotchas & lessons (quick reference)

- **Split workflows by mode, not by webhook** — greeting, brain, booking and outbound are
  separate workflows; the brain is reused by both inbound and outbound.
- **Per-call memory** — key Simple Memory by `CallSid` so calls don't bleed into each other.
- **Confirm before you book** — the agent reads details back and waits for "yes"; only then
  calls `book_appointment`.
- **The log is the counter** — deriving round-robin from row count survives restarts where
  in-memory state doesn't.
- **Guard model dates** — always roll a hallucinated past date forward so nothing books in the
  past.
