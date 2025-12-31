
# ❄️ Snowflake Alpha Discovery

This repository demonstrates how hedge funds and quantitative asset managers can leverage **Snowflake** and **Snowpark Python** to build a modern alpha research environment. By unifying market data with alternative data, we identify trade signals and implement sophisticated strategies directly within the Data Cloud.

## 🚀 Overview

Traditional alpha discovery often suffers from "Data Gravity"—moving massive datasets to local environments for analysis. This project showcases a **"Compute-to-Data"** approach:

* **Snowpark Python:** For high-performance factor computation without moving data.
* **Cortex AI:** For institutional-grade sentiment analysis of news headlines using `SNOWFLAKE.CORTEX.SENTIMENT`.
* **Time-Series Analytics:** Using `ASOF JOIN` for robust factor lookbacks that handle data gaps and holidays natively.

## 🛠️ Tech Stack

* **Data Platform:** Snowflake.
* **Languages:** Python (Snowpark), SQL.
* **AI/ML:** Snowflake Cortex (LLM Sentiment Analysis).
* **Visualization:** Streamlit and Plotly.
* **Data Source:** Cybersyn Financial & Economic Essentials via Snowflake Marketplace.

---

## 📈 Multi-Factor Alpha Discovery

The core research focuses on three primary drivers of excess return (Alpha):

| Factor | Logic | Implementation |
| --- | --- | --- |
| **Momentum** | Capturing 20-day price trends or mean-reversion signals. | Computed via `ASOF JOIN` for robust historical price anchoring. |
| **Volatility** | Exploiting the low-volatility anomaly or high-volatility outperformance. | 20-day annualized standard deviation of daily returns. |
| **Sentiment** | Quantifying news impact to anticipate market moves before they are priced. | LLM-powered scoring using `SNOWFLAKE.CORTEX.SENTIMENT`. |

---

## ⚖️ Strategic Implementations

### 1. Long-Short Equity Strategy (Alpha Generation)

We combine momentum, volatility, and sentiment into a **Composite Alpha Score**. This strategy targets outperformance by capturing the "spread" between different stock profiles:

* **Systematic Selection:** Automatically identifying "Strong Buy" and "Strong Sell" candidates based on weighted factor ranks.
* **IC-Optimized Weighting:** Applying Information Coefficient (IC) weights to prioritize the most predictive signals.
* **Sector Exposure:** Monitoring net exposure across industry groups (Tech, Financials, Energy) to manage concentration risk.

### 2. Market-Neutral Pairs Trading (Statistical Arbitrage)

To generate returns independent of broader market direction, we implement a **Pairs Trading Engine**:

* **Statistical Correlation:** Identifying highly correlated intra-sector pairs (e.g., `GS` ↔ `JPM`) that move together over time.
* **Divergence Detection:** Calculating real-time Z-scores to identify when the price ratio between a pair deviates from its historical mean.
* **Convergence Execution:** Executing a market-neutral trade by shorting the outperformer and longing the underperformer.

---

## 📂 Production Readiness

* **Cortex UDFs:** Sentiment analysis is packaged into functions for scalable, real-time inference.
* **Automated Pipelines:** Logic is deployable via **Snowflake Tasks** and **Stored Procedures** to refresh alpha scores daily.
* **Secure Governance:** All research and trade signals are auditable within a single secure schema.

---

**Would you like me to help you create a "Getting Started" section for this README, or perhaps a section on how to interpret the Z-scores in the pairs trading module?**
