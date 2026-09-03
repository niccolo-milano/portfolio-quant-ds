# Project 01: Rule-Based Stock Screener

**Status:** Work in Progress — Phase 2 of 3 completed

## Overview
This project builds a rule-based stock screener, starting from exploratory
data analysis of financial time-series (return distributions, volatility
regimes, correlation structure, drawdown) and moving toward rule-based
filtering of an equity universe.

## Tech Stack
* **SQL / Data pipeline:** DuckDB (in-memory analytical SQL)
* **Time-series manipulation:** Pandas, NumPy
* **Statistics:** vectorized statistical moments (Fisher kurtosis),
  annualized volatility (√252 scaling), max drawdown
* **Visualization:** Plotly (interactive), Seaborn / Matplotlib

## Roadmap

### Phase 1: Data pipeline — ✅ Completed
* Daily OHLCV extraction via `yfinance`
* DuckDB pipeline computing daily returns via SQL window functions (`LAG`)
* Explicit cold-start and NaN handling, with structural assertions

### Phase 2: Exploratory Data Analysis — ✅ Completed
* Distribution analysis: return histograms and boxplots, leptokurtic
  behavior in single-stock assets (AAPL, MSFT) vs. benchmark (SPY)
* 30-day rolling annualized volatility, highlighting systemic shocks
  (March 2023, August 2024)
* Pearson correlation matrix across assets
* Max drawdown per asset

### Phase 3: Rule-based screening — 🚧 In progress
* Universe expansion beyond AAPL/MSFT/SPY
* Feature engineering for screening criteria (trend, momentum, volatility
  and drawdown constraints)
* Rule-based filtering logic

## Key Insights (Phase 2)
* SPY showed a contained max drawdown (~10.0%) versus single-stock
  components (AAPL ~16.6%), consistent with diversification benefits
* AAPL and MSFT both showed correlation >0.60 with SPY, reflecting their
  weight in systemic market beta

## Dataset
AAPL, MSFT, SPY — daily OHLCV via yfinance, 2023–2025

## Notebook
`01_eda_market_regime.ipynb`