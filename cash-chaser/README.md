# CashChaser — AI Accounts-Receivable Autopilot (Xero)

**Small businesses don't have a cash-flow problem. They have a "nobody chased that invoice" problem.**
CashChaser polls Xero every morning, finds what's actually overdue, and writes each customer a
reminder in the right tone for how late they are — friendly at 7 days, firm at 14, final at 30+.
It remembers who it already chased, escalates one level at a time, **stops the moment an invoice
is paid** and sends a thank-you instead. On Monday it emails the owner a cash-flow brief.

Final notices never go out on their own — they wait for one click from the owner.

> 📖 **[WALKTHROUGH.md](WALKTHROUGH.md)** explains every node in both workflows, line by line.

---

## Why this exists

**The problem —** chasing invoices is the job nobody wants. It's awkward, it's repetitive, and
it's always the thing that slips when the week gets busy. So a 7-day-late invoice quietly
becomes 40 days late. Xero *does* ship built-in reminders, but they're one fixed template on a
fixed timer: the same words at day 7 and day 60, no memory of what was already sent, no way to
hold a final notice for review, and nothing that tells the owner where the money actually is.

**The result —** every overdue invoice gets chased, in words that escalate as the debt ages,
without anyone remembering to do it. The customer who paid yesterday gets a thank-you, not a
third demand. And the owner starts Monday knowing exactly who owes what and who to lean on.

---

## What it does

- **Finds** genuinely overdue invoices in Xero (`AUTHORISED`, amount still due, past the due date).
- **Tiers** each one by how late it is — `friendly` (7–13d) · `firm` (14–29d) · `final` (30d+).
  The first 6 days are a deliberate grace period.
- **Writes** a personalised reminder with AI — the real invoice number, amount, and due date,
  in a tone that matches the tier.
- **Remembers** every chase in a Google Sheets ledger, so it only sends on **first contact or a
  genuine escalation** — never the same level twice. Run it twice, the second run sends nothing.
- **Holds final notices** for human approval — all of them bundled into **one** email with an
  Approve button.
- **Stops automatically** when an invoice is paid, sends a **thank-you**, and closes the ledger row.
- **Briefs the owner** every Monday: total outstanding, overdue, expected this week, top debtors,
  and an AI read on who to chase — as a designed HTML email, not a data dump.

---

## Architecture

Two workflows.

**1 · The chaser** (daily)

```
Schedule Trigger
  → Get many invoices     (Xero — ALL statuses; the paid branch needs to see PAID)
  → Tier invoices         (Code: overdue filter + friendly/firm/final)
  → Get chases            (Sheets: read the ledger)
     ├─ Decide sends      (Code: dedupe — first contact or escalation only)
     │    → Message a model  (gpt-4o-mini, JSON mode → subject + body)
     │    → Normalize        (Code: parse JSON, re-join invoice data)
     │    → IF tier = final
     │         ├ true  → Summarize finals → Approve with owner (Send-and-Wait)
     │         │           → Split finals → Send a message1 → Update ledger1
     │         └ false → Send a message → Update ledger
     └─ Detect paid       (Code: OPEN ledger rows now paid in Xero)
          → the thank-you → mark-CLOSED
```

**2 · The weekly brief** (Mondays)

```
Schedule Trigger → Get many invoices (AUTHORISED only)
  → Build summary  (Code: outstanding · overdue · due-this-week · top 5 debtors)
  → Message a model (gpt-4o-mini writes the insight)
  → Format brief    (Code: branded HTML)
  → Send a message  (Gmail)
```

### The design decisions that matter

| Choice | Why |
|---|---|
| **A Sheets ledger + a dedupe gate**, not "send on a timer" | This is what makes it safe to schedule. Without it every run re-sends. The gate allows a send only on first contact or a real tier escalation — so running twice sends exactly once. |
| **Tier by *threshold*, not exact day** | `>= 30`, not `== 30`. If the workflow misses a day, nothing slips through the cracks unchased. |
| **Chaser fetches ALL invoice statuses**; the brief filters to `AUTHORISED` | The paid-detection branch has to *see* an invoice flip to `PAID` — filtering it out at source would blind it. The brief only cares about unpaid, so it filters early and never needs pagination. |
| **JSON output mode + a Normalize Code node** | The Structured Output Parser is unreliable on self-hosted n8n. JSON mode plus an explicit parse is the portable pattern. Note the parsed object lands at `output[0].content[0].text`. |
| **Digest approval: aggregate → one approval → split back** | n8n's *Send and Wait* processes one item and **drops the rest of the batch** — 6 final notices became 2. Aggregating them into a single approval, then fanning back out after the click, gates the whole batch reliably *and* is better UX than one email per invoice. |
| **Receivables computed from invoice data** | Avoids depending on the `accounting.reports.read` scope, which n8n's built-in Xero credential doesn't let you set. Same numbers, one less thing to break. |
| **Xero tenant ID pinned as an expression** | The node's org dropdown can hold a *stale* tenant ID and 403 on a perfectly valid token. Read the real one from `GET /connections` and pin it. |
| **Deterministic HTML for the brief, AI for the insight** | Letting the model emit markdown produced a run-on wall of text. The template owns the layout; the model contributes one sentence of judgement. |

---

## Tech stack

- **n8n** (self-hosted) — orchestration
- **Xero Accounting API** — invoices, payments, contacts
- **OpenAI `gpt-4o-mini`** — reminder copy + the weekly insight
- **Google Sheets** — the chase ledger (state + dedupe)
- **Gmail** — reminders, approvals, thank-yous, the weekly brief

---

## Setup

1. **Create a Xero app** at [developer.xero.com](https://developer.xero.com) → *Web app*.
   Set the redirect URI to your n8n callback:
   `https://YOUR_N8N_HOST/rest/oauth2-credential/callback`

2. **Add the Xero OAuth2 credential** in n8n and connect it. Pick the **Demo Company** to try it
   with realistic data (Xero pre-loads it with overdue invoices).

3. **Pin your tenant ID.** Don't trust the node's org dropdown — read the real value:
   ```
   GET https://api.xero.com/connections      (auth: your Xero OAuth2 credential)
   ```
   Copy the `tenantId` and set the Xero node's **Organization** field to an *expression*
   in place of `YOUR_XERO_TENANT_ID`.
   > ⚠️ The Demo Company **resets roughly every 28 days** and gets a **new tenant ID** — re-run
   > `/connections` if it suddenly starts returning `403 AuthenticationUnsuccessful`.

4. **Create the ledger sheet** — a Google Sheet with a tab named `Chases` and this header row:
   ```
   invoice_id | invoice_number | contact | tier | level | last_chased_at | status
   ```
   Set it on the `Get chases`, `Update ledger`, `Update ledger1`, and `mark-CLOSED` nodes
   (replacing `YOUR_SHEET_ID`). All three write nodes use **Append or Update Row**, matching on
   **`invoice_id`**.
   > On `Get chases`, turn **Always Output Data** ON — with an empty ledger the node returns
   > nothing and the whole chase branch would stop on the very first run.

5. **Point the email nodes at a safe address** while testing (`you@example.com` in the export).
   Turn **Append n8n Attribution** off on every Gmail node for client-ready mail.

6. **Import the workflows:**
   [`workflows/cashchaser-chaser.json`](workflows/cashchaser-chaser.json) ·
   [`workflows/cashchaser-weekly-brief.json`](workflows/cashchaser-weekly-brief.json)

---

## Try it on Xero's Demo Company

You don't need a real business to see this work end to end. Xero's **Demo Company** ships with
a stack of genuinely overdue invoices, which is exactly what the chaser needs.

A real run against it produced:

| Invoice | Contact | Overdue | Tier |
|---|---|---|---|
| `AP` | Xero | 7 days | friendly |
| `AP` | Swanston Security | 17 days | firm |
| `INV-0006` | City Limousines | 56 days | final |
| `945-OCon` | Central Copiers | 52 days | final |

The friendly one opens *"We hope this message finds you well… if you've already processed this
payment, please disregard."* The final one: *"…failure to settle this account may result in
escalation procedures."* Same pipeline, same data, different register.

Then marking one invoice paid in Xero: it disappeared from the next run's chase list, a single
thank-you went out, and its ledger row flipped to `CLOSED`.

---

## Security notes

- **No secrets in this repo.** n8n exports *reference* credentials by name — never keys or tokens.
- **Placeholders throughout** — the Xero tenant ID, the Google Sheet ID, and the recipient address
  are all scrubbed (`YOUR_XERO_TENANT_ID`, `YOUR_SHEET_ID`, `you@example.com`).
- **No customer data.** Invoices, contacts, and amounts live in Xero and the ledger sheet — the
  exported JSON contains none of them. Everything shown above comes from Xero's public Demo Company.

---

## Results & highlights

- **Safe to schedule** — the ledger dedupe was proven by running it twice: 7 reminders on the
  first pass, **zero** on the second.
- **Nobody gets chased after paying** — a paid invoice drops out of the chase list on the next
  run, gets exactly one thank-you, and closes itself.
- **Final notices stay under human control** — one approval email, one click, whole batch released.
- **The owner gets a real brief**, not a spreadsheet — totals, top debtors, and a plain-English
  read on where the cash is.

---

## Roadmap

- **Bills inbox** — forward a receipt to a label, AI extracts vendor/amount/lines, creates a
  **draft bill** in Xero. Turns the AR chaser into a bookkeeping copilot.
- **Per-invoice approval loop** — wrap the approval in *Loop Over Items* so each final notice can
  be approved or skipped individually, instead of as one batch.
- **Multi-tenant** — one deployment chasing for many Xero orgs (a bookkeeping practice's whole
  client list), with the ledger moved from Sheets to Postgres.

---

## License

MIT — see `LICENSE` (add your preferred license file).
