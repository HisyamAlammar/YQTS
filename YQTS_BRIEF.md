# YQTS — YAN QUANT TRADE STRATEGY
## Complete Agent Brief untuk Antigravity + Claude Opus 4.6

---

## INSTRUKSI UNTUK AI AGENT

Kamu adalah software engineer dan quant developer yang bertugas membangun sistem **YQTS (Yan Quant Trade Strategy)** secara lengkap. Dokumen ini adalah sumber kebenaran tunggal — baca seluruhnya sebelum menulis satu baris kode. Setiap keputusan arsitektur, matematika, dan parameter sudah ditentukan di sini. Tugasmu adalah mengimplementasikannya dengan benar, bersih, dan teroptimasi.

### Rules of Engagement
1. Baca seluruh brief ini sebelum mulai
2. Implementasi satu komponen selesai sebelum lanjut ke berikutnya
3. Setiap fungsi wajib punya docstring dan unit test minimal
4. Gunakan type hints Python di semua fungsi
5. Jangan ubah arsitektur atau matematika tanpa persetujuan eksplisit
6. Jika ada ambiguitas, tanya sebelum implement
7. Target: production-ready code, bukan prototype

---

## BAGIAN 1 — PROFIL SISTEM

```
Nama Sistem     : YQTS (Yan Quant Trade Strategy)
Tipe            : Quantitative Trading System
Target Aset     : XAU/USD (Gold) — Primary
                  NASDAQ — Secondary (setelah XAU stabil)
Platform Live   : MetaTrader 5 (Akun Cent)
Backend         : Python 3.11+
Data Feed       : MT5 live + yfinance (backtest/simulasi)
Modal Awal      : 1,000 cent (= $10 USD akun cent)
```

---

## BAGIAN 2 — SYSTEM DNA (Trading Rules)

### 2.1 Timeframe Hierarchy
```
H4  → Bias utama + Candle Age Filter
      Kurangi confidence jika candle H4 sudah berjalan > 75% durasi
H1  → Konfirmasi setup + sinyal entry
M15 → Validasi + timing entry presisi (timeframe eksekusi)
```

### 2.2 Trade Execution Rules
```
Max posisi bersamaan  : 2
Max trade per hari    : 2 (profit / loss / BEP = semua dihitung)
Setelah 2 trade       : Sistem LOCK sampai hari berikutnya
Risk Reward           : 1:5 (FIXED — tidak pernah diubah)
SL                    : Fixed dari entry saat posisi dibuka
TP                    : Fixed dari entry saat posisi dibuka
```

### 2.3 Trailing SL Logic
```
Trigger  : Floating profit >= 60% dari jarak TP
Aksi     : Geser SL ke harga entry (Break Even Point)
Post     : Biarkan sampai TP kena atau BEP kena
           TIDAK ada trailing lebih lanjut setelah trigger
```

### 2.4 Trade Outcomes (hanya 3 kemungkinan)
```
WIN      : +100 cent (TP kena)
BREAKEVEN: 0 cent (BEP kena setelah trailing trigger)
LOSS     : -20 cent (SL kena sebelum trailing trigger)
```

---

## BAGIAN 3 — RISK MANAGEMENT (4 Layer)

### Layer 1 — Per Trade
```python
risk_per_trade = 0.02 * balance      # 2% per trade
target_per_trade = 0.10 * balance    # 10% per trade (RR 1:5)
```

### Layer 2 — Daily Loss Limit
```python
daily_loss_limit = 0.05 * initial_balance   # 5% per hari
# Setelah hit: STOP entry baru hari itu
```

### Layer 3 — Weekly Drawdown
```python
weekly_drawdown_limit = 0.10 * initial_balance  # 10% per minggu
# Setelah hit: Pause + review strategi
```

### Layer 4 — Kill Switch (Total Stop)
```python
kill_switch_level = 0.80 * initial_balance  # Drawdown 20%
# Setelah hit: Stop sistem sepenuhnya, review ulang
```

### Circuit Breaker
```python
consecutive_loss_limit = 2  # 2 loss beruntun = stop hari itu
```

---

## BAGIAN 4 — REGIME DETECTION ENGINE

### 4.1 Arsitektur Hybrid
```
PELT  (Offline) → Dijalankan mingguan + event-based
                  Mengkalibrasi parameter λ untuk BOCPD
                  Input: Data historis XAU M15 (minimal 2 tahun)
                  Output: μₖ, σₖ per regime, transition matrix

BOCPD (Online)  → Dijalankan setiap candle M15 baru
                  Deteksi real-time menggunakan λ dari PELT
                  Output: STABLE / TRANSITION / CHANGEPOINT
```

### 4.2 PELT (Pruned Exact Linear Time)
```
Library    : ruptures (pip install ruptures)
Model      : "rbf" atau "normal"
Penalty    : β_adaptive = β_base × (σ_current / σ_historical)
Kompleksitas: O(n log n) average

Rekalibrasi triggered jika:
- λ berubah > 20% dari kalibrasi sebelumnya
- Performance degradation terdeteksi
- Major market event (NFP, FOMC, dll)
```

### 4.3 BOCPD (Bayesian Online Changepoint Detection)
```
Hazard function  : H = 1/λ  (constant hazard)
Prior            : Normal-Inverse-Gamma
Update recursive : P(rₜ|x₁:ₜ) ∝ Σ P(rₜ|rₜ₋₁)·P(xₜ|rₜ₋₁,x^(r))·P(rₜ₋₁|x₁:ₜ₋₁)
Memory           : O(1) dengan truncation max_run_length = 200
Waktu per update : O(1) praktis

BOCPD States:
STABLE      → P(changepoint) < 0.4   → FULL position
TRANSITION  → 0.4 ≤ P < 0.7          → REDUCED 50% position size
CHANGEPOINT → P ≥ 0.7                 → NO ENTRY, tutup posisi
```

### 4.4 Regime Classification
```python
regimes = {
    "trending_up":    {"id": 0, "mu": 0.0008,  "sigma": 0.008},
    "ranging":        {"id": 1, "mu": 0.0001,  "sigma": 0.005},
    "trending_down":  {"id": 2, "mu": -0.0006, "sigma": 0.012}
}
```

---

## BAGIAN 5 — STRATEGI TRADING

### 5.1 Strategy Selector
```
H4 TRENDING + BOCPD STABLE  → Strategi A: Trend Following
H4 RANGING  + BOCPD STABLE  → Strategi B: Mean Reversion
BOCPD TRANSITION/CHANGEPOINT → NO ENTRY (skip semua setup)
```

### 5.2 Strategi A — Order Block (OB)

**Definisi:**
Candle terakhir yang bergerak berlawanan arah sebelum pergerakan impulsif institusi.

**Kriteria Validitas (semua harus terpenuhi):**
1. Candle OB berlawanan arah dengan impuls
2. Impulsive Move: IM = Range_Impuls / ATR(14) > 1.5
3. Break of Structure (BOS) terjadi setelah impuls
4. OB State: FRESH atau TESTED (bukan MITIGATED)

**OB Zone:**
```python
ob_upper = high(ob_candle)
ob_lower = (open(ob_candle) + close(ob_candle)) / 2  # 50% body
```

**OB State Tracker:**
```
FRESH     → Belum pernah dikunjungi     → 100% position
TESTED    → Di-touch sekali, hold       → 50% position
MITIGATED → Ditembus atau di-touch 2×   → SKIP
```

**OB Selection Rule (jika multiple candle):**
1. Last Candle Rule (default)
2. Largest Body Rule (jika candle terakhir doji)
3. Confirmation Rule (low paling dekat titik awal impuls)

**OB Validity Scoring System:**
```python
scoring = {
    "impulsive_move": {
        "IM > 3.0":   3,
        "IM 2.0-3.0": 2,
        "IM 1.5-2.0": 1
    },
    "break_of_structure": {
        "BOS at H1":  3,
        "BOS at M15": 2
    },
    "ob_location": {
        "Fibonacci 0.618/0.786": 3,
        "Previous Week H/L":     3,
        "Previous Day H/L":      2,
        "Swing H/L":             1
    },
    "ob_freshness": {
        "FRESH":  3,
        "TESTED": 2
    },
    "confluence_fvg":   2,  # Ada FVG di dalam OB zone
    "confluence_sweep": 2,  # OB terbentuk setelah sweep
    "bocpd_status": {
        "STABLE":     2,
        "TRANSITION": 1
    }
}
minimum_score = 11
premium_score = 15
maximum_score = 18
```

**Expectancy:**
```
Regular (score 11-14): ~38% win rate, +30 cent/trade
Premium (score >= 15): ~51% win rate, +46 cent/trade
```

**Entry / SL / TP:**
```python
entry = ob_lower               # Batas bawah OB (50% body)
sl    = low(ob_candle) - buffer  # buffer = ATR_M15 * 0.1
tp    = entry + (entry - sl) * 5  # RR 1:5
```

---

### 5.3 Strategi B — Fair Value Gap (FVG)

**Definisi:**
Area harga tanpa transaksi yang terbentuk saat pergerakan impulsif.

**Identifikasi:**
```python
# Bullish FVG
if low(C3) > high(C1):
    fvg_upper = low(C3)
    fvg_lower = high(C1)
    fvg_size  = fvg_upper - fvg_lower

# Bearish FVG
if high(C3) < low(C1):
    fvg_upper = low(C1)
    fvg_lower = high(C3)
    fvg_size  = fvg_upper - fvg_lower
```

**Klasifikasi Ukuran:**
```python
if fvg_size < 0.5 * atr14:    fvg_type = "MICRO"    # SKIP
if fvg_size < 1.5 * atr14:    fvg_type = "REGULAR"  # PRIMARY
else:                          fvg_type = "LARGE"    # Entry di 25% atau SKIP
```

**Klasifikasi Tipe:**
```
BREAKAWAY    → Awal trend baru       → Fill prob ~61%
CONTINUATION → Tengah trend          → Fill prob ~79%  ← PRIMARY YQTS
EXHAUSTION   → Akhir trend           → SKIP
```

**FVG Priority Scoring:**
```python
fvg_scoring = {
    "type":       {"BREAKAWAY": 3, "CONTINUATION": 2, "EXHAUSTION": 0},
    "timeframe":  {"H1": 3, "H4": 2, "M15": 1},
    "direction":  {"with_H4_trend": 3, "counter_trend": 0},
    "size":       {"REGULAR": 2, "LARGE": 1, "MICRO": 0},
    "bocpd":      {"STABLE": 2, "TRANSITION": 1, "CHANGEPOINT": 0},
    "freshness":  {"untouched": 2, "partially_tested": 1}
}
minimum_fvg_score = 10
maximum_fvg_score = 15
```

**Expiry Rule:**
```python
fvg_expiry_candles = 12  # 12 candle H1 = 12 jam
# Jika dalam 12 jam tidak fill → cancel, FVG kadaluarsa
```

**Entry / SL / TP:**
```python
entry = fvg_lower                   # Batas bawah FVG (opsi YQTS)
buffer = max(atr_m15 * 0.15, 2_pips)
sl     = fvg_lower - buffer
tp     = entry + (entry - sl) * 5   # RR 1:5
```

**Expectancy:**
```
Baseline: P(TP)=35%, P(BEP)=18%, P(SL)=47% → +26 cent/trade
Optimis:  P(TP)=44%, P(BEP)=23%, P(SL)=33% → +34 cent/trade
```

---

### 5.4 Strategi C — Liquidity Sweep + Reversal

**Definisi:**
Market maker menyapu stop loss cluster retail untuk mengambil likuiditas, lalu harga berbalik.

**Jenis Liquidity Pool:**
```python
liquidity_pools = [
    "equal_highs_lows",    # Selisih max 5 pip untuk XAU
    "previous_day_hl",     # PDH / PDL
    "previous_week_hl",    # PWH / PWL
    "round_numbers",       # 2000, 1950, 1900, dll
    "swing_hl_obvious"     # Di-test minimal 2×
]
```

**Kriteria Sweep Valid (semua harus terpenuhi):**
```python
criteria = {
    "penetration":      "spike_through > 1 pip",
    "candle_close":     "close kembali di sisi sebelumnya",
    "wick_ratio":       "wick_size / (wick + body) > 0.6",
    "reversal_timing":  "reversal dalam 3 candle M15",
    "level_strength":   "test_count >= 2",
    "session_filter":   "London (15:00-20:00 WIB) atau NY (20:30-23:00 WIB)",
    "bocpd_status":     "P(changepoint) < 0.4",
    "news_filter":      "tidak ada high impact news ±30 menit"
}
```

**One Attempt Rule:**
```
Hanya 1 entry per level per hari
Jika kena SL, level tersebut CLOSED untuk hari itu
```

**Entry / SL / TP:**
```python
entry  = close(sweep_candle)         # Close konfirmasi candle
buffer = atr_m15 * 0.1
sl     = sweep_low - buffer          # Di bawah wick sweep
tp     = entry + (entry - sl) * 5   # RR 1:5
```

**Expectancy:**
```
Konservatif: P(TP)=35%, P(BEP)=25%, P(SL)=40% → +27 cent/trade
Optimis:     P(TP)=45%, P(BEP)=20%, P(SL)=35% → +38 cent/trade
```

---

### 5.5 Triple Confluence System

Kombinasi terkuat — prioritaskan selalu:

```python
confluence_levels = {
    "single_OB":           {"win_rate": 0.38, "expectancy": 30},
    "OB_plus_FVG":         {"win_rate": 0.51, "expectancy": 42},
    "OB_plus_Sweep":       {"win_rate": 0.54, "expectancy": 46},
    "FVG_plus_Sweep":      {"win_rate": 0.52, "expectancy": 44},
    "triple_OB_FVG_Sweep": {"win_rate": 0.62, "expectancy": 56},
    "triple_plus_BOCPD":   {"win_rate": 0.67, "expectancy": 62}
}
```

---

## BAGIAN 6 — MATEMATIKA INTI

### 6.1 Geometric Brownian Motion (GBM Standar)
```
S(t+Δt) = S(t) · exp[(μ - σ²/2)·Δt + σ·√Δt·ε]

Dimana:
S(t)   = Harga pada waktu t
μ      = Drift (expected return)
σ      = Volatilitas (std dev return)
Δt     = Time step (1/(252×24×4) untuk M15)
ε      = Random shock ~ Normal(0,1)
σ²/2   = Ito correction term
```

### 6.2 Regime-Switching GBM (RS-GBM)
```
ρ̃(t) ~ Categorical(π₁(t), π₂(t), π₃(t))  ← dari BOCPD posterior

S(t+Δt) = S(t) · exp[(μ_{ρ̃(t)} - σ_{ρ̃(t)}²/2)·Δt + σ_{ρ̃(t)}·√Δt·ε]

Dimana:
ρ̃(t)       = Regime yang di-sample dari BOCPD posterior
μ_{ρ̃(t)}  = Drift untuk regime ρ̃(t)
σ_{ρ̃(t)}  = Volatilitas untuk regime ρ̃(t)
πₖ(t)      = Probabilitas posterior regime k dari BOCPD
```

### 6.3 Gaussian Mixture Distribution (GMM)
```
         K
f(x) =  Σ  πₖ · φ(x | μₖ_eff, σₖ²·Δt)
        k=1

Dimana:
πₖ       = Mixing weight (probabilitas regime k)
μₖ_eff   = (μₖ - σₖ²/2) · Δt  (mean efektif setelah Ito correction)
φ        = PDF Normal

Mean GMM:
E[X] = Σₖ πₖ · μₖ_eff

Variance GMM (Law of Total Variance):
Var[X] = Σₖ πₖ·σₖ²·Δt  +  Σₖ πₖ·μₖ_eff² - (Σₖ πₖ·μₖ_eff)²
         ↑ Within-regime      ↑ Between-regime variance

Adjusted Volatility:
σ_adj(t) = √(Var_GMM)  ← Selalu > weighted average σ

Dynamic SL:
SL(t) = Z_0.99 × σ_adj(t) × √Δt × S(t)
      = 2.326 × σ_adj(t) × √Δt × S(t)

Expected Shortfall (ES):
ES(t) = Σₖ πₖ(t) · [μₖ_eff - 2.063 · σₖ · √Δt]
```

### 6.4 Expectancy Formula
```python
def expectancy(p_win, p_bep, p_loss, reward=100, risk=20):
    """
    p_win  = Probabilitas TP kena
    p_bep  = Probabilitas BEP kena (trailing trigger)
    p_loss = Probabilitas SL kena  (p_win + p_bep + p_loss = 1)
    reward = Profit saat TP (cent)
    risk   = Loss saat SL (cent)
    """
    return (p_win * reward) + (p_bep * 0) + (p_loss * -risk)

# Breakeven win rate dengan RR 1:5:
# W_min = 1 / (1 + RR) = 1/6 = 16.7%
```

### 6.5 Position Sizing
```python
def position_size(balance, risk_pct, sl_pips, pip_value_per_lot):
    risk_amount = balance * risk_pct           # Dalam cent
    lot_size = risk_amount / (sl_pips * pip_value_per_lot)
    return lot_size

# Default YQTS:
# risk_pct = 0.02 (2%)
# pip_value XAU/USD: 0.001 cent per 0.01 lot per pip
```

### 6.6 Sharpe Ratio
```python
def sharpe_ratio(returns, risk_free=0):
    """
    returns = Array daily returns
    Target: Sharpe > 1.5
    """
    return (np.mean(returns) - risk_free) / np.std(returns) * np.sqrt(252)
```

---

## BAGIAN 7 — MONTE CARLO FRAMEWORK

### 7.1 Arsitektur (3 Layer)

```
LAYER 1 — Market Generator
├── 1.1 Parameter Estimator   (PELT → μₖ, σₖ, transition matrix)
├── 1.2 Regime Path Generator  (Markov Chain simulation)
└── 1.3 Price Path Generator   (RS-GBM simulation)

LAYER 2 — YQTS Engine
├── 2.1 State Machine          (balance, position, drawdown tracking)
├── 2.2 Regime Detector        (Simplified BOCPD dengan detection lag)
├── 2.3 Signal Generator       (Entry conditions per strategy)
├── 2.4 Trade Executor         (GMM-adjusted sizing + safety checks)
└── 2.5 Position Manager       (Trailing SL + exit management)

LAYER 3 — Analytics Engine
├── 3.1 Simulation Runner      (N=10,000 loop)
├── 3.2 Metrics Calculator     (7 kategori metrics)
└── 3.3 Decision Dashboard     (Pass/Fail verdict)
```

### 7.2 Simulation Parameters
```python
SIMULATION_CONFIG = {
    "N_simulations":    10_000,      # Jumlah simulasi
    "T_candles":        480,         # Candle M15 per simulasi (1 bulan)
    "initial_balance":  1_000,       # Cent
    "timeframe_delta":  1/(252*24*4),# Δt untuk M15
    "random_seed":      42,          # Reproducibility
}
```

### 7.3 State Machine Schema
```python
@dataclass
class YQTSState:
    balance:          float = 1000.0
    position:         int   = 0      # 1=long, -1=short, 0=flat
    entry_price:      float = 0.0
    sl_price:         float = 0.0
    tp_price:         float = 0.0
    lot_size:         float = 0.0
    bep_triggered:    bool  = False
    trade_count:      int   = 0      # Reset setiap hari
    daily_loss:       float = 0.0    # Reset setiap hari
    weekly_loss:      float = 0.0    # Reset setiap minggu
    consecutive_loss: int   = 0
    is_active:        bool  = True
    regime_detected:  str   = "ranging"
    bocpd_posterior:  list  = None   # [π₁, π₂, π₃]
```

### 7.4 Simulation Result Schema
```python
@dataclass
class SimulationResult:
    sim_id:           int
    final_balance:    float
    equity_curve:     list   # Balance di setiap candle
    trade_log:        list   # Semua trade yang terjadi
    hit_kill_switch:  bool
    max_drawdown:     float  # Dalam persen
    n_trades:         int
    win_rate:         float
    regime_history:   list
```

### 7.5 Trade Log Entry Schema
```python
@dataclass
class TradeEntry:
    sim_id:       int
    trade_id:     int
    entry_time:   int    # Candle index
    entry_price:  float
    entry_regime: str    # "trending_up", "ranging", "trending_down"
    strategy:     str    # "OB", "FVG", "SWEEP", "CONFLUENCE"
    sl_price:     float
    tp_price:     float
    lot_size:     float
    outcome:      str    # "TP", "BEP", "SL"
    pnl:          float  # Dalam cent
    bep_triggered: bool
    confluence_score: int
```

### 7.6 BOCPD Simplified untuk Simulasi
```python
def get_detected_regime(true_regime_sequence, t, pelt_detection_lag):
    """
    Di simulasi kita tahu true regime (ground truth dari generator).
    Simulasikan detection lag BOCPD:
    lag ~ Poisson(lambda_lag) dimana lambda_lag = 7 candle M15
    """
    lag = np.random.poisson(7)
    t_delayed = max(0, t - lag)
    return true_regime_sequence[t_delayed]
```

### 7.7 Signal Generator Logic
```python
SIGNAL_CONFIG = {
    "setup_frequency": {
        "trending_up":   0.08,   # 8% candle ada valid setup
        "trending_down": 0.07,
        "ranging":       0.10
    },
    "win_rates": {
        "trending_up":   0.40,
        "trending_down": 0.38,
        "ranging":       0.45
    },
    "bep_rates": {
        "trending_up":   0.22,
        "trending_down": 0.20,
        "ranging":       0.25
    }
    # loss_rate = 1 - win_rate - bep_rate
}
```

### 7.8 Pass/Fail Criteria (Semua Harus Lulus)
```python
PASS_CRITERIA = {
    "p_profit_min":         0.65,   # P(final_balance > initial) > 65%
    "p_ruin_max":           0.10,   # P(kill switch) < 10%
    "mean_return_min":      0.20,   # Mean monthly return > 20%
    "median_return_min":    0.10,   # Median monthly return > 10%
    "pct5_return_min":     -0.30,   # 5th percentile > -30%
    "var95_max":            200,    # VaR 95% < 200 cent
    "p_maxdd_20pct_max":    0.15,   # P(MaxDD > 20%) < 15%
}
```

### 7.9 Stress Tests
```python
STRESS_TESTS = {
    "black_swan": {
        "jump_prob":        0.005,   # Per candle M15
        "jump_mean":       -0.02,
        "jump_std":         0.01
    },
    "high_regime_freq": {
        "regime_freq_multiplier": 2.0
    },
    "high_volatility": {
        "sigma_multiplier": 1.5
    },
    "low_win_rate": {
        "win_rate_reduction": 0.10   # Kurangi semua win rate 10%
    },
    "high_transaction_cost": {
        "slippage_multiplier": 2.0
    }
}
```

---

## BAGIAN 8 — STRUKTUR PROJECT

```
YQTS/
├── README.md
├── YQTS_BRIEF.md               ← File ini
├── requirements.txt
├── config/
│   └── yqts_config.yaml        ← Semua parameter terpusat
├── data/
│   ├── raw/                    ← Data XAU dari yfinance
│   └── processed/              ← Data setelah preprocessing
├── src/
│   ├── __init__.py
│   ├── regime/
│   │   ├── __init__.py
│   │   ├── pelt.py             ← PELT parameter estimator
│   │   └── bocpd.py            ← BOCPD online detector
│   ├── market/
│   │   ├── __init__.py
│   │   ├── data_loader.py      ← yfinance / MT5 data fetcher
│   │   ├── parameter_estimator.py  ← μₖ, σₖ dari PELT output
│   │   ├── regime_generator.py    ← Markov Chain sim
│   │   └── price_generator.py     ← RS-GBM price paths
│   ├── strategy/
│   │   ├── __init__.py
│   │   ├── order_block.py      ← OB detection + scoring
│   │   ├── fair_value_gap.py   ← FVG detection + priority
│   │   └── liquidity_sweep.py  ← Sweep detection + validation
│   ├── engine/
│   │   ├── __init__.py
│   │   ├── state.py            ← YQTSState dataclass
│   │   ├── signal_generator.py ← Unified signal logic
│   │   ├── trade_executor.py   ← GMM-adjusted execution
│   │   └── position_manager.py ← Trailing + exit
│   ├── monte_carlo/
│   │   ├── __init__.py
│   │   ├── simulator.py        ← Main MC runner
│   │   ├── analytics.py        ← Metrics calculator
│   │   ├── stress_test.py      ← 5 stress test scenarios
│   │   └── dashboard.py        ← Pass/Fail verdict + plots
│   └── utils/
│       ├── __init__.py
│       ├── math_utils.py       ← GMM, ES, VaR calculations
│       └── plot_utils.py       ← Visualization helpers
├── tests/
│   ├── test_regime.py
│   ├── test_market.py
│   ├── test_strategy.py
│   ├── test_engine.py
│   └── test_monte_carlo.py
├── notebooks/
│   ├── 01_data_exploration.ipynb
│   ├── 02_pelt_calibration.ipynb
│   ├── 03_bocpd_testing.ipynb
│   ├── 04_strategy_analysis.ipynb
│   └── 05_monte_carlo_results.ipynb
└── models/
    ├── pelt_params.pkl         ← Saved PELT parameters
    └── bocpd_model.pkl         ← Saved BOCPD state
```

---

## BAGIAN 9 — REQUIREMENTS

```
# requirements.txt

# Core
numpy>=1.26.0
scipy>=1.11.0
pandas>=2.1.0

# Visualization
matplotlib>=3.8.0
seaborn>=0.13.0
plotly>=5.18.0

# Regime Detection
ruptures>=1.1.9

# Data
yfinance>=0.2.33

# Performance
numba>=0.58.0
joblib>=1.3.0

# Progress
tqdm>=4.66.0

# Config
pyyaml>=6.0.1

# Testing
pytest>=7.4.0
pytest-cov>=4.1.0

# Development
jupyter>=1.0.0
jupyterlab>=4.0.0
```

---

## BAGIAN 10 — CONFIG FILE

```yaml
# config/yqts_config.yaml

system:
  name: "YQTS"
  version: "1.0.0"
  asset: "GC=F"           # Gold futures di yfinance
  mt5_symbol: "XAUUSD"
  timeframe: "M15"
  initial_balance: 1000   # cent

trading:
  max_positions: 2
  max_trades_per_day: 2
  risk_reward: 5
  risk_per_trade: 0.02
  trailing_trigger: 0.60
  min_sl_pips: 2.0
  pip_value: 0.001        # cent per pip per 0.01 lot

risk:
  daily_loss_limit: 0.05
  weekly_limit: 0.10
  kill_switch: 0.20
  circuit_breaker: 2

regime:
  pelt:
    model: "rbf"
    beta_base: 10.0
    min_segment_length: 50
    recalibration_threshold: 0.20
  bocpd:
    max_run_length: 200
    hazard_lambda: 100
    regimes:
      trending_up:
        mu: 0.0008
        sigma: 0.008
      ranging:
        mu: 0.0001
        sigma: 0.005
      trending_down:
        mu: -0.0006
        sigma: 0.012

monte_carlo:
  n_simulations: 10000
  t_candles: 480
  random_seed: 42
  detection_lag_lambda: 7
  n_jobs: -1              # Semua CPU cores

pass_criteria:
  p_profit_min: 0.65
  p_ruin_max: 0.10
  mean_return_min: 0.20
  median_return_min: 0.10
  pct5_return_min: -0.30
  var95_max: 200
  p_maxdd_20pct_max: 0.15
```

---

## BAGIAN 11 — URUTAN IMPLEMENTASI

Implement dalam urutan ini. Jangan skip atau parallelkan:

```
SPRINT 1 — Foundation
□ Setup project structure
□ requirements.txt + config
□ src/utils/math_utils.py (GMM, ES, VaR)
□ Tests untuk math_utils

SPRINT 2 — Data & Regime
□ src/market/data_loader.py
□ src/regime/pelt.py
□ src/regime/bocpd.py
□ Tests untuk regime detection
□ notebooks/01_data_exploration.ipynb
□ notebooks/02_pelt_calibration.ipynb

SPRINT 3 — Market Generator (Layer 1)
□ src/market/parameter_estimator.py
□ src/market/regime_generator.py
□ src/market/price_generator.py
□ Tests untuk setiap generator
□ Validasi: plot 10 sample price paths

SPRINT 4 — YQTS Engine (Layer 2)
□ src/engine/state.py
□ src/strategy/order_block.py
□ src/strategy/fair_value_gap.py
□ src/strategy/liquidity_sweep.py
□ src/engine/signal_generator.py
□ src/engine/trade_executor.py
□ src/engine/position_manager.py
□ Integration test: run 1 simulasi, trace setiap trade

SPRINT 5 — Monte Carlo (Layer 3)
□ src/monte_carlo/simulator.py
□ src/monte_carlo/analytics.py
□ src/monte_carlo/stress_test.py
□ src/monte_carlo/dashboard.py
□ Run N=100 dulu untuk debug
□ Run N=10,000 untuk hasil final

SPRINT 6 — Analysis & Decision
□ notebooks/05_monte_carlo_results.ipynb
□ Generate semua plots
□ Evaluate Pass/Fail criteria
□ Document findings
```

---

## BAGIAN 12 — PERFORMANCE TARGETS

```python
PERFORMANCE_TARGETS = {
    # Code execution
    "mc_10k_runtime_seconds":    20,   # < 20 detik untuk N=10,000
    "mc_100k_runtime_seconds":   200,  # < 200 detik untuk N=100,000

    # Statistical validity
    "min_trades_for_stats":      1000, # Minimum trades untuk analisis valid
    "confidence_interval":       0.95, # CI untuk semua metrics

    # Code quality
    "test_coverage_min":         0.80, # 80% test coverage minimum
    "max_function_length_lines": 50,   # Single responsibility
    "docstring_coverage":        1.00  # 100% public functions
}
```

---

## BAGIAN 13 — VALIDASI CHECKLIST

Sebelum declare setiap sprint selesai, validasi:

```
SPRINT 2 — Regime Validation:
□ PELT menghasilkan minimal 3 regime berbeda di data 2 tahun
□ μ₁ > μ₂ > μ₃ (sesuai trending_up > ranging > trending_down)
□ σ₃ > σ₁ > σ₂ (trending_down paling volatile)
□ Transition matrix rows sum to 1.0
□ BOCPD update time < 1ms per candle

SPRINT 3 — Market Generator Validation:
□ Semua S(t) > 0 (properti GBM)
□ Log-return distribution mendekati GMM visual
□ Autocorrelation log-returns ≈ 0
□ Regime sequence length distribution masuk akal (hours, not minutes)

SPRINT 4 — Engine Validation:
□ Trade count per hari tidak pernah > 2
□ Kill switch terpicu jika balance < 800 cent
□ Trailing trigger aktif tepat di 60% TP distance
□ SL selalu lebih ketat dari TP / 5
□ RR selalu = 5.0 (check setiap trade)

SPRINT 5 — Monte Carlo Validation:
□ N=10,000 simulasi selesai < 20 detik
□ Random seed menghasilkan results yang sama
□ Distribusi final balance smooth (bukan spiky)
□ P(profit) + P(loss) + P(breakeven) = 1.0 (per trade)
□ Stress test semua 5 skenario berjalan tanpa error
```

---

## BAGIAN 14 — OUTPUT YANG DIHARAPKAN

### Dashboard Output Utama
```
═══════════════════════════════════════════════════
        YQTS MONTE CARLO RESULTS
        N=10,000 | T=480 candle (1 bulan)
═══════════════════════════════════════════════════

PROBABILITY ANALYSIS:
P(Profit)           : XX.X%  [Target: > 65%]
P(Ruin/KillSwitch)  : XX.X%  [Target: < 10%]
P(Return > 50%)     : XX.X%
P(Return > 100%)    : XX.X%

RETURN DISTRIBUTION:
Mean Return         : +XX.X% [Target: > 20%]
Median Return       : +XX.X% [Target: > 10%]
Std Deviation       : XX.X%
5th Percentile      : -XX.X% [Target: > -30%]
95th Percentile     : +XX.X%

RISK METRICS:
VaR (95%)           : XX.X cent [Target: < 200]
Expected Shortfall  : XX.X cent
Mean Max Drawdown   : XX.X%
P(MaxDD > 20%)      : XX.X%  [Target: < 15%]

TRADE STATISTICS:
Overall Win Rate    : XX.X%
TP Hit Rate         : XX.X%
BEP Hit Rate        : XX.X%
SL Hit Rate         : XX.X%
Avg Trades/Month    : XX.X

SYSTEM VERDICT:
P(Profit) > 65%     : [PASS/FAIL]
P(Ruin) < 10%       : [PASS/FAIL]
Mean Return > 20%   : [PASS/FAIL]
5th pct > -30%      : [PASS/FAIL]
VaR < 200 cent      : [PASS/FAIL]
P(MaxDD>20%) < 15%  : [PASS/FAIL]

OVERALL: [READY / NOT READY] for live trading
═══════════════════════════════════════════════════
```

### Plots yang Dihasilkan
```
1. equity_curve_distribution.png    ← 100 sample equity curves + percentile bands
2. final_balance_histogram.png      ← Distribusi final balance semua simulasi
3. return_distribution_gmm.png      ← GMM fit vs empirical return distribution
4. drawdown_heatmap.png             ← Heatmap drawdown per simulasi per waktu
5. regime_transition_matrix.png     ← Visualisasi transition matrix
6. stress_test_comparison.png       ← Perbandingan base vs 5 stress tests
7. trade_statistics_by_regime.png   ← Win rate, expectancy per regime
```

---

## BAGIAN 15 — CATATAN PENTING

```
1. MATEMATIKA TIDAK BOLEH DIUBAH
   Semua rumus (GBM, GMM, ES, VaR) sudah final
   Jika ada edge case yang tidak ter-cover,
   tanya dulu sebelum modifikasi

2. PARAMETER REGIME ADALAH ESTIMASI AWAL
   {mu, sigma} per regime akan di-update
   setelah PELT dijalankan pada data XAU nyata
   Nilai di config adalah starting point

3. SIMPLIFIED BOCPD DI SIMULASI ADALAH INTENTIONAL
   Full BOCPD terlalu lambat untuk 10,000 simulasi
   Detection lag Poisson(7) adalah approximasi yang valid

4. AKUN CENT BUKAN DOLLAR
   Semua kalkulasi dalam CENT
   1000 cent = $10 USD
   Jangan convert ke dollar dalam code

5. RR 1:5 ADALAH IMMUTABLE
   Ini bukan parameter yang bisa di-tune
   Merupakan DNA sistem YQTS

6. HASIL MONTE CARLO BUKAN PREDIKSI
   Ini probabilistic stress test
   Pass criteria = sistem sound secara statistik
   Bukan guarantee profit di live trading
```

---

*YQTS Brief v1.0 — Dokumen ini adalah single source of truth untuk implementasi YQTS Monte Carlo Framework*
*Semua keputusan arsitektur telah divalidasi melalui diskusi mendalam sebelum dokumen ini dibuat*
