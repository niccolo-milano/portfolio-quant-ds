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

### Phase 1: Dataset Construction & Target Engineering - ✅ Completed
* Ingested the daily feature store (`screening_features.parquet`) from Project 01.
* Downsampled the dataset to an End-of-Month (EoM) frequency to simulate a discrete monthly evaluation horizon.
* Engineered the target variable (`forward_1m_return`) using grouped chronological shifts, explicitly purging structural NaNs (cold-starts and the terminal period) to avoid mislabeling.
* Mapped the 15-asset universe to their respective GICS sectors for categorical feature expansion.

### Phase 2: Temporal Split & Pipeline Architecture - ✅ Completed
* Enforced a strict chronological split (Train: $\le$ 2024-06-30 | Test: > 2024-06-30) instead of randomized cross-validation, structurally preventing look-ahead bias.
* Excluded the market benchmark (SPY) from the feature matrix $X$ to isolate single-stock predictive power.
* Built a unified `Pipeline`: continuous features standardized via `StandardScaler`, categorical features encoded via `OneHotEncoder(handle_unknown='ignore')`.

### Phase 3: Out-of-Sample Evaluation & Heuristic Comparison - ✅ Completed
* Evaluated the baseline model on the unseen H2 2024 test set (75 observations).
* Computed the Signal Agreement Rate against the heuristic rule-based screener developed in Project 01.

### Phase 4: Custom Walk-Forward Cross-Validation - ✅ Completed
* Engineered a custom expanding-window generator to partition the panel dataset into 4 chronological folds, strictly preventing cross-sectional leakage and look-ahead bias.
* Fixed the out-of-sample test horizon at 3 months per fold, while accumulating historical data for the training sets.

### Phase 5: Hyperparameter Tuning & Cost-Sensitive Learning - ✅ Completed
* Reconfigured the modeling pipeline with `class_weight='balanced'` to actively penalize misclassifications on the minority class (Down months).
* Executed a `GridSearchCV` evaluated on Macro-F1 to optimize the $L_2$ inverse regularization strength ($C$).

### Phase 6: Tuned Out-of-Sample Evaluation - ✅ Completed
* Evaluated the optimized model on the unseen H2 2024 test set, observing a structural shift from aggregate Accuracy maximization to minority-class Recall optimization (Recall Down improved from 28% to 55%).
* Re-computed the Signal Agreement Rate against the heuristic screener, confirming a more balanced divergence distribution.


### Known limitations
* **Sample Size & Effective Degrees of Freedom:** The dataset comprises a panel of 15 equities across ~19 available months (following the 6-month feature burn-in and excluding the terminal period lacking a forward target), yielding ~285–300 training observations. However, these do not represent independent observations: cross-sectional correlation within identical GICS sectors and strong serial autocorrelation (induced by overlapping 126-day rolling windows) compress the effective degrees of freedom ($N_{\text{eff}} \ll N$). 
* **Overfitting & Multicollinearity:** With $P = 8$ features post one-hot encoding (without dummy dropping), the risk of parameter instability is non-trivial. The model relies strictly on $L_2$ Ridge shrinkage to guarantee numerical invertibility and prevent coefficient explosion.
* **Hyperparameter Sensitivity (Parameter $C$):** Analysis of the cross-validation results reveals a flat maximum for $C \ge 1$. The performance variance between $C=1$, $C=10$, and $C=100$ falls entirely within the statistical noise (standard deviation $\sim 0.05$) across the 4 folds. Consequently, there is no statistically significant predictive superiority among these values; the selection of $C=10$ represents a strictly mechanical `argmax` extraction by the Grid Search rather than a definitive structural optimum.

## Key Insights

### Geometric Divergence: ML vs. Heuristic Screener
A formal comparison between the ML predictions and the rule-based screener yields a Signal Agreement Rate of **46.67%**. This divergence highlights the mathematical distinction between the two architectures:
* The Project 01 screener utilizes **non-compensatory orthogonal thresholds** (e.g., strictly rejecting an asset if volatility exceeds the median, regardless of momentum).
* The ML model estimates a **compensatory hyperplane**. An exceptionally strong momentum factor can mathematically offset a marginal breach in the volatility dimension, allowing the model to capture multidimensional risk-reward profiles that rigid Boolean logic discards.

### Model Diagnostics & Regime Drift Mitigation
The uncalibrated baseline evaluation revealed a profound asymmetry (Recall Up: 89% vs. Recall Down: 28%), masking poor downside protection behind an illusory 56% aggregate accuracy. 

*Quantitative Diagnosis & Solution:* The training set (Nov 2022 – Jun 2024) was dominated by a strong tech-driven bull market. Consequently, the baseline model internalized an optimistic prior probability $P(y=1)$, generating an elevated intercept ($\beta_0$) that exhibited severe inertia and over-predicted positive returns in the more lateral regime of H2 2024. 

To mitigate this regime drift, the pipeline was upgraded with a **Custom Walk-Forward Cross-Validation** and **Cost-Sensitive Learning** (`class_weight='balanced'`). This structurally forced the optimizer to re-calibrate the decision boundary, actively penalizing false positives. As a result, the tuned model successfully doubled the minority class detection (**Recall Down: 55%**), demonstrating a strict prioritization of downside risk management over raw aggregate accuracy.

## Dataset
* **Feature Store:** `data/screener_results.parquet`
* **Features:** `momentum_126d`, `rolling_vol_63d`, `current_dd_126d`, `gics_sector`.
* **Universe:** 15 highly liquid U.S. equities spanning 5 GICS sectors. SPY is excluded during the training phase.

## Notebooks
* `01_return_classifier.ipynb`