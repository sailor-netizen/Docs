# VPS Deployment and Bot Operations

## Overview

Running bots locally (on a personal machine) means bots stop when the computer sleeps, loses power, or travels. The solution is deploying bots to a cloud VPS and connecting to them via SSH from any machine, leaving bots running 24/7 regardless of local machine state.

---

## VPS Provider

**Primary provider used:** Cherry Servers (`mundave.com/cherry`)

| Server Spec | Value |
|---|---|
| Type | Dedicated resources (not shared) |
| RAM | 16 GB |
| Monthly cost | ~$48.73/month |
| Hourly billing | ~$0.10/hour (available for testing/tutorials) |
| OS | Ubuntu |
| Location example | Amsterdam |

**Why dedicated over shared:**
- Shared servers have "noisy neighbor" risk — other tenants consume RAM and slow the box
- Running private keys on shared infrastructure introduces security risk
- Dedicated is recommended when running bots with live trading credentials

**Alternative for lower budget:** Contabo servers (mentioned as an alternative to Cherry Servers in a separate session)

---

## SSH Setup via VS Code / Cursor

**Extension required:** Remote-SSH (`remote-ssh` in VS Code/Cursor extensions marketplace)

**Connection setup steps:**

1. Install Remote-SSH extension
2. Click the double-arrows icon (bottom-left of VS Code) → "Open Remote Window" → "Connect to Host" → "Add New Host"
3. Enter SSH config in format:
   ```
   Host <IP_ADDRESS>
     HostName <IP_ADDRESS>
     User root
   ```
4. Save the config file
5. Click "Connect to Host" → select the new host
6. On first connect: confirm host authenticity ("yes"), then enter root password
7. VS Code opens a new window connected to the remote server

**Credentials from Cherry Servers:**
- Root username + password: shown for 24 hours only after deployment — save immediately
- Primary IP address: shown on the deployment dashboard
- Cherry Servers cannot recover the password after 24 hours (security by design)

---

## Running Multiple Bots with `screen`

`screen` is a terminal multiplexer that allows multiple persistent terminal sessions on one server. Each session (screen) can run one bot. Sessions persist even when SSH is disconnected.

### The Four Commands

| Command | Action |
|---|---|
| `screen -S <name>` | Create and enter a new named screen |
| `Ctrl+A` then `D` | Detach from current screen (leaves it running) |
| `screen -ls` | List all screens and their status (attached/detached) |
| `screen -r <name>` | Re-attach to a named screen |

### Example — Two Bots Running Simultaneously

```bash
# Create screen for bot one
screen -S bot_one
# Inside screen: navigate to bot folder and run
cd bots && python3 bot_one.py
# Detach: Ctrl+A then D

# Create screen for bot two
screen -S bot_two
cd bots && python3 bot_two.py
# Detach: Ctrl+A then D

# Verify both running
screen -ls
# Output shows: bot_one (Detached), bot_two (Detached)

# Re-attach to check bot one
screen -r bot_one
# Detach again: Ctrl+A then D
```

Bots continue running even when the SSH connection is closed or the local machine is off.

---

## moon-dash SSH Reference

In transcripts, `SSH moon-dash` refers to the researcher's primary named production server. This is a named SSH host in the local SSH config (`~/.ssh/config`) with an entry like `Host moon-dash` pointing to the production server IP. It is used to quickly connect to the live API/bot server without typing the IP each time.

Usage pattern seen:
- Develop and test code locally
- `ssh moon-dash` to connect to the production server
- Deploy updated scripts or verify running bot/API status

---

## Bot Operations Checklist (Pre-Launch)

Before going live on a VPS, confirm:

- [ ] Risk controls implemented in bot code (stop-loss, take-profit logic, max position size)
- [ ] Strategy matches back-tested logic precisely (entry/exit conditions, indicator parameters)
- [ ] Limit orders used instead of market orders where possible (market orders cost ~3x more in fees)
- [ ] Position existence check before placing new orders (prevents double-entry)
- [ ] P&L monitoring loop running (watches open positions for stop/TP conditions)
- [ ] Loop interval set appropriately (every few seconds to minutes depending on strategy)
- [ ] All debug logging removed or silenced
- [ ] Launched with minimum size ($10 on Hyperliquid, $1 on Poly Market) for incubation phase

---

## API / Server Monitoring

From the HIP3 data layer infrastructure (`mundave.com/docs`):

- **Admin endpoint:** Secure API key (not shared with general API users) for monitoring per-endpoint hit frequency — shows which endpoints users call most, informs which parts of the API to make most robust
- **Tick data lag monitoring:** Median lag across active symbols tracked; target is under 60 seconds; 43–154 second lag observed in some sessions, addressed by backend fixes
- **Process monitoring:** Daemon that polls for new HIP3 symbols every 5 minutes; SQLite database; JSON static files served by API; WebSocket ingestion layer

---

## Fees Impact on Bot Design

With a $25,000 account at 40x leverage, trading 5 times/day using market orders: account loses **13% per day** from fees alone — account zeroed in 31 days.

Using limit orders reduces fee cost to ~1/3 of market orders, adding **717 days** to account lifespan under the same trading frequency.

Bot design should default to **limit orders** with configurable discount (e.g., bid at 10% below current price for entries on directional bets).

---

## Security Notes

- Never share API keys, private keys, or server IP addresses on stream
- Researcher notes a significant hack event after years of public streaming — cost described as "a big big L"
- Private keys must not be provided to third-party bots
- SSH credentials should not be shared on screen; cook off-screen when handling sensitive credentials

---

## Source Transcripts

- `So, I want to show you step by step (contobo instead of cherry servers).txt` (primary — full VPS setup walkthrough, screen commands, Cherry Servers)
- `I'm going to show you step by step.txt` (SSH moon-dash reference, bot deployment workflow)
- `I had six clawed codes here as you.txt` (server infrastructure, API monitoring, SQLite on-demand refactor, WebSocket rate limits)
