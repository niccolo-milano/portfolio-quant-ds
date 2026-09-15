# Project 02: Cross-Sectional Return Direction Classifier

**Status:** Baseline Pipeline Completed

> **Scope note:** This project implements a supervised machine learning pipeline to predict the cross-sectional direction of forward returns. It is designed to evaluate the predictive validity of technical factors out-of-sample, not to serve as a standalone trading strategy. 

## Overview
Building upon the feature store generated in Project 01, this project transitions from heuristic rule-based filtering to probabilistic classification. It aims to predict whether an asset will yield a positive return in the subsequent month ($R_{t \rightarrow t+1} > 0$) using an $L_2$-regularized Logistic Regression model.

## Tech Stack
* **Machine Learning:** `scikit-learn` (`Pipeline`, `ColumnTransformer`, `LogisticRegression`)
* **Data Engineering:** Pandas, Parquet
* **Evaluation Metrics:** Classification Report (Precision, Recall, F1-Score), Confusion Matrix
* **Architecture:** Scikit-Learn standard Pipeline for rigorous separation of preprocessing and inference, guaranteeing zero data leakage.

## Roadmap

### Phase 1: Dataset Construction & Target Engineering — ✅ Completed
* Ingested the daily feature store (`screening_features.parquet`) from Project 01.
* Downsampled the dataset to an End-of-Month (EoM) frequency to simulate a discrete monthly evaluation horizon.
* Engineered the target variable (`forward_1m_return`) using grouped chronological shifts, explicitly purging structural NaNs (cold-starts and the terminal period) to avoid mislabeling.
* Mapped the 15-asset universe to their respective GICS sectors for categorical feature expansion.

### Phase 2: Temporal Split & Pipeline Architecture — ✅ Completed
* Enforced a strict chronological split (Train: $\le$ 2024-06-30 | Test: > 2024-06-30) instead of randomized cross-validation, structurally preventing look-ahead bias.
* Excluded the market benchmark (SPY) from the feature matrix $X$ to isolate single-stock predictive power.
* Built a unified `Pipeline`: continuous features standardized via `StandardScaler`, categorical features encoded via `OneHotEncoder(handle_unknown='ignore')`.

### Phase 3: Out-of-Sample Evaluation & Heuristic Comparison — ✅ Completed
* Evaluated the baseline model on the unseen H2 2024 test set (75 observations).
* Computed the Signal Agreement Rate against the heuristic rule-based screener developed in Project 01.

## Key Insights

### Geometric Divergence: ML vs. Heuristic Screener
A formal comparison between the ML predictions and the rule-based screener yields a Signal Agreement Rate of **46.67%**. This divergence highlights the mathematical distinction between the two architectures:
* The Project 01 screener utilizes **non-compensatory orthogonal thresholds** (e.g., strictly rejecting an asset if volatility exceeds the median, regardless of momentum).
* The ML model estimates a **compensatory hyperplane**. An exceptionally strong momentum factor can mathematically offset a marginal breach in the volatility dimension, allowing the model to capture multidimensional risk-reward profiles that rigid Boolean logic discards.

### Model Diagnostics & Regime Drift
The baseline out-of-sample accuracy landed at 56%. However, the Confusion Matrix reveals a profound asymmetry:
* **Recall (Up):** 89%
* **Recall (Down):** 28%

*Quantitative Diagnosis:* The training set (Nov 2022 – Jun 2024) is dominated by a strong tech-driven bull market. The model could have internalized an optimistic prior probability $P(y=1)$, generating an elevated intercept ($\beta_0$). When evaluated on the more distributive/lateral regime of H2 2024, the model exhibited inertia, over-predicting positive returns (high false positive rate). This highlights the necessity of dynamic intercept adjustment or Walk-Forward validation (planned for future iterations).

## Dataset
* **Feature Store:** `data/screening_features.parquet`
* **Features:** `momentum_126d`, `rolling_vol_63d`, `current_dd_126d`, `gics_sector`.
* **Universe:** 15 highly liquid U.S. equities spanning 5 GICS sectors. SPY is excluded during the training phase.

## Notebooks
* `01_return_classifier.ipynb`