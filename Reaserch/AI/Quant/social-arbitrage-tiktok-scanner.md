# Social Arbitrage — TikTok Scanner

## Concept Origin

**Social arbitrage** is the practice of identifying consumer trends on social media before they become visible in price action, then trading the underlying public equities.

- Originator: **Chris Camillo**, documented in *Market Wizards* (Jack Schwager, independent P&L audit)
- Camillo's documented net worth run-up: $20,000 → $80 million (originally reported as $40M, corrected to $80M from his Twitter)
- Also documented on the **Dumb Money** YouTube channel
- Thesis: Wall Street analysts (typically older, out of touch) systematically miss trends driven by younger demographics spending on TikTok, which later move public equities

**Classic example cited:** Abercrombie & Fitch comeback — no Wall Street analyst saw it coming, but it was visible in social media demand before the chart moved.

---

## Implementation — OpenClaw TikTok Scanner

The scanner is built as an **OpenClaw** (computer-use AI) autonomous agent running cron jobs.

### Architecture

```
Cron job (every 5 min) → OpenClaw opens TikTok browser session
                       → Scrolls For You Page for rotating keywords
                       → Extracts brand/company/ticker mentions
                       → Scores confidence (high/medium/low)
                       → Writes findings to JSON + CSV
                       → Dashboard served on port 3456
```

**Stack:**
- OpenClaw (computer-use agent) controlling headless browser
- Node.js (`server.js`) serving dashboard API
- Dashboard: `dashboard/index.html` — vanilla JavaScript, no frameworks
- Data stored in local JSON files and CSV
- Cron job configuration: `cron-config.md`, `agents.md`

**GitHub repo:** `mundave/social-arb` (not open source — code shown on stream but not published as public repo)

---

## Keyword Rotation (Alpha Signal)

The scanner rotates through a curated keyword list each cycle. Keywords are designed to surface **demand signals** before they hit mainstream awareness:

- Restock alert
- Sold out
- DTC brands
- Retail demand
- Hive (and similar trending consumer terms)
- Brand-specific searches

**Why rotation matters:** Searching the same keyword repeatedly trains the TikTok For You Page algorithm to show that content — rotating keywords gives a broader sample of trending content.

**"This week" filter:** Applied to avoid surfacing old viral content; scanner targets new velocity, not evergreen content.

**Scan depth:** Target 200 videos per scan session.

---

## Confidence Scoring and Ticker Mapping

For each brand mention, the scanner applies:

| Category | Action |
|---|---|
| Publicly traded company | Note ticker at high confidence |
| Private company, public parent | Note parent ticker at medium confidence |
| Pre-IPO company | Tag as pre-IPO opportunity |
| No investment path | Skip (configurable) |

**Example mappings:**
- Adidas / Nike / Jordan mention → NKE
- Starbucks mention → SBUX

Confidence scoring populates the dashboard and powers CSV tracking.

---

## Twitter Alpha Scanner (Bundled)

The same repo includes a **Twitter/X alpha scanner** running in parallel:

- Monitors the researcher's **following timeline** (not firehose)
- Filters for posts with **100+ likes** as signal threshold
- Extracts tickers, companies, and engagement metrics
- Stores in JSON/CSV alongside TikTok findings
- JavaScript file: `twitter-alpha.js`

---

## Dashboard

- Served at `localhost:3456` (configurable port)
- Files: `server.js` + `dashboard/index.html`
- Displays: top signals found, brand mentions, ticker guesses, confidence scores, timestamp of last scan
- Built by OpenClaw during session; no external frameworks

---

## Model Used

- Primary orchestrator: **Claude Opus 4.5** (Haiku 4.5 attempted for cost reduction — weekly rate limits hit by Monday using Opus)
- Researcher's thesis: Opus must be the orchestrator; Haiku acceptable for sub-tasks but not for main agent decision-making
- Haiku labeled as "fine for a lot of this stuff" in cost-sensitive configurations

**Cost reduction experiment:** MiniMax 2.5 API tested as a 98% cheaper alternative to Opus ($0.30/million tokens vs. $15/million for Opus input). Integration with OpenClaw was partially achieved (M2.5 confirmed responding in logs) but unstable in the chat interface at time of testing.

---

## Use Case Positioning

This is identified as one of the **top two OpenClaw use cases** (alongside SSH server self-healing automation). Most OpenClaw use cases at the time were described as "content slop" (Reddit digests, YouTube summaries). Social arbitrage is differentiated because:

1. Output is **actionable investment signal**, not content
2. The underlying strategy is **independently verified** (Market Wizards audit of Chris Camillo)
3. OpenClaw's computer-use capability is genuinely necessary — this is not achievable in a terminal script because it requires a logged-in browser session on TikTok

---

## Limitations and Honest Assessment

- Not plug-and-play — "one out of a hundred signals is actually a good idea"
- Still requires human judgment to filter signals and make investment decisions
- Twitter integration had bugs during recorded session (TikTok working; Twitter alpha scanner had display errors)
- Backtesting this strategy is difficult — researcher's approach: observed Chris Camillo execute it for 15+ years before building the AI version
- Researcher does not open-source the full repo: "This is war. If you can't keep up, you can't keep up."

---

## E-commerce Application

Multiple e-commerce operators noted the scanner is useful for **product trend discovery** (finding what's selling out before it saturates), not just equity trading. The same brand-detection output that maps to a ticker also surfaces trending SKUs.

---

## Source Transcripts

- `People have asked me about this bil.txt` (primary — full tutorial, code walkthrough, Chris Camillo background, cron job setup, keyword list, confidence scoring)
- `So, my buddy Art just dropped this.txt` (OpenClaw use case positioning, Twitter alpha scanner, social arb vs. terminal alternatives)
- `So, I have now verified that this d.txt` (MiniMax 2.5 cost reduction experiment, OpenClaw model comparison)
