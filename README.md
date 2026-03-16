<div align="center">
  <h1>📈 YQTS (Yan Quant Trade Strategy)</h1>
  <p><strong>A quantitative trading system for XAU/USD built on Bayesian regime detection and Monte Carlo risk validation.</strong></p>
  <p><em>"Built with a $10 cent account in mind. Validated before deployed."</em></p>

  [![Python 3.11+](https://img.shields.io/badge/python-3.11+-blue.svg)](https://www.python.org/downloads/)
  [![MetaTrader 5](https://img.shields.io/badge/MetaTrader-5-orange.svg)](https://www.metatrader5.com/)
  [![Status: Monte Carlo Validated](https://img.shields.io/badge/Status-MC_Validated-success.svg)](#)
</div>

<hr />

## 📖 Overview

Quantitative trading system yang menggabungkan Bayesian regime detection (PELT + BOCPD) dengan ICT-based strategies (Order Block, FVG, Liquidity Sweep) dan Monte Carlo risk validation ($N=10,000$).

### 🛠️ Technical Stack
- **Regime Detection**: PELT (offline, weekly) + BOCPD (online, per candle)
- **Price Simulation**: RS-GBM (Regime-Switching GBM)
- **Strategies**: Order Block, Fair Value Gap, Liquidity Sweep
- **Data Source**: MT5 Python API (4.2 tahun XAUUSD M15, 50,001 candles)
- **Validation**: Monte Carlo N=10,000
- **Hazard $\lambda$**: 431 *(empirically calibrated from average segment duration)*

---

## 🧮 Mathematical Foundation

YQTS is built on rigorous mathematical and statistical models.

### 1. RS-GBM (Regime-Switching Geometric Brownian Motion)
$$S(t+\Delta t) = S(t) \cdot \exp \left[ \left( \mu_{\rho(t)} - \frac{\sigma_{\rho(t)}^2}{2} \right) \Delta t + \sigma_{\rho(t)} \cdot \sqrt{\Delta t} \cdot \varepsilon \right]$$
where:
* $\varepsilon \sim \mathcal{N}(0,1)$
* $\rho(t) =$ regime index dari BOCPD posterior

> **Note**: The $-\sigma^2/2$ term (Itô correction) prevents upward bias in log-normal simulation.

### 2. BOCPD Hazard Function
$$H(r) = \frac{1}{\lambda}$$
* $\lambda = 431$ candle M15 (empirically calibrated)

### 3. GMM Variance (Law of Total Variance)
$$\text{Var}[X] = \sum_{k} \pi_k \sigma_k^2 \Delta t + \sum_{k} \pi_k \mu_{\text{eff}, k}^2 - \left( \sum_{k} \pi_k \mu_{\text{eff}, k} \right)^2$$
*(within-regime + between-regime)*

### 4. Expected Shortfall
$$\text{ES}(t) = \sum_{k} \pi_k(t) \left[ \mu_{\text{eff}, k} - 2.063 \cdot \sigma_k \cdot \sqrt{\Delta t} \right]$$

### 5. Trade Expectancy
$$\mathbb{E}[\text{PnL}] = P_{\text{win}} \times 100 + P_{\text{bep}} \times 0 + P_{\text{loss}} \times (-20)$$
* Breakeven WR $= \frac{1}{1+\text{RR}} = \frac{1}{5+1} \approx 16.7\%$

### 6. Adaptive PELT Penalty
$$\beta_{\text{adaptive}} = \beta_{\text{base}} \times \left( \frac{\sigma_{\text{current}}}{\sigma_{\text{historical}}} \right)$$

---

## 🎯 Monte Carlo Results

Validated on **N=10,000 simulations**, **T=2,112 candles (22 trading days)** on XAU/USD M15.

### Base Run Results
| Metric | Result | Target | Status |
| :--- | :--- | :--- | :--- |
| **P(Profit)** | 80.4% | $\ge 65\%$ | ✅ PASS |
| **P(Ruin)** | 0.0% | $\le 10\%$ | ✅ PASS |
| **Mean Return** | +42.1% | $\ge 20\%$ | ✅ PASS |
| **5th Percentile** | -4.0% | $\ge -30\%$ | ✅ PASS |
| **VaR (95%)** | 39.6¢ | $\le 200¢$ | ✅ PASS |
| **P(MaxDD > 20%)**| 0.0% | $\le 15\%$ | ✅ PASS |

### Stress Tests
| Scenario | Result | Mean Return |
| :--- | :--- | :--- |
| Black Swan | ✅ PASS | 36.8% |
| High Regime Frequency | ✅ PASS | 39.9% |
| High Volatility | ✅ PASS | 39.6% |
| Low Win Rate (−10pp) | ⚠️ FAIL | 14.8% *(expected)* |
| High Transaction Cost ×2| ✅ PASS | 35.6% |

> **Note**: $P(\text{ruin}) = 0.0\%$ di **SEMUA 5 skenario stress**.

---

## 🛡️ Risk Management (5-Layer Protection)

1. **Layer 1 (Per trade)**: 2% balance risk (RR 1:5)
2. **Layer 2 (Daily)**: 5% loss limit
3. **Layer 3 (Weekly)**: 10% drawdown limit
4. **Layer 4 (Kill switch)**: 20% total drawdown
5. **Layer 5 (Circuit break)**: 2 consecutive losses → stop trading

### BOCPD Position Sizing
* **STABLE** → 100% position size
* **TRANSITION** → 50% position size
* **CHANGEPOINT** → NO ENTRY

---

## 📊 Regime Parameters

Kalibrasi PELT dari **4.2 tahun** data XAUUSD M15:

* **`trending_up`** : $\mu = +0.00016$, $\sigma = 0.00138$
* **`ranging`** : $\mu = +0.000003$, $\sigma = 0.00100$
* **`trending_down`**: $\mu = -0.00081$, $\sigma = 0.00444$

**Transition Matrix** (Laplace smoothed $\varepsilon=0.05$):
* `up` $\to$ `rng`: 60.8% | `up`: 25.5% | `dn`: 13.7%
* `rng` $\to$ `up`: 37.4% | `rng`: 54.6% | `dn`: 8.0%
* `dn` $\to$ `rng`: 78.9% | `up`: 16.8% | `dn`: 4.3%

---

## 📁 Project Structure

```text
YQTS/
├── src/
│   ├── utils/
│   │   └── math_utils.py          # GMM, ES, VaR, RS-GBM
│   ├── market/
│   │   ├── data_loader.py         # MT5 API, multi-timeframe
│   │   ├── parameter_estimator.py
│   │   ├── regime_generator.py    # Markov Chain
│   │   └── price_generator.py     # RS-GBM vectorized
│   ├── regime/
│   │   ├── pelt.py                # Offline changepoint detection
│   │   └── bocpd.py               # Online Bayesian (λ=431)
│   ├── strategy/
│   │   ├── order_block.py
│   │   ├── fair_value_gap.py
│   │   └── liquidity_sweep.py
│   └── engine/
│       ├── state.py               # YQTSState dataclass
│       ├── signal_generator.py
│       ├── trade_executor.py
│       └── position_manager.py
├── tests/                         # 206 tests, 0 failed
├── config/
│   └── yqts_config.yaml
├── data/
│   └── raw/                       # MT5 historical data (gitignored)
└── notebooks/
    └── 05_monte_carlo_results.ipynb
```

---

## ⚙️ Requirements

* Python 3.11+
* `numpy` 1.26+
* `scipy` 1.11+
* `pandas` 2.1+
* `ruptures` 1.1.9+
* `scikit-learn` 1.3+
* `MetaTrader5` 5.0.45+
* `joblib` 1.3+
* `numba` 0.58+
* `pyyaml` 6.0+

---

## 🚀 Project Status

* [x] Mathematical foundation verified
* [x] MT5 data pipeline (4.2 yr XAUUSD M15)
* [x] PELT + BOCPD regime detection
* [x] RS-GBM market generator
* [x] YQTS trading engine (5-layer risk)
* [x] Monte Carlo N=10,000 (6/6 PASS)
* [ ] Live trader bridge (Sprint 7)
* [ ] Walk-forward backtesting (v2)
* [ ] Multi-asset extension (NASDAQ) (v2)

---

## 🧪 Test Coverage

| Module | Tests | Status |
| :--- | :--- | :--- |
| **Sprint 1** (`math_utils`) | 42 | ✅ PASS |
| **Sprint 2** (`regime`) | 57 | ✅ PASS |
| **Sprint 3** (`market`) | 36 | ✅ PASS |
| **Sprint 4** (`engine`) | 71 | ✅ PASS |
| **Total** | **206** | **0 failed** |