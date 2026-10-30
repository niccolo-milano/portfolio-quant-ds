# Project 03: Volatility Forecasting & VaR Backtesting (WIP)

> **Status:** Work in Progress. **Phase 1** (data architecture, vendor quality audit, stylized facts) and **Phase 2** (walk-forward GARCH/GJR and HAR-type forecasting, with out-of-sample evaluation) are completed. **Phase 3** (VaR backtesting) has not been started.

## 1. Project Overview & Financial Motivation
Financial return series exhibit near-zero linear autocorrelation in levels ($r_t$) but strong temporal persistence in conditional variance ($r_t^2$), alongside leptokurtic tails and asymmetric reactions to negative shocks.

This project builds a volatility forecasting and model validation pipeline across European equity indices and foreign exchange. It compares parametric conditional heteroskedasticity models (**GARCH(1,1)-t**, **GJR-GARCH(1,1)-t**) against linear **HAR-type** baselines, one fitted on a range-based variance proxy and one on squared returns.

## 2. Asset Universe & Data Quality Audit (2009–2025)
Raw daily `OHLC` data are extracted via `yfinance` and frozen in `data/raw_market_data_2009_2025.parquet` for exact reproducibility. All series are mapped onto the **EURO STOXX 50** master trading calendar (`2009-01-05` to `2025-12-30`, **4,263 trading days** per asset; **12,789 total rows**).

| Ticker | Asset Class | Total Obs (`n_obs`) | Holiday FFill (`n_ffilled`) | Repaired OHLC (`n_repaired`) | Zero-Return Days (`n_zero_return`) | Scorable Days (`n_scorable`) |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| **`^STOXX50E`** | European Equity Index | 4,263 | 0 | 0 | 2 | 4,260 |
| **`FTSEMIB.MI`** | Italian Equity Index | 4,263 | 15 | 0 | 21 | 4,226 |
| **`EURUSD=X`** | Foreign Exchange | 4,263 | 5 | 5 | 26 | 4,226 |

### Data Hygiene & Repair Policy
* **Causal Holiday Alignment:** Missing closes on local holidays are causally forward-filled (`is_ffilled`), while intraday `Open/High/Low` remain `NaN`.
* **Vendor OHLC Repair (`ohlc_tol_pct = 0.1%`):** Any observation where `Close` falls outside `[Low, High]` by more than $0.1\%$ of price is treated as a vendor feed error (`is_repaired = True`), masked to `NaN`, and causally forward-filled. The rule catches 5 prints on `EURUSD=X` (e.g., `2010-07-01`, Close $2.16\%$ above the daily High). Repairs move the excess kurtosis of `EURUSD=X` from 2.32 to 2.49 and leave the equity indices unchanged.
* **Spike Scan:** one spike-and-reversal is flagged (`FTSEMIB.MI`, `2020-03-13`); it was cross-checked against `^STOXX50E` and kept as a genuine market move.
* **Scorable Mask (`is_scorable`):** Non-trading days, repaired days, zero-return days, and post-gap days (`follows_gap`) are excluded from loss evaluation so that multi-day returns and zero variance proxies do not distort `QLIKE`.

## 3. Empirical Stylized Facts (2009–2025 Sample)

| Ticker | Excess Kurtosis (Fisher) | Min Daily Return (%) | Max Daily Return (%) | Days $\Vert{}r_t\Vert{} > 10\%$ | Leverage $\text{Corr}(r_t, r_{t+1}^2)$ | $\mathbb{E}[r_{t+1}^2 \mid r_t < 0]$ | $\mathbb{E}[r_{t+1}^2 \mid r_t > 0]$ |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **`^STOXX50E`** | 6.91 | -13.24% | +9.85% | 1 | -0.0840 | 1.9845 | 1.4347 |
| **`FTSEMIB.MI`** | 9.92 | -18.55% | +10.68% | 4 | -0.0788 | 2.7135 | 1.9823 |
| **`EURUSD=X`** | 2.49 | -3.31% | +3.40% | 0 | +0.0025 | 0.3110 | 0.3093 |

1. **Heavy Tails:** Positive excess kurtosis across all three series motivates **Student-t innovations** over Gaussian likelihoods.
2. **Volatility Clustering:** Raw returns $r_t$ show near-zero autocorrelation, whereas squared returns $r_t^2$ are clearly positively autocorrelated, with slow decay on the equity indices and a persistent plateau on `EURUSD=X`. This motivates **GARCH(1,1)** dynamics.
3. **Asymmetric Response:** On `^STOXX50E` and `FTSEMIB.MI`, average next-day squared returns are about 37–38% higher after negative return days than after positive ones, which motivates the asymmetric **GJR-GARCH** specification. `EURUSD=X` shows no such asymmetry (`0.311` vs `0.309`). These are descriptive comparisons without confidence intervals.

## 4. Phase 2: Walk-Forward Forecasting & Evaluation (2020–2025)

### Design
* **Test window:** `2020-01-03` to `2025-12-30`, 1,510 days per series (4,530 forecasts per model). Expanding window, first calibrated on 2009–2019.
* **GARCH(1,1)-t and GJR-GARCH(1,1)-t** (`arch`): constant mean, Student-t innovations, percentage returns (`rescale=False`). Parameters are re-estimated every 21 trading days (72 refits per series); between refits the conditional variance is filtered daily with fixed parameters. The forecast for day $t$ uses data through $t-1$ (asserted in code).
* **Two HAR-type baselines** (adapted from Corsi, 2009, who used intraday realized variance): expanding-window OLS with the same 21-day refit, on lagged daily, 5-day and 22-day means of a base series.
  * **`HAR-Parkinson`:** base series is the Parkinson High-Low variance estimator. A causal guard replaces implausible High/Low ranges and missing ranges with the squared return. Over the full sample it flags 27 days: 25 missing ranges (holidays and repaired days) and 2 implausible ranges (one on `EURUSD=X`, one on `^STOXX50E`), none in the test window. The guard constants (20 and 0.5) were chosen after inspecting the data.
  * **`HAR-SqRet`:** base series is the squared return. Like GARCH, it uses closes only and targets the same quantity.
* **Evaluation:** QLIKE (primary) and MSE against two proxies (squared return, Parkinson range), on the same scorable rows for every model within a series: 1,510 (`^STOXX50E`), 1,501 (`FTSEMIB.MI`), 1,503 (`EURUSD=X`). Loss differences come with 95% moving-block-bootstrap intervals (21-day blocks, 1,000 draws, seed 42), 72 in total (36 comparisons × 2 losses).

### Parameter Estimates (last refit, training data through early December 2025; no standard errors)

| Ticker | Model | $\hat\omega$ | $\hat\alpha$ | $\hat\gamma$ | $\hat\beta$ | $\hat\nu$ | Persistence |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| `^STOXX50E` | GARCH-t | 0.0341 | 0.1170 | – | 0.8687 | 5.56 | 0.9857 |
| `^STOXX50E` | GJR-t | 0.0367 | 0.0000 | 0.2233 | 0.8681 | 6.15 | 0.9798 |
| `FTSEMIB.MI` | GARCH-t | 0.0390 | 0.0941 | – | 0.8909 | 6.13 | 0.9851 |
| `FTSEMIB.MI` | GJR-t | 0.0462 | 0.0000 | 0.1595 | 0.8931 | 6.65 | 0.9728 |
| `EURUSD=X` | GARCH-t | 0.0009 | 0.0342 | – | 0.9631 | 7.49 | 0.9972 |
| `EURUSD=X` | GJR-t | 0.0007 | 0.0196 | 0.0192 | 0.9685 | 7.62 | 0.9977 |

On both equity indices $\hat\alpha$ sits on its lower bound, so the whole shock response is carried by $\hat\gamma$ and positive returns add nothing to next-day variance beyond the constant. Boundary estimates make standard inference unreliable, so these values are descriptive. On `EURUSD=X`, $\hat\gamma$ is about the size of $\hat\alpha$; the point estimates alone do not settle whether the asymmetry is real.

HAR-type coefficients (last refit):

| Ticker | Variant | $\hat\beta_0$ | $\hat\beta_d$ | $\hat\beta_w$ | $\hat\beta_m$ | Sum of $\hat\beta_d,\hat\beta_w,\hat\beta_m$ |
| :--- | :--- | :---: | :---: | :---: | :---: | :---: |
| `^STOXX50E` | HAR-Parkinson | 0.2237 | 0.2440 | 0.4734 | 0.0777 | 0.7951 |
| `FTSEMIB.MI` | HAR-Parkinson | 0.3453 | 0.3894 | 0.2641 | 0.1119 | 0.7655 |
| `EURUSD=X` | HAR-Parkinson | 0.0384 | 0.0356 | 0.2870 | 0.5020 | 0.8246 |
| `^STOXX50E` | HAR-SqRet | 0.5544 | -0.1072 | 0.4754 | 0.3008 | 0.6690 |
| `FTSEMIB.MI` | HAR-SqRet | 0.9670 | 0.0040 | 0.3358 | 0.2423 | 0.5821 |
| `EURUSD=X` | HAR-SqRet | 0.1075 | -0.0057 | 0.0856 | 0.5695 | 0.6494 |

### Out-of-Sample Results

Mean QLIKE (lower is better):

| Ticker | Proxy | GARCH-t | GJR-t | HAR-Parkinson | HAR-SqRet |
| :--- | :--- | :---: | :---: | :---: | :---: |
| `^STOXX50E` | squared return | 1.584 | 1.539 | 1.607 | 1.613 |
| `^STOXX50E` | Parkinson | 0.547 | 0.510 | 0.395 | 0.603 |
| `FTSEMIB.MI` | squared return | 1.548 | 1.506 | 1.562 | 1.605 |
| `FTSEMIB.MI` | Parkinson | 0.555 | 0.536 | 0.396 | 0.682 |
| `EURUSD=X` | squared return | 1.443 | 1.451 | 1.327 | 1.462 |
| `EURUSD=X` | Parkinson | 0.369 | 0.374 | 0.309 | 0.413 |

Paired QLIKE differences, mean [95% interval]; negative means the first model has the lower loss; **bold** = interval excludes zero (HAR-P = HAR-Parkinson, HAR-S = HAR-SqRet):

| Ticker | Proxy | GJR − GARCH | GJR − HAR-S | GARCH − HAR-S | HAR-P − HAR-S |
| :--- | :--- | :---: | :---: | :---: | :---: |
| `^STOXX50E` | squared return | **−0.045** [−0.082, −0.008] | **−0.074** [−0.129, −0.023] | −0.029 [−0.061, +0.007] | −0.006 [−0.069, +0.071] |
| `^STOXX50E` | Parkinson | **−0.037** [−0.062, −0.014] | **−0.093** [−0.126, −0.058] | **−0.056** [−0.075, −0.032] | **−0.208** [−0.244, −0.172] |
| `FTSEMIB.MI` | squared return | **−0.042** [−0.079, −0.012] | **−0.099** [−0.151, −0.049] | **−0.057** [−0.093, −0.016] | −0.043 [−0.111, +0.037] |
| `FTSEMIB.MI` | Parkinson | −0.020 [−0.040, +0.0001] | **−0.146** [−0.185, −0.104] | **−0.126** [−0.158, −0.092] | **−0.286** [−0.329, −0.238] |
| `EURUSD=X` | squared return | +0.007 [−0.005, +0.027] | −0.011 [−0.037, +0.033] | −0.018 [−0.040, +0.014] | **−0.134** [−0.165, −0.102] |
| `EURUSD=X` | Parkinson | +0.005 [−0.008, +0.028] | −0.039 [−0.067, +0.008] | **−0.044** [−0.066, −0.014] | **−0.104** [−0.126, −0.078] |

The comparisons of `HAR-Parkinson` with the GARCH models are in the notebook (`paired_diff_df`). Under MSE, no GJR − GARCH difference is significant and the point estimates favour GARCH on the equity indices; `HAR-SqRet` does not differ significantly from the GARCH models, except on `EURUSD=X` with the Parkinson proxy, where both GARCH models have lower MSE.

### What the Results Do and Do Not Show
1. **GJR vs GARCH:** GJR has lower QLIKE on both equity indices, significantly so under the squared-return proxy and, for `^STOXX50E`, also under Parkinson (for `FTSEMIB.MI` the interval only just includes zero). No gain is detectable on `EURUSD=X`. This is consistent with the asymmetry found in Phase 1, but it does not carry over to MSE.
2. **Like-for-like comparison:** GARCH family vs `HAR-SqRet` (both use closes only and target the same quantity, so the comparison is neutral on both proxies). `HAR-SqRet` is never significantly better in QLIKE. GJR beats it significantly on both equity indices under both proxies, and GARCH does so in 4 of 6 cells; on `EURUSD=X` the GJR difference is not detectable.
3. **The `HAR-Parkinson` edge comes from its input, not its structure.** It has the lowest QLIKE in 4 of 6 cells, but it uses High/Low data and is fitted on the Parkinson proxy, so its wins on that proxy are circular. The informative case is squared returns on `EURUSD=X`: `HAR-Parkinson` beats `HAR-SqRet`, while `HAR-SqRet` does not beat GARCH. On the indices the differences against squared returns are not significant.

## 5. Known Limitations
* **2009 Sample Start:** The sample starts in January 2009 because 2008 `yfinance` data for `EURUSD=X` shows recurring spike-and-reversal values that are inconsistent with neighbouring prices (e.g., `2008-01-08`). The cause was not identified (a day/month inversion was tested and rejected). The sample therefore excludes the acute late-2008 phase of the Global Financial Crisis.
* **Different Information Sets:** `HAR-Parkinson` uses High/Low data that the other models do not see, and is fitted on one of the scoring proxies. Conclusions about the model structure rely on the `HAR-SqRet` comparison.
* **High/Low Quality:** the Phase 1 consistency check cannot detect an inflated High or Low when Open and Close stay inside the range. Two such ranges were found in the training sample and handled by an ad hoc guard with constants chosen after inspecting the data. HAR inputs also contain zero-variance days from holidays and repaired closes. Minor discrepancies below the $0.1\%$ tolerance remain in the free FX feed.
* **Single Test Window:** one 2020–2025 window that includes the March 2020 shock; no sub-period results. MSE on squared returns is very sensitive to a few extreme days.
* **Inference:** 72 bootstrap intervals with no multiple-testing adjustment; block length fixed at 21 days, not tuned. Parameter estimates have no standard errors, and $\hat\alpha$ is on its bound for GJR on both indices. The `HAR-SqRet` regressors are noisy, which may understate that model.
* **Data:** single vendor (Yahoo Finance). The five repaired days rely on an internal-consistency rule and were not cross-checked against an independent source. Data after December 2025 is not used.

## 6. Roadmap & Next Steps
* **Phase 1:** completed. **Phase 2:** completed (see limitations).
* **Phase 3:** VaR from the forecast volatilities and backtest (Kupiec test).

## 7. Reproducibility
Install the requirements (including `arch` and `pyarrow`) and run `01_volatility_forecast.ipynb` with Restart & Run All. The first run downloads the data and freezes it under `data/`; later runs read the frozen snapshot.