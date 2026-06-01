# Agent Ops — "10 Hermes Agent Hacks for a 24/7 Assistant"

> Source: YouTube transcript (creator runs an "Open Claw / Hermes" mission-control setup —
> "max HQ", agents named Nova/Sage). Verbatim transcript:
> [`10-hermes-agent-hacks-24-7-assistant.txt`](./10-hermes-agent-hacks-24-7-assistant.txt)

**Why it's here:** the video is about making an AI assistant feel like a **living 24/7
mission control** — agents you *supervise*, not a tool you poke. Most of these patterns map
directly onto Project Quant's autonomous agent mesh; the big takeaway is **make the agents'
work visible** (running / waiting on me / blocked / changed since yesterday) instead of buried
in a chat thread.

## The 10 hacks → mapped to Quant

| # | Hack (video) | In Quant today | What it inspires |
|---|---|---|---|
| 1 | **Mission control** — one place where the whole system is visible (running / waiting / blocked / needs approval / changed since yesterday) | Dashboard + Agents tab + Decision Inbox | A true **command-center home**: live activity + approval inbox + a "since yesterday" digest |
| 2 | **State/event triggers** — Hermes reacts when a board changes state | Event-driven mesh (agents react to market data); `hermes/decisions.py` | Surface **"what changed"**; add inbound triggers (e.g. wallet connected, strategy passed) |
| 3 | **Cron jobs** — time-based, work arrives before you ask | RBI scheduler (`hermes/rbi/scheduler.py`) + per-agent cadences | A visible **"scheduled jobs"** panel; morning/weekly digests to Telegram |
| 4 | **Slash-goals** — objective + sources + constraints + deliverable | Goal system (`hermes/goals.py`) + Growth Manager (Sterling) | Let the operator set a structured fund goal; agent writes the goal spec by interview |
| 5 | **Sub-agents as a research team** | Agent teams (Research/Quant/Trading/Risk/PM), `hermes/supervisor.py` | Already core; keep specialization explicit |
| 6 | **Telegram topics as workspaces** — different work, different rooms | Telegram alerts (`hermes/alerts/telegram.py`) | Route alerts per-topic (trades / risk / research / wallets) |
| 7 | **Kanban for agent task management** — ready / running / done / blocked, who owns what | RBI pipeline (research→backtested→incubating→live) + decision inbox | A **kanban board** for the strategy pipeline + agent tasks (the UI redesign target) |
| 8 | **Skills / SOPs** — reusable standard operating procedures per agent | Agent system prompts + roles | Formalize repeatable processes as named, reusable **skills** |
| 9 | **Webhooks / event agents** — "the world changed" triggers | Data-driven mesh reacting to liquidations/flow/news | Add inbound **webhooks** (e.g. CEX liquidation spikes, PR opened, wallet event) |
| 10 | **Specialized agents** — own model/tools/memory/permissions ("don't let your dentist fly your plane") | Per-agent provider/model/role/team; tiered autonomy + guardrails | Already core; make per-agent capabilities visible in the UI |

## The one line to remember

> *"When your workflow changes state, the agent should know what to do next."* — and the operator
> should be able to **see** it happen. That visibility (mission control + kanban + approval inbox +
> a since-yesterday digest) is the spine of the Quant UI redesign brief.
