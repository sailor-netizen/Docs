# HIP3 Strategies and Oil Drift

## What is HIP3

**HIP3 (Hyperliquid Improvement Proposal 3)** enables Hyperliquid to list and trade stocks, ETFs, futures, and commodities — not just crypto perpetuals. Stocks and traditional assets trade on HIP3 through dedicated decentralized exchanges (dexes) integrated into Hyperliquid.

- As of the research session: **136–137 HIP3 symbols** tracked across **7 dexes**
- Symbols are discovered automatically by a daemon polling every 5 minutes
- HIP3 symbols include: oil (CL / WTI crude, Brent, US oil, USDH), gold, silver, XYZ (S&P 500 proxy), HIMS, Costco, Lily, NVDA, and others
- Trading is **24/7/365** — HIP3 does not follow traditional market hours despite tracking underlying equities/futures

---

## Data Retention on Hyperliquid API (HIP3)

| Timeframe | Max Lookback (Hyperliquid Public API) |
|---|---|
| 1-minute | 3.6 days (hard cap at API level) |
| 5-minute | 17.4–17.5 days |
| 15-minute | ~52 days |
| 1-hour | Full listed history |

- Hyperliquid returns max **5,000 candles per API call**
- The Mundav API collects tick data continuously from listing date, giving **unlimited historical depth** starting April 26, 2026 — this is a structural data edge over any party relying solely on the public Hyperliquid API
- 1-minute retention verified identical across multiple symbols: 3.6 / 3.6 / 3.5 days (three symbols tested)

---

## Basket Strategy — 5.37 Sharpe (After-Hours)

- Discovered using 6 parallel Claude Code agents during a private Zoom session
- Strategy type: **mean reversion during closed-market (after-hours) window**
- Best result: **Sharpe 5.37**, achieved by **narrowing the trading window to the UTC dead zone** (overnight, when US equity markets are closed)
- The "closed hours" window was identified by gating standard indicators to after-hours only — the same indicators in RTH (regular trading hours) showed substantially lower Sharpe
- Two top uncorrelated strategies identified:
  - **RSI extreme fade** — fires on RSI oversold/overbought extremes (24 trades over sample period)
  - **Bollinger Band touch entry** — fires on BB rejection (13 trades over sample period)
  - These two strategies share no co-firing bars — they are independently triggered, making them stackable with ~50/50 capital allocation; combined expected Sharpe > 5

---

## Oil Drift Thesis

### Weekend Drift (CL / WTI 1-hour data)

**Thesis:** The oil price during market closure (Friday close → Sunday CME open) is noise that reverts back to Friday's closing price by Monday.

- **Tested on:** CL (WTI crude oil), using hourly data (~110 days / 15–16 weeks of data)
- **Finding:** Large positive weekend drifts reverted by **+4.47% on average** by Monday open
- **Agent A (winner):** One trade per weekend, entry at **Sunday 21:00 UTC** (1 hour before CME reopens at 6pm Eastern / 23:00 UTC), fade whatever drift accumulated over 48 hours
  - Win rate: high; draw-down: minimal
  - Avoids Agent B's failure mode: midweek news bleed-through and real catalyst moves on Saturday/Sunday
- **CME oil hours:** Opens 6:00 PM Eastern Sunday, closes 5:00 PM Eastern Friday
- **Agent entry timing:** UTC 21:00 Sunday = 5:00 PM Eastern = 1 hour *before* CME reopens; drift has had full 48 hours to accumulate

### Daily Gap Drift (CL intraday)

- CL futures close for approximately 1 hour each day
- Three agents tested the daily gap fade
- **Two of three agents found real edge**, disagreeing only on magnitude
- Best result: Agent D — Sharpe 6.25, 62 trades, **100% win rate Mon/Wed/Thu** (Tuesday loses)
- Agent E (more conservative): confirms Thursday mean reversion is real; Monday/Wednesday wins attributed to mechanical setup artifacts
- Honest estimated Sharpe for the daily gap strategy: **3–4.6** after accounting for overcount
- **Caveat:** Only ~16 events per day-of-week — not statistically significant at this stage; more data needed as HIP3 history accumulates

---

## Oil Instrument Breakdown (Hyperliquid HIP3)

| Symbol | Description | Notes |
|---|---|---|
| CL | WTI Crude (futures code) | Most data, most liquidity — primary instrument |
| WTI | West Texas Intermediate (alternate listing) | High correlation with CL |
| US Oil | WTI under consumer-friendly name (Kinetic DEX) | Same underlying |
| Brent Oil | Brent crude (ICE benchmark) | ~88% correlation with CL; lower volume |
| USDH | Oil/USD pair | Minimal volume — not tradable |

- All five oil instruments: **>88% correlation** on 1-hour returns
- Brent is the lone outlier with slightly different behavior (~70% correlation with US Oil in some windows)
- Practical conclusion: only CL and WTI have sufficient volume for bot trading on weekends

---

## Gold and Silver (HIP3)

- Tested the same closed-hours mean reversion thesis on gold and silver
- **Finding:** "Fade is brutal on metals" — the thesis does not transfer cleanly
- Small sample size (tight number of weekends tested)
- Gold and silver require a different strategic approach than oil

---

## HIMS Case Study

- HIMS (HIM pharmaceuticals) listed 32 days before the research session
- 1-hour data: full listed history available
- Back-tested: ORB (Opening Range Breakout) 4-hour → 45% return (buy-and-hold benchmark)
- Closed-hours strategy on HIMS: Sharpe 5.37 achieved but **symbol-specific** — basket of other HIP3 symbols showed negative Sharpe, indicating HIMS result was **overfit to 30-day window**, not a general HIP3 edge
- Lesson: closed-hours thesis works for specific symbols and windows; cross-symbol robustness testing is essential

---

## Infrastructure Notes

- **Server capacity:** 24 cores, 375 GB free disk at time of session
- **Data export pipeline:** WebSocket → in-memory buffer → SQLite → JSON file → API serves static file
- Export interval: originally 10 seconds, moved to 30 seconds (acceptable for 1-minute candles; 30-second lag is invisible to strategy)
- **On-demand refactor:** switched from pre-generating JSON for every symbol every N seconds to SQL-on-demand (query SQLite when client hits endpoint) — eliminates write load, scales to 2,000–5,000 symbols
- **Rate limit:** Hyperliquid WebSocket — 1,000 subscribe messages on connect; batch/throttle required at scale
- **API endpoint tracking:** admin-only endpoint added to monitor per-endpoint hit frequency

---

## Source Transcripts

- `I had six clawed codes here as you.txt` (primary — basket strategy, infrastructure, oil thesis, data retention)
- `and oil. So, this is what's neat in.txt` (HIP3 oil volume, funding rate signal)
- `I'm really excited for Hyperlid now.txt` (HIP3 as diversification from crypto bear market)
- `Wall Street doesn't have this and I.txt` (funding rate scanner, HIP3 sentiment)
