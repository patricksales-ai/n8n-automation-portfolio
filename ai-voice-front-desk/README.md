# AI Voice Front Desk (Twilio + n8n + OpenAI)

An AI voice agent that answers the phone, **qualifies callers, books appointments, and
transfers to a human** — and can also run **outbound**, dialing a lead list and pitching
each one. One shared "brain" powers both directions, and every booking lands on a real
Google Calendar with a Google Sheet as the system of record.

Built to mirror what a real AI receptionist / AI SDR needs in production: turn-based
speech over the phone, mid-call calendar booking, fair round-robin across staff, an
availability guard so nobody gets double-booked, and a clean hand-off to a human when the
caller asks for one.

---

## What it does

**📞 Inbound receptionist**
1. Greets the caller and asks how it can help.
2. Understands free-form speech (booking requests, questions, "talk to a person").
3. **Reads the details back and waits for a "yes"** before it books anything.
4. Books the appointment, **round-robins across three stylists**, and only offers a slot
   that is **actually free** (checks each calendar's availability first).
5. Confirms the booking out loud (with the assigned stylist), logs it to a sheet, and
   fires an SMS confirmation.

**📤 Outbound SDR**
- Reads a lead list from a Google Sheet, dials each lead, and opens with a **personalized
  pitch**. The lead's reply funnels into the exact same brain — so it can qualify and book
  on the outbound call too.

**🙋 Human transfer**
- If the caller asks for a real person (or is clearly frustrated), the agent stops the AI
  loop and **`<Dial>`s a human**.

---

## How it's built

```
Twilio number ──▶ Inbound Greeting ──▶ AI Voice Agent ──┬─▶ Smart Booking (webhook)
   (voice)          (TwiML <Say>+       (gpt-4o-mini +   │      round-robin + availability
                     <Gather speech>)    memory + tools)  │      → Google Calendar + Sheet + SMS
                                                          └─▶ Human transfer (<Dial>)
```

- **Turn-based voice** — each turn is a Twilio `<Gather input="speech">` → n8n webhook →
  OpenAI → TwiML response. No streaming server required; it's pure n8n.
- **Per-call memory** — keyed by Twilio `CallSid`, so the agent remembers the conversation
  within a call.
- **Deterministic booking** — the LLM decides *what* to book; a dedicated booking workflow
  decides *who* and *whether the slot is free*, so scheduling logic never depends on the
  model guessing.
- **Booking runs as its own top-level webhook** and the agent calls it over HTTP. This
  keeps the calendar/round-robin logic decoupled and reusable across inbound *and*
  outbound, and sidesteps a self-hosted task-runner limitation with code steps inside
  agent-invoked sub-workflows.

**Stack:** Twilio (voice + SMS) · n8n · OpenAI `gpt-4o-mini` · Google Calendar · Google
Sheets.

---

## Workflows

| File | Role |
|------|------|
| [`inbound-greeting.json`](workflows/inbound-greeting.json) | The first thing a caller hears; returns the greeting TwiML and hands the conversation to the agent. |
| [`ai-voice-agent.json`](workflows/ai-voice-agent.json) | The brain — OpenAI chat + per-call memory, the `book_appointment` tool, and the human-transfer branch. |
| [`smart-booking.json`](workflows/smart-booking.json) | Round-robin across 3 stylists, per-calendar availability check, creates the event, logs the row, sends the SMS. |
| [`outbound-sdr-dialer.json`](workflows/outbound-sdr-dialer.json) | Reads a lead list and places a personalized outbound call to each lead. |

---

## Notes

- Phone numbers, host URLs, calendar IDs, and credentials in the JSON are placeholders or
  references — swap in your own Twilio number, n8n host (or a tunnel for local dev), Google
  Calendar/Sheet IDs, and credentials to run it.
- For a stable public URL (so Twilio can reach a locally-hosted n8n), use a tunnel
  (cloudflared / ngrok) or a hosted n8n instance, and point the Twilio number's Voice
  webhook at `https://YOUR_N8N_HOST/webhook/voice-inbound`.
