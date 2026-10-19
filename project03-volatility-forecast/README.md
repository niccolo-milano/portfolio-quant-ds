# Project 03: Volatility Forecasting & VaR Backtesting (WIP)

> **Status:** Work in Progress — **Phase 1 Completed** (Data Architecture, Vendor Quality Audit & Stylized Facts). Phase 2 (Walk-Forward GARCH/GJR & HAR Forecasting) and Phase 3 (VaR Backtesting) in progress.

## 1. Project Overview & Financial Motivation
Financial return series exhibit near-zero linear autocorrelation in levels ($r_t$) but strong temporal persistence in conditional variance ($r_t^2$), alongside leptokurtic tails and asymmetric reactions to negative shocks. 

This project builds an end-to-end volatility forecasting and model validation pipeline across European equity indices and foreign exchange, comparing parametric conditional heteroskedasticity models (**GARCH(1,1)-t**, **GJR-GARCH(1,1)-t**) against a linear long-memory baseline (**HAR**).

## 2. Asset Universe & Data Quality Audit (2009–2025)
 raw daily `OHLC` data are extracted via `yfinance` and frozen in `data/raw_market_data_2009_2025.parquet` for exact reproducibility. All series are mapped onto the **EURO STOXX 50** master trading calendar (`2009-01-05` to `2025-12-30`, **4,263 trading days** per asset; **12,789 total rows**).

| Ticker | Asset Class | Total Obs (`n_obs`) | Holiday FFill (`n_ffilled`) | Repaired OHLC (`n_repaired`) | Zero-Return Days (`n_zero_return`) | Scorable Days (`n_scorable`) |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **`^STOXX50E`** | European Equity Index | 4,263 | 0 | 0 | 2 | 4,260 |
| **`FTSEMIB.MI`** | Italian Equity Index | 4,263 | 15 | 0 | 21 | 4,226 |
| **`EURUSD=X`** | Foreign Exchange | 4,263 | 5 | 5 | 26 | 4,226 |

### Data Hygiene & Repair Policy
* **Causal Holiday Alignment:** Missing closes on local holidays are causally forward-filled (`is_ffilled`), while intraday `Open/High/Low` remain `NaN`.
* **Vendor OHLC Repair (`ohlc_tol_pct = 0.1%`):** Any observation where `Close` falls outside `[Low, High]` by more than $0.1\%$ of price is treated as a vendor feed corruption (`is_repaired = True`), masked to `NaN`, and causally forward-filled. This rule catches 5 corrupted prints on `EURUSD=X` (e.g., `2010-07-01`, $+2.16\%$ above daily High) while leaving genuine market stress untouched (`kurtosis_raw = 2.32` vs `kurtosis_repaired = 2.49`).
* **Scorable Mask (`is_scorable`):** Non-trading days, repaired days, zero-return days, and post-gap days (`follows_gap`) are excluded from loss evaluation so that multi-day returns and zero variance proxies do not distort `QLIKE`.

## 3. Empirical Stylized Facts (2009–2025 Sample)

| Ticker | Excess Kurtosis (Fisher) | Min Daily Return (%) | Max Daily Return (%) | Days $\Vert{}r_t\Vert{} > 10\%$ | Leverage $\text{Corr}(r_t, r_{t+1}^2)$ | $\mathbb{E}[r_{t+1}^2 \mid r_t < 0]$ | $\mathbb{E}[r_{t+1}^2 \mid r_t > 0]$ |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`^STOXX50E`** | 6.91 | -13.24% | +9.85% | 1 | -0.0840 | 1.9845 | 1.4347 |
| **`FTSEMIB.MI`** | 9.92 | -18.55% | +10.68% | 4 | -0.0788 | 2.7135 | 1.9823 |
| **`EURUSD=X`** | 2.49 | -3.31% | +3.40% | 0 | +0.0025 | 0.3110 | 0.3093 |

1. **Heavy Tails:** Positive excess kurtosis across all three series motivates **Student-t innovations** over Gaussian likelihoods.
2. **Volatility Clustering:** Raw returns $r_t$ show near-zero autocorrelation, whereas squared returns $r_t^2$ exhibit significant positive autocorrelation across 40 lags (slow decay on equity indices; flat persistence on `EURUSD=X`), motivating **GARCH(1,1)** dynamics.
3. **Asymmetric Leverage Effect:** On `^STOXX50E` and `FTSEMIB.MI`, average next-day squared returns are ~37–38% higher following negative return days than positive return days, motivating the asymmetric **GJR-GARCH** specification. Conversely, `EURUSD=X` shows symmetric next-day variance (`0.311` vs `0.309`).

## 4. Known Limitations (Phase 1)
* **2009 Sample Start:** The sample starts in January 2009 because a preliminary audit of 2008 `yfinance` data for `EURUSD=X` revealed systematic vendor spike-and-reversal corruptions (inflating sample excess kurtosis to 99.70). Starting in 2009 preserves a uniform 17-year cross-asset window while excluding the acute late-2008 phase of the Global Financial Crisis.
* **Intraday Range Noise on FX:** Minor `OHLC` discrepancies below the $0.1\%$ tolerance ($0 < v_{\text{rel}} \le 0.1\%$) are present in free `yfinance` FX feeds due to asynchronous close/range timestamps; their impact on range-based proxies (`Parkinson`) will be evaluated in Phase 2.