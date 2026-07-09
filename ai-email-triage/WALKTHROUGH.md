# Build Walkthrough — node by node

How the AI Email Triage works, explained so you can read and present it. One inbound email is
classified by an LLM, then **routed to a different action per category** — urgent gets a Slack
alert, reply-worthy gets a *draft* (never auto-sent), leads get flagged, newsletters get filed,
spam gets binned. Every message is labelled so it's never processed twice.

> n8n note: an **Information Extractor** is an LLM node that returns **structured JSON** against
> a schema (here: a category + fields). A **Switch** routes an item down a different branch
> depending on a value.

```
Gmail Trigger ─▶ Normalize ─▶ Information Extractor ─▶ Switch ─┬─ urgent    ─▶ Slack alert + label
                              (classify)              (route)  ├─ reply     ─▶ draft reply + label
                                                               ├─ lead      ─▶ Slack alert + label
                                                               ├─ newsletter─▶ mark read + label
                                                               └─ spam      ─▶ delete
```

**1. Gmail Trigger** — fires on each new incoming email.

**2. Normalize** (Set) — pulls the fields the model needs into a clean shape: `from`,
`subject`, and the body/snippet. Keeps the prompt small and predictable.

**3. Information Extractor** (+ **OpenAI Chat Model**, `gpt-4o-mini`) — the classifier. It reads
the normalized email and returns structured JSON — a **category** (urgent / reply-needed / lead /
newsletter / spam) plus any supporting fields it extracts. Because it's an Information Extractor,
the output is a typed object the Switch can branch on, not free text.

**4. Switch** (5 rules) — the router. It sends the email down exactly one branch based on the
category:

- **Urgent → Send a message** (Slack) — posts an alert to the team channel so a human sees it
  immediately, then **Add label** tags the email (e.g. `Urgent`).
- **Reply-needed → Basic LLM Chain** (+ **OpenAI Chat Model1**) — drafts a suitable reply, then
  **Create a draft** (Gmail) saves it to the thread **as a draft**. A human reviews and sends —
  nothing goes out automatically. The email is labelled too.
- **Lead → Send a message** (Slack) — flags the potential lead to sales and labels it.
- **Newsletter → Mark a message as read** (Gmail) + label — files it quietly without cluttering
  the inbox.
- **Spam → Delete a message** (Gmail) — removes it.

**5. Add label to message** (Gmail, several) — each branch applies a category label. Labelling
also acts as the **"already handled" marker** — the trigger/flow skips anything already tagged,
so an email is never triaged twice.

> ⚠️ **Design choice — nothing is sent without a human.** The reply path deliberately stops at a
> **draft**. The bot does the reading, sorting and first-draft writing; a person keeps the send
> button. That's what makes it safe to point at a real inbox.

## Gotchas & lessons

- **Classify once, branch many** — a single structured-classification step feeds a Switch, so
  adding a new category is one rule + one branch, not a new pipeline.
- **Label = idempotency** — tagging each email both categorizes it *and* prevents re-processing
  on the next trigger.
- **Draft, don't send** — keep a human in the loop for anything outbound.
