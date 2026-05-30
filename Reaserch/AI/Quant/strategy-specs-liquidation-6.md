# Strategy Specs — 6 Liquidation Strategies
> Project Quant — Source of Truth  
> Derived from `Inspo/Transcripts/Moondev/` — live sessions, back test reviews, and strategy walkthroughs

---

## Overview

All six strategies share the same data foundation and execution skeleton. They differ only in their trigger logic. Each was back tested against real Hyperliquid liquidation data (18–20 months, 28M+ events, $60B in liquidations) using `backtesting.py` on 5-minute OHLCV bars.

**Core insight from the data:**
- Momentum (trade *with* the cascade) dominates on the **1-hour** time frame
- Reversal (fade the cascade) dominates on the **6-hour** time frame
- The 5-minute time frame is the best window for intraday cascade detection
- Contrarian strategies (fade the lick) have only a 3% success rate overall — the exception being BTC 6H specifically
- High-threshold events (top 1% of liquidation magnitude) outperform low-threshold events every time

---

## Common Infrastructure (All 6 Strategies)

Every strategy calls the same data endpoints and uses the same bot template. Nothing gets built without this plumbing confirmed working first.

### Data Required (per bar)
```
recent_liquidations     # every 30–60 seconds; shape below
recent_ohlcv            # 5-minute bars, limit 300
current_price           # live tick
```

### Liquidation Row Shape
```python
{
  "symbol":    str,      # e.g. "BTC"
  "side":      str,      # "long" or "short"
  "size_usd":  float,    # USD value of the liquidation
  "timestamp": int,      # unix ms
  "exchange":  str       # "hyperliquid", "binance", "okx", etc.
}
```

### Data Sources
- **MoonDev API** — `moondev.com/docs` — primary data layer (Hyperliquid node, direct feed)
- **Hyperliquid liquidations** — default; can layer in Binance + OKX for multi-exchange confirmation
- **HLP (Hyperliquid Market Maker)** — optional overlay for smart money context

### Position Sizing
- **95% of account maximum** — never 100%
- **$10 incubation size** — all bots start here

### Execution Template (all strategies)
```
- Limit orders only — no market orders
- Cancel all open orders before new entry
- Dust adjustment applied
- while True loop — reset on exception, never die silently
- Stop-loss: required, hardcoded
- Take-profit: required, hardcoded
```

---

## Strategy 1 — Cascade Filter

**Plain English:** Wait until you see three consecutive 5-minute bars where the *same side* is getting liquidated above threshold. That's a real cascade, not noise. Trade with it.

### Trigger Logic
```python
# State: rolling buffer of last 3 bars, per side
long_liq_buffer  = deque(maxlen=3)   # long liquidation USD per bar
short_liq_buffer = deque(maxlen=3)   # short liquidation USD per bar

# Entry condition (short cascade → go long)
if all(bar > 50_000 for bar in short_liq_buffer) and sum(short_liq_buffer) > 500_000:
    enter_long()

# Entry condition (long cascade → go short)
if all(bar > 50_000 for bar in long_liq_buffer) and sum(long_liq_buffer) > 500_000:
    enter_short()
```

### Parameters (configurable)
| Variable | Default | Notes |
|---|---|---|
| `min_per_bar` | $50,000 | Minimum liq per bar to count |
| `cascade_sum_threshold` | $500,000 | Total across 3 bars to confirm cascade |
| `bars_required` | 3 | Consecutive bars same side |
| `time_frame` | 5m | Back tested on 5-minute bars |

### Back Test Results (confirmed passing)
| Stat | Value |
|---|---|
| Return | 98% |
| vs. Buy & Hold | 33% |
| Max Drawdown | -25% |
| Sharpe | 1.64 |
| Sortino | 5.33 |
| Win Rate | ~50% |

### Trade Direction
- Short liquidation cascade → **long entry**
- Long liquidation cascade → **short entry**

### Notes
- This is the simplest and most readable of the six
- Low number of trades — exposure time ~28–38%
- Time-scaled variant tested (scale in over 1–4 hours after trigger) — mixed results; unoptimised max drawdown hit 71%

---

## Strategy 2 — Z-Score Momentum

**Plain English:** Watch the imbalance between short liquidations and long liquidations on every bar. When that imbalance spikes to 4+ standard deviations above the 24-hour rolling average, you've got a clean, statistically significant cascade. Trade with it.

### Trigger Logic
```python
# State: rolling 288-bar window (24 hours on 5m bars)
net_liq = short_liq_usd - long_liq_usd   # positive = short cascade (bullish signal)

rolling_mean = mean(net_liq[-288:])
rolling_std  = std(net_liq[-288:])

z_score = (net_liq[-1] - rolling_mean) / rolling_std

if z_score > 4.0:
    enter_long()    # shorts getting crushed → price going up

if z_score < -4.0:
    enter_short()   # longs getting crushed → price going down
```

### Parameters (configurable)
| Variable | Default | Notes |
|---|---|---|
| `lookback_bars` | 288 | 24 hours at 5m resolution |
| `z_threshold` | 4.0 | Standard deviations to trigger |
| `time_frame` | 5m | |

### Back Test Results
| Stat | Value |
|---|---|
| Return | 114% |
| vs. Buy & Hold | 33% |
| Max Drawdown | -27% |
| SQN | 3.96–4.6 |
| Profit Factor | high |

### Notes
- Statistically cleaner entry than Strategy 1 — fewer false triggers
- Expectancy reported as strong across multiple runs
- Works best when liq imbalance is extreme and not just elevated

---

## Strategy 3 — Volume-Confirmed Momentum

**Plain English:** Liquidation signals alone aren't enough. This strategy also requires that the bar's total volume be in the top 10% of the last 100 bars. That confirms the whole exchange is participating — not just one whale's position getting blown out.

### Trigger Logic
```python
# Same cascade signal as Strategy 1 or 2 (configurable)
cascade_triggered = detect_cascade(...)

# Volume confirmation
volume_percentile = percentileofscore(volume_history[-100:], current_volume)
volume_confirmed  = volume_percentile >= 90.0   # top 10%

if cascade_triggered and volume_confirmed:
    enter_position()
```

### Parameters (configurable)
| Variable | Default | Notes |
|---|---|---|
| `volume_percentile_threshold` | 90.0 | Top 10% of last N bars |
| `volume_lookback` | 100 | Bars for percentile calculation |
| `cascade_type` | "cascade_filter" | Can use S1 or S2 as the cascade trigger |

### Back Test Results (Revisit Momentum Volume Confirmed V2)
| Stat | Value |
|---|---|
| Return | 514% |
| vs. Buy & Hold | 33% |
| Max Drawdown | -28% |
| Win Rate | 48% |
| Profit Factor | 4.246 |
| Expectancy | 5.777 |
| Trades | 47 |

**This is the highest-conviction back test result from the session.** 514% return, 28% drawdown, 4.2+ profit factor. Flagged as the best code to share.

### Entry Detail (from the code walkthrough)
1. Detect liquidation event above threshold
2. Record the low at that bar
3. Wait for price to *re-test below that low* (revisit)
4. Confirm with RSI oversold **or** MACD bullish cross
5. Enter long

### Notes
- More selective than S1/S2 — waits for a revisit, not just the initial signal
- The volume confirmation is key: removes single-whale noise
- Position sizing at 95% of account in the back test

---

## Strategy 4 — One-Sided Cascade

**Plain English:** When one liquidation side is at least 5x bigger than the other in a single bar, you've got a pure one-sided event. Trade it fast with tight stops. No history needed — just current bar data.

### Trigger Logic
```python
short_liq = sum(liq.size_usd for liq in bar_liquidations if liq.side == "short")
long_liq  = sum(liq.size_usd for liq in bar_liquidations if liq.side == "long")

ratio = short_liq / long_liq if long_liq > 0 else float('inf')

if ratio >= 5.0:
    enter_long()    # shorts 5x+ bigger → strong bullish signal

inv_ratio = long_liq / short_liq if short_liq > 0 else float('inf')

if inv_ratio >= 5.0:
    enter_short()   # longs 5x+ bigger → strong bearish signal
```

### Parameters (configurable)
| Variable | Default | Notes |
|---|---|---|
| `imbalance_ratio` | 5.0 | Min ratio of dominant to other side |
| `min_dominant_size` | $100,000 | Minimum absolute size to avoid tiny ratios |
| `stop_loss_pct` | tight | Tighter than other strategies — fast trade |
| `take_profit_pct` | tight | Fast scalp target |

### Back Test Characteristics
- Fewest state requirements — no rolling history needed
- Fastest signal — fires immediately on single-bar imbalance
- **Tighter stops mandatory** — this is a momentum scalp, not a swing trade
- Higher trade frequency than S1–S3

### Notes
- Simpler to implement than S1–S3 (no buffer management)
- Risk: can fire on a single whale without broader cascade confirmation
- Best paired with a minimum absolute size filter ($100K+) to avoid noise from small accounts

---

## Strategy 5 — Adaptive Percentile

**Plain English:** Instead of a fixed dollar threshold (e.g. $500K), the trigger adapts to recent market conditions. The threshold is the **99th percentile** of all liquidation events from the last 7 days. You only fire on the top 1% of cascade events relative to recent history — always calibrated to what "big" means right now.

### Trigger Logic
```python
# State: rolling 7-day liquidation history (per side)
lookback_events = liquidation_history_7d()

p99_short = np.percentile([e.size_usd for e in lookback_events if e.side == "short"], 99)
p99_long  = np.percentile([e.size_usd for e in lookback_events if e.side == "long"],  99)

# On each bar
short_liq_bar = sum_short_liquidations(current_bar)
long_liq_bar  = sum_long_liquidations(current_bar)

if short_liq_bar >= p99_short:
    enter_long()

if long_liq_bar >= p99_long:
    enter_short()
```

### Parameters (configurable)
| Variable | Default | Notes |
|---|---|---|
| `percentile_threshold` | 99.0 | Top 1% of events |
| `lookback_days` | 7 | Rolling window for percentile calc |
| `recalculate_every` | 1 bar | Recompute threshold each bar |

### Back Test Results (Cascade P99 H8 variant)
| Stat | Value |
|---|---|
| Return | 33–425% (across variants) |
| Max Drawdown | 8–13% |
| Sharpe | 1.73–2.98 |
| Win Rate | 62–75% |
| Trades | 180–1,000+ |

**Best risk-adjusted result of all six strategies.** 3% drawdown relative to return. Survived all robustness tests:
- Out-of-sample: profitable (84% win rate OS in some variants)
- Monte Carlo: 100% of simulations profitable
- Parameter sensitivity: 100% of 500 combos profitable
- Walk-forward: all 4 quarters profitable, worst quarter +71%

### Notes
- This is the most production-ready of the six
- The adaptive threshold means it doesn't go stale as market volatility changes
- "H8" variant refers to holding for 8 bars (40 minutes at 5m); test different hold durations
- The edge is in selectivity: only the most extreme 1% of events trigger
- Requires more data infrastructure than S1–S4 (7-day rolling history)

---

## Strategy 6 — Adaptive Percentile Reversion (Fade)

**Plain English:** Same P99 trigger as Strategy 5 — but instead of trading *with* the cascade, you *fade* it. You're betting that after an extreme liquidation event, price snaps back. Only take countertrend fades: buy dumps that happen in an uptrend, sell rips that happen in a downtrend. Tight scalp targets for the bounce.

### Trigger Logic
```python
# Reuse S5 percentile trigger
extreme_short_liq = short_liq_bar >= p99_short
extreme_long_liq  = long_liq_bar  >= p99_long

# Trend filter (countertrend only)
price_above_ema = current_price > ema_200   # uptrend
price_below_ema = current_price < ema_200   # downtrend

# Entry: fade the short cascade ONLY if in uptrend (price likely to snap back up)
if extreme_short_liq and price_above_ema:
    enter_long()    # fade: expect mean reversion up

# Entry: fade the long cascade ONLY if in downtrend (price likely to snap back down)
if extreme_long_liq and price_below_ema:
    enter_short()   # fade: expect mean reversion down
```

### Parameters (configurable)
| Variable | Default | Notes |
|---|---|---|
| `percentile_threshold` | 99.0 | Same as S5 |
| `lookback_days` | 7 | Same as S5 |
| `trend_ema_period` | 200 | Bars for trend filter |
| `take_profit_pct` | tight | Scalp — not a swing |
| `stop_loss_pct` | tight | Tight; the cascade can continue |
| `leverage` | low | Noted as using leverage in original spec |

### Back Test Characteristics
- Low trade count — most extreme events in the wrong trend direction are filtered out
- High win rate on the trades that do fire (80%+ in some robustness tests)
- Maximum drawdown very low (7.5% in some variants)
- The trend filter is critical — fading without it is the dominant failure mode

### Key Risk
The data showed that **pure fade strategies (without trend filter) fail 97% of the time.** The trend filter is what makes this work. Without it, momentum almost always wins.

### Notes
- This is the most contrarian of the six — emotionally hardest to trade by hand, which is exactly why it's automated
- Works best on BTC 6-hour time frame specifically (most robust across robustness tests)
- The "Lick BB Reversal" variant (adds Bollinger Band confirmation on top of the percentile trigger) showed 80%+ win rate out-of-sample
- Low trade count is a feature, not a bug — this strategy waits for the perfect setup

---

## Strategy Comparison Matrix

| # | Name | Direction | Trigger | Time Frame | Best Return | Best Drawdown | Sharpe |
|---|---|---|---|---|---|---|---|
| 1 | Cascade Filter | Momentum | 3 consecutive bars same side | 5m | 98% | -25% | 1.64 |
| 2 | Z-Score Momentum | Momentum | 4σ net imbalance vs 24h mean | 5m | 114% | -27% | ~1.5 |
| 3 | Volume Confirmed | Momentum | Cascade + top 10% volume | 5m | 514% | -28% | — |
| 4 | One-Sided Cascade | Momentum | 5x ratio single bar | 5m | — | tight | — |
| 5 | Adaptive Percentile | Momentum | P99 of 7-day liq history | 5m/1h | 425% | -8% | 2.98 |
| 6 | Adaptive Reversion | Countertrend | P99 trigger + trend filter | 6h | moderate | -7.5% | — |

**Key findings from robustness testing:**
- S5 (Adaptive Percentile) is the most robust overall — passes all four robustness tests
- S3 (Volume Confirmed) has the best raw return (514%) but fewer trades
- S6 (Reversion) is the most robust on BTC 6H specifically; lowest drawdown of all
- All six are uncorrelated in trigger logic — they can be run simultaneously as a portfolio

---

## Portfolio Stacking Recommendation

Per the multi-bot rules in `execution-checklist-and-risk-rules.md`:

**Recommended starting portfolio (3 bots):**

| Bot | Strategy | Why |
|---|---|---|
| Bot A | S5 — Adaptive Percentile | Most robust, best risk-adjusted return |
| Bot B | S3 — Volume Confirmed | Highest raw return, different trigger logic |
| Bot C | S6 — Adaptive Reversion | Uncorrelated (countertrend), low drawdown hedge |

These three trigger on different conditions:
- S5 fires on P99 magnitude alone
- S3 fires on cascade + volume spike
- S6 fires on P99 magnitude + wrong-direction trend

They will not all lose at the same time. That's the point.

---

*Last updated: 2026-05-31*  
*Source: Moondev transcripts — `Inspo/Transcripts/Moondev/`*  
*Primary sessions: "All right, so Claude Code built me...", "I found the holy grail trading stra...", "What do I do with liquidation data...", "back test stats are here..."*
