# SolBot v2.0 — Solana Auto-Trading Bot

A professional Telegram-controlled Solana trading bot with full safety features,
multi-wallet support, new token scanning, and detailed analytics.
Deployed on [Render](https://render.com).

---

## File Structure

```
solana-trade-bot/
├── main.py                  ← Entry point
├── config.py                ← All configuration & constants
├── requirements.txt         ← Python dependencies
├── Procfile                 ← Render process definition
├── render.yaml              ← Render build & start config
├── runtime.txt              ← Python 3.11 version pin
├── .env.example             ← Environment variable template
├── .gitignore
├── README.md                ← This file
├── LICENSE                  ← MIT License
├── core/
│   ├── database.py          ← SQLite persistence layer
│   ├── wallet.py            ← Wallet create/import/balance/transfer
│   ├── jupiter.py           ← Jupiter DEX quotes, swaps, simulation
│   ├── safety.py            ← Honeypot, freeze auth, liquidity checks
│   ├── trader.py            ← Buy/sell engine + position monitor
│   ├── scanner.py           ← New token scanner (pump.fun, Raydium, Meteora)
│   ├── alert_monitor.py     ← Price alert background task
│   └── scheduler.py         ← APScheduler daily report
├── handlers/
│   ├── commands.py          ← All /command handlers
│   └── signal_handler.py    ← Channel signal message parser
└── utils/
    ├── state.py             ← In-memory bot state singleton
    ├── crypto.py            ← AES-256-GCM key encryption
    └── parser.py            ← Solana address extractor
```

---

## Step-by-Step Render Deployment

### STEP 1 — Create a Telegram Bot

1. Open Telegram → search **@BotFather**
2. Send `/newbot`
3. Follow the prompts — give it a name and a username ending in `bot`
4. Copy the token it returns, e.g. `7123456789:AAF-abc123...`
   This is your `TELEGRAM_BOT_TOKEN`

---

### STEP 2 — Get Your Telegram User ID

1. Open Telegram → search **@userinfobot**
2. Send `/start`
3. It replies with your numeric ID, e.g. `987654321`
   This is your `TELEGRAM_OWNER_ID`

---

### STEP 3 — Generate an Encryption Key

Run this on any machine that has Python installed:

```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

Copy the 64-character hex string it prints.
This is your `ENCRYPTION_KEY`.

> ⚠️ Save this key somewhere safe outside Render — in a password manager
> or offline note. It encrypts every wallet private key stored in the
> database. If you lose it you lose access to all stored wallets.

---

### STEP 4 — Get a Solana RPC Endpoint

The free public Solana RPC rate-limits aggressively and will cause missed
trades. A free-tier paid RPC is strongly recommended.

| Provider | Free Tier | Notes |
|---|---|---|
| Public (no sign-up) | Yes (very limited) | `https://api.mainnet-beta.solana.com` |
| [Helius](https://helius.xyz) | Yes — 100k req/day | Best free option |
| [QuickNode](https://quicknode.com) | Trial available | Reliable |
| [Triton](https://triton.one) | Paid only | Professional grade |

**Recommended — Helius free tier:**
1. Go to [helius.xyz](https://helius.xyz) → Sign up → Create a new API key
2. HTTP URL: `https://mainnet.helius-rpc.com/?api-key=YOUR_KEY`
3. WebSocket URL: `wss://mainnet.helius-rpc.com/?api-key=YOUR_KEY`

---

### STEP 5 — Upload Code to GitHub

1. Go to [github.com](https://github.com) → click **New repository**
2. Name it `solana-trade-bot` → set to **Private** → click **Create repository**
3. Extract the zip file you downloaded
4. Upload **all files** into the repository so the structure looks exactly
   like this at the **root level** of the repo:

```
your-repo/
├── main.py           ← must be at root
├── config.py         ← must be at root
├── requirements.txt  ← must be at root
├── Procfile          ← must be at root
├── render.yaml       ← must be at root
├── runtime.txt       ← must be at root
├── core/
├── handlers/
└── utils/
```

> ⚠️ Do NOT upload the outer `solana-trade-bot/` folder itself.
> Open the extracted folder, select everything **inside** it, and drag
> those files into the GitHub browser uploader. `main.py` must be
> visible at the root of the repo — not inside a subfolder.

---

### STEP 6 — Create a Render Account

1. Go to [render.com](https://render.com) → **Get Started for Free**
2. Click **Sign up with GitHub** — this links your repositories automatically
3. Verify your email if prompted

---

### STEP 7 — Create a Background Worker Service

Telegram bots use long-polling — they do **not** need an HTTP web server.
On Render you must create a **Background Worker**, not a Web Service.
Background Workers run continuously without needing to serve HTTP traffic.

1. From the Render dashboard click **New +**
2. Select **Background Worker**
3. Click **Connect account** if GitHub is not yet linked, then select
   your `solana-trade-bot` repository
4. Click **Connect**

---

### STEP 8 — Configure the Service

Render shows you a configuration screen. Fill in every field as follows:

| Field | Value |
|---|---|
| **Name** | `solana-trade-bot` |
| **Region** | Closest to your location |
| **Branch** | `main` |
| **Runtime** | `Python 3` |
| **Build Command** | `pip install -r requirements.txt` |
| **Start Command** | `python main.py` |
| **Instance Type** | **Starter ($7/month)** — see note below |

> ⚠️ **Do not use the Free instance type.**
> Render Free tier suspends background workers after inactivity and
> terminates long-running processes. Your bot will go offline randomly.
> The **Starter plan at $7/month** keeps the bot running 24/7
> without interruption. This is mandatory for a trading bot.

---

### STEP 9 — Add Environment Variables

On the same configuration screen scroll down to the
**Environment Variables** section.
Click **Add Environment Variable** and add each one:

| Key | Value | Required |
|---|---|---|
| `TELEGRAM_BOT_TOKEN` | Token from BotFather — Step 1 | ✅ Yes |
| `TELEGRAM_OWNER_ID` | Your numeric Telegram user ID — Step 2 | ✅ Yes |
| `ENCRYPTION_KEY` | 64-char hex string — Step 3 | ✅ Yes |
| `SOLANA_RPC_URL` | Your RPC HTTP URL — Step 4 | ✅ Yes |
| `SOLANA_WS_URL` | Your RPC WebSocket URL — Step 4 | ✅ Yes |
| `DATABASE_PATH` | `/var/data/bot.db` | ✅ Yes |
| `HELIUS_API_KEY` | Your Helius key (token age checks) | Optional |
| `BIRDEYE_API_KEY` | BirdEye key (enhanced price data) | Optional |

---

### STEP 10 — Add a Persistent Disk

Without a persistent disk your SQLite database resets on every redeploy,
wiping all wallets, trade history, and settings.

Still on the same configuration screen, scroll to the **Disks** section:

1. Click **Add Disk**
2. Fill in:
   - **Name**: `solbot-data`
   - **Mount Path**: `/var/data`
   - **Size**: `1 GB` (sufficient for years of data)
3. Confirm the `DATABASE_PATH` environment variable above is set to
   `/var/data/bot.db` — it must match this mount path exactly

> This disk survives all redeploys and restarts. Your wallets, trade
> history, blacklists, and all settings are permanently preserved.

---

### STEP 11 — Deploy

1. Click **Create Background Worker** at the bottom of the page
2. Render starts building — watch the progress in the **Logs** tab
3. A successful deployment looks like this in the logs:

```
==> Installing dependencies with pip...
Successfully installed python-telegram-bot solana solders aiosqlite ...

==> Starting service with 'python main.py'
Starting SolBot v2.0...
Database initialised.
Bot command menu registered.
Position monitor started.
Token scanner started.
Alert monitor started.
Scheduler started. Daily report at 08:00 UTC.
SolBot ready.
```

4. Open **Telegram** — the bot sends you this startup message automatically:

```
🤖 SolBot v2.0 is online
Use /status for dashboard or /help for commands.
⚠️ Bot is STOPPED by default. Send /run to activate trading.
```

---

### STEP 12 — First-Time Bot Setup in Telegram

```
# 1. Create a dedicated trading wallet
/createwallet main

#    The bot shows you:
#      • Public address  → send SOL here to fund the bot
#      • Private key     → shown ONCE, save it immediately and securely

# 2. Set your trading parameters
/setprofit 2.0          ← sell at 2× (100% gain)
/setposition 5          ← risk 5% of balance per trade
/setslippage 500        ← allow up to 5% slippage
/setmaxloss 2.0         ← halt if down 2 SOL today
/setdailytrades 10      ← maximum 10 trades per day
/setcooldown 30         ← 30-second gap between trades

# 3. Always test with paper trading before using real money
/paper                  ← enables paper mode (no real SOL spent)
/run                    ← starts the bot

# 4. Check everything looks correct
/status

# 5. Add a signal channel to monitor for contract addresses
/addchannel @channelname

# 6. Optionally enable the automatic new token scanner
/scanner                ← toggles scanner ON/OFF

# 7. Once satisfied with paper results, disable paper mode
/paper                  ← toggles paper mode OFF (now using real SOL)
```

---

## Updating the Bot

Render auto-deploys on every GitHub push.

```bash
# Make your changes, then:
git add .
git commit -m "Update settings"
git push origin main
# Render detects the push and redeploys within ~60 seconds
# Your persistent disk (wallets, history, settings) is untouched
```

---

## Monitoring

**From Telegram:**

| Command | What it shows |
|---|---|
| `/status` | Full live dashboard |
| `/positions` | Open positions with live P&L |
| `/logs` | Last 20 audit log lines |
| `/history` | Recent closed trades |
| `/pnl` | All-time profit & loss |

**From Render Dashboard:**
- Service → **Logs** tab — real-time application output
- Service → **Metrics** tab — CPU and memory usage
- Render retains 7 days of logs on Starter tier

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| Bot not responding in Telegram | Wrong token or service not running | Check Render Logs for errors |
| `⛔ Unauthorised` on every command | Wrong `TELEGRAM_OWNER_ID` | Confirm exact numeric ID via @userinfobot |
| Service keeps restarting | Crash on startup | Check Render Logs — usually a missing env var |
| Trades fail: insufficient balance | Wallet not funded | Send SOL to address in `/balance` |
| "No Jupiter route found" | Token has no liquidity | Check token on dexscreener.com |
| Database resets on each redeploy | No persistent disk | Complete Step 10 — add the disk |
| Build fails | Missing file or bad dependency | Confirm `requirements.txt` is at repo root |
| Bot goes offline randomly | Free instance type selected | Upgrade to Starter ($7/mo) in Render settings |
| `ENCRYPTION_KEY` error on startup | Key is wrong length | Must be exactly 64 hex chars (32 bytes) |

---

## Safety Features

| Feature | What It Does |
|---|---|
| Honeypot detection | Simulates a full buy + sell route before committing any SOL |
| Freeze authority check | Rejects tokens where the issuer can freeze your token account |
| Mint authority check | Warns when token supply can still be inflated by the issuer |
| Liquidity validation | Rejects tokens with no viable Jupiter swap route |
| Top holder concentration check | Warns when top 5 wallets hold over 60% of supply |
| Transaction simulation | Every swap is dry-run on-chain before broadcasting |
| Daily loss cap | Auto-halts all trading when your loss limit is reached |
| Cooldown timer | Enforces a gap between trades to prevent rapid-fire losses |
| Duplicate signal prevention | Will not open a second position on a token already held |
| Emergency kill switch | `/kill` force-closes all positions immediately |
| Auto-blacklist | Blacklists tokens that triggered a stop-loss automatically |
| Paper trading mode | Complete trade simulation — zero real SOL at risk |

---

## Default Trading Parameters

| Parameter | Default | Command |
|---|---|---|
| Slippage | 500 bps (5%) | `/setslippage <bps>` |
| Position size | 5% of balance | `/setposition <pct>` |
| Take-profit | 2× (100% gain) | `/setprofit <multiplier>` |
| Trailing stop | Disabled (0) | `/settrailing <pct>` |
| Cooldown | 30 seconds | `/setcooldown <sec>` |
| Max daily trades | 20 | `/setdailytrades <n>` |
| Daily loss cap | 2.0 SOL | `/setmaxloss <sol>` |
| Min pool liquidity | $5,000 USD | `/setminliq <usd>` |
| Min token age | 5 minutes | `/setminage <hours>` |
| Trading window | 00:00–23:59 UTC | `/setwindow <HH:MM> <HH:MM>` |

---

## Environment Variables Reference

```env
# ── Required ──────────────────────────────────────────────────────────────────
TELEGRAM_BOT_TOKEN=      # From @BotFather
TELEGRAM_OWNER_ID=       # Your numeric Telegram user ID
ENCRYPTION_KEY=          # 64-char hex string
SOLANA_RPC_URL=          # Solana RPC HTTP endpoint
SOLANA_WS_URL=           # Solana RPC WebSocket endpoint
DATABASE_PATH=           # /var/data/bot.db  — must match disk mount path

# ── Optional but Recommended ──────────────────────────────────────────────────
HELIUS_API_KEY=          # Enables token age checks and richer metadata
BIRDEYE_API_KEY=         # Enhanced price feeds

# ── Optional Overrides ────────────────────────────────────────────────────────
JUPITER_API_URL=         # Defaults to https://quote-api.jup.ag/v6
```

---

## Important Disclaimer

Automated cryptocurrency trading involves **substantial risk of financial loss**.
Degen and meme coin trading is extremely high risk — tokens can lose all value
in seconds. This software:

- Provides no guarantee of profit or positive returns
- Is provided as-is under the MIT License
- Does not constitute financial or investment advice
- Is used entirely at your own risk

Always start with paper trading mode (`/paper`) before committing real funds.
Never trade more than you can afford to lose completely.
