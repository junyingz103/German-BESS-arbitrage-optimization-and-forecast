# BESS Forecast, Optimization, and Arbitrage Backtesting

An end-to-end quantitative study of a 100 MWh / 50 MW grid-scale Battery Energy Storage System (BESS) in the German/Luxembourg day-ahead market. This repository integrates Pyomo physical optimization (Perfect Foresight), weather/neighbor-price correlation analysis, and a Machine Learning forecasting backtest (walk-forward), settled against actual real-time market prices to evaluate practical arbitrage profitability.

---

## Table of Contents

1. [Repository Structure](#repository-structure)
2. [Data Preparation](#data-preparation)
3. [Preliminary Analysis](#preliminary-analysis)
4. [Model Assumptions](#model-assumptions)
5. [Model Formulation](#model-formulation)
6. [Key Results](#key-results)
7. [Sensitivity Analysis](#sensitivity-analysis)
8. [Weather-Price Analysis](#weather-price-analysis)
9. [Neighbor Market Coupling](#neighbor-market-coupling)
10. [ML Forecasting & Arbitrage Backtest](#ml-forecasting--arbitrage-backtest)
11. [Installation](#installation)
12. [Limitations](#limitations)

---

## Repository Structure

| File | Role |
|---|---|
| `1_price.ipynb` | Raw day-ahead price data |
| `2_weather.ipynb` | Raw weather data |
| `3_merge.ipynb` | Merge and clean the two datasets |
| `4_preliminary analysis.ipynb` | Price distribution, skewness, kurtosis, negative-price frequency, intraday time periods |
| `5_German BESS price arbitrage optimization.ipynb` | Core optimization model (Pyomo), strategy identification, and sensitivity analysis |
| `6_weather-price analysis.ipynb` | Weather–price correlation and feature importance |
| `7_neighbor market-price analysis.ipynb` | Market coupling analysis with neighboring countries |
| `8_ML price forecast and arbitrage.ipynb` | ML forecasting benchmark, Pyomo physical optimization, and real-world arbitrage backtest |

---

## Data Preparation

- **Price data**: DE/LU day-ahead prices (15-min resolution), including neighbor-country prices.
- **Weather data**: temperature, sunshine duration, wind speed/direction, precipitation, shortwave radiation, is_day flag.
- **Merge**: 15-min time-aligned inner join -> **61,144 rows** covering **2025-01-01 to 2026-09-29**.

---

## Preliminary Analysis

**Price distribution:** Strongly non-normal - bimodal, right-skewed (skew 0.42), fat-tailed (kurtosis 8.28). Range: **-500 to +747 EUR/MWh**.

**Intraday pattern:** Dual-trough / dual-peak - noon and afternoon cheapest (~50-61 EUR/MWh), evening and night most expensive (~121-134 EUR/MWh).

**Low-price frequency:** **7.9%** of time <= 0 EUR/MWh; **17.5%** <= 50 EUR/MWh. Negative prices concentrated in April-June, absent in Nov-Dec.

---

## Model Assumptions

- Nominal capacity: 100 MWh; SOC range 20-100 MWh (80% DoD)
- Max power: 50 MW (12.5 MWh per 15-min step)
- Round-trip efficiency: 85% (sqrt(0.85) each direction)
- Initial SOC: 50 MWh
- Degradation cost: 5 EUR/MWh discharged
- Perfect foresight of prices (theoretical upper bound)
- No simultaneous charge/discharge (binary constraint)
- **No Dmax** - the optimizer determines cycle frequency endogenously

---

## Model Formulation

### Symbols

| Symbol | Description | Unit |
|---|---|---|
| $t$ | Time step (96 steps/day, 15-min each) | -- |
| $\pi_t$ | Day-ahead price at step $t$ | EUR/MWh |
| $E^{in}_t$ | Energy charged at step $t$ | MWh |
| $E^{out}_t$ | Energy discharged at step $t$ | MWh |
| $S_t$ | State of charge after step $t$ | MWh |
| $u_t$ | Binary (1 = charge, 0 = discharge/idle) | -- |
| $S_{max}$ | Max capacity = 100 | MWh |
| $S_{min}$ | Min capacity = 20 | MWh |
| $R_{max}$ | Max charge/discharge per step = 12.5 | MWh |
| $\eta$ | Round-trip efficiency = 0.85 | -- |
| $\eta_{rt}$ | One-way efficiency = sqrt(0.85) | -- |
| $c_{deg}$ | Degradation cost = 5 | EUR/MWh |

### Objective

    max  sum_t [ pi_t * E_out_t - pi_t * E_in_t - c_deg * E_out_t ]

Revenue from discharging minus cost of charging minus degradation cost.

### Constraints

**SOC transition:**

    S_t = S_{t-1} + eta_rt * E_in_t - E_out_t / eta_rt

**No simultaneous charge & discharge:**

    E_in_t  <= R_max * u_t
    E_out_t <= R_max * (1 - u_t)

**Discharge availability:**

    E_out_t / eta_rt <= S_{t-1}

**Charge headroom:**

    eta_rt * E_in_t <= S_max - S_{t-1}

**Bounds:**

    0 <= E_in_t <= R_max
    0 <= E_out_t <= R_max
    S_min <= S_t <= S_max
    u_t in {0, 1}

**End-of-day SOC >= start-of-day SOC:**

    S_96 >= S_start

---

## Key Results

### Strategy Findings

- The model **endogenously identifies** a two-charge / two-discharge pattern on high-spread days - no Dmax assumption imposed.
- **60% of days idle**, 24% do 1-2 cycles, 15% do >=3 cycles.
- On days when the model does cycle, the average intraday pattern shows:
  - **Charging**: pre-dawn (~03:00) and midday/afternoon (12:00 winter, 15:00 summer - shifted by solar).
  - **Discharging**: morning peak (~07:00-08:00) and evening peak (~16:00 winter, ~18:00-22:00 summer).

### Financial Result

| Metric | Value |
|---|---|
| Net profit (637 days) | 6,384,488 EUR |
| Annualized net profit | 3,658,301 EUR / year |
| Per installed MW | 73,166 EUR / MW / year |
| Avg cycles/day | 0.95 |

**Scope-matched comparison:** Modo Energy reports ~55% of German BESS revenues come from ancillary services in 2026. Stripping this share from the full-stack benchmark (125k-190k EUR/MW/year) yields an arbitrage-only range of **56k-86k EUR/MW/year**. Our result (73,166 EUR/MW/year) sits within this range, confirming consistency.

---

## Sensitivity Analysis

| c_deg (EUR/MWh) | Net Profit (EUR) | Annualized (EUR/year) | Delta vs base |
|---|---|---|---|
| 0 | 6,872,476 | 3,938,286 | +7.7% |
| **5 (base)** | **6,384,488** | **3,658,301** | -- |
| 20 | 4,908,345 | 2,812,858 | -23% |
| 40 | 3,688,243 | 2,113,562 | -42% |

**Conclusion:** Even under conservative degradation cost assumptions (c_deg = 20 and 40 EUR/MWh), the strategy remains strongly profitable, with annualized net profit still above **2M EUR/year**.

---

## Weather-Price Analysis

Solar radiation is the strongest weather driver (r = -0.50), followed by sunshine (-0.31) and wind speed (-0.23). Weather correlates strongly with intraday price shapes (midday troughs and negative prices), but a pure weather-based forecast achieves a very low R² (0.04). Adding lagged prices and neighbor-country prices raises the R² to 0.93, with neighbor prices dominating (94.6% feature importance).

---

## Neighbor Market Coupling

The DE/LU price is highly coupled with neighboring markets. The average of all neighbors (`∅ DE/LU neighbours`) achieves an **r = 0.969**. 

- **Top Coupled Markets**: Netherlands (0.966), Denmark 1 (0.945), Belgium (0.938).
- **Seasonal Variation**: Coupling is strongest in summer (r ≈ 0.98) due to synchronized solar generation, and weakest in winter (r ≈ 0.93) when localized wind generation dominates.

---

## ML Forecasting & Arbitrage Backtest

We benchmarked four models (Persistence, RidgeCV, XGBoost, Random Forest) for a 7-day Day-Ahead forecast (2026-09-30 → 2026-10-07). 

**Crucial Setup**: The ML models only provided the predicted price curves. The actual charge/discharge schedule was generated by integrating these predictions into the **Pyomo Optimization Engine** (100 MWh / 50 MW), and the final profit was **settled using actual real-time prices** to simulate real trading.

### 1. Price Magnitude & Timing Accuracy
- **Magnitude**: Machine learning struggles with a "Smoothing Penalty," failing to predict extreme spikes.
- **Timing**: A **±1-hour tolerance** is physically justified because a 50 MW battery requires **1.6 hours** to discharge 80 MWh. Within this window, models successfully captured the macro intraday shape.

### 2. Commercial Profit Capture Rate
We define the **Profit Capture Rate** as `(ML Strategy Real Profit / Perfect Foresight Profit) × 100`.

| Model | Perfect Foresight (€) | Pyomo Strategy Real Profit (€) | Profit Capture Rate (%) | Overall Timing Accuracy (±1h) |
|---|---|---|---|---|
| **Perfect Foresight** | **€102,490** | **€102,490** | **100.0%** | **100.0%** |
| **Persistence (Baseline)** | €102,490 | €90,243 | **88.1%** | **92.9%** |
| **RidgeCV** | €102,490 | €87,720 | **85.6%** | **71.4%** |
| **Random Forest** | €102,490 | €87,292 | **85.2%** | **85.7%** |
| **XGBoost** | €102,490 | €80,745 | **78.8%** | **92.9%** |

### 3. Key Conclusion: "Time is Money, but Shape is King"
1. **The XGBoost Paradox**: XGBoost achieved an excellent 92.9% timing accuracy but yielded the lowest profit (78.8%). Its predicted magnitudes were wildly distorted (e.g., 579.9 instead of 261.4 EUR/MWh), which confused the Pyomo optimizer. This proves that **accurate timing cannot compensate for a distorted price shape.**
2. **The Smoothness Advantage**: RidgeCV and Persistence produced smooth, realistic curves. Even with timing deviations, Pyomo used these stable shapes to execute continuous 1.6-hour discharge schedules, capturing 85%+ of the profit.
3. **Justification for Perfect Foresight (Notebook 5)**: The 12%-15% gap between the best realistic models (Persistence/RidgeCV) and the theoretical maximum represents the real-world **"Cost of Forecast Uncertainty."** This validates using Perfect Foresight in Notebook 5 to establish the ultimate revenue ceiling.

---

## Installation

```bash
pip install pandas numpy matplotlib seaborn scikit-learn pyomo pyomo_windows xgboost
---

## Installation

```bash
pip install pandas numpy matplotlib seaborn scikit-learn pyomo pyomo_windows xgboost
```

CBC solver required. On Windows:

```python
from pyomo_windows.solvers import DownloadSolvers
DownloadSolvers().download_cbc()
```

---

## Limitations

1. **Perfect foresight** — theoretical upper bound, not achievable in live trading.
2. **Energy arbitrage only** — excludes ancillary services (FCR, aFRR), intraday trading, and capacity markets.
3. **No market impact** — assumes 50 MW / 100 MWh does not affect prices.
4. **Constant 85% efficiency** — real efficiency varies with C-rate and SOC.
5. **German day-ahead market only** — no cross-border trading.
6. **Data period** — 2025-01-01 to 2026-09-29 may not represent future market conditions.
