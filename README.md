---

# YQTS — Yan Quant Trade Strategy

A quantitative trading system for XAU/USD
built on Bayesian regime detection and
Monte Carlo risk validation.

## Overview

YQTS combines offline changepoint detection
(PELT) with online Bayesian inference (BOCPD)
to identify market regimes in real time,
then executes probabilistic ICT-based strategies
with multi-layer risk management.

Monte Carlo simulation (N=10,000) validates
system robustness before any live deployment.

## Architecture
MT5 Data Feed (XAUUSD M15)
│
▼
┌───────────────────┐
│  Regime Detection  │  PELT (offline, weekly)
│                   │  BOCPD (online, per candle)
└────────┬──────────┘
│ STABLE / TRANSITION / CHANGEPOINT
▼
┌───────────────────┐
│  Signal Generator  │  Order Block · FVG · Sweep
│                   │  Regime-aware direction bias
└────────┬──────────┘
│ Signal + win_rate + bep_rate
▼
┌───────────────────┐
│  Trade Executor   │  5-layer risk management
│                   │  BOCPD-adjusted lot sizing
└────────┬──────────┘
│
▼
┌───────────────────┐
│ Position Manager  │  Trailing SL · BEP trigger
│                   │  Fixed RR 1:5
└───────────────────┘

## Monte Carlo Results

Validated on N=10,000 simulations,
T=2,112 candles (22 trading days), XAU/USD M15.

| Metric | Result | Target | Status |
|---|---|---|---|
| P(Profit) | 80.4% | ≥ 65% | ✅ PASS |
| P(Ruin) | 0.0% | ≤ 10% | ✅ PASS |
| Mean Return | +42.1% | ≥ 20% | ✅ PASS |
| 5th Percentile | -4.0% | ≥ -30% | ✅ PASS |
| VaR (95%) | 39.6 cent | ≤ 200 | ✅ PASS |
| P(MaxDD > 20%) | 0.0% | ≤ 15% | ✅ PASS |

### Stress Tests

| Scenario | Result | Mean Return |
|---|---|---|
| Black Swan | ✅ PASS | 36.8% |
| High Regime Frequency | ✅ PASS | 39.9% |
| High Volatility | ✅ PASS | 39.6% |
| Low Win Rate (−10pp) | ⚠️ FAIL | 14.8% |
| High Transaction Cost | ✅ PASS | 35.6% |

P(ruin) = 0.0% across all 5 stress scenarios.

## Project Structure
YQTS/
├── src/
│   ├── utils/
│   │   └── math_utils.py        # GMM, ES, VaR, RS-GBM
│   ├── market/
│   │   ├── data_loader.py       # MT5 API, multi-timeframe
│   │   ├── parameter_estimator.py
│   │   ├── regime_generator.py  # Markov Chain
│   │   └── price_generator.py   # RS-GBM vectorized
│   ├── regime/
│   │   ├── pelt.py              # Offline changepoint detection
│   │   └── bocpd.py             # Online Bayesian detection
│   ├── strategy/
│   │   ├── order_block.py
│   │   ├── fair_value_gap.py
│   │   └── liquidity_sweep.py
│   └── engine/
│       ├── state.py             # YQTSState dataclass
│       ├── signal_generator.py
│       ├── trade_executor.py
│       └── position_manager.py
├── tests/                       # 206 tests, 0 failed
├── config/
│   └── yqts_config.yaml
├── data/
│   └── raw/                     # MT5 historical data (gitignored)
└── notebooks/
    └── 05_monte_carlo_results.ipynb

## Key Design Decisions

**Regime Detection — PELT + BOCPD Hybrid**
PELT calibrates prior parameters weekly from
4.2 years of M15 data. BOCPD updates posteriors
online per candle with λ=431 (empirically
calibrated from average segment duration).

**RS-GBM with Ito Correction**
Price paths use exact formula from brief §6.2:
S(t+Δt) = S(t)·exp[(μ_ρ − σ_ρ²/2)·Δt + σ_ρ·√Δt·ε]
The −σ²/2 term prevents upward bias in
log-normal simulation.

**Probabilistic Trade Outcomes**
Monte Carlo uses pre-determined outcomes
sampled at trade entry (WIN/BEP/LOSS per §6.4),
consistent with regime-specific win rates
from brief §7.7.

**Multi-Layer Risk Management**
Layer 1: Per trade    — 2% balance risk
Layer 2: Daily        — 5% loss limit
Layer 3: Weekly       — 10% drawdown limit
Layer 4: Kill switch  — 20% total drawdown
Layer 5: Circuit break— 2 consecutive losses

## Data Source

Live and historical data via MetaTrader5
Python API. XAUUSD M15, 4.2 years available
from broker (50,001 candles after gap removal).

No dependency on yfinance or external APIs.

## Requirements
Python        3.11+
numpy         1.26+
scipy         1.11+
pandas        2.1+
ruptures      1.1.9+
scikit-learn  1.3+
MetaTrader5   5.0.45+
joblib        1.3+
numba         0.58+
pyyaml        6.0+

## Test Coverage
Sprint 1 — math_utils    :  42 tests
Sprint 2 — regime        :  57 tests
Sprint 3 — market        :  36 tests
Sprint 4 — engine        :  71 tests
Total                    : 206 tests
Status                   : 0 failed

## Status

- [x] Mathematical foundation verified
- [x] MT5 data pipeline (4.2 yr XAUUSD M15)
- [x] PELT + BOCPD regime detection
- [x] RS-GBM market generator
- [x] YQTS trading engine
- [x] Monte Carlo N=10,000 validated
- [ ] Live trader bridge (Sprint 7)
- [ ] Walk-forward backtesting (v2)
- [ ] Multi-asset extension (NASDAQ)

---

*Built with a $10 cent account in mind.
Validated before deployed.*

---