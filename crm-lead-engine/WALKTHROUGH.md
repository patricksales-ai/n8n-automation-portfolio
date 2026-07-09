# Build Walkthrough — node by node

How the CRM Lead Engine works, explained so you can read and present it. It's **two AI lanes on
one HubSpot CRM**: a **scheduled enrichment agent** that cleans + scores existing contacts, and
an **inbound capture** flow that turns incoming emails into qualified HubSpot leads. Both talk to
HubSpot over its REST API.

> n8n note: **Information Extractor** returns structured JSON from an LLM. **Split Out** turns an
> array field into one item per element. **Loop Over Items** (Split In Batches) processes items in
> controlled batches and has a *loop* output (per batch) and a *done* output (after all batches).

---

## Workflow 1 — CRM Enrichment & Scoring Agent (scheduled)

Runs on a timer, finds contacts that haven't been enriched yet, infers missing attributes from
what's already on the record, scores them, and writes it all back to HubSpot.

```
Schedule ─▶ HTTP (list contacts) ─▶ Split Out ─▶ Filter ─▶ Loop Over Items ─┬─(loop)▶ Normalize ─▶ Extractor ─▶ HubSpot Update ─┐
                                                                            │                                                   └─▶ back to loop
                                                                            └─(done)▶ Code ─▶ Slack summary
```

**1. Schedule Trigger** — fires the run (e.g. daily).

**2. HTTP Request** (`GET api.hubapi.com/crm/v3/objects/contacts`) — pulls the contact list from
HubSpot. (Uses the REST API with a private-app token rather than the native node, so it runs
cleanly on self-hosted n8n.)

**3. Split Out** — explodes the returned `results` array into one item per contact.

**4. Filter** — keeps only contacts **not already enriched**
(`{{ $json.properties.ai_enriched }}` ≠ `"true"`), so each contact is processed once and the run
is cheap on repeat.

**5. Loop Over Items** — walks the un-enriched contacts one batch at a time.
- **Loop output → Normalize** (Set) → **Information Extractor** (+ **OpenAI Chat Model**) — infers
  attributes from what's *already on the record* (e.g. company / seniority) and assigns a score —
  **no external data-append service**, just reasoning over existing fields → **HubSpot Update**
  (`PATCH …/contacts/{id}`) writes the inferred fields + score + `ai_enriched = true` back, then
  returns to the loop.
- **Done output → Code in JavaScript** → **Send a message** (Slack) — after all contacts are
  processed, posts a run summary to the team.

---

## Workflow 2 — Inbound Lead Capture

Watches the inbox, decides whether an email is a real lead, writes it into HubSpot, and routes it
by temperature — auto-replying to the hot ones.

```
Gmail Trigger ─▶ Get message ─▶ Lead Qualifier ─▶ If(is_lead) ─▶ HubSpot Upsert ─▶ Switch ─┬─ hot  ─▶ label ─▶ draft+Reply + Slack
                                (classify)                                                  ├─ warm ─▶ label
                                                                                            └─ cold ─▶ label
```

**1. Gmail Trigger → Get a message** — catches a new email and fetches its full content.

**2. Lead Qualifier** (Information Extractor + **OpenAI Chat Model**) — scores the email against
BANT-style criteria and returns structured JSON, including `is_lead` and a temperature.

**3. If** — `{{ $json.output.is_lead }}` true? Non-leads stop here; leads continue.

**4. HubSpot Upsert Lead** (`POST …/contacts/batch/upsert`) — creates or updates the contact in
HubSpot keyed by email, so the same person is never duplicated.

**5. Switch** (3 rules) — routes by temperature:
- **Hot → Add label** → **Basic LLM Chain** (+ **OpenAI Chat Model1**) drafts a tailored reply →
  **Reply to a message** (Gmail) sends the acknowledgment → **Send a message** (Slack) alerts
  sales. Hot leads get an instant, personalized response.
- **Warm → Add label** — captured + tagged for follow-up (even incomplete inquiries aren't
  dropped).
- **Cold → Add label** — filed without noise.

> ⚠️ **Design choice — infer, don't buy.** The enrichment agent deliberately reasons over the
> data already in HubSpot instead of calling a paid data-append API — cheaper, and it works on any
> CRM record. The `ai_enriched` flag makes the scheduled run idempotent.

## Gotchas & lessons

- **HubSpot on self-hosted n8n** — use the **REST API + private-app token** via HTTP Request; the
  native node expects an app token it can't supply on self-host.
- **Upsert, don't insert** — batch/upsert keyed by email prevents duplicate contacts.
- **Flag processed rows** — `ai_enriched = true` keeps the scheduled agent from re-charging you for
  the same contact.
