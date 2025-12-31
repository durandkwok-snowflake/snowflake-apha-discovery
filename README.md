To incorporate your screenshot as a sample of the output, I have updated the **Strategic Implementations** section of the README. This visual sample directly correlates to the signals generated in your notebook's "Today's Alpha Signals" query.

---

# ❄️ Snowflake Alpha Discovery

This repository demonstrates how hedge funds and quantitative asset managers can leverage **Snowflake** and **Snowpark Python** to build a modern alpha research environment. By unifying market data with alternative data, we identify trade signals and implement sophisticated strategies directly within the Data Cloud.

## 🚀 Overview

The project shifts quantitative research from local environments to the data, utilizing:

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

## 🛒 Load Market Data from Snowflake Marketplace

We utilize real market data from the Cybersyn Financial & Economic Essentials dataset available on the Snowflake Marketplace.
	•	Zero ETL: Hedge funds can instantly access high-quality historical OHLCV data without building complex ingestion pipelines.
	•	Dynamic Stock Universe: The notebook expands beyond standard benchmarks to include Tech giants, Financials, Payments, Healthcare, Consumer, Energy, and Industrials.
	•	Freshness: Data is queried directly from the provider's share, ensuring research is always conducted on the most up-to-date figures.

```PYTHON
# Create database and schema for hedge fund analytics
session.sql("CREATE DATABASE IF NOT EXISTS HEDGE_FUND_DEMO").collect()
session.sql("CREATE SCHEMA IF NOT EXISTS HEDGE_FUND_DEMO.ANALYTICS").collect()
session.sql("USE SCHEMA HEDGE_FUND_DEMO.ANALYTICS").collect()

print("✅ Created HEDGE_FUND_DEMO.ANALYTICS schema")

```

## LOADING UNIVERSE
```PYTHON
# Load REAL market data from Cybersyn (Snowflake Marketplace)
# This pulls historical OHLCV data for our expanded stock universe
# Includes: Tech, Financials, Payments, Healthcare, Consumer, Energy, Industrials

market_data_sql = """
SELECT 
    t.DATE,
    t.TICKER AS SYMBOL,
    MAX(CASE WHEN t.VARIABLE_NAME = 'Pre-Market Open' THEN t.VALUE END) AS OPEN,
    MAX(CASE WHEN t.VARIABLE_NAME = 'All-Day High' THEN t.VALUE END) AS HIGH,
    MAX(CASE WHEN t.VARIABLE_NAME = 'All-Day Low' THEN t.VALUE END) AS LOW,
    MAX(CASE WHEN t.VARIABLE_NAME = 'Post-Market Close' THEN t.VALUE END) AS CLOSE,
    MAX(CASE WHEN t.VARIABLE_NAME = 'Nasdaq Volume' THEN t.VALUE END) AS VOLUME
FROM FINANCIAL__ECONOMIC_ESSENTIALS.CYBERSYN.STOCK_PRICE_TIMESERIES t
WHERE t.TICKER IN (
        -- Tech Giants
        'AAPL', 'GOOGL', 'MSFT', 'AMZN', 'META', 'NVDA', 'TSLA',
        -- Financials (Investment Banks)
        'JPM', 'GS', 'MS', 'BAC', 'WFC', 'C',
        -- Payments & FinTech
        'V', 'MA', 'PYPL', 'SQ',
        -- Healthcare & Pharma
        'JNJ', 'UNH', 'PFE', 'MRK', 'ABBV',
        -- Consumer & Retail
        'WMT', 'COST', 'HD', 'NKE', 'SBUX',
        -- Energy
        'XOM', 'CVX', 'COP',
        -- Industrials
        'CAT', 'BA', 'UPS', 'HON'
    )
  AND t.VARIABLE_NAME IN ('Pre-Market Open', 'All-Day High', 'All-Day Low', 'Post-Market Close', 'Nasdaq Volume')
  AND t.DATE >= DATEADD(year, -1, CURRENT_DATE())  -- Last 1 year of data
GROUP BY t.DATE, t.TICKER
HAVING CLOSE IS NOT NULL  -- Ensure we have closing prices
ORDER BY t.TICKER, t.DATE
"""

market_df = session.sql(market_data_sql).to_pandas()
print(f"✅ Loaded {len(market_df):,} market data records from Cybersyn")
print(f"   Date range: {market_df['DATE'].min()} to {market_df['DATE'].max()}")
print(f"   Symbols ({len(market_df['SYMBOL'].unique())} stocks): {', '.join(sorted(market_df['SYMBOL'].unique()))}")
market_df.head(10)
```
<img width="1106" height="403" alt="image" src="https://github.com/user-attachments/assets/97421957-d097-4fa2-820e-e491a3949ad9" />



## 📈 Multi-Factor Alpha Architecture

The research engine focuses on three primary drivers of excess return (Alpha), utilizing **Cross-Sectional Ranking** to ensure signals are comparable across the universe.

### 1. Momentum Alpha (Relative Strength)

* **Logic:** Captures stocks outperforming or underperforming their peers over a ~20 trading day window.
* **Implementation:** Uses `ASOF JOIN` to anchor the price exactly 28 calendar days ago, providing a robust calculation even with market gaps.
* **Normalization:** Raw returns are converted into a `PERCENT_RANK()` (0.0 to 1.0).

### 2. Volatility Alpha (Risk-Adjusted Anomaly)

* **Logic:** Measures the annualized standard deviation of daily returns over the last 20 sessions.
* **Implementation:** Utilizes Snowpark window functions (`STDDEV` over `19 PRECEDING`).
* **Normalization:** Stocks are ranked such that a high rank (1.0) represents the **lowest** volatility.

### 3. Sentiment Alpha (Alternative Data Edge)

* **Logic:** Quantifies news mood using Large Language Models (LLMs).
* **Implementation:** Leverages `SNOWFLAKE.CORTEX.SENTIMENT()` to score headlines from -1 to 1.
* **Normalization:** Daily scores are averaged and ranked cross-sectionally.

### 4. Analyze Apha Signals
```python
# Get latest signals - Today's trading recommendations
latest_signals_sql = """
SELECT 
    SYMBOL,
    ROUND(CLOSE, 2) AS PRICE,
    ROUND(MOMENTUM_RANK, 3) AS MOMENTUM,
    ROUND(VOLATILITY_RANK, 3) AS VOLATILITY,
    ROUND(SENTIMENT_RANK, 3) AS SENTIMENT,
    ROUND(COMPOSITE_ALPHA, 3) AS ALPHA_SCORE,
    TRADING_SIGNAL
FROM ALPHA_SIGNALS
WHERE DATE = (SELECT MAX(DATE) FROM ALPHA_SIGNALS)
ORDER BY COMPOSITE_ALPHA DESC
"""

print("🎯 TODAY'S ALPHA SIGNALS")
print("=" * 60)
session.sql(latest_signals_sql).show()

```

<img width="686" height="298" alt="image" src="https://github.com/user-attachments/assets/08070486-89db-4bae-8d0c-a654149db583" />

---

## ⚖️ Strategic Implementations

### 1. Long-Short Equity Strategy

Factors are combined into a **Composite Alpha Score** to target outperformance by capturing the "spread" between relative winners and losers:

* **Weighted Signal Calculation:** 
* **Signal Thresholds:**
* **STRONG_BUY**: Alpha Score > 0.2.
* **BUY**: Alpha Score > 0.05.
* **HOLD**: Alpha Score between -0.05 and 0.05.
* **SELL**: Alpha Score < -0.05.
* **STRONG_SELL**: Alpha Score < -0.2.



**Sample Strategy Output:**
Below is a sample of the real-time trading recommendations generated by the model:
<img width="421" height="307" alt="image" src="https://github.com/user-attachments/assets/ca9c5c5c-e001-4e67-8a8c-16e7cdcfec36" />


### 2. Market-Neutral Pairs Trading (Statistical Arbitrage)

To generate returns independent of market direction, the engine includes a pairs trading module:

* **Correlation Analysis:** Identifying highly correlated intra-sector pairs (e.g., `GS` ↔ `JPM`).
* **Divergence Detection:** Calculating real-time Z-scores to identify when price ratios deviate from their historical mean.
* **Convergence Execution:** Executing a market-neutral trade by shorting the outperformer and longing the underperformer, profiting as the pair converges.

---

## 📂 Production Readiness

* **Cortex UDFs:** Sentiment analysis is packaged into functions for scalable, real-time inference.
* **Automated Pipelines:** Logic is deployable via **Snowflake Tasks** and **Stored Procedures** to refresh alpha scores daily.
* **Secure Governance:** All research and trade signals are auditable within a single secure schema.
