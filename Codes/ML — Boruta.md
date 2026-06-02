## What & When

**Boruta** is a **wrapper feature-selection** algorithm built on Random Forest importance. It compares each real feature against **shadow** (permuted) features and keeps only attributes that consistently beat their random counterparts.

Use Boruta when:

- You have **many correlated features** and want an all-relevant subset (not just top-k by univariate score)
- Building a [[ML — scikit-learn]] pipeline before [[ML — XGBoost]] / [[ML — LightGBM]]
- Reducing dimensionality without hand-picking a fixed `k`
- Pairing with [[ML — Optuna]] on the reduced feature set

```bash
pip install boruta
```

Boruta depends on **scikit-learn** `RandomForestClassifier` / `RandomForestRegressor`. Track selected features and model metrics in [[ML — MLflow]]. Overview: [[Machine Learning]].

---

## Boruta vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| All-relevant features (RF-based) | **Boruta** | Shadow features, iterative tests |
| Univariate filter | `SelectKBest`, `mutual_info_classif` | Fast, ignores interactions |
| Model-based importance | `SelectFromModel` | Threshold on single fit |
| Recursive elimination | `RFECV` | Greedy, expensive on wide data |
| Hyperparameters after selection | [[ML — Optuna]] | Search on Boruta-confirmed columns |
| Explain kept features | [[ML — SHAP]] | Post-hoc on final model |

---

## Basic Classification Workflow

```python
import pandas as pd
from sklearn.ensemble import RandomForestClassifier
from boruta import BorutaPy
from sklearn.model_selection import train_test_split

df = pd.read_csv("data/churn.csv")
X = df.drop(columns=["churn"])
y = df["churn"].astype(int)

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

rf = RandomForestClassifier(
    n_jobs=-1,
    class_weight="balanced",
    max_depth=5,
    random_state=42,
)

boruta = BorutaPy(
    estimator=rf,
    n_estimators="auto",      # default: max(100, n_features)
    perc=100,                 # percentile of shadow importance to beat
    alpha=0.05,               # p-value threshold (two-sided test)
    two_step=True,            # tentative features get a second chance
    max_iter=100,
    random_state=42,
    verbose=2,
)

boruta.fit(X_train.values, y_train.values)

confirmed = X.columns[boruta.support_].tolist()
tentative = X.columns[boruta.support_weak_].tolist()
rejected = X.columns[~boruta.support_ & ~boruta.support_weak_].tolist()

print("Confirmed:", confirmed)
print("Tentative:", tentative)
```

---

## Regression

```python
from sklearn.ensemble import RandomForestRegressor
from boruta import BorutaPy

rf_reg = RandomForestRegressor(
    n_jobs=-1,
    max_depth=7,
    random_state=42,
)

boruta_reg = BorutaPy(
    estimator=rf_reg,
    n_estimators="auto",
    random_state=42,
)

boruta_reg.fit(X_train.values, y_train.values)
selected_mask = boruta_reg.support_
X_train_sel = X_train.loc[:, selected_mask]
```

---

## sklearn Pipeline Integration

Keep selection **inside** cross-validation when tuning — fit Boruta only on training folds.

```python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression

# Manual pattern: fit Boruta on train, transform both splits
boruta.fit(X_train.values, y_train.values)
cols = X_train.columns[boruta.support_]

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("clf", LogisticRegression(max_iter=1000)),
])

pipe.fit(X_train[cols], y_train)
score = pipe.score(X_test[cols], y_test)
```

For production, **persist the column list** (or mask) alongside the model artifact in [[ML — MLflow]].

---

## Parameters That Matter

| Parameter | Effect |
| --- | --- |
| `perc` | Higher → stricter (fewer confirmed features); default 100 |
| `alpha` | Statistical threshold; lower → fewer features |
| `max_iter` | Stop after N rounds if features still undecided |
| `two_step` | Re-test tentative features; usually leave `True` |
| `estimator` | Any sklearn RF with `feature_importances_` |

```python
# Stricter selection (fewer features)
boruta_strict = BorutaPy(
    estimator=rf,
    perc=90,
    alpha=0.01,
    max_iter=200,
    random_state=42,
)
```

---

## Handling Tentative Features

**Confirmed** (`support_`) — consistently beat shadow features. **Tentative** (`support_weak_`) — ambiguous; domain experts often include or drop them explicitly.

```python
def feature_sets(columns, boruta: BorutaPy) -> dict[str, list[str]]:
    return {
        "confirmed": columns[boruta.support_].tolist(),
        "tentative": columns[boruta.support_weak_].tolist(),
        "rejected": columns[~boruta.support_ & ~boruta.support_weak_].tolist(),
    }

sets = feature_sets(X.columns, boruta)
# Policy: train production model on confirmed only
use_cols = sets["confirmed"]
```

---

## End-to-End with MLflow & Serving

```python
import mlflow
import joblib

with mlflow.start_run(run_name="boruta-lr"):
    boruta.fit(X_train.values, y_train.values)
    cols = X_train.columns[boruta.support_].tolist()
    mlflow.log_param("n_features_confirmed", len(cols))
    mlflow.log_dict({"features": cols}, "selected_features.json")

    from sklearn.pipeline import Pipeline
    from sklearn.preprocessing import StandardScaler
    from sklearn.linear_model import LogisticRegression

    model = Pipeline([
        ("scaler", StandardScaler()),
        ("clf", LogisticRegression(max_iter=2000)),
    ])
    model.fit(X_train[cols], y_train)
    mlflow.sklearn.log_model(model, "model", input_example=X_train[cols].head(1))

# Serve via [[ML — BentoML]] or expose predictions behind [[API - FastAPI]]
```

---

## Pitfalls

- **High-dimensional sparse data** — RF + Boruta can be slow; consider univariate pre-filter first.
- **Leakage** — never fit Boruta on the full dataset before splitting.
- **Class imbalance** — use `class_weight="balanced"` on the underlying RF.
- **Categorical columns** — encode (one-hot / target encoding) before Boruta; it expects numeric matrices.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Fit | `BorutaPy(estimator=rf).fit(X.values, y.values)` |
| Confirmed mask | `boruta.support_` |
| Tentative mask | `boruta.support_weak_` |
| Ranking | `boruta.ranking_` (1 = best, 2 = tentative, -1 = rejected) |
| Selected columns | `X.columns[boruta.support_]` |
| Regression | `BorutaPy(estimator=RandomForestRegressor(...))` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — Optuna]]
- [[ML — SHAP]]
- [[ML — MLflow]]
- [[ML — pandas]]

---

## Tags

#python #machine-learning #boruta #feature-selection #sklearn #random-forest #tabular
