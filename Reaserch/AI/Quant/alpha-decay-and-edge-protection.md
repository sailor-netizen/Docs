# Alpha Decay and Edge Protection

## Core Concept

**Alpha decay** is what happens when a profitable trading strategy stops working because too many people are using it. The alpha *is* the edge. Once a strategy is widely known or replicated, the edge compresses to zero.

---

## Why This Explains Quant Secrecy

- No professional quant shares a live, working bot — the act of sharing is the act of destroying the edge
- Public "earn 3–5 ETH/day, just plug in your private key" bots on YouTube/ads are either scams or already-decayed strategies
- Private keys should never be handed to a third-party bot regardless
- The quant community is structurally secretive **because the economics demand it** — sharing is self-defeating

---

## The Edge Lifecycle

| Phase | Description |
|---|---|
| Discovery | Researcher identifies anomaly in historical data |
| Back-test | Edge confirmed on out-of-sample data |
| Incubation | Live deployment with minimum size ($10 on Hyperliquid) |
| Scale | Increase position size as live performance validates back-test |
| Decay | Strategy becomes known → too many players → edge narrows → retires |

---

## Implications for Bot Strategy

- Never rely on a single bot — one bot decay event should not destroy an account
- Stack **multiple uncorrelated bots** so each bot's bad period is carried by the others
- Three bots with different trigger logic (e.g., RSI extremes vs. Bollinger Band rejection) that don't co-fire on the same bars is better than one "holy grail" bot with higher absolute returns
- Quant Elite / community code drops should be treated as **starting points for ideas**, not plug-and-play edges — running the same code as hundreds of others accelerates decay

---

## What Preserves Edge

- Proprietary data (e.g., tick data collected since day of listing — not obtainable from public API retroactively)
- Proprietary infrastructure (lower-latency access, node-level data)
- Unique signal combination or time-window that others are not looking at
- Keeping the strategy private — even from close associates

---

## RBI Framework (Research → Back-test → Incubate)

The three-step process designed to find and validate edges before risking capital:

1. **Research** — source ideas from books (Market Wizards, The Quants), podcasts (Chat With Traders, ~300 episodes), Google Scholar papers, and live market observation at minimum size
2. **Back-test** — run the idea against historical data; if it never worked in the past it won't work in the future; eliminate bad ideas before they kill the account
3. **Incubate** — deploy with minimum live size ($10 on Hyperliquid, $1 on Poly Market); demos lie, incubation doesn't; hidden costs (bad fills, 3am slippage) only appear in live runs

---

## Alpha Decay vs. Sharing Code

- Releasing back-tests (e.g., Quant Elite drops) is lower-risk than releasing live bots — the back-test is a completed piece of research, not the running edge
- Even back-tests should be treated as idea starters: "take it and make it yours"
- Community learning compounds the *researcher's* edge because feedback from hundreds of smart traders generates new ideas faster than solo work

---

## Key Quotes (Source Material)

> "Alpha decay is what happens when a profitable trading strategy stops working because too many people are using it. The alpha is your edge."

> "Nothing is plug-and-play. Something called alpha decay means that if everybody's running the same bot, it's going to go to zero."

> "If you can live longer, you can compound more." *(on why edge preservation = time in the game)*

---

## Source Transcripts

- `Can you go explain Alpha Decay for.txt`
- `I had six clawed codes here as you.txt` (RBI framework detail, multi-bot stacking)
- `Today I'm going to show you step by.txt` (Poly Market bot build, RBI walkthrough)
