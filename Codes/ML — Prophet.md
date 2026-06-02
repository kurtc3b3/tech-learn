
## What & When

**Prophet** (Meta) is a **forecasting** library for business time series with daily/weekly/yearly seasonality, holidays, and trend changepoints. It expects a simple two-column schema (`ds`, `y`) and produces interpretable components — a strong baseline before custom [[ML — scikit-learn]] lag features or [[ML — PyTorch]] sequence models.

Use Prophet when:

- Forecasting **univariate** metrics (revenue, signups, load) with clear seasonality
- Stakeholders want **decomposable** plots (trend + seasonality + holidays)
- You need quick **holiday** and **regressor** support without building a full state-space model
- Historical data has missing days or outliers Prophet can absorb

For multivariate or high-frequency industrial series, combine with domain features or other tools in [[Machine Learning]]. Log forecasts and params in [[ML — MLflow]]; tune changepoints and seasonality with [[ML — Optuna]].

```bash
pip install prophet
# depends on cmdstanpy / Stan backend — first fit may compile
```

---

## Prophet vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Business univariate forecast | Prophet | Fast baseline |
| Tabular ML / lags | [[ML — scikit-learn]], [[ML — XGBoost]] | Feature engineering |
| Deep sequence models | [[ML — PyTorch]] | More flexible, more ops |
| AutoML tables | [[ML — H2O]] | Not specialized for series |
| Experiment tracking | [[ML — MLflow]] | Params + forecast CSV artifacts |
| Seasonality / CP tuning | [[ML — Optuna]] | Wrap `Prophet` params |

---

## Required Data Format

```python
import pandas as pd

# Columns: ds (datetime), y (numeric target)
df = pd.read_csv("data/daily_orders.csv", parse_dates=["date"])
df = df.rename(columns={"date": "ds", "orders": "y"})
df = df.sort_values("ds").reset_index(drop=True)

# Prophet ignores rows with NaN y if you drop them explicitly
df = df.dropna(subset=["y"])
```

---

## Fit & Forecast

```python
from prophet import Prophet

train = df[df["ds"] < "2024-01-01"]
test = df[df["ds"] >= "2024-01-01"]

m = Prophet(
    growth="linear",
    yearly_seasonality=True,
    weekly_seasonality=True,
    daily_seasonality=False,
    seasonality_mode="multiplicative",  # or "additive"
    changepoint_prior_scale=0.05,
    interval_width=0.95,
)

m.fit(train)

future = m.make_future_dataframe(periods=90, freq="D")
forecast = m.predict(future)

# Point forecast columns: yhat, yhat_lower, yhat_upper
print(forecast[["ds", "yhat", "yhat_lower", "yhat_upper"]].tail())
```

---

## Plot Components

```python
fig1 = m.plot(forecast)           # matplotlib forecast
fig2 = m.plot_components(forecast)  # trend + seasonality
```

---

## Holidays

```python
import pandas as pd

holidays = pd.DataFrame({
    "holiday": "black_friday",
    "ds": pd.to_datetime(["2023-11-24", "2024-11-29"]),
    "lower_window": -1,
    "upper_window": 1,
})

m = Prophet(holidays=holidays)
m.fit(train)
```

Add country holiday calendars via pre-built CSVs or `holidays` package helpers.

---

## Extra Regressors

```python
df["promo"] = (df["ds"].dt.month == 11).astype(int)
train = df[df["ds"] < "2024-01-01"]

m = Prophet()
m.add_regressor("promo")
m.fit(train)

future = m.make_future_dataframe(periods=30, freq="D")
future["promo"] = (future["ds"].dt.month == 11).astype(int)
forecast = m.predict(future)
```

Future regressor values must be known or forecast separately — do not leak test promo flags from actuals.

---

## Cross-Validation & Metrics

```python
from prophet.diagnostics import cross_validation, performance_metrics

# Requires initial training history; horizon in days
df_cv = cross_validation(
    m,
    initial="365 days",
    period="30 days",
    horizon="30 days",
    parallel="processes",
)
df_p = performance_metrics(df_cv)
print(df_p[["horizon", "mape", "rmse"]].head())
```

---

## Evaluate Holdout

```python
import numpy as np

merged = test.merge(forecast[["ds", "yhat"]], on="ds", how="left")
rmse = np.sqrt(np.mean((merged["y"] - merged["yhat"]) ** 2))
mape = np.mean(np.abs((merged["y"] - merged["yhat"]) / merged["y"])) * 100
print(f"RMSE: {rmse:.2f}, MAPE: {mape:.2f}%")
```

---

## Key Parameters

| Param | Role |
| --- | --- |
| `changepoint_prior_scale` | Flexibility of trend (higher → more wiggly) |
| `seasonality_prior_scale` | Strength of seasonality |
| `seasonality_mode` | `additive` vs `multiplicative` |
| `holidays_prior_scale` | Holiday effect strength |
| `interval_width` | Uncertainty band width |
| `growth` | `linear` or `logistic` (needs cap column) |

---

## MLflow Integration

```python
import mlflow

with mlflow.start_run(run_name="prophet-orders"):
    mlflow.log_params({
        "changepoint_prior_scale": 0.05,
        "seasonality_mode": "multiplicative",
    })
    mlflow.log_metric("holdout_rmse", rmse)
    forecast.to_csv("forecast.csv", index=False)
    mlflow.log_artifact("forecast.csv")
    # Serialize model with pickle/joblib or custom flavor
    import joblib
    joblib.dump(m, "prophet_model.joblib")
    mlflow.log_artifact("prophet_model.joblib")
```

---

## Optuna Tuning

```python
import optuna
from prophet.diagnostics import cross_validation, performance_metrics

def objective(trial):
    cps = trial.suggest_float("changepoint_prior_scale", 0.001, 0.5, log=True)
    sps = trial.suggest_float("seasonality_prior_scale", 0.01, 10.0, log=True)
    mode = trial.suggest_categorical("seasonality_mode", ["additive", "multiplicative"])

    pm = Prophet(
        changepoint_prior_scale=cps,
        seasonality_prior_scale=sps,
        seasonality_mode=mode,
        yearly_seasonality=True,
        weekly_seasonality=True,
    )
    pm.fit(train)
    cv = cross_validation(pm, initial="180 days", period="30 days", horizon="14 days")
    metrics = performance_metrics(cv)
    return metrics["mape"].mean()

study = optuna.create_study(direction="minimize")
study.optimize(objective, n_trials=40)
```

See [[ML — Optuna]] for parallel trials; CV is expensive — cache `train` frame.

---

## Serialization

```python
import joblib

joblib.dump(m, "models/prophet_orders.joblib")
loaded = joblib.load("models/prophet_orders.joblib")
loaded.predict(loaded.make_future_dataframe(periods=7))
```

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| Irregular timestamps | Resample to daily/weekly first |
| Short history | Reduce seasonality flags; simpler model |
| Outliers | `m = Prophet(); df["y"] = cap outliers` or use `holidays` |
| Multiple series | One model per series or global features + sklearn |
| COVID-style shifts | Add changepoints or segment data |
| Slow CV | Shorter `horizon`, fewer trials in [[ML — Optuna]] |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Schema | columns `ds`, `y` (datetime + numeric) |
| Fit | `m = Prophet(); m.fit(df)` |
| Future | `future = m.make_future_dataframe(periods=90, freq="D")` |
| Predict | `forecast = m.predict(future)` |
| Holidays | `Prophet(holidays=holidays_df)` |
| Regressor | `m.add_regressor("promo")` |
| CV | `cross_validation(m, initial=..., period=..., horizon=...)` |
| Metrics | `performance_metrics(df_cv)` |
| Save | `joblib.dump(m, path)` |

---

## Related Notes

- [[Machine Learning]] — forecasting in the stack
- [[ML — pandas]] — resampling and date features
- [[ML — scikit-learn]] — lag features alternative
- [[ML — MLflow]], [[ML — Optuna]]
- [[ML — matplotlib]] — custom forecast plots

---

## Tags

#python #prophet #forecasting #time-series #seasonality #machine-learning #mlflow #optuna
