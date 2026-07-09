# Build Walkthrough — node by node

How the Social Content Bot works, explained so you can read and present it. It turns **one idea
from a sheet** into platform-tailored posts, **critiques its own draft** to raise quality, then
**pauses for a human Approve/Decline in Slack** before anything publishes — and logs the live link
back.

> n8n note: a **Basic LLM Chain** is a single prompt→completion step. **Slack "Send message and
> wait for response"** posts a message with buttons and **pauses the workflow** until someone
> clicks, then resumes with their choice.

```
Schedule ─▶ Get rows ─▶ Draft (LLM) ─▶ Normalize ─▶ Critique/Refine (LLM) ─▶ Normalize ─▶ Slack approve? ─▶ If ─┬─ approved ─▶ Ayrshare post ─▶ log link
           (the idea)                                                                     (human gate)          └─ declined ─▶ (stops)
```

**1. Schedule Trigger** — runs the bot on a cadence (e.g. daily).

**2. Get row(s) in sheet** (Google Sheets) — reads the next content idea/topic from the content
calendar sheet (the queue of things to post).

**3. Basic LLM Chain** (+ **OpenAI Chat Model**) — the **drafter**. Turns the idea into
platform-tailored post copy.

**4. Normalize Draft** (Code) — cleans the model output into a predictable shape (strips
formatting, pulls out the post text) so the next step has stable input.

**5. Basic LLM Chain1** (+ **OpenAI Chat Model1**) — the **critic/refiner**. It reviews the draft
against quality criteria and rewrites it — a self-check pass that noticeably lifts the output
versus a single-shot generation.

**6. Normalize Final** (Code) — shapes the refined post into its final form, ready to publish and
ready to show a human.

**7. Send message and wait for response** (Slack) — the **human gate**. Posts the finished post to
a Slack channel with **Approve / Decline** buttons and **pauses the workflow** until someone
clicks. Nothing is published while it waits.

**8. If** — `{{ $json.data.approved }}` true?
- **true → HTTP Request** (`POST api.ayrshare.com/api/post`) — publishes the post to the connected
  social accounts via Ayrshare → **Update row in sheet** writes the status + the **live post link**
  back to the calendar, closing the loop.
- **false →** the flow stops; nothing is posted.

> ⚠️ **Design choice — generate, self-critique, *then* gate.** Two things make this safe and good:
> the **critique pass** (step 5) catches weak drafts before a human ever sees them, and the
> **Slack approval** (step 7) means nothing reaches a real audience without a person clicking
> Approve. Autonomy with a human hand on the publish button.

## Gotchas & lessons

- **Normalize between LLM steps** — a small Code/Set node between chained LLM calls keeps each
  prompt's input stable and the workflow debuggable.
- **`send-and-wait` = built-in approval** — Slack's wait-for-response node is a clean way to pause
  a workflow for a human decision without extra infra.
- **Write the result back** — logging the live link to the sheet turns the calendar into an audit
  trail of what actually shipped.
