# Polymarket Bot Patterns & Strategies
> Project Quant — Source of Truth  
> Market types, confirmed bot patterns, ML experiments, API notes  
> Derived from `Inspo/Transcripts/Moondev/` — 7 dedicated Polymarket sessions

---

## Overview

Polymarket is the highest-priority non-Hyperliquid trading venue in this project. The core thesis: Polymarket is early, retail is still hand-trading it, and Wall Street isn't there yet — so the edge is large right now. This window closes as the platform matures.

**Key paradigm shift:** Unlike a casino sportsbook where you take whatever price they offer, Polymarket is peer-to-peer. You can place limit orders at any price you want — including a 30% discount to current market. Nobody else is doing this systematically. That asymmetry is the edge.

---

## Market Types on Polymarket

### 1. 5-Minute BTC Up/Down Markets
Every 5 minutes, a new market opens asking: will BTC be up or down vs the last price?

**Characteristics:**
- Resolves in exactly 5 minutes — forced exit, removes one of the hardest human decisions (when to close)
- Highly volatile, large swings possible (233%, 150%, 113% seen in sessions)
- 288 markets per day
- Volume per 5-minute market: ~$150K average; $53K off-hours, up to $216K during US trading hours
- More liquid during US hours; off-hours is where edge may concentrate (lower competition)
- These are the most actively developed bot targets in the transcripts

**Bot ideas tested:**
- Liquidation momentum (S → go up, L → go down)
- Liquidation cascade reversal (capitulation → mean reversion)
- Off-hours vs US-hours timing
- AI predictor (Claude Opus 4.6 given 30 min of 5-second tick data, asked to predict direction)
- ML models (XGBoost, LightGBM, TCN, TabNet) on raw candles

**Critical operating rule:** Cancel all open orders before placing new entry. Failure to do this caused runaway order loops draining accounts in live sessions.

### 2. Sports Markets
One-day-to-one-week markets on individual game outcomes.

**Markets built:**
- **WTA Tennis (women's):** Best-of-3 format = more volatility than ATP best-of-5. More upsets = more panic selling = more stink bid fills.
- **NBA:** First half only (Q1+Q2). Cancel all at halftime. Favorite goes down early → panic dump → buy cheap.
- **Cricket (IPL + PSL):** First innings only. Cancel all at innings boundary. Similar logic.
- **NHL scanner, Soccer scanner, UFC scanner:** Built for discovery, not yet running full bots.
- **MLB:** Mentioned but not detailed.
- **E-sports:** Dota 2 discovered via scanner, e-sports bot started.

**Core sports strategy — Stink Bids:**
```python
BET_SIZE = 5        # $5 incubation size
DISCOUNT = 0.30     # Place bid 30% below current market price
# If filled: hold to expiration — favorite usually wins → collect $1/share
# Only bet during first half / first innings
# Cancel all orders at halftime / innings boundary
# Refresh every 15 minutes
# Source matchups from ESPN API (free)
```

**Why this works:** On Polymarket, if a whale panic-sells during a game and dumps the price, the order book has to fill from wherever it can find buyers. Stink bids sitting 30% below market get swept up at that discounted price. Hold to expiration — the favorite still usually wins.

**Why it doesn't always work:** Low liquidity on smaller markets means some stink bids never fill. Upsets still happen. Not backtested — pure incubation.

### 3. Weather Markets
Daily markets on high temperature in major cities worldwide (Wellington, Shenzhen, Atlanta, etc.).

**Characteristics:**
- Resets daily — one market per city per day
- Temperature "buckets" (ranges of 1-2°C)
- Volume: ~$98K per market observed
- Less competition than 5-minute markets
- Edge: forecast data + scanner to see where market price deviates from actual weather forecast

**Bot built:** Scanner showing all city markets, forecast vs market price, gap analysis. Stink bid on the bucket the forecast favors when market is mispriced vs the forecast.

**Results seen:** 52-82% returns on individual fills (smaller than 5-minute markets but more consistent).

### 4. News / Politics / Events Markets
Longer-horizon markets (days to months) on outcomes like elections, regulatory decisions, geopolitical events.

**AI Agent approach:** Multi-model consensus agent that:
1. Scans all Polymarket markets for large trades (whales trading over $500)
2. Presents the market question to 6 AI models in parallel (Claude, GPT-5-mini, Qwen, XAI, DeepSeek, GLM)
3. Each model votes Yes / No / No Trade with reasoning
4. Consensus model aggregates votes → only trade when clear majority agree
5. Ignores crypto markets (too emotional) and sports markets (for this agent)

```python
# Multi-model swarm agent config (confirmed working)
MIN_TRADE_SIZE = 500   # Only watch whales above $500
MODELS_CALLED = [      # Via Open Router
    "claude-opus-4",
    "gpt-5-mini",
    "qwen",
    "x-ai",
    "deepseek-chat",
    "glm-4-7"
]
# Waits for 25 new trades before re-running consensus
# Ignores prices near $0.02 or $0.98 (near-certain outcomes)
```

### 5. Kalshi Markets
Not Polymarket but same approach — Kalshi is a regulated US prediction market.

**Kalshi whale scanner built:**
- Scans all Kalshi trades over $1,000
- Saves to CSV
- Clickable links to each market
- Same use case: discover liquid markets, then build stink bid bots

---

## ML Experiments on 5-Minute Markets

**Dataset:** 100 weeks of 1-minute BTC data (from MoonDev API), resampled to 5-minute windows for market simulation.

**Models tested and results:**

| Model | Test Accuracy | Notes |
|---|---|---|
| TabNet | 50.86% | Barely above random. Top feature: price position at 42% importance. Found 11-trade edge cluster at 65% WR → useless sample size. |
| TCN (Temporal CNN) | 49.91% | **Worse than random.** Loss never moved from 0.6932. Predicted "up" on every single trade. No signal learned. |
| XGBoost (raw candles) | 50.73% | Best overall. 57.34% WR on 143 high-confidence trades. Top features: green candle count over 5 and 15 bars (momentum counting). |
| LightGBM (micro-structure) | 50.91% | Best aggregate accuracy but still near-random overall. Top features: up-window counts, green ratio skew. 56% WR on high-confidence subset. |

**Key finding:** No model has edge trading every 5-minute window. All sit at ~51% when forced to trade every market. But every model shows meaningful edge at **high-confidence thresholds** — i.e., when the model is very confident, it's right more often.

**The real alpha from ML:** Confidence filtering. XGBoost at high confidence = 57% WR over 143 trades (reasonable sample). If multiple models agree with high confidence simultaneously → that's where to trade.

**Andrew Ng principle applied:** Everything in ML comes down to feature engineering. The raw candle features were insufficient. Better features to explore:
- Liquidation counts per bar (from MoonDev API — nobody else has this)
- HLP position changes in the preceding 5 minutes
- Funding rate delta
- CVD (Cumulative Volume Delta)
- Open interest change

---

## API & Technical Setup

### Polymarket API
- Public WebSocket available for live trade data (free)
- REST endpoints for market data, order placement
- Requires credential validation on connect
- Traded via private key (not exchange API key)
- Multi-account system required (no sub-accounts on Polymarket yet): separate private keys in `.env` file

```python
# Multi-account setup (confirmed pattern)
ACCOUNT_OG   = os.getenv("POLY_PRIVATE_KEY_OG")
ACCOUNT_2    = os.getenv("POLY_PRIVATE_KEY_2")
ACCOUNT_3    = os.getenv("POLY_PRIVATE_KEY_3")

# 5-minute market URL construction
# Format: polymarket.com/event/{series_lower}/{title_slug}/{event_lower}
# Market ID is a Unix timestamp: e.g., "17732358800"
```

### ESPN API
- Free, no auth required
- Used for: WTA match status (which inning / half), NBA quarter status, cricket innings boundaries
- Called every 15 minutes to update game state → triggers cancel of open orders when game phase changes

### MoonDev API (tick data advantage)
- Hyperliquid 1-minute data: max 3.6 days retention from public API; MoonDev API has months
- Hyperliquid 5-minute data: max 17.5 days from public API; MoonDev API has unlimited (started collecting April 26, 2026)
- This is the data edge for ML training — nobody else has Hyperliquid micro-structure data at this depth

---

## Bot Template (5-Minute Markets)

The base bot that all variants derive from:

```python
"""
POLY_HYPER_OG — Base liquidation stink bot for Poly 5-min markets

Config:
- Liquidation threshold: $2,000 short liq → take Up position
- Liquidation threshold: $2,000 long liq → take Down position  
- No entry in last 2 minutes of 5-minute window
- Cancel all orders before placing new
- Bet size: $5 (incubation)

Variants:
- V2 (no_last_two_minutes): Bumped no-entry zone from 60s to 120s
- V3 (25k_threshold): Raised liq threshold to $25K — only trade on big liquidations
- V4 (pullback): Wait for 3-minute pullback after liq signal before entering
- V5 (reversal/capitulation): Opposite of momentum — 500K+ liq → mean reversion play
"""
```

**Key bugs encountered in live sessions:**
1. Arrow keys in terminal sent escape sequences that triggered buys/sells — fix: only execute on explicit `B` or `S` keypress
2. Missing order duplicate check → bot placed multiple orders, drained account → fix: always query open orders before placing, cancel all first
3. Stale market detection → bot tried to place orders after market closed → fix: validate market is still active before order

---

## Polymarket vs Hyperliquid Risk Profile

| | Polymarket | Hyperliquid |
|---|---|---|
| Max loss | Fixed — what you bet ($5, $10) | Unlimited — leverage magnifies both ways |
| Leverage | None — peer to peer | Up to 40x+ |
| Fees | Per trade, spread-based | Funding rates + trade fees + liquidation |
| Forced exit | Yes — every 5 minutes on 5-min markets | No — can hold forever (usually bad) |
| Emotional risk | Lower — loss is bounded | High — open P&L visible, hard to close |
| Market hours | 24/7 | 24/7 |
| Edge source | Stink bids, whale following, ML consensus | Liquidation momentum/reversion, HLP mirroring |

**The argument for Polymarket over hand-trading Hyperliquid:** Your max loss is $5. You can't leverage yourself into oblivion. The 5-minute forced exit removes the "I can't close the trade" human failure mode. Fees are visible and bounded.

---

## Discovery Workflow (Finding New Markets)

1. Run **whale scanner** (Polymarket) — filter trades > $500 USD, collect CSV
2. Run **Kalshi scanner** — filter trades > $1,000 USD
3. Let scanner run 2-3 days → review CSV → look for markets with consistent volume
4. Find markets where: (a) multiple whales trade, (b) volume is $50K+ per market, (c) market resolves daily or sub-daily
5. Build stink bid bot for that market category
6. Incubate at $5 size for 2 weeks
7. If profitable: scale on server, run 24/7

**The "No" bot / no-markets:** Among the best performing bots at time of transcripts. "No markets" = prediction markets where the current price is heavily skewed toward "Yes" but the actual outcome is uncertain. Details withheld from public — in the vault.

---

## Confirmed Results (from Live Sessions)

These are returns observed on individual trades / sessions — not annualized backtests:

| Market Type | Returns Seen |
|---|---|
| 5-min BTC (momentum bots) | 72%, 100%, 104%, 113%, 150%, 203%, 233%, 245%, 317% |
| 5-min BTC (Opus 4.6 predictor) | Mixed — "did alright" |
| Weather markets | 52%, 57%, 80%, 82% |
| Tennis stink bids | Profitable over first 24 hours post-launch |
| Cricket stink bids | Running, early results positive |
| NBA stink bids | Launched, early stage |

**Caveat:** These are individual trade returns, not compounded. Win rate on 5-minute markets is ~50% when trading every window. Profit comes from: (a) getting good fills on stink bids, (b) only trading high-confidence ML signals, (c) not overtrading.

---

## Alpha Decay Warning

If you share your exact Polymarket strategies with large audiences, the edge decays. This is why:
- If everyone places stink bids at 30% below market, the market fills at a higher price (no more discount)
- If everyone runs the same liquidation momentum bot on 5-minute markets, the edge arbitrages away

The correct approach: use these docs as a framework and build your own variant. Different thresholds, different timing rules, different market categories = different alpha.

---

*Last updated: 2026-05-31*  
*Source: Moondev transcripts — `Inspo/Transcripts/Moondev/`*  
*Primary sessions: "If you want to build bots on Polyma", "Man, these poly market five minute", "These poly market five minute BTC u", "The Poly Market bot that I built he", "All right, Cloud Code plus Polymark", "These Cloud Code Poly Market bots f", "You haven't heard of Poly Market ye"*