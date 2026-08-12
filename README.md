# Multi-Market Regime-Aware Trading Intelligence Platform
*(working title — rename freely)*

[![Status](https://img.shields.io/badge/status-in--development-yellow)]()
[![License](https://img.shields.io/badge/license-MIT-green)]()

A trading research platform that predicts short-term market direction and manages a simulated portfolio across four market categories — Indian equities, crypto, forex, and global (USA/Japan/China/Europe) equities — using one fused, explainable decision engine instead of a pile of disconnected models.

This started as a submission for a hackathon challenge ("AI-Powered Algorithmic Trading & Delta Optimization"). It's now being built as its own project regardless of that outcome — the architecture and goals below are ours, not scoped to a competition brief.

This document is the shared knowledge base for the group. Everyone building any component should read this before writing code — it defines what data looks like at every handoff point, so multiple people's work fits together instead of needing to be glued at the end.

> **Disclaimer:** This is a research/education tool. It surfaces analytics and recommendations, not personalized investment advice, and does not auto-execute trades without explicit confirmation. Not registered with SEBI or any regulator. Not intended for live capital deployment as-is.

---

## Table of contents

- [The core idea](#the-core-idea)
- [Architecture](#architecture)
- [The data contract](#the-data-contract)
- [Fusion neural network](#fusion-neural-network)
- [Multi-market data](#multi-market-data)
- [Components](#components)
- [What's original here](#whats-original-here)
- [Tech stack](#tech-stack)
- [Repo structure](#repo-structure)
- [Team](#team)
- [License](#license)

---

## The core idea

Most trading projects show a pile of independent models — a forecast here, a sentiment score there — with no story connecting them. Ours fuses everything into one neural network that decides, per market, whether the near-term move is up, flat, or down, and how confident it is.

```
 Market data (multi-market)   News headlines
      │                             │
      ▼                             ▼
┌───────────┐┌──────────┐┌────────┐┌───────────┐
│Forecasting││Volatility││ Regime ││ Sentiment │
│  (ARIMA)  ││ (GARCH)  ││ (HMM)  ││ (FinGPT)  │
└─────┬─────┘└────┬─────┘└───┬────┘└─────┬─────┘
      └────────────┴─────┬────┴───────────┘
                          ▼
           ┌───────────────────────────┐
           │  Fusion Neural Network      │  ◄── our original piece
           │  Buy / Hold / Sell + conf.  │
           └──────────────┬─────────────┘
                           ▼
           ┌───────────────────────────┐
           │  Strategy & risk layer      │
           │  sizing · stops · regime    │
           │  caps                       │
           └──────────────┬─────────────┘
              ┌────────────┴────────────┐
              ▼                          ▼
   ┌─────────────────┐        ┌───────────────────┐
   │ Backtest engine   │        │ Live execution     │
   │ (historical)       │        │ (broker/exchange)  │
   └─────────┬─────────┘        └─────────┬─────────┘
              └────────────┬─────────────┘
                            ▼
              ┌───────────────────────┐
              │ Portfolio & risk        │
              │ analytics + dashboard   │
              └───────────────────────┘
```

## Architecture

Two rules govern everything, because this is a multi-person build across four very different markets:

1. **Single source of truth for raw data, per market.** Every model reads from the same raw OHLCV/news pull for a given market. Pulling data independently, or at different times, means the fusion network and the backtest will silently disagree about what actually happened on a given day.
2. **Everyone's output speaks the same language.** Whoever builds forecasting, regime detection, volatility, or sentiment never hands teammates a custom format — everyone writes to the schema below. The fusion network never needs to know or care whose model produced a given signal.

## The data contract

Any model (forecasting, regime, volatility, sentiment) outputs a record in this shape:

| Field | Type | Description |
|---|---|---|
| `ticker` | string | Instrument the signal applies to |
| `market` | enum | `india` \| `crypto` \| `forex` \| `global_usa` \| `global_japan` \| `global_china` \| `global_europe` |
| `timestamp` | date | As-of date, always using data available up to `t-1` — never leak future information |
| `signal_type` | enum | `forecast` \| `regime` \| `volatility` \| `sentiment` |
| `value` | float | Signal's core output |
| `confidence` | float [0,1] | Model's own confidence |
| `metadata` | dict | Signal-specific extras |

The fusion network consumes a batch of these per ticker per day (plus raw technical features) and outputs:

| Field | Type | Description |
|---|---|---|
| `class` | enum | `buy` (high) \| `hold` (flat) \| `sell` (low) |
| `confidence` | float [0,1] | Softmax probability of the predicted class |
| `market` | enum | Which market/calibration this prediction used |

## Fusion neural network

The centerpiece. A PyTorch MLP that takes engineered features plus every signal model's output and predicts direction — replacing a hand-coded regime→weight table with a learned fusion function.

**Inputs (~18 dimensions, standardized per-asset):**

| Group | Features |
|---|---|
| Volatility | Realized volatility, implied volatility (VIX/India VIX where available, else realized-vol proxy) |
| Price action | SMA, EMA, RSI, Bollinger Bands, VWAP, OHLCV, ATR |
| Context | Timeframe, market tag |
| Model outputs | Forecasted price (with confidence), predicted volatility, detected regime, analyzed news sentiment |

**Output:** 3-class softmax — `Buy` (high) / `Hold` (flat) / `Sell` (low) — with the softmax probability doubling as the confidence score. This is simultaneously the price-prediction and signal-generation deliverable.

**Two architectures being built and benchmarked against each other:**

| | Shared trunk + per-market heads | Fully separate per-market networks |
|---|---|---|
| Structure | One shared trunk (Dense 64→32) trained on pooled data from all markets, feeding four small per-market heads | Four fully independent networks, no shared weights at all |
| Hypothesis | Universal patterns (e.g. "vol spike → crisis regime") get learned once from more data; each head only specializes on genuinely market-specific decision boundaries | Each market's dynamics are different enough that full specialization outperforms any sharing |
| Status | To be benchmarked head-to-head once training data is ready | |

Training uses a chronological split (never shuffled across time) and walk-forward-computed features, so no model ever sees information from its own future during training.

## Multi-market data

Four required categories, ~152 symbols total, one primary source (`yfinance`) covering nearly all of it for historical/pre-event training:

| Category | Count | Notes |
|---|---|---|
| India | 50 stocks + index + India VIX | Nifty 50-style universe across banking, IT, energy, auto, pharma, FMCG, metals |
| Crypto | 22 coins | Majors, smart-contract platforms, higher-beta alts |
| Forex | 18 pairs | Majors, minors, and USD crosses against INR/CNY |
| Global — USA | 20 stocks + 3 indices + VIX | |
| Global — Japan | 10 stocks + Nikkei | |
| Global — China | 9 tickers (direct A-shares + ETF proxies) + Hang Seng | Direct A-shares can be unreliable on `yfinance`; FXI/ASHR/KWEB/MCHI are the fallback |
| Global — Europe | 12 stocks + 3 indices | UK, France, Germany, Switzerland |

See `data_pipeline/config.py` for the exact ticker lists and `data_pipeline/fetch_data.py` for the pull script.

**Normalization rules, non-negotiable:**
- Convert everything to log returns before it touches any model — never feed raw price levels.
- Z-score every feature against that asset's own trailing mean/std, never a global scaler across markets — crypto's volatility scale and a blue-chip's are not comparable in absolute terms.
- Resample each market to its own local trading calendar (fixed session for equities, 24/7 for crypto, 24/5 for forex) rather than forcing one global calendar.
- Use relative volume (today's volume ÷ trailing average), never absolute volume.
- Settle all portfolio-level P&L in one currency (USD is the natural choice, given crypto and USA markets are already USD-denominated) before computing portfolio-wide Delta/Sharpe/CAGR.
- Forex has no reliable volume data — the volume feature is dropped or zero-filled for FX rows, and the model is trained to handle that gracefully.

## Components

| Component | Method | Produces |
|---|---|---|
| Multi-market data pipeline | `yfinance`, per-market raw pull | Raw OHLCV, all four categories |
| Forecasting | ARIMA/SARIMA, pooled | `forecast` signal |
| Volatility | GARCH(1,1)/EGARCH | `volatility` signal |
| Regime detection | HMM, 3 states (trending / range-bound / crisis) | `regime` signal |
| Sentiment | Pretrained FinGPT sentiment model (swap-in, no training needed) | `sentiment` signal |
| Fusion neural network | PyTorch, shared-trunk and separate-network variants | Buy/Hold/Sell + confidence |
| Strategy & risk layer | Confidence-weighted sizing, volatility targeting, fractional Kelly, ATR-based stops, regime exposure caps | Target portfolio weights |
| Backtest engine | Vectorized, pluggable strategy interface, walk-forward | Performance report |
| Live execution | Broker/exchange API per market, manual confirm gate | Execution log |
| Portfolio & risk analytics | Delta, returns, Sharpe, CAGR, Max Drawdown | Report + dashboard feed |
| Dashboard | Per-model pages + unified decision view | — |

## What's original here

- **The fusion neural network** — learns the signal-weighting logic directly from data instead of a hand-coded regime→weight table, while still taking the regime/vol/sentiment context as explicit inputs so its behavior stays traceable.
- **Shared-trunk-vs-separate-networks benchmark** — rather than assuming one architecture is right, we're building and comparing both, which also produces a genuinely interesting ablation result on its own.
- **True cross-market generalization** — one architecture, calibrated per market, validated against four structurally different asset classes (fixed-session equities, 24/7 crypto, 24/5 forex, multiple currencies and timezones) rather than tuned on a single market and assumed to transfer.
- **Explainability as design principle, not an afterthought** — every prediction traces back to a specific regime, sentiment score, and confidence value, which also happens to keep the system clear of stricter regulatory treatment reserved for opaque ("black-box") algo strategies.

## Tech stack

| Layer | Choice |
|---|---|
| Data | `yfinance` (all four market categories), news API for headlines |
| Forecasting | `statsmodels` / `pmdarima` |
| Volatility | `arch` (GARCH/EGARCH) |
| Regime | `hmmlearn` |
| Sentiment | Pretrained FinGPT sentiment model (HuggingFace) |
| Fusion network | PyTorch |
| Backtest | custom vectorized engine |
| Backend | FastAPI |
| Frontend | Streamlit |
| Storage | SQLite |
| Live execution | Kite Connect / Upstox (India), Alpaca (USA), Binance WebSocket (crypto), OANDA (forex) |

## Repo structure

```
├── data_pipeline/
│   ├── config.py              # ticker universe, all four market categories
│   └── fetch_data.py          # historical OHLCV pull → data/raw/
├── data/
│   ├── raw/                   # single source of truth, untouched pulls, per market
│   └── processed/             # per-model feature sets, all derived from raw/
├── models/
│   ├── forecasting.py
│   ├── volatility.py
│   ├── regime.py
│   └── sentiment.py
├── fusion/
│   ├── shared_trunk_net.py    # shared trunk + per-market heads
│   └── separate_nets.py       # fully independent per-market networks
├── strategy/
│   └── risk_layer.py
├── backtest/
│   └── engine.py
├── execution/
│   └── live_gate.py
├── portfolio/
│   └── analytics.py
├── dashboard/
│   └── app.py
├── PLAN.md
└── README.md
```

## Team

See the tracker and handoff map in [`PLAN.md`](./PLAN.md) for component ownership, current status, and who delivers what to whom.

## License

MIT — for research/educational use. Not licensed or intended for live capital deployment as-is.# Trading_Neural_Network
