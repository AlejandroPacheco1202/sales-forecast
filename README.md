# Sales Demand Forecasting: A Hybrid ML Approach

**A complete walkthrough of building a production sales forecast model: from data exploration to validation to deployment.**

---

## 📋 Executive Summary

This project demonstrates how to build a **sales forecast that outperforms traditional methods** by combining:
1. **Statistical baseline** (seasonal factors + day-of-week patterns)
2. **Machine Learning** (Gradient Boosting with Poisson loss)
3. **Hybrid reconciliation** (best of both worlds)

**Key Results (backtested on May-June 2026):**

| Metric | Previous Method | New Forecast | Improvement |
|--------|-----------------|--------------|-------------|
| **Daily accuracy (store-level)** | 27.0% error | 12.6% error | **↓ 53% better** |
| **Monthly accuracy (SKU-level)** | 25.1% error | 23.1% error | ↓ 8% better |
| **Forecast bias** | −10.5% | −2.2% | **↓ 79% better** |

The new forecast catches **real patterns in your business** (weekends vs. weekdays, monthly seasonality, special events) that the linear method missed.

---

## 🎯 Problem Statement

**The old method:**
- Calculated the average of the last 2 months
- Divided that evenly across every day of the next month
- Assumed all Mondays are equal to all Wednesdays, etc.

**What was wrong:**
- Saturday sales are **2–3× higher** than Tuesday sales, but the forecast didn't know
- September (winter down-cycle) is **14% weaker** than May, but the forecast treated all months the same
- Forecast systematically **underestimated by −10.5%** over a 2-month period

**The cost:**
- Production team didn't know how much to bake each day → stockouts or waste
- Procurement bought based on a flat forecast → wrong inventory levels
- Decisions were made without understanding the real demand curve

---

## 💡 Solution: Hybrid Forecast Model

### Three Components

#### 1. **Level: Recent Base** (What is the current demand?)

```
Base per product = average sales, May-June 2026, per store
```

- Captures current business momentum (2026 is ~19% above 2025)
- Anchors in recent data, not the distant past
- Result: ~45,908 units/month baseline

#### 2. **Shape: Monthly Seasonality** (Which months are strong/weak?)

```
Seasonal factor = sales(month in 2025) / average(May-June 2025)
```

- July–August: ×1.03 (winter break, school holidays)
- September: ×0.86 (weakest month of the year)
- October: ×0.98 (recovery)
- November–December: ×0.95–0.85 (end of year)

Calculated **by product family** so new products inherit the seasonality of their category. Capped at [0.4, 2.5] to prevent outliers.

#### 3. **Timing: Daily Distribution** (When within the month?)

```
daily_sales = monthly_total × (model prediction for this day / Σ predictions in month)
```

**Model:** Gradient Boosting Regressor (Poisson loss)
- Trained on Jan 2025 – Jun 2026 (16 months of daily data)
- 146,664 rows including zeros (53% of days had zero sales for a given SKU)
- Key features:
  - `roll28`: mean of last 28 days (level persistence)
  - `dow_mean4`: mean of same day of week over past 4 weeks (weekly pattern)
  - `dow`, `mes`, `dia_mes`: calendar features

**Why Poisson loss?** Demand is a count (non-negative integers, many zeros). Poisson is the standard loss for retail demand.

**Why not ARIMA/SARIMA?** Those model a *single* time series. You have ~350 product-store combinations. Gradient Boosting learns across all series at once (cross-learning).

---

## 📊 Methodology: Backtesting Protocol

To prove the model works **without cheating**, we used a proper temporal split:

1. **Data available:** Jan 2025 – Apr 2026
2. **Forecast:** May–June 2026 (the model doesn't see these months)
3. **Measure:** Compare forecast vs. actual sales in May–June 2026

**Results on the holdout set:**

| Model | Store-Day WMAPE | SKU-Month WMAPE | Bias |
|-------|---|---|---|
| Previous Method | 27.0% | 25.1% | −10.5% |
| Method A (Baseline) | 14.0% | 23.1% | −2.2% |
| **Hybrid (A + B)** | **12.6%** | **23.1%** | **−2.2%** |

---

## 🔍 Feature Importance

```
dow_mean4:     0.886  ← Day-of-week pattern (biggest signal)
roll28:        0.697  ← Recent average (second biggest)
dow:           0.118  ← Calendar day of week
FAMILIA:       0.106  ← Product family
mes:           0.055  ← Month (seasonality)
Tienda:        0.050  ← Store
[others]:      < 0.05
```

**Key insight:** ~90% of value comes from "recent average" + "day-of-week pattern." A demand forecast is an intelligent recent average.

---

## 📁 Project Structure

```
├── 01_carga_limpieza.ipynb          # Load & clean data
├── 02_eda.ipynb                     # Explore patterns
├── 03_modelo_baseline.ipynb         # Statistical baseline
├── 04_modelo_gbm.ipynb              # ML model (Gradient Boosting)
├── 05_presupuesto_final.ipynb       # Final forecast & export
├── Sales_Forecast_Anonymized.xlsx   # Output forecast
├── environment.yml                   # Dependencies
└── README.md                         # This file
```

---

## ⚙️ Setup & Reproduction

### Install Dependencies

```bash
conda env create -f environment.yml
conda activate sales-forecast
```

### Run the Pipeline

```bash
jupyter lab 01_carga_limpieza.ipynb
jupyter lab 02_eda.ipynb
jupyter lab 03_modelo_baseline.ipynb
jupyter lab 04_modelo_gbm.ipynb
jupyter lab 05_presupuesto_final.ipynb
```

---

## 🔐 Data Anonymization

**All values in this repository have been scaled by 0.2 (÷5) for confidentiality.**

- Store names: Ugarteche → StoreA, Sinclair → StoreB
- Products: Medialuna Dulce → Product001, etc.
- Quantities: all multiplied by 0.2

**The methodology is fully reproducible** with any dataset; all logic is in the code.

---

## 🎓 Portfolio Skills Demonstrated

- Data engineering (ETL, causal feature derivation)
- Exploratory data analysis (seasonality, ABC curves)
- Baseline modeling (competitive, interpretable)
- Machine Learning (Gradient Boosting, Poisson regression)
- Evaluation & validation (WMAPE, temporal backtesting)
- Production & deployment (Excel export, reconciliation)
- Communication (executive summary, documentation)

---

## 📚 References

- M5 Forecasting Competition (Walmart, Kaggle): retail demand; Gradient Boosting won
- Poisson Regression for Count Data: standard in demand modeling
- Temporal Cross-Validation: Hyndman's forecasting best practices
- WMAPE Metric: industry standard in retail forecasting

---

**Created:** July 2026  
**Status:** Complete (backtest validated)
