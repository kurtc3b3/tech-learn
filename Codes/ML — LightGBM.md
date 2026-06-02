
## What & When

**LightGBM** (Light Gradient Boosting Machine) is Microsoft’s **gradient boosting** framework optimized for speed and memory on large tabular datasets. It grows trees **leaf-wise** (best-first) rather than level-wise, which often yields better accuracy with fewer leaves when tuned carefully.

Use LightGBM when:

- Training on **large** or high-dimensional tabular data
- You need **fast iteration** during [[ML — Optuna]] studies
- Categorical features are common (`categorical_feature` without one-hot explosion)
- [[ML — XGBoost]] is accurate but too slow for your data volume

Part of the boosting tier in [[Machine Learning]]. Log runs with [[ML — MLflow]]; compare against [[ML — scikit-learn]] baselines first.

```bash
pip install lightgbm
```

---

## LightGBM vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Similar accuracy, classic API | [[ML — XGBoost]] | Level-wise vs leaf-wise |
| Pipelines & interpretability | [[ML — scikit-learn]] | Simpler ops |
| AutoML / Spark | [[ML — H2O]] | Managed clusters |
| Deep learning | [[ML — PyTorch]] | Images, sequences |
| Hyperparameters | [[ML — Optuna]] | Pruning callbacks available |
| Experiments | [[ML — MLflow]] | `mlflow.lightgbm.log_model` |

---

## sklearn API

```python
import lightgbm as lgb
import numpy as np
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, stratify=y, random_state=42
)

clf = lgb.LGBMClassifier(
    n_estimators=500,
    learning_rate=0.05,
    num_leaves=31,
    max_depth=-1,
    min_child_samples=20,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.0,
    reg_lambda=0.1,
    random_state=42,
)

clf.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    eval_metric="auc",
    callbacks=[lgb.early_stopping(30, verbose=False), lgb.log_evaluation(0)],
)

proba = clf.predict_proba(X_test)[:, 1]
print("AUC:", roc_auc_score(y_test, proba))
print("Best iteration:", clf.best_iteration_)
```

---

## Native API — `Dataset` + `train`

```python
train_data = lgb.Dataset(X_train, label=y_train)
valid_data = lgb.Dataset(X_test, label=y_test, reference=train_data)

params = {
    "objective": "binary",
    "metric": "auc",
    "boosting_type": "gbdt",
    "num_leaves": 31,
    "learning_rate": 0.05,
    "feature_fraction": 0.8,
    "bagging_fraction": 0.8,
    "bagging_freq": 5,
    "verbose": -1,
    "seed": 42,
}

gbm = lgb.train(
    params,
    train_data,
    num_boost_round=1000,
    valid_sets=[train_data, valid_data],
    valid_names=["train", "valid"],
    callbacks=[lgb.early_stopping(30), lgb.log_evaluation(50)],
)

pred = gbm.predict(X_test, num_iteration=gbm.best_iteration)
```

---

## Categorical Features

```python
import pandas as pd

df = pd.read_csv("data/events.csv")
cat_cols = ["region", "device_type", "channel"]

for c in cat_cols:
    df[c] = df[c].astype("category")

X = df.drop(columns=["converted"])
y = df["converted"]

clf = lgb.LGBMClassifier(n_estimators=300, learning_rate=0.05)
clf.fit(
    X, y,
    categorical_feature=cat_cols,
)
```

With native API, pass column names or category indices in `Dataset(..., categorical_feature=cat_cols)`.

---

## Regression Example

```python
from sklearn.datasets import load_diabetes
from sklearn.metrics import mean_squared_error

X, y = load_diabetes(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

reg = lgb.LGBMRegressor(
    objective="regression",
    n_estimators=800,
    learning_rate=0.03,
    num_leaves=63,
    random_state=42,
)
reg.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    eval_metric="rmse",
    callbacks=[lgb.early_stopping(40, verbose=False)],
)
rmse = np.sqrt(mean_squared_error(y_test, reg.predict(X_test)))
print(f"RMSE: {rmse:.3f}")
```

---

## Key Hyperparameters

| Param | Role | Notes |
| --- | --- | --- |
| `num_leaves` | Main complexity knob | Keep `< 2^max_depth` if `max_depth` set |
| `learning_rate` | Step size | Lower → more `n_estimators` |
| `min_child_samples` | Min data in leaf | Increase to reduce overfit |
| `feature_fraction` | Column subsample | Same idea as `colsample_bytree` |
| `bagging_fraction` | Row subsample | Use with `bagging_freq` |
| `max_depth` | Limit depth | `-1` = no limit (watch `num_leaves`) |

> [!warning] Leaf-wise overfitting High `num_leaves` on small data overfits quickly — increase `min_child_samples` or use cross-validation.

---

## MLflow Integration

```python
import mlflow
import mlflow.lightgbm

with mlflow.start_run(run_name="lgbm-breast"):
    mlflow.log_params({"num_leaves": 31, "learning_rate": 0.05})
    mlflow.lightgbm.log_model(clf, artifact_path="model")
    mlflow.log_metric("test_auc", roc_auc_score(y_test, proba))
```

---

## Optuna Tuning

```python
import optuna

def objective(trial):
    params = {
        "num_leaves": trial.suggest_int("num_leaves", 16, 128),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.2, log=True),
        "min_child_samples": trial.suggest_int("min_child_samples", 5, 100),
        "subsample": trial.suggest_float("subsample", 0.6, 1.0),
        "colsample_bytree": trial.suggest_float("colsample_bytree", 0.6, 1.0),
        "reg_lambda": trial.suggest_float("reg_lambda", 1e-3, 10.0, log=True),
    }
    m = lgb.LGBMClassifier(**params, n_estimators=500, random_state=42)
    m.fit(
        X_train, y_train,
        eval_set=[(X_test, y_test)],
        eval_metric="auc",
        callbacks=[lgb.early_stopping(20, verbose=False), optuna.integration.LightGBMPruningCallback(trial, "auc")],
    )
    return roc_auc_score(y_test, m.predict_proba(X_test)[:, 1])

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=80)
```

See [[ML — Optuna]] for study storage and parallel jobs.

---

## Model Export

```python
clf.booster_.save_model("models/lgbm.txt")
# reload
bst = lgb.Booster(model_file="models/lgbm.txt")
```

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| Too many leaves | Cap `num_leaves`, raise `min_child_samples` |
| Categorical NA | Explicit category or impute before `category` dtype |
| Small datasets | Prefer shallower trees or [[ML — scikit-learn]] |
| Reproducibility | Set `random_state`, `bagging_freq`, fixed seeds |
| Warning spam | `verbose=-1` or `log_evaluation(0)` callback |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Classifier | `lgb.LGBMClassifier(...)` |
| Regressor | `lgb.LGBMRegressor(...)` |
| Early stop | `callbacks=[lgb.early_stopping(30)]` |
| Eval set | `eval_set=[(X_val, y_val)]` |
| Categories | `categorical_feature=cat_cols` |
| Native train | `lgb.train(params, train_data, ...)` |
| CV | `lgb.cv(params, dataset, nfold=5)` |
| MLflow | `mlflow.lightgbm.log_model(clf, "model")` |

---

## Related Notes

- [[Machine Learning]] — boosting in the stack
- [[ML — XGBoost]] — sibling booster
- [[ML — scikit-learn]] — pipelines and baselines
- [[ML — Optuna]], [[ML — MLflow]]
- [[ML — SHAP]] — `TreeExplainer` for LightGBM

---

## Tags

#python #lightgbm #gradient-boosting #tabular #machine-learning #categorical-features #mlflow #optuna
