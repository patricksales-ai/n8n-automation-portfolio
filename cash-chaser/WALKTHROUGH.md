# CashChaser — Node-by-Node Walkthrough

Two workflows: **the chaser** (daily) and **the weekly brief** (Mondays).
Every node below is in the exported JSON under [`workflows/`](workflows/).

---

# Workflow 1 — The Chaser

## 1 · Schedule Trigger
Fires once a day. Everything downstream is poll-based, so this build needs **no public webhook** —
it runs anywhere n8n runs.

## 2 · Get many invoices — Xero
`Resource: Invoice · Operation: Get Many`

Pulls invoices from the Xero organisation. Two things matter here:

- **Organization is an expression**, not the dropdown. The dropdown can carry a *stale* tenant ID
  and return `403 AuthenticationUnsuccessful` even on a healthy token. Pin the real value from
  `GET https://api.xero.com/connections`.
- **No status filter.** The chaser deliberately fetches *all* statuses, because the paid-detection
  branch (node 16) has to see invoices that have flipped to `PAID`. Filtering here would blind it.

> **Return All** is left off and a **Limit** is used instead — Return All threw a pagination error
> on this node, and the invoice volume never approaches the limit.

## 3 · Tier invoices — Code (Run Once for All Items)
The overdue detector. Keeps only invoices that are genuinely chaseable, then assigns a tone tier.

```javascript
if (inv.Status !== 'AUTHORISED') continue;   // approved but unpaid
if (!(inv.AmountDue > 0)) continue;          // something still owed
const daysOverdue = Math.floor((today - due) / 86400000);
if (daysOverdue <= 0) continue;              // not late yet

let tier;
if (daysOverdue >= 30) tier = 'final';
else if (daysOverdue >= 14) tier = 'firm';
else if (daysOverdue >= 7) tier = 'friendly';
else continue;                               // 1-6 days = grace period
```

Tiers are **thresholds, not exact days** (`>= 30`, not `== 30`) so a missed run never lets an
invoice slip past its tier unchased. Outputs a clean record per invoice: `invoiceId`,
`invoiceNumber`, `amountDue`, `currency`, `dueDate`, `daysOverdue`, `tier`, `contactName`.

## 4 · Get chases — Google Sheets (Get Row(s))
Reads the `Chases` ledger — the memory that makes this safe to schedule.

> **Settings → Always Output Data: ON.** On the very first run the ledger is empty, and a node
> with no output halts its branch — the chaser would never send anything. This one setting is the
> difference between "works" and "silently does nothing on day one".

## 5 · Decide sends — Code (Run Once for All Items)
**The dedupe gate — the most important node in the build.**

```javascript
const tierLevel = { friendly: 1, firm: 2, final: 3 };
const existing = byId[inv.invoiceId];

if (!existing) shouldSend = true;                                  // never chased
else if (String(existing.status).toUpperCase() === 'CLOSED') shouldSend = false;
else if (level > Number(existing.level || 0)) shouldSend = true;   // escalation only
```

Send on **first contact** or a **genuine tier escalation** — never the same level twice. This is
what makes a daily schedule safe: run it twice in a row and the second run sends *nothing*.

It reads invoices from `$('Tier invoices').all()` and the ledger from `$('Get chases').all()`,
so duplicate or blank ledger rows can't confuse it.

## 6 · Message a model — OpenAI (`gpt-4o-mini`)
Writes the reminder. A system message fixes the tone ladder; the user message injects the facts.

```
friendly: warm, appreciative, assume good faith.
firm:     professional and direct, reference how overdue it is.
final:    serious, state this is a final notice before escalation, stay professional.
```

**Output Format = JSON** so it reliably returns `{ subject, body }`.

> Placed **after** the dedupe gate on purpose — only invoices actually being chased cost tokens.

## 7 · Normalize — Code (Run Once for All Items)
Parses the model output and re-joins it to the invoice.

> **Gotcha:** with JSON output mode this node's parsed object sits at
> `items[i].json.output[0].content[0].text` — *not* `message.content`. Matching is done **by
> index** against `$('Decide sends').all()`, which is robust even when `pairedItem` is lost
> across the AI node.

Output = the full invoice record **plus** `subject` and `body`.

## 8 · If — `{{ $json.tier }}` equals `final`
The human-in-the-loop gate. `true` → approval branch. `false` → send immediately.

## 9 · Summarize finals — Code
Collapses every final-tier invoice into **one** item carrying a readable list *and* the original
array:

```javascript
const list = finals.map(r =>
  `• ${r.contactName} — ${r.invoiceNumber}: ${r.currency} ${r.amountDue} (${r.daysOverdue} days overdue)`
).join('\n');
return [{ json: { count: finals.length, list, finals } }];
```

## 10 · Approve with owner — Gmail (Send and Wait for Response)
`Response Type: Approval`. Emails the owner one message listing every final notice, with an
**Approve** button. Execution pauses here until it's clicked.

> **Why aggregate first:** *Send and Wait* only processes the first input item and **drops the
> rest of the batch** — with 6 final invoices it released 2 and lost 4. Bundling them into a single
> item sidesteps that entirely, and one approval for the batch is nicer than six separate emails.

## 11 · Split finals — Code
After approval, fans the batch back out to individual invoices:

```javascript
return $('Summarize finals').item.json.finals.map(f => ({ json: f }));
```

The Send-and-Wait node replaces `$json` with the approval response (`{ data: { approved } }`), so
the invoice array is pulled back from `Summarize finals`.

## 12 · Send a message1 — Gmail (final notices)
Subject `{{ $json.subject }}` · Message `{{ $json.body }}`.

> **Both fields must read `$json`.** Referencing a *named node* here silently mismatches subject
> and body across different invoices (a subject for INV-0006 on a body for 945-OCon), because the
> Code-node fan-out doesn't carry `pairedItem`.

## 13 · Update ledger1 — Google Sheets (Append or Update Row)
Match column **`invoice_id`**. Writes `tier`, `level`, `last_chased_at`, `status = OPEN`, pulling
fields from `$('Split finals').item.json…` (Gmail replaces `$json` with its API response).

## 14 · Send a message — Gmail (friendly / firm)
The `false` branch. Same fields, no approval — these go out immediately.

## 15 · Update ledger — Google Sheets
Same as node 13 but sourced from `$('Normalize').item.json…`. Both branches converge on the same
sheet and the same `invoice_id` match key.

## 16 · Detect paid — Code
The auto-stop. Cross-references **OPEN ledger rows** against the *current* Xero status:

```javascript
if (seen.has(row.invoice_id)) continue;   // one thank-you per invoice
const inv = invoices[row.invoice_id];
const paid = inv && (inv.Status === 'PAID' || Number(inv.AmountDue) === 0);
```

> The `seen` Set matters: without it, duplicate ledger rows produce **one thank-you email per
> row** — a paid invoice once triggered seven identical emails.

Note the chase-stop itself is *structural*: a paid invoice fails the `AUTHORISED / AmountDue > 0`
test back at node 3, so it simply stops appearing. This branch exists to say thank you and close
the row.

## 17 · the thank-you — Gmail
A short thanks referencing the invoice number. Not AI-generated — there's nothing to reason about.

## 18 · mark-CLOSED — Google Sheets (Append or Update Row)
Match on `invoice_id`, set `status = CLOSED`. Updates the row **in place** — the ledger doesn't
grow. That closed row is also what stops node 16 from ever thanking the same customer twice.

---

# Workflow 2 — The Weekly Brief

## 1 · Schedule Trigger
Weekly, Monday morning.

## 2 · Get many invoices — Xero
Filtered to **`Statuses = AUTHORISED`**. Receivables *are* the unpaid invoices, so filtering at
source keeps the result well under the limit and removes any need for pagination.

## 3 · Build summary — Code
Pure arithmetic over the invoice list:

```javascript
outstanding += inv.AmountDue;
if (due < today) overdue += inv.AmountDue;
else if (due <= wk) dueThisWeek += inv.AmountDue;
debtors[name] = (debtors[name] || 0) + inv.AmountDue;
```

Emits `outstanding`, `overdue`, `dueThisWeek`, and the top 5 `topDebtors`.

> Deliberately **not** the Xero Aged Receivables report — that needs the
> `accounting.reports.read` scope, which n8n's built-in Xero credential doesn't expose. The
> numbers are identical and there's one less dependency.

## 4 · Message a model — OpenAI (`gpt-4o-mini`)
Writes **one or two sentences** of insight from those figures — who to chase, what's expected in.
Nothing else; the layout isn't its job.

## 5 · Format brief — Code
Builds the email as deterministic HTML: a totals table (overdue in red), the debtor list, and the
AI insight in a highlighted callout.

> Earlier versions let the model produce the whole email. Plain-text mail collapsed its line
> breaks and rendered literal `**asterisks**`. Splitting the roles — template owns layout, model
> owns judgement — fixed it permanently.

## 6 · Send a message — Gmail
`Email Type: HTML`, body `{{ $json.html }}`.

> In n8n's expression editor, type `{{ … }}` **without** a leading `=`. Adding one emits a stray
> `=` at the top of the email body.

---

## Gotchas worth stealing

| Symptom | Cause | Fix |
|---|---|---|
| Xero `403 AuthenticationUnsuccessful` on one org only | The org dropdown holds a stale tenant ID | Read the real one from `GET /connections`, pin it as an expression |
| Chaser sends nothing on the very first run | `Get chases` returns 0 rows on an empty ledger and halts the branch | `Always Output Data: ON` |
| Every run re-sends the same reminders | No ledger gate | Dedupe on first-contact-or-escalation |
| Only some final notices send | *Send and Wait* drops batch items beyond the first | Aggregate → one approval → split back |
| Subject and body describe different invoices | A named-node reference after a Code fan-out (no `pairedItem`) | Use `$json` for both |
| Seven identical thank-you emails | Duplicate ledger rows, one email per row | `seen` Set keyed on `invoice_id` |
| Ledger row count exploding | Cell contents cleared instead of rows deleted — blank rows still count | Delete *rows*, not contents |
