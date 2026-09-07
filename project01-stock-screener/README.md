# Project 01: Rule-Based Stock Screener

**Status:** Completed

> **Scope note:** this is a rule-based screening tool, not a backtested
> trading strategy. No P&L, position sizing, or transaction costs are
> modeled. Signal stability over time is evaluated in Phase 3b (rolling
> evaluation), not as a performance backtest.

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
* **Architecture:** Polyglot ELT pipeline — DuckDB for vectorized SQL 
  window functions (window functions, cross-sectional quantiles); Pandas 
  for downstream list-based aggregation.

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

### Phase 3: Rule-based screening — ✅ Completed
* Cross-sectional filtering logic — ✅ Completed
* Rolling screening evaluation (monthly turnover/stability) — ✅ Completed

**Universe expansion.** The investable universe comprises 15 individual
equities across 5 GICS sectors (Technology, Financials, Healthcare, Energy,
Consumer Staples) plus SPY. SPY is included as a full constituent of the
cross-sectional calculations — not held out as a passive-only benchmark.
This means SPY contributes to the daily volatility median used as the
relative filter threshold, and can itself receive an `is_investable` signal.
This is a deliberate choice: since SPY is a broadly diversified basket, its
volatility is structurally lower than most single-name constituents, which
pulls the daily median down and makes the relative volatility filter
modestly stricter for individual stocks than it would be with SPY excluded.

**Feature engineering.** Three features are computed for each asset,
on each trading day, with explicit cold-start protection (values are
forced to `NULL` until the full lookback window is available):
* `momentum_126d` — return over a ~6-month window (126 trading days)
* `rolling_vol_63d` — annualized volatility over a ~3-month window
  (63 trading days)
* `current_dd_126d` — drawdown from the rolling 6-month peak (not
  global max drawdown — this gives an ex-ante, regime-agnostic
  investability signal rather than a historical worst-case figure)

**Screening criteria.** Two absolute filters and one relative
(cross-sectional) filter, all of which must be true for an asset to be
flagged as investable on a given day:
* `momentum_126d > 0`
* `rolling_vol_63d` below the cross-sectional median volatility of the
  universe on that date
* `current_dd_126d >= -0.15`

### Known limitations
* Some parameters are hardcoded rather than configurable: the number of
  assets in the universe and the window sizes for momentum, volatility
  and drawdown. Changing any of these requires updating both the SQL
  queries and the validation assertions (see inline `NOTE` comments in
  `02_stock_screener.ipynb`).
* SPY is included in the cross-sectional median calculation (see
  "Universe expansion" above). An alternative design would exclude SPY
  from that calculation and use it only as a passive reference — this
  was considered but not implemented in the current version.

## Key Insights

### Phase 2
* SPY showed a contained max drawdown (~10.0%) versus single-stock
  components (AAPL ~16.6%), consistent with diversification benefits
* AAPL and MSFT both showed correlation >0.60 with SPY, reflecting their
  weight in systemic market beta

### Phase 3
* Cross-sectional screening yields ~29% investable signals (3,015 out of
  10,400 observations), indicating the filter is selective without being
  overly restrictive
* Monthly portfolio size (EoM snapshots) ranges from 3 to 8 assets across
  the evaluation period (Nov 2022 – Dec 2024), with troughs of 3 assets
  in February 2023, May 2023, October 2023, and December 2024
* NVDA and MSTR (the two highest-volatility constituents) fail the
  relative volatility filter in every observed month, while defensive
  names (PG, KO, JNJ, PEP) and SPY pass most consistently. This is a
  direct consequence of the cross-sectional median volatility design
  (see "Universe expansion"): high-beta growth/crypto-proxy assets are
  structurally excluded regardless of momentum or drawdown conditions
* Portfolio troughs loosely coincide with known periods of market stress
  (e.g. Oct 2023 Treasury yield spike), though a monthly-resolution
  screen cannot establish causality or intra-month dynamics

## Dataset
* **Phase 1–2 (EDA):** AAPL, MSFT, SPY — daily OHLCV via yfinance,
  2023–2024
* **Phase 3 (Screening):** expanded to 16 assets — NVDA, MSFT, MSTR, JPM,
  BAC, MS, LLY, JNJ, ABBV, XOM, CVX, COP, PG, KO, PEP, SPY — daily OHLCV
  via yfinance, June 2022–2025 (start date moved back to absorb the
  126-day burn-in period required by the rolling features)

## Notebooks
* `01_eda_market_regime.ipynb`
* `02_stock_screener.ipynb`