# 🤖 Multi-Tool Telegram AI Agent

A production-ready AI agent built in n8n that handles both text and voice 
messages via Telegram, routes them intelligently, and responds using 
multi-tool function calling with persistent memory.

> 📖 **[WALKTHROUGH.md](WALKTHROUGH.md)** explains every node, line by line.

---

## 💡 Why This Exists

**The problem —** quick tasks (check the calendar, fire off an email, look something
up) are annoying to do from your phone, and typing them out is slower than just
*saying* them — especially on the move.

**The result —** a personal assistant that lives in Telegram and takes **voice or
text**. Send a voice note, it transcribes it with Whisper, figures out which tool to
call, does the work, and replies — by text or generated audio — all from a chat you
already have open.

---

## 🧠 What It Does

- Accepts **text and voice messages** from Telegram
- Transcribes voice notes via OpenAI Whisper
- Routes messages through a **Switch node** (text vs. audio path)
- Passes input to an **AI Agent** with memory and tool access
- Responds via **text or generated audio** back to Telegram

---

## 🛠 Tools Available to the Agent

Each tool is its own **sub-workflow** running its own mini-agent — not a single node. The
main agent decides *which* capability a request needs and delegates; the sub-agent then
decides *which specific operation* within that capability to run.

| Tool | Purpose | Workflow |
|------|---------|----------|
| **Sub Calender Agent** | Calendar — create, read, update, delete events, and create events with attendees | [`sub-agent-calendar.json`](workflows/sub-agent-calendar.json) |
| **Sub Workflow Mails** | Email — read, send, reply, and **save drafts** (so a reply can be prepared for a human to approve rather than sent blind) | [`sub-agent-email.json`](workflows/sub-agent-email.json) |
| **Search Agent** | Research — Wikipedia, live web search via SerpApi, and Hacker News | [`sub-agent-research.json`](workflows/sub-agent-research.json) |

**Why sub-workflows instead of attaching every tool to one agent:** it keeps the main
agent's decision small (three capabilities, not a dozen operations), each domain's tools
and prompt stay isolated and independently testable, and a sub-agent can be reused by
other workflows without dragging the Telegram front end along.

---
## 📸 Workflow Screenshot

![Workflow](docs/workflow.png)

---

## 🎬 Demo

![Demo](docs/demo.gif)

A live run — a Telegram message routes through the agent to the right tool
(calendar / email / web search) and the reply lands right back in the chat.

## ⚙️ Tech Stack

- **Platform:** n8n (self-hosted / cloud)
- **LLM:** OpenAI GPT-4o
- **Memory:** n8n Simple Memory
- **Voice:** OpenAI Whisper (transcription) + TTS (audio reply)
- **Trigger:** Telegram Bot API

---

## 🚀 How to Use

1. **Import all four workflows** — the three sub-agents *first*, then the main agent:
   - [`workflows/sub-agent-calendar.json`](workflows/sub-agent-calendar.json)
   - [`workflows/sub-agent-email.json`](workflows/sub-agent-email.json)
   - [`workflows/sub-agent-research.json`](workflows/sub-agent-research.json)
   - [`workflows/multi-tool-telegram-agent.json`](workflows/multi-tool-telegram-agent.json)

2. ⚠️ **Re-point the three tool references.** n8n assigns new workflow IDs on import, so
   the main agent's `Call '…'` tool nodes will still hold *this* instance's IDs. Open each
   of the three tool nodes and re-select the matching sub-workflow from the dropdown.

   > This is the one step people skip, and it fails **silently** — the agent keeps working
   > and simply loses that capability, with no error until someone asks for it. It bit this
   > very build: the mail tool pointed at a workflow that no longer existed, so the agent
   > quietly couldn't do email at all. If a capability seems to be ignored, check this first.

3. Set credentials: Telegram Bot, OpenAI, Gmail, Google Calendar, SerpApi
4. Set your calendar in the calendar sub-agent (replace `YOUR_CALENDAR_ID` — for a primary
   Google calendar this is just your Google account address)
5. Activate the main workflow and message your Telegram bot

---

## ✨ Results & Highlights

- **Voice-first** — speak instead of type; Whisper transcription makes it hands-friendly.
- **One agent, many tools** — calendar, email, and web search behind a single chat, with
  the agent routing each request to the right sub-workflow itself.
- **Replies in kind** — answers by text or generated audio, right inside Telegram.
- **Persistent memory** — follow-up messages resolve against the running conversation.

---

## 👤 Author

**Patrick Sales** — Senior Automation & AI Engineer  
[LinkedIn](https://www.linkedin.com/in/patrickomarsales/) | [GitHub](https://github.com/fatquicksales0022-hash)
