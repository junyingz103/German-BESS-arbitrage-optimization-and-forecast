# Battery Storage Arbitrage Optimization

A Pyomo-based optimization study of a 100 MWh / 50 MW grid-scale battery participating in the German/Luxembourg day-ahead market. The model endogenously identifies charge/discharge strategies from historical prices, with **no pre-imposed cycling assumption (Dmax)**.

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
9. [Installation](#installation)
10. [Limitations](#limitations)

---

## Repository Structure

| File | Role |
|---|---|
| `1_price.ipynb` | Raw day-ahead price data |
| `2_weather.ipynb` | Raw weather data |
| `3_merge.ipynb` | Merge and clean the two datasets |
| `4_preliminary analysis.ipynb` | Price distribution, skewness, kurtosis, negative-price frequency, intraday time periods |
| `5_German BESS price arbitrage optimization.ipynb` | Project assumptions, model formulation, optimization results, visualization summary, sensitivity analysis, findings & recommendations |
| `6_weather-price analysis.ipynb` | Weather-price correlation and price prediction |

---

## Data Preparation

- **Price data**: DE/LU day-ahead prices (15-min resolution), including neighbor-country prices.
- **Weather data**: temperature, sunshine duration, wind speed/direction, precipitation, shortwave radiation, is_day flag.
- **Merge**: 15-min time-aligned inner join → **61,144 rows** covering **2025-01-01 to 2026-09-29**.

---

## Preliminary Analysis

**Price distribution:** Strongly non-normal — bimodal, right-skewed (skew 0.42), fat-tailed (kurtosis 8.28). Range: **-500 to +747 EUR/MWh**.

**Intraday pattern:** Dual-trough / dual-peak — noon and afternoon cheapest (~50–61 EUR/MWh), evening and night most expensive (~121–134 EUR/MWh).

**Low-price frequency:** **7.9%** of time ≤ 0 EUR/MWh; **17.5%** ≤ 50 EUR/MWh. Negative prices concentrated in April–June, absent in Nov–Dec.

**Summary:** Frequent low/negative prices provide charging windows; evening peaks provide discharge opportunities — a solid economic basis for battery arbitrage.

---

## Model Assumptions

- Nominal capacity: 100 MWh; SOC range 20–100 MWh (80% DoD)
- Max power: 50 MW (12.5 MWh per 15-min step)
- Round-trip efficiency: 85% (√0.85 each direction)
- Initial SOC: 50 MWh
- Degradation cost: 5 EUR/MWh discharged
- Perfect foresight of prices (theoretical upper bound)
- No simultaneous charge/discharge (binary constraint)
- **No Dmax** — the optimizer determines cycle frequency endogenously

---

## Model Formulation

### Symbols

| Symbol | Description | Unit |
|---|---|---|
| `t` | Time step (96 steps/day, 15-min each) | — |
| `π_t` | Day-ahead price at step `t` | EUR/MWh |
| `E_in_t` | Energy charged at step `t` | MWh |
| `E_out_t` | Energy discharged at step `t` | MWh |
| `S_t` | State of charge after step `t` | MWh |
| `u_t` | Binary (1 = charge, 0 = discharge/idle) | — |
| `S_max` | Max capacity = 100 | MWh |
| `S_min` | Min capacity = 20 | MWh |
| `R_max` | Max charge/discharge per step = 12.5 | MWh |
| `η` | Round-trip efficiency = 0.85 | — |
| `η_rt` | One-way efficiency = sqrt(0.85) | — |
| `c_deg` | Degradation cost = 5 | EUR/MWh |

### Objective

Maximize net profit:

```
max  Σ_t [ π_t · E_out_t − π_t · E_in_t − c_deg · E_out_t ]
```

Revenue from discharging minus cost of charging minus degradation cost.

### Constraints

**SOC transition:**

```
S_t = S_(t−1) + η_rt · E_in_t − E_out_t / η_rt
```

**No simultaneous charge & discharge:**

```
E_in_t  ≤ R_max · u_t
E_out_t ≤ R_max · (1 − u_t)
```

**Discharge availability:**

```
E_out_t / η_rt ≤ S_(t−1)
```

**Charge headroom:**

```
η_rt · E_in_t ≤ S_max − S_(t−1)
```

**Bounds:**

```
0 ≤ E_in_t  ≤ R_max
0 ≤ E_out_t ≤ R_max
S_min ≤ S_t ≤ S_max
u_t ∈ {0, 1}
```

**End-of-day SOC ≥ start-of-day SOC:**

```
S_96 ≥ S_start
```

---

## Key Results

### Strategy Findings

- The model **endogenously identifies** a two-charge / two-discharge pattern on high-spread days — no Dmax assumption imposed.
- **60% of days idle**, 24% do 1–2 cycles, 15% do ≥3 cycles.
- On days when the model does cycle, the average intraday pattern shows:
  - **Charging**: pre-dawn (~03:00) and midday/afternoon (12:00 winter, 15:00 summer — shifted by solar).
  - **Discharging**: morning peak (~07:00–08:00) and evening peak (~16:00 winter, ~18:00–22:00 summer).

### Actionable Recommendations

- **Charge** during pre-dawn lows and midday solar troughs; **discharge** into morning and evening peaks.
- **Shift the second charge later in summer** (~15:00) to align with solar oversupply.
- **Do not cycle on low-spread days** — 60% of days are better left idle.
- Accept **1–2 cycles/day as the realistic operating baseline**.

### Financial Result

| Metric | Value |
|---|---|
| Net profit (637 days) | 6,384,488 EUR |
| Annualized net profit | 3,658,301 EUR / year |
| Per installed MW | 73,166 EUR / MW / year |
| Per installed MWh | 36,583 EUR / MWh / year |
| Avg cycles/day | 0.95 |
| Days idle | 60% |
| Simultaneous charge/discharge | 0 |

**Scope-matched comparison:** Modo Energy reports ~55% of German BESS revenues come from ancillary services in 2026. Stripping this share from the full-stack benchmark (125k–190k EUR/MW/year) yields an arbitrage-only range of **56k–86k EUR/MW/year**. Our result (73,166 EUR/MW/year) sits within this range, confirming consistency.

---

## Sensitivity Analysis

| c_deg (EUR/MWh) | Net profit (EUR) | Annualized (EUR/year) | Δ vs base |
|---|---|---|---|
| 0 | 6,872,476 | 3,938,286 | +7.7% |
| **5 (base)** | **6,384,488** | **3,658,301** | — |
| 20 | 4,908,345 | 2,812,858 | −23% |
| 40 | 3,688,243 | 2,113,562 | −42% |

**Key findings:**

- Increasing c_deg from 0 to 40 reduces cycling by **96%** (2.63 → 0.09 cycles/day), while net profit declines by only **46%**.
- The base case (c_deg = 5) captures **93%** of the theoretical maximum profit while cutting cycling by **64%**.
- Unit margin stabilizes at ~69–70 EUR/MWh for c_deg ≥ 5.

**Conclusion:** Even under conservative degradation cost assumptions (c_deg = 20 and 40 EUR/MWh), the strategy remains strongly profitable, with annualized net profit still above **2M EUR/year**.

---

## Weather-Price Analysis

### Correlation with Price

Overall, solar radiation is the strongest driver (**r = -0.50**), followed by sunshine (-0.31) and wind speed (-0.23). Temperature, wind direction, and precipitation have negligible effects.

The dominant driver shifts seasonally:

| Season | Dominant driver | r |
|---|---|---|
| Winter | Wind speed | -0.37 |
| Spring | Solar radiation | -0.65 |
| Summer | Solar radiation | -0.55 |
| Autumn | Wind speed | -0.35 |

### Price Prediction

Two Random Forest models were trained to assess the predictive value of weather variables.

| Model | Features | R-squared |
|---|---|---|
| Model 1 | Weather + Time | 0.04 |
| Model 2 | Weather + Time + Lagged/Neighbor Prices | 0.93 |

- **Model 1 (weather + time only)**: R-squared = 0.04, very limited predictive power. Top features: solar radiation (0.30), wind speed (0.17), temperature (0.15).
- **Model 2 (adding lagged prices and neighbor prices)**: R-squared jumps to 0.93. However, feature importance is overwhelmingly dominated by **neighbor-country prices (94.6%)**, with weather variables contributing less than 2% combined.

### Interpretation

Day-ahead prices are driven primarily by market factors — neighbor-country prices, price autocorrelation, demand, and fuel costs — that are not captured by weather alone. Weather explains the *direction* of price pressure but not the *magnitude* of day-to-day variation. Once market coupling is accounted for, weather effects are transmitted indirectly through neighbor prices rather than directly driving German prices.

**Conclusion:** Weather variables alone are insufficient for standalone price forecasting. However, the strong weather-price correlation (r = -0.50 for radiation) confirms that weather remains a physical driver of price formation — explaining the midday price troughs and frequent negative prices.

---

## Installation

```bash
pip install pandas numpy matplotlib seaborn scikit-learn pyomo pyomo_windows
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
