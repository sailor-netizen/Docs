# Execution Checklist & Risk Rules
> Project Quant — Source of Truth  
> Derived from 5 years of live sessions, transcripts, and confirmed patterns in `Inspo/Transcripts/Moondev/`

---

## Core Philosophy

**Bots don't feel. That's exactly why they win.**

Every loss in manual trading traces back to emotion — fear, greed, FOMO, revenge, hope. The entire purpose of this system is to remove the human from the execution loop. No overrides. No exceptions. The bot flies the plane.

> *"You can't force yourself to stop being human. Don't do that."*

The framework that governs everything: **RBI — Research → Backtest → Incubate.**

---

## 1. The RBI Framework

Every strategy that goes live must pass all three stages in order. Skipping any stage is a protocol violation.

### Stage 1 — Research
- Find an idea with a real edge: a reason the market pays you repeatedly
- Source ideas from: liquidation data patterns, academic papers (via Quant Strategies), your own observations
- Document the thesis clearly before touching any code: what triggers entry, what triggers exit, why this should work

**Disqualifiers at research stage:**
- The idea is based on a chart pattern you saw once
- No clear hypothesis for why the edge exists
- The idea relies on predicting direction without a trigger

---

### Stage 2 — Backtest

Run the idea against historical data before risking a single dollar. The backtest is a flight simulator — crashes don't cost you real money, but the lessons are real.

#### Required Stats — Minimum Thresholds to Proceed

| Stat | Minimum to Proceed | Notes |
|---|---|---|
| Return (annualised) | > 30% | Must beat buy-and-hold for the same period |
| Max Drawdown | < 40% | Preferred < 30%. Above 50% = reject |
| Sharpe Ratio | ≥ 1.0 | Target > 1.5. Below 1.0 = not ready |
| Sortino Ratio | ≥ 1.5 | Penalises downside volatility only |
| Profit Factor | ≥ 2.0 | Gross profit / gross loss |
| Expectancy (EV) | > 0 | Positive expected value per trade required |
| Win Rate | Context-dependent | A 30% win rate with 5:1 R is fine. A 70% win rate with 0.5:1 R is not. |
| Number of Trades | ≥ 30 | Less than 30 trades = statistically insufficient |
| Exposure Time | < 80% | Always in market = overfit risk |
| SQN | ≥ 2.0 | System Quality Number |

**Robustness Tests (required before incubation):**
- Run across at least 3 different symbols (not just BTC)
- Run across at least 2 different time periods (split data in half)
- Sharp drop in performance on out-of-sample data = overfitting, reject
- If stats look "too good" (e.g. 20,000% return) = almost certainly overfit, reject

#### Backtest Hygiene Rules
- Position sizing: 95% of account maximum (never 100%)
- Use $1,000,000 minimum simulated capital when backtesting BTC (BTC > $100K; smaller capital causes position-sizing distortions)
- Never run the optimiser during initial assessment — get raw unoptimised stats first
- All stats go in the triple-quoted docstring at the top of the backtest file
- All stats are also logged to `backtest_results.csv` with columns: `strategy_name`, `symbol`, `return_pct`, `max_drawdown_pct`, `sharpe`, `sortino`, `profit_factor`, `ev`, `num_trades`, `exposure_time`
- Label every file with date: e.g. `liquidation_cascade_v1_2026-05-31.py`

---

### Stage 3 — Incubate

**Paper trading is banned.** Paper mode lies — it doesn't account for slippage, bad fills, or 3am weirdness. The only valid incubation is real money at tiny size.

#### Incubation Rules
- Starting size: **$10 per trade**
- Duration: minimum **2–4 weeks** of live runs before scaling
- If the bot breaks or errors, it must reset and loop — never die silently
- Keep a log of live trades during incubation; compare to backtest expectations
- Only after incubation confirms the edge does size scale up

**Scaling decision:**
- Incubation profitable → scale up incrementally (2x, then 4x, etc.)
- Incubation losing → back to research stage, do not force it

---

## 2. Bot Construction Checklist

Every bot built for live trading must satisfy all items before deployment. This is non-negotiable.

### Pre-Launch

- [ ] Strategy has a written thesis (one paragraph minimum, committed to `Research/`)
- [ ] Backtest has been run across ≥ 3 symbols and ≥ 2 time periods
- [ ] All required stats pass minimum thresholds (see table above)
- [ ] Backtest stats are documented in both the file docstring and `backtest_results.csv`
- [ ] Position size variable is exposed at the top of the file and set to `$10` for incubation

### Execution Logic

- [ ] **Limit orders only** — never market orders. Enter on limit, cancel and re-loop until filled
- [ ] **Cancel all open orders** before opening a new position — no stacking
- [ ] **Dust adjustment** applied to position sizing to avoid micro-remainder positions
- [ ] Loop structure is `while True` with a reset on exception — the bot must never silently die
- [ ] Bot logs its own trades and state to a file for auditability

### Risk Controls

- [ ] Stop-loss is defined and hardcoded — not optional, not commented out
- [ ] Take-profit is defined
- [ ] Maximum position size cap is set (even if size is $10, the cap must exist as a variable)
- [ ] Leverage is explicit and logged — no implicit leverage
- [ ] Bot handles the case where the exchange returns an error without blowing up
- [ ] No manual override path exists in the production bot — if you want to override, stop the bot and trade manually in a separate account

### Infrastructure

- [ ] Bot is deployed on a VPS (Cherry Servers or equivalent), not a local machine
- [ ] Bot restarts automatically on crash (e.g. `supervisor`, `systemd`, or equivalent)
- [ ] SSH key access configured — no password login
- [ ] Environment variables used for API keys — keys never hardcoded in source

---

## 3. Runtime Risk Rules

These rules govern a bot once it is live. Violating these is a protocol breach.

### The No-Override Rule

**The model is the boss. No trade is ever made because a human decides to override the bot.**

If you believe the bot is wrong:
1. Stop the bot
2. Let the position close naturally or close manually in a separate session
3. Go back to research to understand why the bot is wrong
4. Fix the strategy, backtest again, re-incubate

There is no Step 4 that is "just this once override the signal."

> *"Jim Simons ran 100% model-driven. Not 90%, not 80%. 100%. The religious sticking to the model is the only way."*

### Position Sizing Rules

| Stage | Max Size per Trade |
|---|---|
| Incubation | $10 |
| Live (early) | ≤ 1% of account |
| Live (proven) | ≤ 5% of account |
| Max ever | 95% of account (for strategies explicitly designed for full allocation) |

Never size a position based on how confident you feel. Confidence is an emotion.

### Drawdown Kill Switch

- If any single bot exceeds **25% drawdown** from its equity peak during live trading, stop it automatically
- If the entire portfolio (all bots combined) exceeds **20% drawdown** in a single week, halt all bots and review
- Drawdown thresholds are hardcoded as bot variables, not mental rules

### Leverage Rules

- Default: **no leverage** during incubation
- If leverage is used: it must be backtested with the same leverage applied
- 40x leverage is explicitly banned — this is the number one cause of retail liquidation
- Any bot that uses leverage must have a tighter stop-loss to compensate

### Liquidation Awareness

The platform (Hyperliquid) shows all open positions and their liquidation distances. Bots that trade around liquidations must:
- Never bet solely on cascading — confirm with RSI, MACD, or volume
- Track whether recent liq events were isolated or part of a cascade (3 consecutive bars, same side)
- Minimum liquidation threshold to trigger entry: **$500K** for majors, configurable per symbol

---

## 4. Strategy Retirement Rules

Not every bot runs forever. Anomalies fade. These are the conditions that trigger retirement or review:

| Condition | Action |
|---|---|
| Live Sharpe drops below 0.5 for 4 consecutive weeks | Pause, review, possibly retire |
| Strategy loses on 3 consecutive months | Halt, re-backtest on recent data |
| Market structure changes (e.g. new major exchange rule) | Flag for immediate review |
| Number of trades drops significantly vs. backtest expectation | Investigate data feed or trigger logic |
| Win rate drops > 15% below backtest win rate | Halt, diagnose |

---

## 5. Multi-Bot Portfolio Rules

Running multiple bots concurrently is the goal — not one bot, but a stack of uncorrelated edges.

- **Target: ≥ 3 live bots at all times**, each with a different trigger type
- Bots should be **uncorrelated in trigger logic** — don't run two RSI-based bots simultaneously; diversify across liquidation momentum, volume-confirmed, mean-reversion, etc.
- Combined portfolio Sharpe should exceed any individual bot's Sharpe (if it doesn't, the bots are too correlated)
- If one bot has a bad week, the others carry it. Size accordingly

**Capital allocation per bot:**
- Start equal weight during incubation
- After 4+ weeks live, allocate proportional to Sharpe ratio
- No single bot should ever represent > 40% of deployed capital

---

## 6. Quick Reference — What Never Changes

These are the immovable rules of the system. No exceptions, no "just this once."

1. **RBI only** — no live trading without backtest
2. **Limit orders only** — never market orders in production
3. **$10 incubation size** — always start small
4. **No overrides** — if you override the bot, you've become an emotional trader again
5. **95% max position size** — never go full Kelly
6. **Cancel all before new entry** — no stacked orders
7. **Bot must loop forever** — `while True`, reset on error
8. **API keys in env vars** — never hardcoded
9. **Drawdown kill switch** — 25% per bot, 20% portfolio weekly
10. **No paper trading** — real money, tiny size, real lessons

---

*Last updated: 2026-05-31*  
*Source: Moondev transcripts, 5 years of live sessions — `Inspo/Transcripts/Moondev/`*
