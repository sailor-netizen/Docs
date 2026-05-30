# UI/UX Spec — Quant App (MoonDev App)
> Project Quant — Source of Truth  
> Tab layout, components, data panels, interaction patterns  
> Derived from `Inspo/Transcripts/Moondev/` — live app walkthroughs, Quant App demos, and build sessions

---

## Overview

The Quant App (also called MoonDev App) is a downloadable desktop application — the operational hub for the entire Project Quant stack. It combines live market data, bot management, terminal access, and backtesting in one frictionless interface.

**Primary design principle:** No context switching. Everything visible at once. Stop flipping between tabs on a browser and different terminal windows. The whole operation runs in one screen.

**Platform:** Electron (cross-platform desktop app, Mac + Windows). Available at `moondev.com` — download and unlock with API key.

---

## Tab Structure (5 Confirmed Tabs)

```
[ Trade ] [ Code ] [ Data ] [ Backtest ] [ HLP ]
```

---

### Tab 1 — Trade

The primary operations tab. Shows live positions closest to liquidation and serves as the execution interface.

#### Main Panel — Positions Near Liquidation

Two tables, side by side or stacked:
- **Longs Near Liquidation** — top 50 longs sorted by distance from liq price (ascending)
- **Shorts Near Liquidation** — top 50 shorts sorted the same way

**Each row contains:**
```
Address (truncated)  |  Symbol  |  Side  |  Size (USD)  |  Liq Distance %  |  Liq Price  |  Entry Price  |  Label  |  Actions
```

**Row interactions:**
- **Click on address/label** → opens rename dialog → set a custom label (e.g. "hype_bull")
- **Click on middle section** → opens deeper position info: fills, P&L, full trade history
- **Click on liquidation price** → pre-fills trade entry form to schedule open/close at that liq level
- **Alert button (per row)** → toggle to receive notification every time that wallet places a trade

**Filter bar (above table):**
- Symbol filter dropdown — e.g. filter to BTC only, or HYPE only
- Default shows all symbols (all top 50 longs + all top 50 shorts)
- Symbol-specific filtering helps when bot only supports one market

**Saved Traders sidebar / section:**
- Addresses you've named appear in a "Saved Traders" list
- Add notes per address (e.g. "this guy always longs BTC before big moves")
- Click into any saved trader for full fill history from the API

**Liquidation Stats Bar (right side panel):**
```
Liq summary — rolling windows
• 10 min:  $X longs / $Y shorts
• 1 hour:  $X longs / $Y shorts  
• 4 hours: $X longs / $Y shorts
• 12 hours: $X longs / $Y shorts
• 24 hours: $X longs / $Y shorts
```
Updated on each polling cycle. Used to eyeball cascade momentum without running a strategy.

**Trade Execution (bottom or side panel):**
- Triggered from clicking a liquidation price row
- Fields: symbol, side (long/short), size, price (pre-filled from click), TP, SL
- Order type: limit only
- Submit button: places order on Hyperliquid via API

**Live Liquidation Feed (optional overlay/section):**
- Audio ding on HYPE liquidations (confirmed feature — "dings mean hype liquidation")
- Color-coded: green = short liq (bullish), red = long liq (bearish)

---

### Tab 2 — Code

Multi-terminal workspace. The core coding environment for running bots, scripts, and Claude Code sessions.

**Layout: 4×4 xterm.js grid**
- Up to 16 independent terminal panes on screen simultaneously
- Each pane runs a fully independent shell session
- Used to run: bots (live/incubating), Claude Code instances, API scripts, RBI agents, dashboards

**Controls:**
- Add new pane / close pane buttons
- Resize panes by dragging dividers
- Each pane has a label (editable) so you know what's running where
- Optional: color-coded borders per pane for quick identification

**Purpose:** Eliminate context switching between 4–6 terminal windows. Every agent, bot, and script visible simultaneously.

**Design notes:**
- Dark background (terminal standard)
- Monospaced font (JetBrains Mono or similar)
- No fancy chrome — this is a workspace, not a dashboard
- The context-switching problem is the reason this tab exists; the design must solve it

---

### Tab 3 — Data

Live data explorer. Surfaces the API endpoints in readable form without needing to write code.

**Sections:**

**OHLCV Panel:**
- Symbol selector dropdown
- Interval selector: 5m / 15m / 1h / 4h / 6h / 1d
- "Get Data" button → fetches and displays as a table or mini chart
- Shows timestamp, open, high, low, close, volume per row

**Live Liquidations Feed:**
- Exchange filter: Hyperliquid / Binance / OKX / All
- Symbol filter
- Real-time stream of liquidation events
- Columns: time, symbol, side, size_usd, exchange
- Color coded: long liq = red, short liq = green

**HIP3 Markets Panel:**
- Toggle: show HIP3 only
- Lists active HIP3 symbols: CL (oil), XAU (gold), XAG (silver), NVDA (Nvidia), XYZ100 (S&P)
- Position and liquidation data same as main market

**CVD (Cumulative Volume Delta):**
- Tick-based; shows buy vs sell pressure over rolling window
- Per-symbol toggle

**Data notes:**
- Everything fed live from the MoonDev API (requires API key entered in settings)
- No external calls — all data routed through the node

---

### Tab 4 — Backtest

Integrated backtesting runner. Run strategies without leaving the app.

**Sections:**

**Data Source:**
- Pre-loaded datasets from `moondev.com/data` download
- Symbol selector + interval selector (same as Data tab)
- Option to load custom CSV

**Strategy Selector:**
- Dropdown or file browser to pick a Python strategy file
- Shows docstring preview (stats from previous runs stored in the file header)

**Run Controls:**
- Starting cash: default $1,000,000
- Date range: start / end
- Run button → executes `backtesting.py` and streams results

**Results Panel:**
- Return %, Max Drawdown %, Sharpe, Sortino, SQN, Profit Factor, EV, # Trades, Win Rate
- P&L equity curve chart
- Trade log table: entry time, exit time, side, size, P&L per trade

**CSV Export:**
- One-click export of stats + trade log
- Feeds the master CSV tracker

---

### Tab 5 — HLP

Dedicated HLP (Hyperliquid Market Maker) tracking tab.

**What it shows:**
- All current HLP wallet positions (the 4–5 component strategy wallets)
- HLP trade feed — every fill in real time
- HLP P&L over time (the vault has grown from $1K initial to $118M+)

**HLP Wallet breakdown:**
- HLP Strategy A (wallet ending in 303)
- HLP Strategy B (wallet ending in 74B)
- HLP Liquidator (wallet ending in D14)
- HLP Liquidator 4 (wallet ending in 308)
- Shows each wallet's current positions, entry price, size, P&L

**Why this tab exists:**
HLP is the exchange's own market maker. They run the largest positions on the platform. Watching their moves provides the closest thing to institutional flow data available. When HLP starts building a position, it's worth knowing before it shows up in the price.

---

## Global UI Elements

### Header / Nav Bar
- Tab navigation (Trade / Code / Data / Backtest / HLP)
- API key status indicator — green = connected, red = expired/invalid
- Bot status row — shows each running bot name + P&L since start (green/red)
- Clock (UTC)

### API Key Entry
- Prompt on first launch if no key stored
- Settings area to update key
- Visual confirmation when key validates against the node

### Settings Panel
- API key input
- Default symbol preference
- Alert sound toggle (on/off for liquidation dings)
- Terminal font size
- Theme: dark only (no light mode — this is a trading terminal)

### Notifications / Alerts
- In-app alert banner when a saved trader places a trade
- Sound alert on configurable liquidation threshold
- Bot kill switch alert if drawdown threshold hit (25% per bot / 20% portfolio weekly)

---

## Design Principles

**1. Data density over decoration.** Every pixel should show something actionable. No hero sections, no marketing copy inside the app. It's a cockpit.

**2. No purple.** The specific instruction from sessions: "Don't hold back on design or fun" but "no purple." Color palette should lean dark with accent colors for signal: green (bullish/profit), red (bearish/loss), amber (warning), white/gray text.

**3. Terminal-forward.** The Code tab is the heart of the app. The xterm.js grid should feel native — not like a novelty. Users should forget they're in an Electron app, not a terminal.

**4. Fun is not optional.** Dashboards should be visually engaging: flashing updates, color-coded rows, live counters. "Make it feel fun like a slot machine but no goofy shit — just fire updates." Think Pump.fun aesthetic applied to institutional data.

**5. Frictionless execution.** Clicking a liquidation price should auto-fill the trade form. One click from seeing a setup to submitting the order.

**6. No ads, no breaks, no upsell.** The app is the product. It's powered by the API key. Nothing behind a paywall once you're inside.

---

## Component Inventory

| Component | Tab | Notes |
|---|---|---|
| Positions table (longs) | Trade | Sortable, filterable, click-to-trade |
| Positions table (shorts) | Trade | Same |
| Liq stats bar (10m/1h/4h/12h/24h) | Trade | Right panel |
| Saved traders list | Trade | Named addresses + notes |
| Trade entry form | Trade | Limit orders only |
| Live liq event feed | Trade / Data | Color-coded stream |
| xterm.js 4×4 grid | Code | Full independent shells |
| OHLCV table/chart | Data | Per-symbol, per-interval |
| HIP3 market panel | Data | 136+ symbols |
| CVD chart | Data | Tick-based |
| Strategy file selector | Backtest | Reads .py files |
| Results stats panel | Backtest | All 7 core stats |
| Equity curve chart | Backtest | P&L over time |
| Trade log table | Backtest | Per-trade breakdown |
| HLP wallet cards | HLP | 4–5 component wallets |
| HLP trade feed | HLP | Real-time fills |
| API key input | Settings | With validation status |
| Bot status row | Header | All bots, live P&L |
| Alert system | Global | Saved trader triggers |

---

*Last updated: 2026-05-31*  
*Source: Moondev transcripts — `Inspo/Transcripts/Moondev/`*  
*Primary sessions: "In this video I want to help you understand the longs near lick", "Good morning — Quaba is here for trading", "All right, so Claude Code built me six strategies", "data layer that I built here"*