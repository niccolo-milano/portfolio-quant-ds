# Project 02: Cross-Sectional Return Direction Classifier

**Status:** Phases 1–7 completed (baseline, walk-forward validation, tuning, interpretability and challenger benchmark)

> **Scope note:** This project implements a supervised machine learning pipeline to predict the cross-sectional direction of forward returns. It is designed to evaluate the predictive validity of technical factors out-of-sample, not to serve as a standalone trading strategy. 

## Overview
Building upon the feature store generated in Project 01, this project transitions from heuristic rule-based filtering to probabilistic classification. It aims to predict whether an asset will yield a positive return in the subsequent month ($R_{t \rightarrow t+1} > 0$) using an $L_2$-regularized Logistic Regression model. The main outcome is a validated pipeline and a documented null result: no out-of-sample predictive skill is detectable at this sample size.

## Tech Stack
* **Machine Learning:** `scikit-learn` (`Pipeline`, `ColumnTransformer`, `LogisticRegression`, `HistGradientBoostingClassifier`, `permutation_importance`), `shap`
* **Data Engineering:** Pandas, Parquet
* **Evaluation Metrics:** Classification Report (Precision, Recall, F1-Score), Confusion Matrix, ROC-AUC (per fold, pooled and held-out), month-clustered bootstrap intervals
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

### Phase 7: Model Interpretability & Validation Audit - ✅ Completed
* Standardized coefficients of the tuned model, read alongside mean |SHAP| (coefficients of 0/1 dummies and of standardized numeric features are not directly comparable).
* Out-of-sample permutation importance (30 repeats), for single features and for the numeric block permuted jointly.
* SHAP decomposition of the log-odds, with a quadrant analysis of where the model and the screener disagree.
* Coefficient stability across the 4 walk-forward folds.
* Untuned `HistGradientBoostingClassifier` challenger on the same folds and on the held-out test.

### Known limitations
* **Sample Size & Effective Degrees of Freedom:** The panel comprises 15 equities over 20 training months (300 observations) and 5 test months (75 observations), after the 6-month feature burn-in and excluding the terminal period lacking a forward target. These are not independent observations: cross-sectional correlation within months and sectors and strong serial autocorrelation (induced by overlapping 126-day rolling windows) compress the effective degrees of freedom ($N_{\text{eff}} \ll N$). Statements about the H2 2024 test have low statistical power: the permutation test can only exclude dependence above roughly 8 accuracy points. 
* **Overfitting & Multicollinearity:** With $P = 8$ features post one-hot encoding (without dummy dropping), the risk of parameter instability is non-trivial. The model relies strictly on $L_2$ Ridge shrinkage to guarantee numerical invertibility and prevent coefficient explosion.
* **Hyperparameter Sensitivity (Parameter $C$):** Analysis of the cross-validation results reveals a flat maximum for $C \ge 1$. The performance variance between $C=1$, $C=10$, and $C=100$ falls entirely within the statistical noise (standard deviation $\sim 0.05$) across the 4 folds. Consequently, there is no statistically significant predictive superiority among these values; the selection of $C=10$ represents a strictly mechanical `argmax` extraction by the Grid Search rather than a definitive structural optimum.
* **Challenger Benchmark:** Only one untuned Gradient Boosting configuration was tested. The results do not establish either the presence or the absence of a nonlinear signal (see Key Insights).
* **Universe Selection:** The universe consists of 15 hand-selected, highly liquid equities; selection effects may drive sector-level patterns.
* **Interpretation:** SHAP values and permutation importance describe the fitted models, not causal market effects.

## Key Insights

### Geometric Divergence: ML vs. Heuristic Screener
A formal comparison between the ML predictions and the rule-based screener yields a Signal Agreement Rate of **46.67%**. This divergence highlights the mathematical distinction between the two architectures:
* The Project 01 screener utilizes **non-compensatory orthogonal thresholds** (e.g., strictly rejecting an asset if volatility exceeds the median, regardless of momentum).
* The ML model estimates a **compensatory hyperplane**. An exceptionally strong momentum factor can mathematically offset a marginal breach in the volatility dimension, allowing the model to capture multidimensional risk-reward profiles that rigid Boolean logic discards.

### Model Diagnostics & Regime Drift Mitigation
The uncalibrated baseline evaluation revealed a profound asymmetry (Recall Up: 89% vs. Recall Down: 28%), masking poor downside protection behind an illusory 56% aggregate accuracy. 

*Quantitative diagnosis:* The share of positive months fell from 58% in the Train period (Nov 2022 – Jun 2024) to 47% in the Test period (H2 2024): the baseline fitted an optimistic prior $P(y=1)$ that did not carry over to the more lateral test regime. The data used here do not allow attributing this shift to a specific sector: the `Tech` up-rate moved only from 67% to 60% (15 observations per sector in Test).

*Mitigation and its cost:* The pipeline was upgraded with a **Custom Walk-Forward Cross-Validation** and **Cost-Sensitive Learning** (`class_weight='balanced'`), with $C$ selected on Macro-F1 inside the Train set. The tuned model raised Recall Down from 28% to 55%, but at the cost of Recall Up (89% → 43%) and aggregate accuracy (56% → 49%). Class weighting moves the operating point rather than adding information: the tuned model's ROC-AUC on the H2 2024 test is 0.524, so the recall shift should be read as a different risk/return trade-off, not as improved discrimination.

### Interpretability & Challenger Benchmark
* **No detectable out-of-sample signal.** ROC-AUC is 0.524 on the H2 2024 test ($N = 75$) and 0.515 on average across the 4 walk-forward folds; pooled over the 12 walk-forward months, the 95% month-clustered bootstrap interval is 0.34–0.57. Permuting any single feature, or the numeric block jointly, does not move test accuracy beyond noise.
* **Attribution is spread across three features; momentum is negligible.** Mean |SHAP| is ≈ 0.23 for drawdown, ≈ 0.22 for volatility and ≈ 0.20 for the `Tech` dummy, versus ≈ 0.01 for momentum.
* **Contrarian drawdown weight.** The drawdown coefficient is negative in all 4 walk-forward folds and in the final model, and its marginal correlation with the target has the same sign in both periods (−0.12 Train, −0.10 Test). Coefficient magnitudes are not stable across folds (e.g. the momentum weight falls from 0.66 to 0.04): stable signs are partly mechanical, since the expanding windows are nested.
* **The challenger's ranking depends on the evaluation set.** An untuned Gradient Boosting reaches a train AUC of 0.92 but 0.45 on the validation folds (an overfitting signature), yet scores higher than the baseline on the H2 2024 test (0.591 vs 0.524). The paired AUC difference over the pooled walk-forward months includes zero, so with this sample size the comparison cannot rank the two models.

## Dataset
* **Feature Store:** `data/screener_results.parquet`
* **Features:** `momentum_126d`, `rolling_vol_63d`, `current_dd_126d`, `gics_sector`.
* **Universe:** 15 highly liquid U.S. equities spanning 5 GICS sectors. SPY is excluded during the training phase.

## Reproducibility
Run the notebooks top to bottom (Restart & Run All). `02_interpretability.ipynb` includes a reproducibility gate that asserts the reference model matches the metrics reported above (accuracy 49%, Recall Down 55% / Up 43%, Signal Agreement Rate 56.00%) before any interpretation is computed.

## Notebooks
* `01_return_classifier.ipynb`
* `02_interpretability.ipynb`