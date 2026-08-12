# Build Plan & Project Tracker

This is the group's single source of truth for scope, ownership, status, and who hands off what to whom. Update the tracker tables directly as tasks move — don't let status live only in chat.

No fixed deadline drives this anymore — it's being built as its own project. Phases are ordered by dependency, not by day count; work through them in order, and don't start a phase whose inputs aren't ready yet.

---

## Table of contents

- [Status legend](#status-legend)
- [Phase overview](#phase-overview)
- [Handoff map](#handoff-map)
- [Phase 1 — Multi-market data pipeline](#phase-1--multi-market-data-pipeline)
- [Phase 2 — Signal models](#phase-2--signal-models)
- [Phase 3 — Fusion neural network](#phase-3--fusion-neural-network)
- [Phase 4 — Strategy & risk layer](#phase-4--strategy--risk-layer)
- [Phase 5 — Backtest engine](#phase-5--backtest-engine)
- [Phase 6 — Live execution](#phase-6--live-execution)
- [Phase 7 — Portfolio analytics & dashboard](#phase-7--portfolio-analytics--dashboard)
- [Team assignment](#team-assignment)
- [Risk register](#risk-register)
- [Open decisions](#open-decisions)

---

## Status legend

| Symbol | Meaning |
|---|---|
| 🔴 | Not started |
| 🟡 | In progress |
| 🟢 | Done |
| ⚪ | Blocked — see notes |

Priority: **P0** = blocks everything downstream · **P1** = important, not blocking · **P2** = nice to have

---

## Phase overview

| Phase | Focus | Status |
|---|---|---|
| 1 | Multi-market data pipeline | 🟡 In progress — config + fetch script built and tested |
| 2 | Signal models (forecasting, volatility, regime, sentiment) | 🔴 Not started |
| 3 | Fusion neural network (shared-trunk vs. separate benchmark) | 🔴 Not started |
| 4 | Strategy & risk layer | 🔴 Not started |
| 5 | Backtest engine | 🔴 Not started |
| 6 | Live execution | 🔴 Not started |
| 7 | Portfolio analytics & dashboard | 🔴 Not started |

---

## Handoff map

| From (owner) | Delivers | Format | To (owner) |
|---|---|---|---|
| Data pipeline owner | Raw OHLCV, all four market categories | `data/raw/<market>/<ticker>.csv` | Forecasting, volatility, regime owners |
| Data pipeline owner | Raw news headlines | Ticker-tagged headline set | Sentiment owner |
| Forecasting owner | Forecast signal | Schema record, `signal_type: forecast` | Fusion network owner |
| Volatility owner | Vol forecast | Schema record, `signal_type: volatility` | Regime owner, fusion network owner, strategy layer owner |
| Regime owner | Regime label + probability | Schema record, `signal_type: regime` | Fusion network owner |
| Sentiment owner | Sentiment score | Schema record, `signal_type: sentiment` | Fusion network owner |
| Fusion network owner | Buy/Hold/Sell + confidence, per market | Schema record (see README) | Strategy & risk layer owner |
| Strategy layer owner | Target portfolio weights | Weight vector per ticker | Backtest engine, live execution owners |
| Backtest engine owner | Performance report | Standard report shape, reusable for either fusion architecture | Dashboard owner |
| Live execution owner | Executed trades | Trade log | Portfolio analytics owner |
| Portfolio analytics owner | Delta, Sharpe, CAGR, Max Drawdown | Report shape | Dashboard owner |

Anyone changing the shape of what they hand off flags it to the group before merging — a silent schema change upstream breaks everyone downstream without an obvious error.

---

## Phase 1 — Multi-market data pipeline

| ID | Task | Owner | Priority | Status | Notes |
|---|---|---|---|---|---|
| 1.1 | Ticker universe defined across all four categories | | P0 | 🟢 | `data_pipeline/config.py` — ~152 symbols: 50 India, 22 crypto, 18 forex, 51 global (USA/Japan/China/Europe) |
| 1.2 | Historical pull script | | P0 | 🟢 | `data_pipeline/fetch_data.py` — tested, fails gracefully on bad tickers |
| 1.3 | Run the actual pull, populate `data/raw/` | | P0 | 🔴 | Needs to run somewhere with real network access, not the dev sandbox |
| 1.4 | Verify China A-share tickers; fall back to ETF proxies where they fail | | P1 | 🔴 | FXI/ASHR/KWEB/MCHI already in config as backups |
| 1.5 | News headline pull (separate raw source) | | P0 | 🔴 | Needs a news API key — not yet wired up |
| 1.6 | Currency conversion table (settle all P&L in one currency) | | P0 | 🔴 | USD recommended — crypto and USA markets are already USD-denominated |

## Phase 2 — Signal models

| ID | Task | Owner | Priority | Status | Depends on | Notes |
|---|---|---|---|---|---|---|
| 2.1 | Feature engineering — returns, realized vol, RSI, MACD, Bollinger, VWAP, ATR | | P0 | 🔴 | 1.3 | Per-asset z-scoring, never a global scaler |
| 2.2 | Forecasting model (ARIMA/SARIMA), pooled across tickers | | P0 | 🔴 | 2.1 | |
| 2.3 | Volatility model (GARCH/EGARCH) | | P0 | 🔴 | 2.1 | |
| 2.4 | Regime detection (HMM, 3-state) | | P0 | 🔴 | 2.3 | Validate labels visually against known events before trusting them downstream |
| 2.5 | Sentiment — wire up pretrained FinGPT sentiment model | | P0 | 🔴 | 1.5 | No training needed, HuggingFace pull |
| 2.6 | Implied-vol proxy for markets without a free VIX-equivalent | | P1 | 🔴 | 2.3 | Only India and USA have real implied-vol indices; realized vol fills the gap elsewhere |

**Phase 2 done when:** each model runs independently and emits a valid signal-schema record for every market.

## Phase 3 — Fusion neural network

| ID | Task | Owner | Priority | Status | Depends on | Notes |
|---|---|---|---|---|---|---|
| 3.1 | Build feature vector (~18 dims) from Phase 2 outputs | | P0 | 🔴 | Phase 2 | Chronological split, walk-forward computed — no lookahead |
| 3.2 | Shared-trunk + per-market-heads model | | P0 | 🔴 | 3.1 | Dense 64→32 shared trunk, small head per market |
| 3.3 | Fully separate per-market networks (baseline) | | P0 | 🔴 | 3.1 | Same total data, partitioned per market — fair comparison |
| 3.4 | Training loop, both architectures | | P0 | 🔴 | 3.2, 3.3 | Cross-entropy loss, class-weighted if `hold` dominates |
| 3.5 | Benchmark comparison (accuracy, F1, per-market breakdown) | | P0 | 🔴 | 3.4 | This result decides which architecture ships |

**Phase 3 done when:** both architectures are trained and benchmarked against each other, with a documented winner (or a documented reason to keep both).

## Phase 4 — Strategy & risk layer

| ID | Task | Owner | Priority | Status | Depends on | Notes |
|---|---|---|---|---|---|---|
| 4.1 | Confidence-weighted position sizing | | P0 | 🔴 | Phase 3 | Softmax probability scales size |
| 4.2 | Volatility targeting, calibrated per asset's own trailing vol | | P0 | 🔴 | 2.3 | Fixes the crypto-vs-equity vol-scale mismatch |
| 4.3 | Fractional Kelly sizing | | P1 | 🔴 | Phase 5 (needs backtested win rate) | |
| 4.4 | ATR-based stops (volatility-adjusted, not fixed %) | | P1 | 🔴 | 2.3 | |
| 4.5 | Regime-conditioned exposure caps (crisis-regime circuit breaker) | | P0 | 🔴 | 2.4 | Hard rule, not learned |

## Phase 5 — Backtest engine

| ID | Task | Owner | Priority | Status | Depends on | Notes |
|---|---|---|---|---|---|---|
| 5.1 | Vectorized engine, pluggable `strategy(data, positions) → weights` interface | | P0 | 🔴 | — | Can start in parallel with Phase 2/3 |
| 5.2 | No-look-ahead enforcement (structural) | | P0 | 🔴 | 5.1 | |
| 5.3 | Transaction cost / slippage / bid-ask spread modeling, per market | | P0 | 🔴 | 5.1 | Costs differ meaningfully by asset class — don't use one global assumption |
| 5.4 | Portfolio accounting, multi-currency aware | | P0 | 🔴 | 5.1, 1.6 | |
| 5.5 | Performance report: Delta, Sharpe, CAGR, Max Drawdown, win rate | | P0 | 🔴 | 5.4 | Confirm the exact Delta definition being targeted before building this |
| 5.6 | Run: buy-and-hold baseline per market | | P0 | 🔴 | 5.1–5.5 | |
| 5.7 | Run: shared-trunk fusion network strategy | | P0 | 🔴 | Phase 3, 5.1–5.5 | |
| 5.8 | Run: separate-networks fusion strategy | | P0 | 🔴 | Phase 3, 5.1–5.5 | Same report shape, compare against 5.7 |
| 5.9 | Two-period robustness check (calm vs. volatile period, per market) | | P1 | 🔴 | 5.7, 5.8 | Strongest proof-of-value result |

## Phase 6 — Live execution

| ID | Task | Owner | Priority | Status | Notes |
|---|---|---|---|---|---|
| 6.1 | India — Kite Connect / Upstox integration | | P1 | 🔴 | Read-only sync first |
| 6.2 | USA — Alpaca integration | | P1 | 🔴 | Paper trading |
| 6.3 | Crypto — Binance WebSocket integration | | P1 | 🔴 | True tick-by-tick |
| 6.4 | Forex — OANDA integration | | P2 | 🔴 | |
| 6.5 | Manual confirm-before-execute gate | | P0 | 🔴 | No auto-trading without explicit approval |
| 6.6 | Live feed adapter matches the backtest engine's `strategy()` interface exactly | | P0 | 🔴 | This is what makes swapping to live low-risk |

## Phase 7 — Portfolio analytics & dashboard

| ID | Task | Owner | Priority | Status | Depends on | Notes |
|---|---|---|---|---|---|---|
| 7.1 | Portfolio Delta / returns / risk metrics module | | P0 | 🔴 | Phase 5 | |
| 7.2 | Dashboard scaffold | | P0 | 🔴 | — | |
| 7.3 | Per-model pages (forecast, vol/regime, sentiment per market) | | P0 | 🔴 | Phase 2 | |
| 7.4 | Unified "fusion network decision" page | | P0 | 🔴 | Phase 3 | |
| 7.5 | Backtest results page — shared-trunk vs. separate comparison front and center | | P0 | 🔴 | Phase 5 | |
| 7.6 | Multi-market view (switch/compare across the four categories) | | P1 | 🔴 | 7.3 | |

---

## Team assignment

| Component | Owner | Status | Contact |
|---|---|---|---|
| Data pipeline | | 🟡 | |
| Forecasting | | 🔴 | |
| Volatility | | 🔴 | |
| Regime detection | | 🔴 | |
| Sentiment | | 🔴 | |
| Fusion network (shared-trunk) | | 🔴 | |
| Fusion network (separate) | | 🔴 | |
| Strategy & risk layer | | 🔴 | |
| Backtest engine | | 🔴 | |
| Live execution | | 🔴 | |
| Portfolio analytics | | 🔴 | |
| Dashboard | | 🔴 | |

---

## Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| China A-share tickers fail on `yfinance` | High | Low | ETF proxies (FXI/ASHR/KWEB/MCHI) already in config as fallback |
| No free implied-vol index for Japan/China/Europe | High | Medium | Realized volatility as documented proxy — accept the approximation, don't fake a number |
| News API historical depth too short for backtesting sentiment | High | Medium | Confirm depth early before building around it |
| Regimes don't align with intuition / too noisy | Medium | High | Validate visually against known events (2020 crash, 2022 selloff) before trusting them downstream |
| Fusion network overfits to the data it's backtested on | Medium | High | Hold out a final untouched period for the robustness check (5.9) |
| Vol-scale mismatch across markets miscalibrates position sizing | Medium | High | Per-asset trailing-vol calibration (4.2), not a global constant — already designed for, watch it in implementation |
| Schema mismatches between team members' models | High | Medium | Enforce the shared schema strictly — reject any output that doesn't conform |
| "Portfolio Delta" definition ambiguous outside options context | Medium | High | Confirm the exact formula wanted before building 5.5 — flagged as an open decision below |

---

## Open decisions

- [ ] **Exact definition of "Portfolio Delta"** for this system (it's not a standard term outside options trading) — needs to be pinned down before Phase 5.5
- [ ] **Shared-trunk vs. separate networks** — resolved by the Phase 3 benchmark, not by assumption
- [ ] **Settlement currency** — USD recommended, needs to be confirmed and locked before Phase 5.4
- [ ] **Live data feed for Japan/China/Europe** — no confirmed free real-time source yet; may need delayed data or a paid feed