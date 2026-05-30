# Backtest Framework & Stat Thresholds
> Project Quant — Source of Truth  
> RBI system, backtesting.py setup, pass/fail criteria, robustness tests  
> Derived from `Inspo/Transcripts/Moondev/`

---

## The RBI System

Every strategy goes through three gates before live capital is allocated. No exceptions.

```
R → Research      Find an idea. Come up with a thesis.
B → Backtest      Prove it worked in the past. Kill bad ideas cheaply.
I → Incubate      Run live with $10 size. Forward test in real market conditions.
```

**Why this order matters:**  
The Backtest stage exists to kill ideas before they kill your account. If a strategy never worked in the past, it won't start working now. The Incubate stage exists because backtests can't capture slippage, bad fills, and 3 a.m. market conditions. You must see it run on real money at small size before scaling.

**The no-override rule (Jim Simons principle):**  
Once a bot is live, the model decides 100% of entries and exits. No human override. If you're overriding, you're not running a quant system — you're running a slot machine with extra steps.

---

## Backtesting Environment

### Library
```python
from backtesting import Backtest, Strategy
# backtesting.py — the confirmed standard for this project
```

### Data source
- Primary: MoonDev API (`moondev.com/data`) — `moondev.com/data` with `_QE` API key
- Time frames tested: 5m, 1h, 4h, 6h, 1d
- Minimum data for meaningful results: 30+ trades across the test period
- BTC starting capital: **$1,000,000 minimum** — BTC price > $100K means $100K starting capital is insufficient and causes calculation errors

### CSV output format (standard)
```python
# Log these stats per back test run — goes in the CSV tracker
stats_to_csv = [
    "strategy_name",    # Python filename
    "return_pct",       # annualized return %
    "max_drawdown",     # maximum drawdown % (negative number)
    "sharpe_ratio",     # Sharpe ratio
    "sortino_ratio",    # Sortino ratio
    "sqn",              # SQN (System Quality Number)
    "profit_factor",    # gross profit / gross loss
    "expectancy",       # expected value (EV) per trade
    "num_trades",       # total number of closed trades
    "win_rate",         # % of trades that were profitable
    "exposure_time",    # % of time the strategy was in a position
]

# Also paste full stats block into the top of each .py file as a docstring
```

---

## Pass Criteria — Minimum Thresholds

A strategy must clear **all** of the following before moving to Incubate.

| Stat | Minimum | Target | Notes |
|---|---|---|---|
| Return (annualized) | > 0% (beat buy & hold) | 2x buy & hold | Must beat passive holding the asset |
| Max Drawdown | < 40% | < 25% | Strategies with >40% DD get skipped regardless of return |
| Sharpe Ratio | ≥ 1.0 | ≥ 1.5 | Hard floor; iterate until 10 Sharpes are above 1.5 |
| Sortino Ratio | ≥ 1.0 | ≥ 2.0 | Downside-adjusted; directional complement to Sharpe |
| SQN | ≥ 2.0 | ≥ 3.0 | System Quality Number — overall system health score |
| Profit Factor | ≥ 2.0 | ≥ 3.0 | Gross profit / gross loss; below 2.0 is too thin |
| Expectancy (EV) | > 0 | > 1.0 | Expected value per trade in % or $; must be positive |
| Number of Trades | ≥ 30 | ≥ 50 | Below 30 trades is statistically meaningless |
| Win Rate | any | > 50% | Not a hard requirement — low WR is fine if PF and EV pass |

### The 2x drawdown rule
The return-to-drawdown ratio must be at least 2:1. A strategy returning 50% on a 40% drawdown is marginal. A strategy returning 50% on a 10% drawdown is a keeper.

```
✅ Return 77%,  Drawdown -25%  →  ratio 3.1x  → pass
✅ Return 98%,  Drawdown -25%  →  ratio 3.9x  → pass
⚠️ Return 149%, Drawdown -71%  →  ratio 2.1x  → borderline, skip
❌ Return 93%,  Drawdown -37%  →  ratio 2.5x  → drawdown too high, skip
❌ Return 276%, Drawdown -91%  →  ratio 3.0x  → DD catastrophic, skip
```

### Overfit flags
Any of the following signals likely overfitting — proceed with extreme skepticism:
- Return > 10,000% (20,000%+ is almost certainly overfit)
- Very few trades (< 15) combined with very high return
- Strategy only profitable in one specific date range
- Parameters optimized to exact decimal values with no robustness

---

## Robustness Tests (Run Before Incubating)

A strategy that passes the stat thresholds must then survive **all four** robustness tests. A strategy that passes backtests but fails robustness is not production-ready.

### Test 1 — Out-of-Sample (OOS) / Walk-Forward
Split the data 70/30. Train on 70%, test on the untouched 30%.

```python
# 70/30 split
train_end   = int(len(data) * 0.7)
train_data  = data[:train_end]
test_data   = data[train_end:]

bt_train = Backtest(train_data, MyStrategy, cash=1_000_000)
bt_test  = Backtest(test_data,  MyStrategy, cash=1_000_000)
```

**Pass criteria:** Strategy remains profitable on the OOS period. Some return degradation is expected and acceptable (in-sample → out-of-sample decay). The P99 Cascade strategy showed 84% OOS win rate — that's the bar to aim for.

### Test 2 — Parameter Sensitivity
Run the strategy across a wide grid of parameter combinations. If only a tiny slice of parameters produces good results, the strategy is fragile.

```python
# Example: sweep threshold parameters
for threshold in [0.05, 0.1, 0.15, 0.2, 0.25, 0.3]:
    for lookback in [100, 200, 288, 500]:
        run_backtest(threshold=threshold, lookback=lookback)
```

**Pass criteria:** ≥ 80% of parameter combinations are profitable. The best-performing S5 (Adaptive Percentile) strategy passed with 100% of 500 parameter combinations profitable.

### Test 3 — Monte Carlo
Randomly resample or shuffle the trade sequence 100+ times. Check if the strategy survives in the majority of simulations.

```python
# Simplified Monte Carlo: run with 20% of trades randomly removed
import random

def monte_carlo_run(trades, n_simulations=100, drop_pct=0.2):
    results = []
    for _ in range(n_simulations):
        sampled = random.sample(trades, int(len(trades) * (1 - drop_pct)))
        results.append(simulate_trades(sampled))
    return results
```

**Pass criteria:** ≥ 95% of Monte Carlo simulations are profitable. The S5 strategy survived 100% of simulations even with 20% of trades removed.

### Test 4 — Rolling Window (Quarterly)
Chop the data into 4 equal periods (Q1–Q4). The strategy must be profitable in all four.

```python
quarters = [data[i:i+len(data)//4] for i in range(0, len(data), len(data)//4)]
for q_num, q_data in enumerate(quarters[:4], 1):
    stats = Backtest(q_data, MyStrategy).run()
    print(f"Q{q_num}: Return {stats['Return [%]']:.1f}%")
```

**Pass criteria:** All 4 quarters profitable. The S5 strategy's worst quarter was +71% — that's the quality bar.

### Robustness Summary Table (from transcripts)
| Test | S5 Adaptive Percentile | Notes |
|---|---|---|
| OOS (70/30) | Profitable, 84% WR OOS | Minimal edge decay |
| Parameter Sensitivity | 100% of 500 combos profitable | Edge is in the signal, not parameters |
| Monte Carlo (20% drop) | 100% of simulations profitable | Tight draw downs in shuffled world |
| Walk-Forward (4 quarters) | 4/4 quarters profitable, worst Q +71% | Max DD only -10% |

---

## Bot Template Checklist (Before Incubating)

Once a strategy passes all four robustness tests, it gets built into a bot. Every bot must include:

```
[ ] Limit orders only — no market orders
[ ] Cancel all open orders before placing a new entry
[ ] Dust adjustment (account for minimum trade sizes)
[ ] while True loop with try/except — reset on exception, never silent-die
[ ] $10 incubation size — scale only after 4+ weeks live
[ ] Stop-loss hardcoded — never move it wider
[ ] Take-profit hardcoded — never move it wider
[ ] Kill switch: pause if daily bot loss > 25% of bot capital
[ ] Kill switch: pause if weekly portfolio loss > 20%
[ ] CSV trade log — every fill recorded with timestamp, size, P&L
```

---

## Multi-Bot Portfolio Rules

After Incubation, winning bots graduate into the live portfolio.

**Minimum portfolio size:** 3 uncorrelated bots running simultaneously.

**Correlation requirement:** Bots must have different trigger logic so they don't all lose in the same week. Example of acceptable diversity:
- Bot A: Fires on P99 liquidation magnitude alone (S5)
- Bot B: Fires on cascade + volume spike (S3)
- Bot C: Fires countertrend on P99 + trend filter (S6)

These three will not all lose simultaneously. That's the point.

**Sizing rule:** After 4 weeks of incubation, size each bot proportional to its Sharpe ratio.

```python
# Proportional sizing by Sharpe
sharpes = {"bot_a": 2.98, "bot_b": 1.64, "bot_c": 1.80}
total   = sum(sharpes.values())
alloc   = {k: v / total for k, v in sharpes.items()}
# bot_a gets 45%, bot_b gets 25%, bot_c gets 27% of portfolio
```

---

## What "Good" Looks Like — Real Examples From Transcripts

These are confirmed back test results from actual sessions, not hypotheticals.

| Strategy | Return | Drawdown | Sharpe | EV | Trades | Verdict |
|---|---|---|---|---|---|---|
| Cascade Filter (S1) | 98% | -25% | 1.64 | — | low | ✅ Pass |
| Z-Score Momentum (S2) | 114% | -27% | ~1.5 | 3.96 | — | ✅ Pass |
| Volume Confirmed V2 (S3) | 514% | -28% | — | 5.777 | 47 | ✅ Pass (high conviction) |
| Adaptive Percentile (S5) | 33–425% | -8–13% | 1.73–2.98 | — | 180–1K | ✅ Pass all robustness |
| Time-scaled variant | 149% | -71% | — | — | — | ❌ Fail (DD > 40%) |
| ADX+MACD cross | 34% | -3.4% | high | — | — | ✅ Pass (10:1 ratio) |
| BB squeeze breakout | 20% | -5.3% | 1.25+ | 3.4 | low | ✅ Pass (low exposure) |
| 20K% overfit example | 20,484% | — | — | — | 9 | ❌ Overfit (9 trades) |
| 91% DD example | 276% | -91% | — | — | — | ❌ Fail (catastrophic DD) |

---

*Last updated: 2026-05-31*  
*Source: Moondev transcripts — `Inspo/Transcripts/Moondev/`*  
*Primary sessions: "I found the holy grail trading stra", "All right, so Claude Code built me", "back test pass goals", "What do I do with liquidation data", "I had six clawed codes here"*