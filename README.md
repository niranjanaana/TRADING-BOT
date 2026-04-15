# Binance Futures Testnet — Trading Bot

A Python CLI application for placing **MARKET**, **LIMIT**, and **STOP_MARKET** orders on the Binance USDT-M Futures Testnet. Built with a clean layered architecture, structured rotating-file logging, and full input validation.

---

## Table of Contents

- [Project Structure](#project-structure)
- [Setup](#setup)
- [How to Run](#how-to-run)
- [Example Commands](#example-commands)
- [Sample Output](#sample-output)
- [CLI Reference](#cli-reference)
- [Logging](#logging)
- [Error Handling](#error-handling)
- [Assumptions](#assumptions)

---

## Project Structure

```
trading_bot/
├── bot/
│   ├── __init__.py          # package exports
│   ├── client.py            # Binance REST API wrapper (HMAC signing, HTTP)
│   ├── orders.py            # order placement logic per order type
│   ├── validators.py        # input validation (symbol, side, type, qty, price)
│   └── logging_config.py    # rotating file + console logging setup
├── cli.py                   # CLI entry point (argparse)
├── logs/
│   └── trading_bot.log      # auto-created on first run
├── requirements.txt
└── README.md
```

The code is separated into two clear layers:
- **`bot/`** — core library (API client, order logic, validation, logging). No CLI concerns.
- **`cli.py`** — presentation layer only. Parses arguments, prints formatted output, calls into `bot/`.

---

## Setup

### Prerequisites

- Python 3.8 or higher
- A Binance Futures Testnet account

### 1. Get Testnet API Credentials

1. Go to [https://testnet.binancefuture.com](https://testnet.binancefuture.com)
2. Log in with your GitHub account
3. Click **API Key** in the top navigation
4. Click **Generate Key** — save both the API Key and Secret Key immediately (the secret is shown only once)

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

Only one external dependency: `requests`.

### 3. Set Your Credentials

**Option A — Environment variables (recommended):**

```bash
# macOS / Linux
export BINANCE_API_KEY="your_testnet_api_key"
export BINANCE_API_SECRET="your_testnet_api_secret"

# Windows (Command Prompt)
set BINANCE_API_KEY=your_testnet_api_key
set BINANCE_API_SECRET=your_testnet_api_secret

# Windows (PowerShell)
$env:BINANCE_API_KEY="your_testnet_api_key"
$env:BINANCE_API_SECRET="your_testnet_api_secret"
```

**Option B — Inline flags:**

```bash
python cli.py --api-key YOUR_KEY --api-secret YOUR_SECRET --symbol BTCUSDT ...
```

---

## How to Run

### Basic syntax

```
python cli.py --symbol SYMBOL --side BUY|SELL --type TYPE --quantity QTY [--price PRICE]
```

`--price` is **required** for `LIMIT` and `STOP_MARKET` orders. It is ignored for `MARKET` orders.

---

## Example Commands

### Place a MARKET BUY order

```bash
python cli.py \
  --symbol BTCUSDT \
  --side BUY \
  --type MARKET \
  --quantity 0.001
```

### Place a LIMIT SELL order

```bash
python cli.py \
  --symbol ETHUSDT \
  --side SELL \
  --type LIMIT \
  --quantity 0.01 \
  --price 3200.00
```

### Place a STOP_MARKET BUY order *(bonus order type)*

Triggers a market buy when the price reaches 66000:

```bash
python cli.py \
  --symbol BTCUSDT \
  --side BUY \
  --type STOP_MARKET \
  --quantity 0.001 \
  --price 66000.00
```

### Dry-run — validate inputs without sending the order

No credentials needed. Useful for testing your inputs first:

```bash
python cli.py \
  --symbol BTCUSDT \
  --side BUY \
  --type LIMIT \
  --quantity 0.001 \
  --price 60000 \
  --dry-run
```

### Verbose debug output

Prints full request parameters and raw API responses to the console:

```bash
python cli.py \
  --symbol BTCUSDT \
  --side BUY \
  --type MARKET \
  --quantity 0.001 \
  --log-level DEBUG
```

---

## Sample Output

```
════════════════════════════════════════════════════════
  ORDER REQUEST SUMMARY
════════════════════════════════════════════════════════
  Symbol      BTCUSDT
  Side        BUY
  Type        MARKET
  Quantity    0.001
  Price       —
────────────────────────────────────────────────────────

════════════════════════════════════════════════════════
  ORDER RESPONSE
════════════════════════════════════════════════════════
  Order ID:             4234876321
  Symbol:               BTCUSDT
  Side:                 BUY
  Type:                 MARKET
  Status:               FILLED
  Orig Qty:             0.001
  Executed Qty:         0.001
  Avg Price:            65432.10
────────────────────────────────────────────────────────

  ✓ Order placed successfully!
```

---

## CLI Reference

| Flag | Required | Default | Description |
|---|---|---|---|
| `--symbol` / `-s` | Yes | — | Trading pair, e.g. `BTCUSDT`, `ETHUSDT` |
| `--side` | Yes | — | `BUY` or `SELL` |
| `--type` / `-t` | Yes | — | `MARKET`, `LIMIT`, or `STOP_MARKET` |
| `--quantity` / `-q` | Yes | — | Contract quantity (must be > 0) |
| `--price` / `-p` | Conditional | — | Limit price for `LIMIT`; trigger price for `STOP_MARKET` |
| `--time-in-force` | No | `GTC` | `GTC`, `IOC`, or `FOK` — applies to `LIMIT` orders only |
| `--api-key` | No | env var | Overrides `BINANCE_API_KEY` environment variable |
| `--api-secret` | No | env var | Overrides `BINANCE_API_SECRET` environment variable |
| `--log-level` | No | `INFO` | Console verbosity: `DEBUG`, `INFO`, `WARNING`, `ERROR` |
| `--dry-run` | No | false | Validate and preview without sending the order |

---

## Logging

All activity is logged automatically to `logs/trading_bot.log`.

- The file rotates at **5 MB** and keeps **3 backups**
- The file always captures **DEBUG** level and above (full request params, raw responses)
- The console captures **INFO** and above by default (adjustable with `--log-level`)

Example log entries:

```
2025-06-15T09:12:01 | INFO  | trading_bot.orders | Preparing MARKET order | symbol=BTCUSDT side=BUY qty=0.001
2025-06-15T09:12:01 | INFO  | trading_bot.client | Placing order | symbol=BTCUSDT side=BUY type=MARKET qty=0.001
2025-06-15T09:12:02 | INFO  | trading_bot.client | Order placed | orderId=4234876321 status=FILLED executedQty=0.001 avgPrice=65432.10
2025-06-15T09:12:02 | INFO  | trading_bot.cli    | Order completed successfully
```

---

## Error Handling

The bot handles the following failure scenarios gracefully, printing a clear message and logging the detail:

| Scenario | Example |
|---|---|
| Invalid input | Negative quantity, missing price for LIMIT, unknown order type |
| Missing credentials | No API key/secret provided |
| Binance API error | Invalid symbol (`-1121`), insufficient margin (`-2019`) |
| Network failure | No internet connection, DNS failure |
| Request timeout | Testnet unresponsive |

All errors exit with code `1`. Successful orders exit with code `0`.

**Common Binance error codes:**

| Code | Meaning | Fix |
|---|---|---|
| `-1121` | Invalid symbol | Check spelling — must be e.g. `BTCUSDT` not `BTC-USDT` |
| `-1102` | Missing parameter | Add `--price` for LIMIT orders |
| `-2019` | Insufficient margin | Reset your testnet account balance in the web UI |
| `-1111` | Too many decimal places | Round your quantity or price |
| `-4061` | Hedge mode mismatch | Switch your testnet account to one-way position mode |

---

## Assumptions

1. **USDT-M Futures testnet only.** The base URL is fixed to `https://testnet.binancefuture.com`. To use mainnet, change `TESTNET_BASE_URL` in `bot/client.py`.

2. **One-way position mode.** Orders are placed with `positionSide=BOTH`. If your testnet account is in hedge mode, switch it to one-way mode in the testnet web UI under *Position Mode*.

3. **No `python-binance` library.** The bot uses direct REST calls via `requests`. This keeps dependencies minimal and makes the HMAC signing logic explicit and auditable.

4. **Quantity and price precision.** Binance enforces per-symbol step sizes (e.g. BTCUSDT quantity must be a multiple of 0.001). If you get error `-1111`, reduce the number of decimal places in your quantity or price.

5. **Testnet account resets.** The Binance Futures Testnet resets periodically. If orders fail with `-2019` (insufficient margin), log into the testnet web UI and click *Reset* to restore your test balance.
