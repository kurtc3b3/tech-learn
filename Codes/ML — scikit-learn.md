
## What & When

**scikit-learn** (imported as `sklearn`) is the default **classical ML** library in Python — consistent APIs for preprocessing, models, pipelines, cross-validation, and metrics. It is the first stop in this vault for tabular classification, regression, clustering, and baseline models before boosting or deep learning.

Use scikit-learn when:

- Building **end-to-end tabular pipelines** (impute → scale → model)
- You need **reproducible CV**, metrics, and model selection without heavy dependencies
- Prototyping before [[ML — XGBoost]] / [[ML — LightGBM]]
- Feature engineering with `ColumnTransformer`, `Pipeline`, and `GridSearchCV`
- Time-series baselines with lag features (not native forecasting)

Part of the stack in [[Machine Learning]]. Track runs with [[ML — MLflow]]; tune hyperparameters with [[ML — Optuna]] or `GridSearchCV`.

```bash
pip install scikit-learn
# often with:
pip install pandas numpy scipy
```

---

## sklearn vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Tabular baseline / pipelines | scikit-learn | Unified `fit` / `predict` API |
| Higher tabular accuracy | [[ML — XGBoost]], [[ML — LightGBM]] | Gradient boosting |
| AutoML on JVM cluster | [[ML — H2O]] | Separate runtime |
| Neural networks | [[ML — PyTorch]] | Custom architectures |
| Business forecasting | [[ML — Prophet]] | Additive seasonality |
| Experiment tracking | [[ML — MLflow]] | Params, metrics, artifacts |
| Hyperparameter search | [[ML — Optuna]] | Pruning, parallel trials |

---

## Core Pattern — Train / Evaluate

```python
import pandas as pd
from sklearn.datasets import load_breast_cancer
from sklearn.model_selection import train_test_split, cross_val_score
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import LogisticRegression
from sklearn.pipeline import Pipeline
from sklearn.metrics import classification_report, roc_auc_score

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("clf", LogisticRegression(max_iter=10_000, random_state=42)),
])
pipe.fit(X_train, y_train)

y_pred = pipe.predict(X_test)
y_proba = pipe.predict_proba(X_test)[:, 1]
print(classification_report(y_test, y_pred))
print("ROC-AUC:", roc_auc_score(y_test, y_proba))

cv_scores = cross_val_score(pipe, X_train, y_train, cv=5, scoring="roc_auc")
print(f"CV ROC-AUC: {cv_scores.mean():.3f} ± {cv_scores.std():.3f}")
```

---

## Pipelines & ColumnTransformer

Keep preprocessing inside the pipeline so CV and deployment see the same transforms.

```python
from sklearn.compose import ColumnTransformer
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import OneHotEncoder, StandardScaler
from sklearn.ensemble import RandomForestClassifier

numeric_features = ["age", "income"]
categorical_features = ["region", "tier"]

preprocessor = ColumnTransformer(
    transformers=[
        ("num", Pipeline([
            ("imputer", SimpleImputer(strategy="median")),
            ("scaler", StandardScaler()),
        ]), numeric_features),
        ("cat", Pipeline([
            ("imputer", SimpleImputer(strategy="most_frequent")),
            ("onehot", OneHotEncoder(handle_unknown="ignore")),
        ]), categorical_features),
    ]
)

model = Pipeline([
    ("prep", preprocessor),
    ("clf", RandomForestClassifier(n_estimators=200, random_state=42)),
])

# df: pandas DataFrame with columns above + target "churn"
# model.fit(df, df["churn"])
```

> [!tip] Always `fit` on training data only Fit preprocessor + estimator via `Pipeline.fit(X_train, y_train)` — never fit the scaler on the full dataset before splitting.

---

## Model Selection

```python
from sklearn.model_selection import GridSearchCV, StratifiedKFold

param_grid = {
    "clf__n_estimators": [100, 200],
    "clf__max_depth": [None, 10, 20],
    "clf__min_samples_leaf": [1, 5],
}

search = GridSearchCV(
    model,
    param_grid,
    cv=StratifiedKFold(5, shuffle=True, random_state=42),
    scoring="roc_auc",
    n_jobs=-1,
    refit=True,
)
search.fit(X_train, y_train)
print(search.best_params_, search.best_score_)
best_model = search.best_estimator_
```

For large search spaces, prefer [[ML — Optuna]] with `optuna.integration.sklearn.OptunaSearchCV` or a custom objective.

---

## Common Estimators (Quick Map)

| Task | Estimators |
| --- | --- |
| Linear classify/regress | `LogisticRegression`, `Ridge`, `Lasso`, `ElasticNet` |
| Trees / ensembles | `RandomForestClassifier`, `GradientBoostingClassifier`, `HistGradientBoostingClassifier` |
| SVM | `SVC`, `SVR` (scale features first) |
| Neighbors | `KNeighborsClassifier` |
| Clustering | `KMeans`, `DBSCAN`, `AgglomerativeClustering` |
| Dim reduction | `PCA`, `TruncatedSVD` |
| Anomaly | `IsolationForest`, `LocalOutlierFactor` |

`HistGradientBoostingClassifier` is sklearn’s fast boosting — a middle ground before [[ML — XGBoost]].

---

## Persistence

```python
import joblib

joblib.dump(best_model, "models/churn_pipeline.joblib")
loaded = joblib.load("models/churn_pipeline.joblib")
loaded.predict(X_test)
```

Log the artifact path and metrics in [[ML — MLflow]] (`mlflow.sklearn.log_model`).

---

## MLflow Integration

```python
import mlflow
import mlflow.sklearn

with mlflow.start_run(run_name="rf-churn"):
    mlflow.log_params(search.best_params_)
    mlflow.sklearn.log_model(best_model, artifact_path="model")
    mlflow.log_metric("test_roc_auc", roc_auc_score(y_test, best_model.predict_proba(X_test)[:, 1]))
```

---

## Optuna + sklearn

```python
import optuna
from sklearn.model_selection import cross_val_score

def objective(trial):
    params = {
        "clf__n_estimators": trial.suggest_int("n_estimators", 50, 400),
        "clf__max_depth": trial.suggest_int("max_depth", 3, 20),
        "clf__min_samples_leaf": trial.suggest_int("min_samples_leaf", 1, 10),
    }
    pipe.set_params(**params)
    return cross_val_score(pipe, X_train, y_train, cv=5, scoring="roc_auc").mean()

study = optuna.create_study(direction="maximize")
study.optimize(objective, n_trials=50)
pipe.set_params(**{f"clf__{k}": v for k, v in study.best_params.items()})
pipe.fit(X_train, y_train)
```

See [[ML — Optuna]] for pruning, storage, and parallel workers.

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| Data leakage | `Pipeline` + split before any `fit` |
| Imbalanced classes | `class_weight="balanced"` or `imblearn` (external) |
| High-cardinality categoricals | Target encoding (careful CV) or hashing |
| Sparse text | `TfidfVectorizer` in pipeline, then linear model |
| Slow grid search | [[ML — Optuna]] or `HalvingGridSearchCV` |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Split | `train_test_split(X, y, stratify=y, random_state=42)` |
| Pipeline | `Pipeline([("prep", prep), ("clf", clf)])` |
| Mixed columns | `ColumnTransformer([...])` |
| CV score | `cross_val_score(est, X, y, cv=5, scoring="roc_auc")` |
| Grid search | `GridSearchCV(pipe, param_grid, cv=5)` |
| Metrics | `classification_report`, `mean_squared_error` |
| Save model | `joblib.dump(model, path)` |
| MLflow | `mlflow.sklearn.log_model(model, "model")` |

---

## Related Notes

- [[Machine Learning]] — stack map and lifecycle
- [[ML — pandas]], [[ML — NumPy]] — data foundation
- [[ML — XGBoost]], [[ML — LightGBM]] — tabular performance
- [[ML — Optuna]], [[ML — MLflow]] — tune and track
- [[ML — SHAP]] — explain predictions

---

## Tags

#python #sklearn #scikit-learn #machine-learning #tabular #pipeline #classification #regression #mlflow #optuna
