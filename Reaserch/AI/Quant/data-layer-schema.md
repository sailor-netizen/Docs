# Data Layer Schema
> Project Quant — Source of Truth  
> MoonDev Hyperliquid Data Layer API — endpoints, field shapes, auth, and infrastructure  
> Derived from `Inspo/Transcripts/Moondev/` — live sessions, API walkthroughs, and Quant App builds

---

## Overview

MoonDev runs a self-hosted direct Hyperliquid node on a VPS (`ssh moon-dash`). This node powers the data layer API — it is not a wrapper around the public Hyperliquid REST API. The direct connection is approximately 20x faster than the public API and includes data not available through any other source (positions scanner, liquidation history, HLP trades).

The API exposes 55+ confirmed endpoints. A self-healing monitor checks all endpoints continuously and alerts on failures.

**Docs:** `moondev.com/docs`  
**Bulk data download:** `moondev.com/data`  
**Cost comparison:** CoinGlass charges ~$900/month for 12 days of 1-minute liquidation data. This API has 18–20 months.

---

## Authentication

```python
headers = {
    "x-api": "YOUR_API_KEY"    # NOT "Authorization: Bearer ..."
}
```

The API key is passed as an `x-api` header. Bearer token format is not supported and will return 401.

Keys are tiered:
- **Standard key** (stream key) — access to all live endpoints
- **Quant Elite key** (`_QE` suffix) — standard + bulk historical data download at `moondev.com/data`

Keys rotate daily in public live streams. Permanent keys are issued through the paid membership.

---

## Endpoints

### 1. Liquidations (Live)

**What it returns:** Per-bar liquidation totals, split by long/short side and by exchange. Updated every 30–60 seconds.

**Use case:** All 6 liquidation strategies (S1–S6) pull from this endpoint as their primary trigger signal.

```python
# Field shape — one row per bar per exchange
{
    "symbol":     str,    # e.g. "BTC", "ETH", "HYPE"
    "side":       str,    # "long" or "short"
    "size_usd":   float,  # USD value liquidated on this bar
    "count":      int,    # number of liquidation events
    "timestamp":  int,    # unix ms (bar open time)
    "exchange":   str     # "hyperliquid", "binance", "okx"
}
```

**Exchanges available:** Hyperliquid (primary), Binance, OKX. Multi-exchange aggregation possible — sum `size_usd` across exchanges for total bar liquidation pressure.

**In-bot usage:**
```python
# Aggregate per bar for strategy triggers
short_liq_bar = sum(r["size_usd"] for r in bar_data if r["side"] == "short")
long_liq_bar  = sum(r["size_usd"] for r in bar_data if r["side"] == "long")
```

---

### 2. Liquidation History (Bulk)

**What it returns:** 18–20 months of Binance futures liquidation data. 28M+ events totaling $60B in liquidated value.

**Access:** Quant Elite tier only (`_QE` API key). Download via `moondev.com/data`.

**Available datasets (confirmed):**
| Dataset | Resolution | History |
|---|---|---|
| BTC daily | 1d | 10 years |
| BTC 6-hour | 6h | 10 years |
| BTC 1-hour | 1h | 500 weeks |
| ETH | varies | from Oct 2016 |
| Binance futures liq | varies | 18–20 months |

**Download size:** ~2 GB compressed.

**How to load:**
```python
import pandas as pd

# After bulk download from moondev.com/data
df = pd.read_csv("binance_liquidations.csv")
# Columns: symbol, side, size_usd, timestamp, exchange
```

---

### 3. Positions — Longs Near Liquidation

**What it returns:** Top 50 long positions on Hyperliquid closest to their liquidation price. Updates on a 1–2 second priority scan.

**Architecture:**
- Full scan: 23,000 wallet addresses, every 90 seconds
- Priority scan: top 50 longs + top 50 shorts (100 addresses total), every 1–2 seconds
- Per-symbol priority scan: top 10 per symbol for BTC, ETH, SOL, HYPE

**Field shape:**
```python
{
    "address":          str,    # wallet address (HL)
    "symbol":           str,    # e.g. "BTC"
    "side":             str,    # "long"
    "size_usd":         float,  # position size in USD
    "liq_distance_pct": float,  # % away from liquidation price
    "liq_price":        float,  # liquidation price
    "entry_price":      float,  # average entry price
    "label":            str     # custom name if set by user
}
```

**Notes:**
- The per-symbol priority scan fires approximately every 1–2 seconds for the tracked symbols
- Non-priority symbols update only in the full 90-second scan
- 90-second lag on specific symbol lookups was a confirmed bug being addressed (patch: add per-symbol priority queue)

---

### 4. Positions — Shorts Near Liquidation

Same shape and update cadence as Endpoint 3, filtered to short positions.

```python
{
    "address":          str,
    "symbol":           str,
    "side":             str,    # "short"
    "size_usd":         float,
    "liq_distance_pct": float,
    "liq_price":        float,
    "entry_price":      float,
    "label":            str
}
```

---

### 5. OHLCV

**What it returns:** Open/high/low/close/volume data for any Hyperliquid symbol on any available time frame.

**Time frames available:** `"5m"`, `"15m"`, `"1h"`, `"4h"`, `"6h"`, `"1d"`

**Fetch pattern:**
```python
symbol     = "HYPE"    # any HL symbol
interval   = "1h"
batch_size = 300       # rows per API call

# Loop calls to get unlimited history
# Each call returns up to batch_size rows
# Decrement start_time by (batch_size * interval) for each loop
# Returns UTC timestamps — convert for display
```

**Field shape (pandas DataFrame):**
```python
# Columns: open, high, low, close, volume
# Index: timestamp (UTC, unix ms)

import pandas as pd
df = pd.DataFrame(response)
df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms", utc=True)
```

**Key detail:** Exchanges cap historical pulls at ~30 days per call. To get 500 weeks of data, loop the call and concatenate results. This is the same approach used for all exchanges (Binance, Kraken, Coinbase).

---

### 6. HLP Trades

**What it returns:** Every trade executed by HLP (Hyperliquid's internal market maker). Includes entry, exit, size, and direction.

**Use case:** HLP runs a $118M+ vault. Watching their trades in real time is a meaningful edge signal.

```python
{
    "address":    str,    # HLP wallet address
    "symbol":     str,
    "side":       str,    # "long" or "short"
    "size_usd":   float,
    "price":      float,
    "action":     str,    # "open" or "close"
    "timestamp":  int     # unix ms
}
```

---

### 7. Wallet Trade History

**What it returns:** Full trade history for any Hyperliquid address. All fills, entries, exits.

**Use case:** Reverse-engineering whale strategies; tracking addresses in the Quant App.

```python
# Request: pass wallet address
# Response: list of trades
{
    "address":   str,
    "symbol":    str,
    "side":      str,
    "size_usd":  float,
    "price":     float,
    "pnl":       float,   # realized P&L on close
    "timestamp": int
}
```

**Quant App feature:** You can name any address with a custom label for tracking (e.g., "hype_bull"). Labels persist in the app.

---

### 8. Polymarket Whale Scanner

**What it returns:** The top Polymarket traders by P&L. 6,200+ wallets indexed.

```python
{
    "address":    str,    # Polymarket wallet
    "pnl_1d":    float,  # P&L last 24 hours
    "pnl_30d":   float,  # P&L last 30 days
    "pnl_all":   float,  # total P&L all time
    "label":      str     # custom name if set
}
```

**Filter example:**
```python
# Only show wallets up at least $100K all-time
whales = [w for w in response if w["pnl_all"] >= 100_000]
```

---

### 9. Tick Data (WebSocket)

**What it returns:** Every tick for all Hyperliquid tokens in real time.

**Use case:** CVD (cumulative volume delta) calculations; sub-second signal generation; any strategy where open/high/low/close is too coarse.

```python
# WebSocket stream — one message per trade tick
{
    "symbol":    str,
    "price":     float,
    "size":      float,   # size in base token
    "side":      str,     # "buy" or "sell"
    "timestamp": int      # unix ms
}
```

**Note:** Tick data is confirmed available but WebSocket endpoint URL is not published in transcripts — access via `moondev.com/docs` with API key.

---

### 10. Funding Rates

**What it returns:** Current funding rate per symbol on Hyperliquid.

```python
{
    "symbol":       str,
    "funding_rate": float,  # e.g. 0.0001093 = 0.01093%
    "timestamp":    int
}
```

---

### 11. HIP3 Markets (Oil, Gold, Silver, Nvidia, S&P)

Hyperliquid's HIP3 protocol adds traditional market tokens: CL (crude oil/WTI), XAU (gold), XAG (silver), NVDA (Nvidia), XYZ100 (S&P proxy).

**All position and liquidation endpoints work identically for HIP3 symbols.** Use the same field shapes as Endpoints 3 and 4, just pass the HIP3 symbol.

```python
# Example: fetch longs near liquidation for crude oil
symbol = "CL"
# Same endpoint, same shape — just different symbol
```

**Why this matters:** There is always a trending market somewhere. When crypto is in a drawdown, HIP3 exposes the war trade (oil), the safe-haven trade (gold/silver), or the tech trade (Nvidia). HLP participates in HIP3, so their trade data covers these symbols too.

---

## Data Infrastructure Notes

### Server Setup
```bash
# VPS SSH access
ssh moon-dash

# Switch user
su mundav

# API monitor runs continuously — checks all 55 endpoints
# Alerts on failures; self-healing where possible
```

### Bulk Download
```bash
# Download historical datasets
# Requires _QE API key
curl -H "x-api: YOUR_QE_KEY" https://moondev.com/data/btc-1h.csv -o btc_1h.csv
```

### Latency Advantage
The direct node connection is approximately 20x faster than the public Hyperliquid API. This is the primary reason the API was built — the Quant App needed position data updating at 1–2 second intervals across 23,000 wallets, which is not achievable through the public API.

### Rate Limits
No documented rate limits in transcripts, but the priority scanner architecture implies:
- Priority addresses: 1–2 second refresh
- Full scan: 90-second cycle
- Live liquidation stream: 30–60 second update cadence
- Tick data: real-time (WebSocket, no polling)

---

## Python Integration Patterns

### Standard request template
```python
import requests

BASE_URL = "https://moondev.com"  # confirm exact base from moondev.com/docs
API_KEY  = "YOUR_API_KEY"

headers = {"x-api": API_KEY}

def get_liquidations(symbol: str = "BTC") -> list[dict]:
    resp = requests.get(
        f"{BASE_URL}/liquidations",
        headers=headers,
        params={"symbol": symbol}
    )
    resp.raise_for_status()
    return resp.json()
```

### Unlimited OHLCV history
```python
import pandas as pd
from datetime import datetime, timedelta

def get_ohlcv_unlimited(symbol: str, interval: str, weeks: int) -> pd.DataFrame:
    """Loop OHLCV calls to get unlimited history."""
    all_data = []
    batch    = 300
    end_time = int(datetime.utcnow().timestamp() * 1000)

    while True:
        resp = requests.get(
            f"{BASE_URL}/ohlcv",
            headers=headers,
            params={"symbol": symbol, "interval": interval,
                    "limit": batch, "endTime": end_time}
        )
        data = resp.json()
        if not data:
            break
        all_data.extend(data)
        end_time = data[0]["timestamp"] - 1   # move window back
        # add break condition based on total weeks desired

    df = pd.DataFrame(all_data)
    df["timestamp"] = pd.to_datetime(df["timestamp"], unit="ms", utc=True)
    return df.sort_values("timestamp").reset_index(drop=True)
```

### Liquidation aggregation for strategy triggers
```python
def aggregate_liq_bar(liq_data: list[dict]) -> dict:
    """Aggregate raw liquidation rows into per-bar totals."""
    return {
        "short_liq_usd": sum(r["size_usd"] for r in liq_data if r["side"] == "short"),
        "long_liq_usd":  sum(r["size_usd"] for r in liq_data if r["side"] == "long"),
        "short_count":   sum(1 for r in liq_data if r["side"] == "short"),
        "long_count":    sum(1 for r in liq_data if r["side"] == "long"),
    }
```

---

## Quant App Data Tabs

The Quant App (`moondev.com` — downloadable app) surfaces the following data panels, all powered by the same API key:

| Tab | Data Source | Update Rate |
|---|---|---|
| **Trade** | Positions near liquidation (longs + shorts) | 1–2s (priority) / 90s (full) |
| **Code** | xterm.js 4×4 terminal grid | n/a |
| **Data** | Liquidation levels, multi-exchange feed | 30–60s |
| **Backtest** | OHLCV + liq history | on-demand |
| **HLP** | HLP trade feed | real-time |

---

## Endpoint Summary Table

| # | Name | Update Rate | Tier | Key Use |
|---|---|---|---|---|
| 1 | Liquidations (live) | 30–60s | Standard | Strategy triggers (S1–S6) |
| 2 | Liquidation history (bulk) | static | QE only | Backtesting |
| 3 | Longs near liquidation | 1–2s (priority) | Standard | Entry signals, whale tracking |
| 4 | Shorts near liquidation | 1–2s (priority) | Standard | Entry signals, whale tracking |
| 5 | OHLCV | on-demand | Standard | Trend filter, volume confirmation |
| 6 | HLP trades | real-time | Standard | Smart money overlay |
| 7 | Wallet trade history | on-demand | Standard | Whale reverse-engineering |
| 8 | Polymarket whales | on-demand | Standard | Copy/stink-bid signals |
| 9 | Tick data | real-time (WS) | Standard | CVD, sub-second signals |
| 10 | Funding rates | periodic | Standard | Market regime context |
| 11 | HIP3 markets | same as 3/4 | Standard | Oil, gold, Nvidia, S&P exposure |

---

*Last updated: 2026-05-31*  
*Source: Moondev transcripts — `Inspo/Transcripts/Moondev/`*  
*Primary sessions: "data layer that I built here", "asked me how do you get data for testing crypto", "In this video I want to help you understand the longs near lick", "I used GPT 5.4 across six different terminals", "What do I do with liquidation data"*
