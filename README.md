# BTC 15-Minute Live Trading Bot

Automated trading bot for **Polymarket BTC Up/Down 15-minute binary markets**. The bot monitors real-time market data via WebSocket, calculates technical indicators (VWAP, deviation, momentum, z-score), generates entry signals, executes Fill-And-Kill orders, and optionally hedges positions with Good-Till-Date limit orders on the opposite token.

## What This Bot Does

Every 15 minutes, Polymarket opens a new market asking: "Will BTC go up or down in the next 15 minutes?" Two tokens are available:

- **UP token** pays $1.00 if BTC rises, $0.00 if it falls
- **DOWN token** pays $1.00 if BTC falls, $0.00 if it rises

The bot identifies the "favorite" (the token with higher probability), waits for specific technical conditions to align, then buys it. If the prediction is correct, the token resolves to $1.00 for a profit. If wrong, it resolves to $0.00 for a loss.

### Key Features

- Real-time terminal dashboard with Rich library (order book, indicators, signals, position, P&L)
- VWAP-based signal generation with deviation and momentum filters
- Historical win rate filtering by price range and time bin
- FAK order execution with retry logic and WebSocket fill confirmation
- Optional hedging via GTD orders on the opposite token at $0.02
- Timeout recovery: detects fills via User WebSocket even after network timeouts
- Chainlink BTC/USD oracle tracking: real-time BTC price and deviation from market start
- Auto-redemption of winning positions on-chain
- Telegram notifications with trade alerts and equity charts
- Per-trade drawdown tracking with logging
- Persistent trade history in JSON format (survives restarts)

## Project Structure

```
btc-15m-live/
|-- main.py                 # Main bot: dashboard, signals, execution, all core logic
|-- config.json             # Trading parameters (strategy, entry, hedge, etc.)
|-- .env.example            # Environment variables template (copy to .env)
|-- requirements.txt        # Python dependencies
|-- chart_pnl.py            # P&L chart generator (run separately)
|-- PROJECT_LOGIC.md        # Detailed technical documentation with formulas
|-- data/
|   +-- win_rate.csv        # Historical win rate matrix (10 price ranges x 15 time bins)
+-- src/
    |-- __init__.py
    |-- config_loader.py    # Loads config.json + .env, validates settings
    |-- order_executor.py   # FAK order placement with retry logic
    |-- hedge_manager.py    # GTD hedge order management
    |-- market_finder.py    # Discovers active markets via Gamma API
    |-- position_tracker.py # Position and P&L tracking
    |-- auto_redeemer.py    # On-chain redemption of resolved positions
    |-- telegram_notifier.py# Telegram alerts and chart sending
    |-- user_websocket.py   # User channel WebSocket (order/fill tracking)
    +-- websocket_client.py # Market data WebSocket (prices, trades, book)
```

## Installation (From Scratch on a Clean Machine)

### Prerequisites

- Linux server (Ubuntu 22.04+ recommended) or macOS
- Python 3.11+
- Polymarket account with funded USDC balance (on Polygon), POL for gas fees, and API credentials
- Private key of your trading wallet

### Step 1: System Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y python3 python3-pip python3-venv git
python3 --version
```

### Step 2: Clone the Repository

```bash
cd ~
git clone https://github.com/txbabaxyz/btc-15m-live.git
cd btc-15m-live
```

### Step 3: Create Virtual Environment

```bash
python3 -m venv venv
source venv/bin/activate
```

### Step 4: Install Dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

### Step 5: Configure Environment Variables

```bash
cp .env.example .env
nano .env
```

Fill in your credentials:

| Variable | Required | Description |
|---|---|---|
| PRIVATE_KEY | Yes | Polygon wallet private key (0x...) |
| FUNDER_ADDRESS | If proxy | Gnosis Safe address (if using proxy wallet) |
| SIGNATURE_TYPE | If proxy | 0=EOA, 1=Poly Proxy, 2=Gnosis Safe |
| POLY_API_KEY | Yes | Polymarket CLOB API key |
| POLY_API_SECRET | Yes | Polymarket CLOB API secret |
| POLY_API_PASSPHRASE | Yes | Polymarket CLOB API passphrase |
| RPC_URL | Recommended | Alchemy/Infura Polygon RPC (default: public RPC) |
| TELEGRAM_BOT_TOKEN | Optional | Telegram bot token from @BotFather |
| TELEGRAM_CHAT_ID | Optional | Your Telegram user/chat ID |

**How to get Polymarket API credentials:**
1. Go to https://polymarket.com and connect your wallet
2. Navigate to your account settings
3. Generate API credentials (key, secret, passphrase)
4. These are used for L2 authentication on the CLOB

### Step 6: Configure Trading Parameters

```bash
nano config.json
```

See the Configuration section below for parameter descriptions.

### Step 7: Create Logs Directory

```bash
mkdir -p logs
```

### Step 8: Run the Bot

```bash
source venv/bin/activate
python3 main.py
```

### Step 9: Run in Background (Production)

```bash
sudo apt install -y tmux
tmux new -s bot

# Inside tmux:
source venv/bin/activate
python3 main.py

# Detach: Ctrl+B then D
# Reattach: tmux attach -t bot
```

## Configuration

The bot is **highly configurable** -- every aspect of the strategy, risk management, execution, hedging, and notifications can be fine-tuned through `config.json` without touching any code. You can adjust the entry window, price filters, indicator sensitivity, bet sizing, and more to match your risk tolerance and trading style.

**For a complete parameter-by-parameter guide with explanations, examples, and ready-made presets (Conservative / Moderate / Aggressive), see [CONFIG.md](CONFIG.md).**

Quick overview of the most important settings:

| Parameter | Default | What it controls |
|---|---|---|
| `strategy.min_price` | 0.75 | Minimum token price to enter (lower = riskier, more profit) |
| `strategy.max_price` | 0.88 | Maximum token price to enter (higher = safer, less profit) |
| `strategy.min_elapsed_sec` | 530 | Wait this many seconds before entering |
| `strategy.min_deviation_pct` | 3 | Minimum VWAP deviation to trigger signal |
| `strategy.no_entry_before_end_sec` | 335 | Stop entering with this many seconds left |
| `entry.bet_amount_usd` | 5 | USD per trade (start small!) |
| `entry.max_entry_price` | 0.88 | Hard price ceiling for safety |
| `hedge.enabled` | false | Automatic hedging on opposite token |
| `telegram.enabled` | false | Trade notifications via Telegram |

## How the Strategy Works

### Signal Generation

The bot evaluates 5 conditions every 250ms. ALL must be true to trigger a BUY:

1. **Price in range**: min_price <= favorite_price <= max_price
2. **Time elapsed**: elapsed_seconds >= min_elapsed_sec
3. **VWAP deviation**: min_deviation_pct < deviation < max_deviation_pct
4. **Positive momentum**: momentum > 0%
5. **Time remaining**: seconds_left > no_entry_before_end_sec

### Indicators

- **VWAP** (Volume-Weighted Average Price): SUM(price * volume) / SUM(volume) over the last N seconds
- **Deviation**: (last_price - VWAP) / VWAP * 100% -- how far price moved from its average
- **Momentum**: (price_now - price_Ns_ago) / price_Ns_ago * 100% -- direction of price movement
- **Z-Score**: (price - mean) / stdev over the last 5 seconds -- statistical outlier detection

### Execution Flow

```
Signal detected
  -> FAK order placed
  -> Fill confirmed via WebSocket
  -> Position recorded
  -> Hedge placed (if enabled)
  -> Drawdown tracked every 250ms
  -> Market ends (10s before expiry)
  -> Position resolved, P&L recorded
  -> Winning positions auto-redeemed on-chain
```

### Risk

Higher entry prices mean higher risk. The break-even win rate equals the entry price:

- Entry at $0.75 needs 75% win rate to break even
- Entry at $0.85 needs 85% win rate to break even
- Entry at $0.88 needs 88% win rate to break even

Start with small bet_amount_usd ($1-5) until you understand the behavior.

## Logs

The bot creates a logs/ directory with:

| File | Description |
|---|---|
| bot.log | Main application log (connections, errors, BTC price ticks) |
| signals.log | Full indicator snapshot at each trade entry and market end |
| orders.log | Detailed order execution log (prices, retries, fills) |
| hedges.log | Hedge order placement and fill tracking |
| trading_log.json | Persistent trade history (survives restarts) |

## Generating Charts

After accumulating trades, generate a P&L chart:

```bash
source venv/bin/activate
python3 chart_pnl.py
# Output: logs/pnl_chart.png
```

## Documentation

For a deep technical dive including all formulas, architecture diagrams, and the complete signal generation logic, see [PROJECT_LOGIC.md](PROJECT_LOGIC.md).

## Disclaimer

This software is provided for educational and research purposes only. Trading on prediction markets involves significant financial risk. You may lose all funds used for trading. The authors are not responsible for any financial losses. Always start with small amounts and understand the risks before increasing position sizes.

## License

MIT
