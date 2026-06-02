
## What & When

**XGBoost** (eXtreme Gradient Boosting) is a high-performance **gradient boosting** library for tabular classification, regression, ranking, and survival tasks. It dominates many Kaggle-style problems and production tabular pipelines when you need strong accuracy with reasonable training time.

Use XGBoost when:

- [[ML — scikit-learn]] baselines plateau on structured/tabular data
- You need **GPU training** (`tree_method="gpu_hist"`) or **distributed** (`DMatrix` + cluster)
- Feature importance and early stopping matter
- You export models to **ONNX** or other runtimes (with extras)

Default “level up” from sklearn in [[Machine Learning]]. Pair with [[ML — MLflow]] for experiments and [[ML — Optuna]] for hyperparameter search.

```bash
pip install xgboost
# optional GPU build — follow platform docs for cuda-enabled wheel
```

---

## XGBoost vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Fast tabular, large data | [[ML — LightGBM]] | Leaf-wise, often faster |
| JVM / AutoML cluster | [[ML — H2O]] | Enterprise AutoML |
| Simple unified API | [[ML — scikit-learn]] | `HistGradientBoostingClassifier` |
| Deep learning | [[ML — PyTorch]] | Unstructured / sequences |
| Tuning | [[ML — Optuna]] | `trial.suggest_*` + `xgb.train` |
| Tracking | [[ML — MLflow]] | `mlflow.xgboost.log_model` |

---

## sklearn API (Recommended Start)

```python
import xgboost as xgb
from sklearn.datasets import load_diabetes
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_squared_error
import numpy as np

X, y = load_diabetes(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = xgb.XGBRegressor(
    n_estimators=500,
    learning_rate=0.05,
    max_depth=6,
    subsample=0.8,
    colsample_bytree=0.8,
    reg_alpha=0.1,
    reg_lambda=1.0,
    random_state=42,
    early_stopping_rounds=30,
    eval_metric="rmse",
)

model.fit(
    X_train, y_train,
    eval_set=[(X_test, y_test)],
    verbose=False,
)

pred = model.predict(X_test)
rmse = np.sqrt(mean_squared_error(y_test, pred))
print(f"RMSE: {rmse:.3f}")
```

---

## Native API — `DMatrix` + `xgb.train`

More control for custom objectives, weights, and distributed training.

```python
dtrain = xgb.DMatrix(X_train, label=y_train)
dvalid = xgb.DMatrix(X_test, label=y_test)

params = {
    "objective": "reg:squarederror",
    "eval_metric": "rmse",
    "max_depth": 6,
    "eta": 0.05,
    "subsample": 0.8,
    "colsample_bytree": 0.8,
    "seed": 42,
}

evals = [(dtrain, "train"), (dvalid, "valid")]
bst = xgb.train(
    params,
    dtrain,
    num_boost_round=1000,
    evals=evals,
    early_stopping_rounds=30,
    verbose_eval=50,
)

pred = bst.predict(dvalid)
```

---

## Classification Example

```python
from sklearn.datasets import load_breast_cancer
from sklearn.metrics import roc_auc_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, stratify=y, random_state=42)

clf = xgb.XGBClassifier(
    n_estimators=300,
    learning_rate=0.1,
    max_depth=4,
    scale_pos_weight=1,          # adjust for imbalance
    eval_metric="auc",
    random_state=42,
)
clf.fit(X_train, y_train, eval_set=[(X_test, y_test)], verbose=False)
print("AUC:", roc_auc_score(y_test, clf.predict_proba(X_test)[:, 1]))
```

---

## Key Hyperparameters

| Param | Role | Typical range |
| --- | --- | --- |
| `learning_rate` / `eta` | Shrinkage per tree | 0.01–0.3 |
| `max_depth` | Tree depth | 3–10 |
| `n_estimators` / `num_boost_round` | Boosting rounds | 100–2000 + early stop |
| `subsample` | Row sampling | 0.6–1.0 |
| `colsample_bytree` | Column sampling | 0.6–1.0 |
| `reg_alpha`, `reg_lambda` | L1 / L2 on weights | 0–10 |
| `min_child_weight` | Min sum hessian in child | 1–10 |

---

## Early Stopping & Best Iteration

```python
# After fit with early_stopping_rounds:
best_rounds = model.best_iteration
# Refit on full train+valid or use best_iteration in predict:
model.predict(X_test, iteration_range=(0, model.best_iteration + 1))
```

---

## Feature Importance

```python
import pandas as pd
series = pd.Series(model.feature_importances_, index=[f"f{i}" for i in range(X.shape[1])])
print(series.sort_values(ascending=False).head(10))
```

See [[ML — SHAP]] (`shap.TreeExplainer`) for per-prediction explanations.

---

## MLflow Integration

```python
import mlflow
import mlflow.xgboost

with mlflow.start_run(run_name="xgb-diabetes"):
    mlflow.log_params({
        "max_depth": 6,
        "learning_rate": 0.05,
        "n_estimators": model.best_iteration + 1 if hasattr(model, "best_iteration") else 500,
    })
    mlflow.xgboost.log_model(model, artifact_path="model")
    mlflow.log_metric("test_rmse", rmse)
```

---

## Optuna Tuning

```python
import optuna

def objective(trial):
    params = {
        "max_depth": trial.suggest_int("max_depth", 3, 10),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
        "subsample": trial.suggest_float("subsample", 0.6, 1.0),
        "colsample_bytree": trial.suggest_float("colsample_bytree", 0.6, 1.0),
        "reg_lambda": trial.suggest_float("reg_lambda", 1e-3, 10.0, log=True),
    }
    m = xgb.XGBRegressor(**params, n_estimators=500, random_state=42, early_stopping_rounds=20)
    m.fit(X_train, y_train, eval_set=[(X_test, y_test)], verbose=False)
    pred = m.predict(X_test)
    return np.sqrt(mean_squared_error(y_test, pred))

study = optuna.create_study(direction="minimize")
study.optimize(objective, n_trials=100)
```

Use `optuna.integration.XGBoostPruningCallback` when calling `xgb.train` directly.

---

## Saving & Loading

```python
model.save_model("models/xgb.json")
loaded = xgb.XGBRegressor()
loaded.load_model("models/xgb.json")
```

Prefer MLflow model registry for versioned promotion (see [[ML — MLflow]]).

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| Overfitting | Lower `max_depth`, more `reg_*`, early stopping |
| Slow on wide sparse data | Consider [[ML — LightGBM]] |
| Categorical columns | `enable_categorical=True` (XGB 2.x) or encode in pipeline |
| Leakage in CV | Wrap in sklearn `Pipeline`; tune inside CV folds |
| Class imbalance | `scale_pos_weight` ≈ neg/pos ratio |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Regressor | `xgb.XGBRegressor(...)` |
| Classifier | `xgb.XGBClassifier(...)` |
| Early stop | `early_stopping_rounds=30`, `eval_set=[(X_val, y_val)]` |
| Native train | `xgb.train(params, dtrain, num_boost_round=...)` |
| DMatrix | `xgb.DMatrix(X, label=y)` |
| GPU | `tree_method="gpu_hist"` (when built with CUDA) |
| Save | `model.save_model("model.json")` |
| MLflow | `mlflow.xgboost.log_model(model, "model")` |

---

## Related Notes

- [[Machine Learning]] — when to use boosting
- [[ML — scikit-learn]] — baselines and pipelines
- [[ML — LightGBM]], [[ML — H2O]] — alternatives
- [[ML — Optuna]], [[ML — MLflow]] — tune and track
- [[ML — SHAP]] — interpretability

---

## Tags

#python #xgboost #gradient-boosting #tabular #machine-learning #classification #regression #mlflow #optuna
