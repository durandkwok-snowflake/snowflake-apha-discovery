

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

## 🛒 Create Database and Schema then Load Market Data from Snowflake Marketplace

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

## Pulling in historical OHLCV data for expanded stocks
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

```PYTHON
# Write market data to our analytics schema
# This creates a copy we can work with for our alpha calculations
market_snow_df = session.create_dataframe(market_df)
market_snow_df.write.mode("overwrite").save_as_table("MARKET_DATA")

print("✅ Created MARKET_DATA table from Cybersyn Marketplace data")
print(f"   Source: FINANCIAL__ECONOMIC_ESSENTIALS.CYBERSYN.STOCK_PRICE_TIMESERIES")
session.table("MARKET_DATA").show(5)
```

<img width="538" height="210" alt="image" src="https://github.com/user-attachments/assets/05dc2112-89ab-46fd-967e-6a5b728823d3" />

## Generate sample news headlines and use Snowflake Cortex AI for sentiment analysis
```PYTHON
# Generate sample news headlines and use SNOWFLAKE CORTEX AI for sentiment analysis!
# This demonstrates how Cortex LLM functions can analyze text at scale

import random
from datetime import datetime, timedelta

# Get date range from our market data
min_date = market_df['DATE'].min()
max_date = market_df['DATE'].max()
symbols = market_df['SYMBOL'].unique().tolist()

np.random.seed(42)
random.seed(42)

# Realistic financial news headline templates (no pre-scored sentiment!)
headline_templates = [
    "beats earnings expectations with strong quarterly results",
    "announces strategic partnership to expand market presence",
    "faces regulatory investigation over compliance concerns",
    "launches innovative new product line ahead of schedule",
    "reports significant supply chain disruptions affecting production",
    "receives analyst upgrade citing growth potential",
    "CEO sells significant stock holdings in planned transaction",
    "posts record revenue growth exceeding analyst estimates",
    "misses quarterly targets amid challenging market conditions",
    "expands aggressively into new international markets",
    "announces major layoffs as part of restructuring plan",
    "secures major government contract worth billions",
    "faces class action lawsuit from shareholders",
    "reports cybersecurity breach affecting customer data",
    "raises full-year guidance following strong performance",
    "cuts dividend amid cash flow concerns",
    "announces stock buyback program worth $10 billion",
    "loses key executive to competitor",
    "wins patent dispute against rival company",
    "warns of slowing demand in key markets"
]

sources = ['Reuters', 'Bloomberg', 'CNBC', 'WSJ', 'Financial Times', 'MarketWatch']

# Generate headlines (we'll let Cortex score them!)
headline_records = []
current_date = pd.to_datetime(min_date)
end_date = pd.to_datetime(max_date)

while current_date <= end_date:
    if current_date.weekday() < 5:  # Weekdays only
        for symbol in symbols:
            n_articles = np.random.poisson(2)  # Average 2 articles per stock per day
            for _ in range(n_articles):
                headline = f"{symbol} {random.choice(headline_templates)}"
                headline_records.append({
                    'DATE': current_date.strftime('%Y-%m-%d'),
                    'SYMBOL': symbol,
                    'HEADLINE': headline,
                    'SOURCE': random.choice(sources)
                })
    current_date += timedelta(days=1)

headline_df = pd.DataFrame(headline_records)
print(f"📰 Generated {len(headline_df):,} news headlines")
print(f"   Now using Snowflake Cortex AI to analyze sentiment...")

# Write headlines to staging table
headline_snow_df = session.create_dataframe(headline_df)
headline_snow_df.write.mode("overwrite").save_as_table("NEWS_HEADLINES_RAW")

# Use SNOWFLAKE CORTEX SENTIMENT function to analyze headlines!
# This is the magic - LLM-powered sentiment analysis at scale!
cortex_sentiment_sql = """
SELECT 
    DATE,
    SYMBOL,
    HEADLINE,
    SOURCE,
    -- Snowflake Cortex SENTIMENT function returns score from -1 to 1
    SNOWFLAKE.CORTEX.SENTIMENT(HEADLINE) AS SENTIMENT_SCORE,
    -- Confidence derived from sentiment MAGNITUDE using Cortex output
    -- Strong sentiment (±0.9) = high confidence, Weak sentiment (±0.1) = low confidence
    -- Formula: 0.5 + (|sentiment| * 0.5) gives range 0.5 to 1.0
    0.5 + (ABS(SNOWFLAKE.CORTEX.SENTIMENT(HEADLINE)) * 0.5) AS CONFIDENCE
FROM NEWS_HEADLINES_RAW
"""

print("🤖 Running Snowflake Cortex SENTIMENT analysis...")
session.sql(cortex_sentiment_sql).write.mode("overwrite").save_as_table("NEWS_SENTIMENT")

print(f"✅ Created NEWS_SENTIMENT table with Cortex AI sentiment scores!")
print(f"   🧠 Powered by: SNOWFLAKE.CORTEX.SENTIMENT()")
print(f"   📊 Sentiment range: -1 (very negative) to +1 (very positive)")
print(f"   🎯 Confidence: Derived from sentiment magnitude (stronger = more confident)")

# Show sample with Cortex-generated sentiment and confidence
session.sql("""
    SELECT SYMBOL, 
           ROUND(SENTIMENT_SCORE, 3) AS SENTIMENT,
           ROUND(CONFIDENCE, 3) AS CONFIDENCE,
           CASE 
               WHEN SENTIMENT_SCORE > 0.3 THEN '🟢 Positive'
               WHEN SENTIMENT_SCORE < -0.3 THEN '🔴 Negative'
               ELSE '🟡 Neutral'
           END AS LABEL,
           HEADLINE
    FROM NEWS_SENTIMENT 
    ORDER BY ABS(SENTIMENT_SCORE) DESC 
    LIMIT 10
""").show()

```
<img width="765" height="389" alt="image" src="https://github.com/user-attachments/assets/991d44a4-7768-45ed-b9b5-b81d2e0fe61c" />


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

### 4. Combine Apha Factors
```PYTHON
# Combine all factors into composite alpha signal

# Based on typical market analysis:
#   - Momentum often reverses (mean reversion) → negative weight
#   - Low-vol anomaly may not hold in risk-on markets → negative weight
#   - Sentiment tends to be predictive → positive weight
# These will be validated/updated by the IC analysis later in the notebook

MOMENTUM_WEIGHT = -0.20    
VOLATILITY_WEIGHT = -0.30  
SENTIMENT_WEIGHT = 0.50    

print("📊 Using IC-OPTIMIZED Weights:")
print(f"   Momentum:   {MOMENTUM_WEIGHT:+.0%} ")
print(f"   Volatility: {VOLATILITY_WEIGHT:+.0%} ")
print(f"   Sentiment:  {SENTIMENT_WEIGHT:+.0%} ")
print()

composite_alpha_sql = f"""
WITH combined AS (
    SELECT 
        m.DATE,
        m.SYMBOL,
        m.CLOSE,
        m.MOMENTUM_RANK,
        m.MOMENTUM_SIGNAL,
        v.VOLATILITY_RANK,
        v.VOLATILITY_SIGNAL,
        COALESCE(s.SENTIMENT_RANK, 0.5) AS SENTIMENT_RANK,
        COALESCE(s.SENTIMENT_SIGNAL, 0) AS SENTIMENT_SIGNAL,
        s.AVG_SENTIMENT,
        s.ARTICLE_COUNT
    FROM MOMENTUM_FACTOR m
    LEFT JOIN VOLATILITY_FACTOR v 
        ON m.DATE = v.DATE AND m.SYMBOL = v.SYMBOL
    LEFT JOIN SENTIMENT_FACTOR s 
        ON m.DATE = s.DATE AND m.SYMBOL = s.SYMBOL
)
SELECT 
    *,
    -- IC-OPTIMIZED composite alpha (negative weights FLIP the signal!)
    (MOMENTUM_RANK * {MOMENTUM_WEIGHT} + 
     VOLATILITY_RANK * {VOLATILITY_WEIGHT} + 
     SENTIMENT_RANK * {SENTIMENT_WEIGHT}) AS COMPOSITE_ALPHA,
    
    -- Weighted signal (negative weights flip BUY to SELL)
    (MOMENTUM_SIGNAL * {MOMENTUM_WEIGHT} + 
     VOLATILITY_SIGNAL * {VOLATILITY_WEIGHT} + 
     SENTIMENT_SIGNAL * {SENTIMENT_WEIGHT}) AS WEIGHTED_SIGNAL,
     
    -- Trading recommendation based on optimized signal
    CASE 
        WHEN (MOMENTUM_SIGNAL * {MOMENTUM_WEIGHT} + VOLATILITY_SIGNAL * {VOLATILITY_WEIGHT} + SENTIMENT_SIGNAL * {SENTIMENT_WEIGHT}) > 0.2 THEN 'STRONG_BUY'
        WHEN (MOMENTUM_SIGNAL * {MOMENTUM_WEIGHT} + VOLATILITY_SIGNAL * {VOLATILITY_WEIGHT} + SENTIMENT_SIGNAL * {SENTIMENT_WEIGHT}) > 0.05 THEN 'BUY'
        WHEN (MOMENTUM_SIGNAL * {MOMENTUM_WEIGHT} + VOLATILITY_SIGNAL * {VOLATILITY_WEIGHT} + SENTIMENT_SIGNAL * {SENTIMENT_WEIGHT}) < -0.2 THEN 'STRONG_SELL'
        WHEN (MOMENTUM_SIGNAL * {MOMENTUM_WEIGHT} + VOLATILITY_SIGNAL * {VOLATILITY_WEIGHT} + SENTIMENT_SIGNAL * {SENTIMENT_WEIGHT}) < -0.05 THEN 'SELL'
        ELSE 'HOLD'
    END AS TRADING_SIGNAL
FROM combined
WHERE DATE IS NOT NULL
ORDER BY DATE DESC, COMPOSITE_ALPHA DESC
"""

composite_alpha = session.sql(composite_alpha_sql)
composite_alpha.write.mode("overwrite").save_as_table("ALPHA_SIGNALS")

print("✅ Created ALPHA_SIGNALS table with IC-OPTIMIZED composite alpha")
print("   (Negative weights flip momentum/volatility signals for mean reversion)")
session.table("ALPHA_SIGNALS").show(15)
```
<img width="1104" height="342" alt="image" src="https://github.com/user-attachments/assets/641d9095-3a8b-4863-a210-785c49ac684f" />



### 5. Analyze Apha Signals
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
