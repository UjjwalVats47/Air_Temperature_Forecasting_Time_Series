# Monthly Mean Air Temperature Forecasting

This project develops and evaluates 12-month-ahead forecasts of monthly mean surface air temperature at Singapore's Changi Climate Station. It uses a leakage-aware, chronological workflow to compare Seasonal Naive, Ridge Regression, Holt-Winters, SARIMA, XGBoost, and a weighted ensemble.

The historical dataset covers **January 1982 through June 2020**. Because it ends in June 2020, the final July 2020–June 2021 projection is a **historical retrospective forecast**, not a current weather forecast.

## Project overview

- **Target:** Monthly mean surface air temperature, in degrees Celsius.
- **Location:** Changi Climate Station, Singapore.
- **Forecast horizon:** 12 months ahead.
- **Dataset:** 462 complete monthly observations, with no missing or duplicated months.
- **Source:** Singapore National Environment Agency (NEA), *Surface Air Temperature Monthly Mean*, available through data.gov.sg under its open-data licence.
- **Primary metric:** Mean Absolute Error (MAE), in degrees Celsius.

## Key results

| Model | Validation MAE (C) | Held-out test MAE (C) | Notes |
|---|---:|---:|---|
| XGBoost | 0.3287 | 0.4540 | Validation-selected primary model |
| Holt-Winters | 0.3480 | 0.4084 | Best observed held-out test MAE |
| Ridge Regression | — | 0.4337 | Calendar/trend features |
| Seasonal Naive | — | 0.5288 | Baseline |
| SARIMA | — | 0.5433 | Did not beat baseline on test data |
| Weighted Ensemble | — | 0.4540 | Equivalent to XGBoost because the selected XGBoost weight was 1.0 |

XGBoost achieved the lowest validation MAE and was selected before the final test period was evaluated. On the 66 held-out test months from January 2015 to June 2020, Holt-Winters produced the lowest observed MAE, demonstrating that validation performance does not always translate directly to final out-of-sample performance.

## Dataset

The source file is:

```text
surface-air-temperature-monthly-mean.csv
```

It contains two columns:

| Column | Type | Description |
|---|---|---|
| `month` | `YYYY-MM` | Calendar month of observation |
| `meantemp` | Numeric | Monthly mean surface air temperature, in degrees Celsius |

Data-quality checks confirmed:

- 462 monthly records from January 1982 through June 2020.
- No missing calendar months, duplicate months, missing temperatures, or non-finite values.
- Historical mean temperature: 27.67 C.
- Observed monthly-mean range: 25.4 C to 29.5 C.
- Warmest calendar month by historical mean: May, at 28.46 C.
- Coolest calendar month by historical mean: December, at 26.63 C.
- Average seasonal contrast: 1.84 C.
- Estimated annual linear trend: approximately 0.236 C per decade.

## Methodology

### Chronological data split

The project uses time-ordered partitions rather than random splitting:

| Partition | Period | Months | Purpose |
|---|---|---:|---|
| Training | January 1982–December 2009 | 336 | Initial model fitting |
| Validation | January 2010–December 2014 | 60 | Feature, hyperparameter, and model selection |
| Held-out test | January 2015–June 2020 | 66 | Final unbiased evaluation |

The test period was kept sealed until model configurations, feature groups, and ensemble weights had been frozen.

### Feature engineering

Eleven past-only features were engineered:

- Cyclical calendar features: `monthsin` and `monthcos`
- Trend feature: `timeindex`
- Temperature lags: `lag1`, `lag3`, `lag6`, and `lag12`
- Rolling statistics: `rollingmean3`, `rollingmean6`, `rollingmean12`, and `rollingstd12`

Lag and rolling features use only observations available before the forecast month. For multi-step XGBoost forecasts, predictions are generated recursively, so future actual temperatures are never used as inputs.

### Models evaluated

- Seasonal Naive
- Ridge Regression
- Holt-Winters Exponential Smoothing
- SARIMA
- XGBoost Regressor
- XGBoost/Holt-Winters weighted ensemble

Model selection used three expanding-window 12-month validation forecasts, producing 36 validation predictions per candidate. The final evaluation used six walk-forward origins, covering all 66 held-out test months exactly once.

## Repository structure

```text
.
├── Air_Temperature_Time_Series.ipynb
├── surface-air-temperature-monthly-mean.csv
├── README.md
├── requirements.txt
└── outputs/
    ├── airtemperaturevalidated.csv
    ├── edasummary.csv
    ├── monthlyclimatology.csv
    ├── m3trainingfeatures.csv
    ├── m3datapartitions.csv
    ├── m3validationfolds.csv
    ├── m4candidateevaluationmatrix.csv
    ├── m4modelleaderboard.csv
    ├── m4validationpredictions.csv
    ├── m5frozenconfiguration.json
    ├── m6testpredictions.csv
    ├── m6finaltestevaluationmatrix.csv
    ├── m6monthlytesterrors.csv
    └── figures/
```

## Requirements

Install the dependencies with:

```bash
pip install -r requirements.txt
```

The notebook uses:

- Python 3
- pandas
- NumPy
- Matplotlib
- Seaborn
- SciPy
- statsmodels
- scikit-learn
- XGBoost
- Jupyter Notebook

## How to run

1. Clone this repository.
2. Place `surface-air-temperature-monthly-mean.csv` beside the notebook or inside a `data/` folder.
3. Install the dependencies.
4. Launch Jupyter Notebook.
5. Run `Air_Temperature_Time_Series.ipynb` from top to bottom.

```bash
git clone <your-repository-url>
cd <repository-name>
pip install -r requirements.txt
jupyter notebook
```

The notebook automatically searches the current directory and nearby `data/` directories for the CSV file. It exports validated data, analysis tables, model results, and figures to the `outputs/` directory.

## Main findings

- Monthly temperature shows a strong recurring annual seasonal pattern.
- STL decomposition found a descriptive seasonal strength of 0.759 and trend strength of 0.565.
- ACF values at lags 12 and 24 were 0.635 and 0.553, respectively, supporting annual temporal dependence.
- XGBoost achieved the best validation MAE, improving on Seasonal Naive by approximately 26.96 percent.
- Holt-Winters achieved the best held-out test MAE, approximately 22.76 percent better than Seasonal Naive.
- The weighted ensemble did not improve final performance because its validation-selected XGBoost weight was 1.0.
- Chronological validation was essential: XGBoost's MAE increased from 0.3287 C during validation to 0.4540 C during final testing.

## Limitations

- The dataset represents one weather station, not all of Singapore.
- The project forecasts monthly averages, not daily temperatures or short-term weather.
- The dataset ends in June 2020, so the final projection is retrospective.
- The analysis uses historical temperature only; external climate variables such as humidity, precipitation, wind, and solar radiation were not included.
- The validation period contains only 36 scored forecast months, so model rankings may be sensitive to individual unusually warm or cool years.
- The annual trend estimate is descriptive and does not establish causation or quantify nationwide climate change.

## Future work

- Update the dataset with observations after June 2020.
- Add meteorological covariates such as humidity, rainfall, wind speed, and solar radiation.
- Compare direct multi-horizon forecasting with recursive forecasting.
- Add calibrated prediction intervals and probabilistic forecasts.
- Explore SHAP-based model explanations.
- Deploy the workflow as a Streamlit dashboard or FastAPI prediction service.

## License

Add an appropriate open-source license before publishing, such as the MIT License. Ensure that the NEA/data.gov.sg dataset license and attribution requirements are followed when redistributing the source data.
