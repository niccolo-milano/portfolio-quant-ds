# Quantitative Finance & Data Science Portfolio

A structured portfolio documenting my journey in Quantitative Finance, Machine Learning and Data Science through hands-on projects, continuous learning and professional development.

## About me

I am a quantitative finance graduate with a strong interest in Machine Learning and Data Science.

I am building this portfolio to consolidate my skills through real-world projects while documenting my continuous learning journey.

This portfolio reflects my commitment to aligning my professional path with the quantitative background developed during my academic studies.

Every project follows a structured and methodological approach, combining documentation, written explanations and well-organized code.

My goal is to contribute to the development of quantitative models and data-driven solutions while continuously improving my technical and analytical skills.

## What You'll Find Here

This portfolio brings together coding exercises and real-world projects focused on Quantitative Finance, Machine Learning and Data Science.

- **Methodology:** A step-by-step approach to the main stages of each project, including data preparation, model development and evaluation.
- **Project Documentation:** Each project includes a clear description of its objective, dataset, methodology and results, including negative results and known limitations.
- **Continuous Development:** The portfolio evolves alongside my learning journey, documenting both new skills and increasingly complex applications.

## Projects

The portfolio is organized as a progressive sequence of exercises and projects, from foundational data analysis to machine learning and time-series modelling.

| Project | Main Focus | Methods | Status |
|---|---|---|---|
| [Rule-Based Stock Screener](project01-stock-screener/) | Financial data analysis and rule-based screening of 15 equities | DuckDB SQL window functions, point-in-time feature engineering, rolling monthly evaluation | Completed |
| [Cross-Sectional Return Direction Classifier](project02-credit-risk/) | Supervised learning on a stock panel with temporal validation | scikit-learn pipelines, custom walk-forward cross-validation, SHAP, gradient boosting challenger | Completed |
| [Volatility Forecasting & VaR Backtesting](project03-volatility-forecast/) | Volatility forecasting on European indices and EUR/USD | GARCH and GJR-GARCH, HAR-type regressions, QLIKE with block-bootstrap intervals | In progress (phases 1–2 of 3) |

Planned next: portfolio optimization and risk models, followed by a final, more ambitious project.

### Results at a Glance

- **Project 01:** a rule-based screener that selects about 29% of stock-month observations; the most volatile stocks in the universe are never selected, as a direct consequence of the filter design.
- **Project 02:** no out-of-sample predictive skill could be detected (test ROC-AUC 0.524). The project documents this negative result, with interpretability and a gradient boosting challenger used as checks.
- **Project 03:** on the two equity indices, GJR-GARCH has lower QLIKE than GARCH, while on EUR/USD no difference is detectable. A HAR-type model fitted on a range-based proxy does better on some comparisons, but the edge comes from its High/Low input rather than from the model structure.

## Technologies & Tools

The following tools support the development, analysis and documentation of the projects in this portfolio:

- **Programming and Data Analysis:** Python, NumPy, pandas, SQL, DuckDB, Parquet
- **Modelling:** scikit-learn, statsmodels, arch (GARCH family), SHAP
- **Data Visualization:** Matplotlib, Seaborn, Plotly
- **Data Sources:** yfinance
- **Development and Workflow:** Git, GitHub, Visual Studio Code, Jupyter Notebook

## Repository Structure

The repository is organized as follows:

```text
portfolio-quant-ds/
├── README.md
├── requirements.txt
├── .gitignore
├── data/                          # Shared datasets
├── docs/                          # Supporting documentation
├── week01-fundamentals/           # Foundational Python, NumPy & Pandas
├── week02-sql-analysis/           # Foundational SQL & DuckDB Market Data Engine
├── project01-stock-screener/      # Applied project 1: rule-based screener
├── project02-credit-risk/         # Applied project 2: return direction classifier
└── project03-volatility-forecast/ # Applied project 3: volatility forecasting
```

## Current Progress

- [x] Repository setup and Git workflow
- [x] Main portfolio documentation
- [x] Data analysis fundamentals (Python, pandas, NumPy, SQL)
- [x] Applied Project 1: Rule-Based Stock Screener
- [x] Applied Project 2: Cross-Sectional Return Direction Classifier
- [ ] Applied Project 3: Volatility Forecasting & VaR Backtesting (phases 1–2 done, VaR backtest pending)
- [ ] Further projects: portfolio optimization and risk models

## Contact

For professional connections or further information about my work:

- [LinkedIn](https://www.linkedin.com/in/niccol%C3%B2-milano/)
- [GitHub](https://github.com/niccolo-milano)
- [E-mail](mailto:niccolo.milano.nm@gmail.com)