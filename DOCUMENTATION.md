# BrainNest Trading Platform Pro — Documentation

> Not financial advice. For research, education, and paper trading only.

---

## Table of Contents

1. [What Is This App?](#1-what-is-this-app)
2. [Project Structure](#2-project-structure)
3. [Architecture Flow](#3-architecture-flow)
4. [Setup](#4-setup)
5. [API Keys](#5-api-keys)
6. [Running the App](#6-running-the-app)
7. [Data Sources](#7-data-sources)
8. [11 Tabs Guide](#8-11-tabs-guide)
9. [Scoring Logic](#9-scoring-logic)
10. [Prediction Accuracy Tracker](#10-prediction-accuracy-tracker)
11. [Live Trading](#11-live-trading)
12. [Slack Bot](#12-slack-bot)
13. [Automation — Auto Trader](#13-automation--auto-trader)
14. [Earnings Engine](#14-earnings-engine)

---

## 1. What Is This App?

BrainNest Trading Platform Pro is a premium Streamlit-based stock research and paper trading dashboard. It pulls live market data from 6 providers (FMP, yfinance, Finnhub, Marketaux, NewsAPI, Polygon) with an optional MCP Live Service for fast local caching, runs technical and fundamental analysis, detects chart patterns, generates buy/sell/hold recommendations with confidence scores via an 8-dimension V4 scoring engine, tracks prediction accuracy over time, and supports paper trading or optional live order routing via Alpaca.

---

## 2. Project Structure

```
BrainNest-AI/
├── app.py                          # Streamlit entry point (11 tabs)
├── requirements.txt                # Python dependencies
├── .env                            # API keys (never commit)
├── start_app.sh                    # Auto-start script (cron, 7:50 AM Mon-Fri)
├── stop_app.sh                     # Auto-stop script (cron, 8:00 PM Mon-Fri)
│
├── myedge_trading_platform/        # Core package
│   ├── v4_engine/                  # V4 Signal Engine (scoring & decisions)
│   │   ├── recommender.py            # 8-dimension scoring, safety rules, No Trade
│   │   ├── patterns.py               # 61 candlestick + 7 chart patterns
│   │   ├── macro.py                  # Macro environment (VIX, Treasury, SPY regime)
│   │   └── sentiment.py              # Multi-source news sentiment
│   │
│   ├── data_providers/             # External API integrations
│   │   ├── fmp_data.py               # Financial Modeling Prep API
│   │   ├── market_data.py            # Finnhub, Marketaux, NewsAPI, Polygon
│   │   └── mcp_bundle.py            # MCP Live Service (localhost:8585)
│   │
│   ├── analysis/                   # Analysis & research
│   │   ├── technical.py              # Indicators, Plotly charts, Fibonacci, S/R
│   │   ├── fundamentals.py           # FMP → yfinance fallback, earnings
│   │   └── scanner.py               # Multi-timeframe batch scanner
│   │
│   ├── trading/                    # Order execution & portfolio
│   │   ├── broker.py                 # Order routing + safety checks
│   │   ├── broker_integrations.py    # Alpaca REST API
│   │   ├── backtest.py               # vectorbt / pandas backtester
│   │   └── journal.py               # Trade journal metrics
│   │
│   ├── config.py                   # Paths, constants, .env loader
│   ├── storage.py                  # JSON persistence
│   ├── data.py                     # yfinance fetching, TickerBundle
│   ├── api_tracker.py              # API usage counter
│   ├── universe.py                 # Preset ticker lists
│   ├── ui.py                       # All UI components
│   └── reporting.py                # PDF reports + CSV download
│
├── automation/                     # Auto trader bot
│   ├── brainnest_auto_trader.py      # Main orchestrator
│   ├── mcp_adapter.py               # MCP data layer
│   ├── mcp_warmup.py                # Pre-cache 100 stocks
│   ├── strategy_selector.py         # Market regime → strategy
│   ├── weight_calibrator.py         # Scoring weight feedback loop
│   ├── breakout_detector.py         # Intraday breakout detection
│   ├── email_service.py             # Branded HTML emails
│   ├── slack_service.py             # Slack alerts
│   ├── trade_journal.py             # SQLite persistence
│   ├── config.py                    # Automation settings
│   └── scheduler.sh                 # Cron wrapper
│
├── communication/                  # 2-way Slack bot
│   ├── slack_bot.py                  # Socket Mode bot (slash commands)
│   └── start_bot.sh                  # Manual start script
│
├── earnings_engine/                # Earnings calendar collector
│   ├── earnings_job.py              # Collection job
│   ├── db.py                        # SQLite + get_reporting_symbols()
│   └── scheduler.sh                 # Cron wrapper
│
├── assets/                         # Logo images
│   ├── BrainNest-icon.png
│   └── BrainNest-logo.png
│
├── docs/                           # Documentation
│   ├── DOCUMENTATION.md              # Full user guide
│   ├── ARCHITECTURE.md               # System architecture diagrams
│   └── BrainNest_Project_Improvement_Report.md
│
├── tests/                          # Test suite
└── .streamlit/config.toml          # Streamlit dark theme
```

---

## 3. Architecture Flow

### Page Layout
```
┌─────────────────────────────────────────────────────────────┐
│              SCROLLING TICKER TAPE (fixed top)               │
│  AAPL $208 ▲+1.2%  NVDA $208 ▲+4.3%  SPY $714 ▲+0.5% ... │
└─────────────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  🧠 BrainNest  ·  Trading Platform Pro  [Live] [Pro] [v3]  │
└─────────────────────────────────────────────────────────────┘
┌──────────┐  ┌───────────────────────────────────────────────┐
│ SIDEBAR  │  │  [Overview] [Charts] [Fundamentals] ...       │
│          │  │                                               │
│ 🧠 Logo  │  │  ┌─────────────────┐  ┌──────────────────┐   │
│ Tickers  │  │  │ Research Summary │  │ Score Breakdown   │   │
│ Settings │  │  │ Signal + Drivers │  │ Analyst Consensus │   │
│ Buttons  │  │  │ Fundamentals    │  │ Volatility Gauge  │   │
│          │  │  └─────────────────┘  │ Sector Comparison │   │
└──────────┘  │                       │ Sentiment + Risk  │   │
              │                       └──────────────────┘   │
              │  ┌──────────────────────────────────────────┐ │
              │  │       Prediction Accuracy Tracker         │ │
              │  └──────────────────────────────────────────┘ │
              └───────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────────────┐
│  ● System Online · 2026-04-25 · BrainNest v3.0             │
└─────────────────────────────────────────────────────────────┘
```

### Data Pipeline
```
User selects ticker
        │
        ├──► MCP Service ──► (if available) Full data bundle from localhost:8585
        │         │
        │    (fallback if MCP unavailable)
        │         │
        ├──► yfinance ──► Price history, earnings, quarterly, info, options, insider
        ├──► FMP ────────► Annual financials, metrics, profile
        ├──► Finnhub ───► Real-time quote, company news
        ├──► Marketaux ─► ML-based entity-level sentiment scores
        ├──► NewsAPI ───► Broader news headlines
        └──► Polygon ───► Quote fallback
                │
                ▼
    ┌───────────────────────┐
    │   Analysis Pipeline   │
    │                       │
    │  build_technical()    │──► EMA, RSI, MACD, ADX, ATR, VWAP, Fib, S/R
    │  build_fundamental()  │──► Revenue/EPS CAGR, margins, ROE, D/E, P/E
    │  analyze_patterns()   │──► Candlestick + chart patterns
    │  fetch_sentiment()    │──► TextBlob polarity on news headlines
    │  get_overview_extras()│──► Analyst consensus, volatility, events
    │  get_sector_compare() │──► Stock vs sector ETF vs SPY
    │  get_rel_strength()   │──► Outperformance vs SPY
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │   Scoring Engine V4   │
    │                       │
    │  Technical:   0-100   │  (EMA, RSI, MACD, ADX, VWAP, ATR, crossovers)
    │  Volume:      0-100   │  (RVOL, spikes, price-volume divergence)
    │  Pattern:     0-100   │  (Bullish/bearish count, chart patterns)
    │  Options:     0-100   │  (Put/call ratio, OI analysis)
    │  Market:      0-100   │  (VIX, SPY, Treasury regime)
    │  Sentiment:   0-100   │  (News headline scoring)
    │  Fundamental: 0-100   │  (Revenue, EPS, margins, ROE, D/E, FCF, PEG)
    │  Insider/Analyst: 0-100│ (Insider buys, analyst consensus)
    │                       │
    │  Composite = weighted │  (T×0.30 + V×0.25 + P×0.15 + O×0.10 +
    │                       │   M×0.10 + S×0.05 + F×0.05) + momentum bonus
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │   Output              │
    │                       │
    │  ≥78 → Strong Buy ✨  │  (blinking green badge)
    │  ≥63 → Buy            │
    │  ≥40 → Hold 🟡        │  (blinking orange badge)
    │  ≥25 → Sell           │
    │  <25 → Strong Sell 🔴 │  (blinking red badge)
    │                       │
    │  + Risk Plan (entry,  │
    │    stop, TP1/2/3)     │
    │  + Accuracy logged    │
    └───────────────────────┘
```

---

## 4. Setup

```bash
# 1. Create virtual environment
python3.12 -m venv venv
source venv/bin/activate

# 2. Install dependencies
pip install --upgrade pip setuptools wheel
pip install -r requirements.txt

# 3. Download TextBlob data
python -m textblob.download_corpora

# 4. Add API keys to .env (see Section 4)

# 5. Run
streamlit run app.py
```

---

## 5. API Keys

Add to `.env` in the project root. All keys are optional — app falls back to yfinance.

```env
# Recommended for best accuracy
FMP_API_KEY=your_key              # https://financialmodelingprep.com
FINNHUB_API_KEY=your_key          # https://finnhub.io

# Optional enhancements
MARKETAUX_API_KEY=your_key         # https://marketaux.com (ML sentiment)
NEWS_API_KEY=your_key              # https://newsapi.org
POLYGON_API_KEY=your_key           # https://polygon.io

# Order routing (optional)
ALPACA_API_KEY=your_key
ALPACA_SECRET_KEY=your_secret
ALPACA_BASE_URL=https://paper-api.alpaca.markets

# Slack Bot (2-way communication)
SLACK_BOT_TOKEN=xoxb-your-token
SLACK_APP_TOKEN=xapp-your-token

# Slack Webhooks (1-way alerts from automation)
SLACK_WEBHOOK_URL=https://hooks.slack.com/...
SLACK_WEBHOOK_SIGNALS=https://hooks.slack.com/...
```

---

## 6. Running the App

```bash
source venv/bin/activate
streamlit run app.py
# Opens at http://localhost:8501
```

Stop: `Ctrl+C` · Change port: `streamlit run app.py --server.port 9502`

Data auto-refreshes every 15 minutes. Click "Refresh Data" for manual refresh.

---

## 6b. Supported Markets & Ticker Format

BrainNest works with any stock available on Yahoo Finance. Use the correct ticker suffix for non-US markets.

### US Stocks (no suffix needed)
```
AAPL, MSFT, NVDA, TSLA, AMZN, META, GOOGL
```

### Indian Stocks (NSE / BSE)
Indian stocks require a `.NS` (NSE) or `.BO` (BSE) suffix:
```
NSE:  RELIANCE.NS, TCS.NS, INFY.NS, HDFCBANK.NS, DMART.NS, TATAMOTORS.NS
BSE:  RELIANCE.BO, TCS.BO, INFY.BO, HDFCBANK.BO
```

Common Indian tickers:
| Company | NSE Ticker | BSE Ticker |
|---|---|---|
| Reliance Industries | RELIANCE.NS | RELIANCE.BO |
| TCS | TCS.NS | TCS.BO |
| Infosys | INFY.NS | INFY.BO |
| HDFC Bank | HDFCBANK.NS | HDFCBANK.BO |
| DMart (Avenue Supermarts) | DMART.NS | DMART.BO |
| ICICI Bank | ICICIBANK.NS | ICICIBANK.BO |
| SBI | SBIN.NS | SBIN.BO |
| Tata Motors | TATAMOTORS.NS | TATAMOTORS.BO |
| Bajaj Finance | BAJFINANCE.NS | BAJFINANCE.BO |
| Asian Paints | ASIANPAINT.NS | ASIANPAINT.BO |

### Other Markets
| Market | Suffix | Example |
|---|---|---|
| London (LSE) | .L | HSBA.L, BP.L |
| Tokyo (TSE) | .T | 7203.T (Toyota) |
| Hong Kong | .HK | 0700.HK (Tencent) |
| Toronto (TSX) | .TO | SHOP.TO |
| Australia (ASX) | .AX | BHP.AX |
| Germany (XETRA) | .DE | SAP.DE |

If you enter an Indian stock name without the suffix (e.g., `DMART` instead of `DMART.NS`), the app will show a helpful error suggesting the correct format.

### Universe Presets
The sidebar includes preset watchlists:
- NASDAQ AI Leaders — top US tech/AI stocks
- NYSE Liquid Leaders — high-volume NYSE stocks
- NSE Large Caps — top Indian large caps (pre-configured with .NS)
- US Momentum Basket — high-momentum US stocks
- Dynamic Top 10 — curated top performers across sectors

---

## 7. Data Sources

```
┌──────────────────┬──────────────────────────────────────────┐
│ Feature          │ Priority Order                           │
├──────────────────┼──────────────────────────────────────────┤
│ All data (auto)  │ MCP (localhost:8585) → individual APIs   │
│ Fundamentals     │ FMP (annual) → yfinance (fallback)       │
│ Quarterly Data   │ yfinance (always)                        │
│ Earnings Summary │ yfinance earnings_dates                  │
│ News Sentiment   │ Finnhub → Marketaux → NewsAPI → yfinance│
│ Real-time Quote  │ Finnhub → Polygon → yfinance            │
│ Price History    │ yfinance (always)                        │
│ Analyst Data     │ yfinance info                            │
│ Sector Compare   │ yfinance download                        │
│ Rel Strength     │ yfinance download (vs SPY)               │
│ Options Flow     │ yfinance option_chain                    │
│ Insider Activity │ yfinance insider_purchases               │
│ Macro Environ.   │ MCP /macro → yfinance (^VIX, ^TNX, etc)│
│ Order Execution  │ Alpaca (paper or live)                   │
└──────────────────┴──────────────────────────────────────────┘
```

| Provider | Data | Free Tier |
|---|---|---|
| MCP Service | All data cached locally (localhost:8585) | Self-hosted |
| FMP | Financial statements, metrics, profile | 250 req/day |
| yfinance | Prices, earnings, quarterly, options, insider data | Always free |
| Finnhub | Real-time quotes, company news | 60 calls/min |
| Marketaux | Financial news with ML sentiment scores | 100 req/day |
| NewsAPI | Broader news headlines | 100 req/day |
| Polygon | Real-time/delayed quotes | 15-min delay |
| Alpaca | Paper + live order execution | Free paper |

---

## 8. 11 Tabs Guide

### 1. Overview
The main dashboard — clean summary of the stock analysis:
- Header metrics: Ticker (green blink badge), Price, Volume, Signal (animated), Confidence
- Safety warnings banner (if any blocking/caution rules triggered)
- Research Summary: recommendation, confidence, score pills, driver explanations
- Fundamental Summary: color-coded metrics table
- Score Breakdown: 8-dimension circular progress rings + gradient bars
- Wall Street Consensus: analyst rating, price targets, upside %
- Volatility & Risk: gauge bar, 30D/90D vol, ATR%, Beta
- Sector Comparison: stock vs sector ETF vs SPY (1W/1M/3M/6M)
- Key Events: next earnings date, ex-dividend
- News Sentiment: label, score gauge, source badge
- Pattern Heat: bullish/bearish/neutral counts
- Risk Plan: entry, stop loss, TP1/2/3 with R:R ratios
- Prediction Accuracy Tracker: historical accuracy + confidence calibration

### 2. Signal Engine (V4)
The full decision pipeline — all V4 intelligence in one place:
- V4 Final Decision card: APPROVED/BLOCKED with signal, confidence, regime, risk level, trade bias, position size, R:R, execution mode
- Safety Alerts: all triggered blocking/caution rules listed
- Daily Protection + Options Flow (side by side)
- Volume Intelligence + Insider Activity (side by side)
- Macro Environment: VIX, Treasury, SPY, QQQ with EMA-based regime
- Risk Plan card

### 3. Charts
- Interactive candlestick with EMA 9/20/50, SMA 200 (color-coded)
- Toggle Fibonacci & S/R levels
- Earnings History strip (beat/miss)
- Quick Stats (52W high/low, market cap, P/E, EPS, revenue, etc.)
- Key Price Levels (all levels with distance % and above/below)
- Momentum Dashboard (RSI, MACD, Stochastic)
- Relative Strength vs SPY
- Annual Revenue & Net Income (dual Y-axis)
- Quarterly Revenue & EPS (dual Y-axis)

### 4. Fundamentals
- Color-coded metrics table
- Strengths/weaknesses summary
- Last 6 quarters earnings (beat/miss table)
- Financial statement expanders (annual + quarterly)

### 5. Technical
- Latest signal table
- Support/resistance/fibonacci levels
- Volume profile

### 6. Patterns
- Candlestick pattern detection table
- Chart pattern detection table

### 7. Recommendations
- Signal badge with animation
- Score rings + gradient bars (8 dimensions)
- Research summary with driver explanations
- Risk plan card

### 8. Portfolio
- Paper trading positions
- Order log

### 9. Backtest
- Equity curve chart
- Trade metrics (return, drawdown, win rate, profit factor)

### 10. Sentiment
- News headlines with sentiment scores
- Source indicator (finnhub/marketaux/newsapi/yfinance)

### 11. Journal
- Order analytics
- Status distribution chart

---

## 9. Scoring Logic (Signal Engine V4)

```
8-Dimension Weighted Scoring:

Technical Score (0-100) × 30%
  + Price above EMA 20/50, SMA 200
  + RSI momentum zone
  + MACD above signal, crossovers
  + ADX trend strength, VWAP, ATR

Volume Score (0-100) × 25%
  + RVOL (relative volume vs 20-day avg)
  + Volume spike detection (>2x)
  + Price-volume divergence

Pattern Score (0-100) × 15%
  + 6 points per bullish pattern
  - 6 points per bearish pattern
  + Chart pattern signal

Options Score (0-100) × 10%
  + Put/Call ratio analysis
  + Call vs put volume comparison

Market Score (0-100) × 10%
  + VIX level and trend
  + S&P 500 monthly trend
  + Treasury yield direction

Sentiment Score (0-100) × 5%
  + Finnhub/Marketaux/NewsAPI headline scoring
  + Positive/negative/neutral classification

Fundamental Score (0-100) × 5%
  + Revenue/EPS CAGR
  + Margins, ROE, D/E, FCF, PEG

Insider/Analyst Score (0-100) × 0% (disabled by default, available for calibration)
  + Insider buying/selling activity
  + Analyst consensus and price targets

Composite = weighted sum + momentum bonus

Signal Labels:
  ≥78 → Strong Buy (blinking green)
  ≥63 → Buy
  ≥40 → Hold (blinking orange)
  ≥25 → Sell
  <25 → Strong Sell (blinking red)

No Trade Override (white dashed badge):
  - Earnings within 2 days
  - SPY trend opposite to signal (>3% monthly)
  - Confidence below 40%
  - Volume extremely low (RVOL < 0.5)
  - Choppy market (sideways + high VIX)
  - Reward/risk ratio below 1.5:1
  - Daily trade limit reached
  - 2 consecutive losses today

Market Regime Classification:
  Strong Bullish / Bullish / Sideways / Bearish / Strong Bearish
  + Volatility: Low / Normal / High / Extreme

Strategy Selector (automation):
  Based on market regime, selects:
  MOMENTUM / MEAN_REVERT / DEFENSIVE / CASH
  Adapts min confidence, position size, TP targets, sector preferences

Weight Calibrator (automation):
  After 20+ closed trades, analyzes which dimensions predicted
  winners vs losers and recalibrates weights automatically.
  Saves to automation/learned_weights.json.
```

---

## 10. Prediction Accuracy Tracker

Every recommendation is automatically logged with:
- Timestamp, ticker, signal label, confidence, entry price

After 7 days: fetches actual price, calculates if prediction was correct
After 30 days: same check for 1-month accuracy

Accuracy rules:
- Buy/Strong Buy → correct if price went up
- Sell/Strong Sell → correct if price went down
- Hold → correct if price stayed within ±5%
- No Trade → not tracked (correctly avoided)

### Confidence Calibration

The tracker buckets past predictions by confidence range and shows actual win rate:

```
Confidence 40-55%  → actual win rate: ??%  (X signals)
Confidence 55-70%  → actual win rate: ??%  (X signals)
Confidence 70-85%  → actual win rate: ??%  (X signals)
Confidence 85-100% → actual win rate: ??%  (X signals)
```

This answers: "When the app says 85%+ confidence, how often is it actually right?"
Requires at least 3 checked signals per bucket. Builds up over 2-4 weeks of use.

---

## 10b. How to Validate New Features

### Validate "No Trade" Intelligence

The app blocks trades (shows "No Trade" white dashed badge) when:
- Earnings reporting TODAY (same day)
- SPY/QQQ trend opposite to signal (>3% monthly)
- Confidence below 45%
- Volume extremely low (RVOL < 0.5)
- Choppy market (sideways + high VIX)
- Reward/risk ratio below 1.5:1
- Daily trade limit reached (3 trades/day)
- 2 consecutive losses today

Earnings within 1-3 days shows CAUTION warning but doesn't block.

To test:
1. Try V or KO (earnings within days) — see CAUTION warning banner
2. During market selloff — Buy signals become "No Trade"
3. Check the Daily Protection card on Overview — shows trades remaining

### Validate Market Regime

Look at the Macro Environment card on the Overview tab:
- It shows the regime label (Bullish/Bearish/Sideways)
- It shows volatility level (Low/Normal/High/Extreme)
- Compare with actual market conditions (check SPY chart, VIX level)

### Validate Confidence Calibration

1. Use the app daily for 2+ weeks, analyzing different tickers
2. After 1 week, the Accuracy Tracker will start showing results
3. The calibration buckets appear once you have 3+ checked signals
4. Compare: does 85%+ confidence actually win more than 55-70%?
5. If not, the scoring weights need tuning

Stats shown on Overview:
- Total tracked, 1W accuracy %, 1M accuracy %
- Confidence calibration buckets (40-55%, 55-70%, 70-85%, 85-100%) showing actual win rate per range
- Recent predictions table with results

Confidence calibration answers: "When the app says 85% confidence, is it actually right 85% of the time?"

---

## 11. Live Trading

Live order routing is OFF by default. To enable:

1. Add Alpaca credentials to `.env`
2. Set `enable_live_routing: true` in `.myedge_data/settings.json`
3. Test in paper mode first (`ALPACA_BASE_URL=https://paper-api.alpaca.markets`)

Safety gates: live routing must be enabled, confidence above threshold (75%), credentials valid.

---

## 12. Slack Bot

BrainNest includes a 2-way Slack bot for real-time communication. Ask it questions from your phone or desktop — no need to open the app.

### Setup
1. Create app at https://api.slack.com/apps
2. Enable Socket Mode → get App Token (`xapp-...`)
3. Add OAuth scopes: `app_mentions:read`, `chat:write`, `im:history`, `im:read`
4. Install to workspace → get Bot Token (`xoxb-...`)
5. Add slash commands: `/bn-stocks`, `/bn-status`, `/bn-pnl`, `/bn-regime`, `/bn-check`, `/bn-logs`
6. Add to `.env`:
   ```
   SLACK_BOT_TOKEN=xoxb-...
   SLACK_APP_TOKEN=xapp-...
   ```

### Slash Commands

| Command | What it does |
|---|---|
| `/bn-stocks` | Today's top 20 scanned stocks with scores |
| `/bn-status` | Open positions + unrealized P&L |
| `/bn-pnl` | Today's realized P&L summary |
| `/bn-regime` | Current market regime + strategy |
| `/bn-check AAPL` | Run full V4 analysis on any stock (buy/sell/hold) |
| `/bn-logs` | Last 5 minutes of auto trader logs |

### How it runs
- Starts/stops with the app (`start_app.sh` / `stop_app.sh`)
- Mon-Fri 7:50 AM → 8:00 PM ET
- Uses Socket Mode (free, no public URL needed)
- Code: `communication/slack_bot.py`

### Alerts (1-way, automation only)
The auto trader also posts to Slack channels via webhooks:
- `SLACK_WEBHOOK_URL` → #brainnest-buys (purchase alerts)
- `SLACK_WEBHOOK_SIGNALS` → #brainnest-trades (all signals)


---

## 13. Automation — Auto Trader

The Auto Trader is a fully automated trading bot that scans stocks, buys approved signals, monitors positions with trailing stops and partial profit taking, and sells based on rules — all without touching the Streamlit UI. Uses MCP Live Service (localhost:8585) for all market data.

### How It Works

```
Every trading day (triggered by cron or manually):

┌─────────────────────────────────────────────────────────────┐
│  0. STRATEGY SELECTION                                      │
│     Reads market regime from MCP /macro endpoint            │
│     Selects: MOMENTUM / MEAN_REVERT / DEFENSIVE / CASH     │
│     Sets min confidence, position size, TP targets          │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  1. SCAN PHASE                                              │
│     GET /top-stocks → 100 curated blue-chip stocks          │
│     Fast 8-rule scan → top 20 candidates                    │
│     GET /stock-data/{symbol} for top 20 → 6-dimension score │
│     Filters: green today + intraday rising + $10B+ cap      │
│     + R/R ≥ 1.5 + not reporting earnings                    │
│     Priority: Mag 7 (+10), Blue chips (+5), high R/R (+3)   │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  2. BUY PHASE                                               │
│     Only APPROVED + Buy/Strong Buy signals pass             │
│     ATR-based position sizing (max $150 risk/trade)         │
│     Max 40 shares per stock, max 25% capital per position   │
│     DRY_RUN=true  → logs what it would buy                  │
│     DRY_RUN=false → submits orders via Alpaca API           │
│     Sends purchase email + Slack alert                      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  3. MONITOR + ADD-ON SCAN (loops every 60 seconds)          │
│     For each open position:                                 │
│       • Trailing stop (1.5% threshold, 30% lock)            │
│       • TP1 → sell 1/3, stop to breakeven                   │
│       • TP2 → sell 1/3, stop to TP1                         │
│       • TP3 → sell remaining 1/3                            │
│       • Stop loss / SPY crash / EOD close                   │
│     Every 15 min: add-on scan for new opportunities         │
│     Real-time SPY crash detection (2%+ drop in 20 min)      │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│  4. DAILY REPORT                                            │
│     After market close:                                     │
│       • Total trades, wins, losses, win rate                │
│       • Total P&L, largest winner/loser                     │
│       • Blocked signals count                               │
│     Sends daily report email + Slack → exits                │
└─────────────────────────────────────────────────────────────┘
```

### Data Source: MCP Live Service

All automation market data comes from a single MCP service at `localhost:8585`:
- `GET /top-stocks?top_n=100` — curated blue-chip stock list
- `GET /stock-data/{symbol}` — full data: price, history, technicals, fundamentals, earnings, analyst, insider, news
- `GET /macro` — market regime (VIX, SPY, regime classification)
- `GET /intraday/{symbol}?interval=5m&bars=20` — real-time 5-min bars from Alpaca
- `POST /warm-up?top_n=100&parallel=3` — pre-cache stocks (runs at 8:00 AM)

Set `USE_MCP_DATA=true` in `.env` to use MCP (default). Set `false` to fall back to yfinance/FMP.

### Files

```
automation/
├── brainnest_auto_trader.py   # Main orchestrator (scan → buy → monitor → report)
├── mcp_adapter.py             # MCP data layer (all market data via localhost:8585)
├── mcp_data.py                # MCP helper utilities
├── mcp_warmup.py              # Pre-cache 100 stocks at 8:00 AM (~35 min)
├── config.py                  # All settings from .env
├── strategy_selector.py       # Market regime → trading strategy
├── weight_calibrator.py       # Feedback loop: recalibrates scoring weights
├── breakout_detector.py       # Breakout pattern detection
├── email_service.py           # Branded HTML emails (SMTP + local fallback)
├── slack_service.py           # Slack alerts (#brainnest-buys, #brainnest-trades)
├── trade_journal.py           # SQLite persistence (signals, trades, daily stats)
├── brainnest_ai_client.py     # AI research client
├── dashboard.py               # Streamlit monitoring dashboard (port 9502)
├── scheduler.sh               # Cron wrapper script
├── tests/test_dry_run.py      # Quick test with dummy data (2 seconds)
├── tests/test_e2e.py          # Full end-to-end test
├── brainnest_trades.db        # SQLite database (auto-created)
├── learned_weights.json       # Calibrated weights (auto-created after 20+ trades)
├── email_logs/                # HTML email backups (if SMTP fails)
└── logs/YYYY-MM-DD/           # Dated log folders (14-day retention)
```

### Configuration (.env)

```env
# ── Trading Mode (SAFETY FIRST) ──
ENABLE_LIVE_TRADING=false       # Must be true + DRY_RUN=false for real orders
DRY_RUN=true                    # true = log only, false = execute real orders

# ── MCP Data Source ──
USE_MCP_DATA=true               # true = all data from MCP, false = yfinance/FMP fallback
BRAINNEST_MCP_URL=http://localhost:8585

# ── Trading Limits ──
MAX_STOCKS_TO_BUY=10            # Max stocks to buy per day
MAX_TRADES_PER_DAY=5            # Daily trade limit
MAX_DAILY_LOSS_PERCENT=2        # Stop trading if daily loss exceeds this %

# ── Position Sizing ──
MAX_SHARES_PER_STOCK=40         # Hard cap per stock
MAX_CAPITAL_PER_STOCK_PCT=25    # Max 25% of capital in one position
ACCOUNT_USAGE_PERCENT=50        # Use 50% of Alpaca balance

# ── Trailing Stop ──
MIN_TRAIL_GAIN_PCT=1.5          # Stock must gain 1.5% before trailing activates
TRAIL_LOCK_PCT=0.30             # Lock in 30% of gain above threshold

# ── Filters ──
MIN_MARKET_CAP_BILLIONS=10      # Skip stocks below $10B

# ── Market Hours ──
MARKET_TIMEZONE=America/New_York
MARKET_OPEN_HOUR=9
MARKET_OPEN_MINUTE=30
MARKET_CLOSE_HOUR=16
MARKET_CLOSE_MINUTE=0
EOD_EXIT_MINUTES_BEFORE=10
CHECK_INTERVAL_SECONDS=60
SCAN_DELAY_AFTER_OPEN_MINUTES=10
SCAN_RETRY_INTERVAL_SECONDS=900

# ── Slack Alerts ──
SLACK_WEBHOOK_URL=https://hooks.slack.com/...       # #brainnest-buys
SLACK_WEBHOOK_SIGNALS=https://hooks.slack.com/...   # #brainnest-trades

# ── Email Alerts ──
ALERT_EMAIL_TO=your_email@gmail.com
ALERT_EMAIL_FROM=sender@gmail.com
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=sender@gmail.com
SMTP_PASSWORD=your_app_password    # Google App Password (not regular password)
```

### Running

```bash
# Activate virtual environment
source venv/bin/activate

# ── Manual run ──

# During market hours (auto-detects open/close):
python -m automation.brainnest_auto_trader

# Anytime (skip market hours check — for testing):
python -m automation.brainnest_auto_trader --force

# ── Quick test with dummy data (no API calls, 2 seconds) ──
python automation/tests/test_dry_run.py

# ── MCP warm-up (pre-cache 100 stocks) ──
python automation/mcp_warmup.py

# ── Monitoring dashboard ──
streamlit run automation/dashboard.py --server.port 9502

# ── Fully automatic via cron (runs every weekday) ──
# Add to crontab (crontab -e):
0 8 * * 1-5   python automation/mcp_warmup.py       # MCP warm-up
35 9 * * 1-5  automation/scheduler.sh                # Auto trader start
```

### Safety Layers

```
Layer 1:  DRY_RUN=true (default) → No orders placed
Layer 2:  ENABLE_LIVE_TRADING=false → Double opt-in for live
Layer 3:  USE_MCP_DATA flag → Easy toggle between data sources
Layer 4:  Market cap filter ($10B+) → Blue chips only
Layer 5:  Green-today filter → Only buy stocks up today
Layer 6:  Intraday momentum → Stock must be rising NOW
Layer 7:  ATR-based sizing → Max $150 risk per trade
Layer 8:  Trailing stop (1.5% threshold, 30% lock)
Layer 9:  Strategy selector → Adapts to market regime
Layer 10: SPY crash detection → Sells all if SPY drops 2%+ in 20 min
Layer 11: Daily loss circuit breaker
Layer 12: No re-buy same stock same day
Layer 13: Earnings engine → Blocks trades on earnings day
```

### Hybrid EOD Logic (hold overnight)

At 3:50 PM, positions are NOT force-closed. Instead:
- **Profitable** → Sell (take the win)
- **Loss < 2.5%** → Hold overnight (give it time to recover)
  - Places a GTC limit sell at TP1 for 1/3 shares (catches morning gap-ups)
- **Loss ≥ 2.5%** → Sell (cut the loser)
- **Held 3+ days** → Sell regardless (don't bag-hold)
- **Stop loss hit** → Sell immediately (any time)

Configurable via `.env`: `MAX_HOLD_DAYS=3`, `EOD_MAX_LOSS_PCT=-2.5`

### Going Live Checklist

1. Test with `DRY_RUN=true` for at least 1-2 weeks, review logs and emails
2. Switch to Alpaca paper trading (`DRY_RUN=false`, keep paper URL) for another week
3. Review win rate, P&L, and blocked signals in daily reports
4. Only then:
   ```env
   DRY_RUN=false
   ENABLE_LIVE_TRADING=true
   ALPACA_BASE_URL=https://api.alpaca.markets   # ← live URL (real money!)
   ```
5. Start with small capital and `MAX_STOCKS_TO_BUY=3`
6. Monitor daily report emails closely for the first week

### Email & Slack Alerts

The bot sends alerts via email and Slack:

1. **Purchase Email/Slack** — sent after buying stocks: symbol, qty, entry price, stop loss, TP targets, confidence, reason
2. **Sell Email/Slack** — sent after each exit: symbol, buy/sell prices, P&L, P&L %, exit reason
3. **Daily Report** — sent at end of day: total trades, wins/losses, win rate, total P&L, largest winner/loser

If SMTP is not configured or fails, emails are saved as HTML files in `automation/email_logs/`.

### Logs and Debugging

```bash
# View today's auto trader log
cat automation/logs/$(date +%Y-%m-%d)/brainnest_auto.log

# Check SQLite database
sqlite3 automation/brainnest_trades.db "SELECT * FROM trades ORDER BY id DESC LIMIT 10;"
sqlite3 automation/brainnest_trades.db "SELECT * FROM signals ORDER BY id DESC LIMIT 10;"
sqlite3 automation/brainnest_trades.db "SELECT * FROM daily_stats ORDER BY date DESC LIMIT 5;"

# Check MCP service health
curl http://localhost:8585/top-stocks?top_n=1
```

---

## 14. Earnings Engine

Standalone earnings calendar collector that feeds the auto trader's "No Trade on earnings day" guard.

### What It Does
- Pulls upcoming earnings events for all tracked symbols
- Uses Finnhub (primary), FMP (optional), yfinance (fallback)
- Deduplicates by symbol + report_date with source priority
- Stores in SQLite: `earnings_engine/earnings_engine.db`
- Exposes `get_reporting_symbols(from_date, to_date)` for the auto trader

### Files

```
earnings_engine/
├── __init__.py
├── config.py              # Earnings-specific config
├── db.py                  # SQLite access + get_reporting_symbols()
├── earnings_job.py        # Main collection job
├── scheduler.sh           # Cron wrapper
├── earnings_engine.db     # SQLite database (auto-created)
└── README.md              # Detailed documentation
```

### Running

```bash
# Current week only
python -m earnings_engine.earnings_job --force

# 2-week look-ahead (recommended)
python -m earnings_engine.earnings_job --force --lookahead 2

# Schedule: every Monday 6:00 AM
0 6 * * 1 /path/to/earnings_engine/scheduler.sh
```

### Integration with Auto Trader

The auto trader calls `get_reporting_symbols()` before buying to block trades on stocks reporting earnings within 3 days.
