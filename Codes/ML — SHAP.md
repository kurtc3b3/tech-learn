## What & When

**SHAP** (SHapley Additive exPlanations) assigns each feature an **additive contribution** to a model's prediction, grounded in cooperative game theory. It works with tree models, linear models, deep nets (approximate), and any black-box via `KernelExplainer`.

Use SHAP when:

- Stakeholders ask **why** a prediction happened (credit, fraud, healthcare)
- Debugging a [[ML — scikit-learn]] / [[ML — XGBoost]] model after [[ML — Optuna]] tuning
- Comparing global vs local importance vs raw `feature_importances_`
- Building explanation APIs alongside [[API - FastAPI]] inference

```bash
pip install shap
# Optional plots: matplotlib already used by shap.plots
```

Log explanation summaries as artifacts in [[ML — MLflow]]. Overview: [[Machine Learning]].

---

## SHAP vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Unified feature attributions | **SHAP** | `TreeExplainer`, `LinearExplainer`, `KernelExplainer` |
| sklearn-only importances | `feature_importances_` | No per-row explanation |
| Partial dependence | `sklearn.inspection` | Marginal effect, not Shapley |
| Fairness / bias metrics | separate tooling | SHAP shows drivers, not legal compliance |
| Feature selection | [[ML — Boruta]] | Upstream of modeling |
| Track explainability runs | [[ML — MLflow]] | Save `shap_values` plots as artifacts |

---

## Tree Models — TreeExplainer (Fast)

```python
import shap
import pandas as pd
from sklearn.ensemble import GradientBoostingClassifier
from sklearn.model_selection import train_test_split

df = pd.read_csv("data/loan.csv")
X = df.drop(columns=["default"])
y = df["default"]

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = GradientBoostingClassifier(random_state=42)
model.fit(X_train, y_train)

explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X_test)   # list for multiclass; ndarray for binary

# Binary classification: use class-1 contributions
sv = shap_values[1] if isinstance(shap_values, list) else shap_values
```

---

## Modern Plots API

```python
import shap

# Global summary (beeswarm)
shap.summary_plot(sv, X_test, show=False)

# Single prediction — waterfall
row = 0
shap.waterfall_plot(
    shap.Explanation(
        values=sv[row],
        base_values=explainer.expected_value[1] if isinstance(explainer.expected_value, list) else explainer.expected_value,
        data=X_test.iloc[row],
        feature_names=X_test.columns.tolist(),
    )
)

# Bar plot of mean |SHAP|
shap.plots.bar(
    shap.Explanation(
        values=sv,
        base_values=explainer.expected_value,
        data=X_test.values,
        feature_names=X_test.columns.tolist(),
    ),
    max_display=15,
)
```

---

## XGBoost / LightGBM

```python
import xgboost as xgb

dtrain = xgb.DMatrix(X_train, label=y_train)
params = {"objective": "binary:logistic", "eval_metric": "auc"}
bst = xgb.train(params, dtrain, num_boost_round=200)

explainer = shap.TreeExplainer(bst)
shap_values = explainer.shap_values(xgb.DMatrix(X_test))

shap.summary_plot(shap_values, X_test)
```

For **LightGBM**, pass the booster or `LGBMClassifier` — `TreeExplainer` handles both when trees are available.

---

## Linear Models — LinearExplainer

```python
from sklearn.linear_model import LogisticRegression
from sklearn.preprocessing import StandardScaler
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ("scaler", StandardScaler()),
    ("clf", LogisticRegression(max_iter=2000)),
])
pipe.fit(X_train, y_train)

# Explain in transformed space or pass masker for original features
explainer = shap.Explainer(
    pipe.predict_proba,
    masker=shap.maskers.Independent(X_train),
    algorithm="linear",
)
shap_values = explainer(X_test.iloc[:100])
shap.plots.beeswarm(shap_values[..., 1])   # class 1 for binary proba
```

---

## KernelExplainer — Model-Agnostic (Slower)

```python
import numpy as np
from sklearn.svm import SVC

svc = SVC(probability=True, random_state=42)
svc.fit(X_train.sample(500, random_state=42), y_train.loc[X_train.sample(500, random_state=42).index])

background = shap.sample(X_train, 100, random_state=42)

def predict_proba_positive(x):
    return svc.predict_proba(x)[:, 1]

explainer = shap.KernelExplainer(predict_proba_positive, background)
shap_values = explainer.shap_values(X_test.iloc[:50], nsamples=200)
```

Use **small background** and **limited rows** — complexity grows quickly.

---

## FastAPI — Explain Endpoint Sketch

```python
from fastapi import FastAPI
from pydantic import BaseModel
import pandas as pd
import joblib

app = FastAPI()
bundle = joblib.load("models/gb_bundle.pkl")  # model + explainer + feature_order
model = bundle["model"]
explainer = bundle["explainer"]
columns = bundle["columns"]

class RowIn(BaseModel):
    features: dict[str, float]

@app.post("/predict/explain")
def explain(row: RowIn):
    X = pd.DataFrame([row.features])[columns]
    pred = model.predict_proba(X)[0, 1]
    sv = explainer.shap_values(X)
    vec = sv[1][0] if isinstance(sv, list) else sv[0]
    return {
        "probability": float(pred),
        "base_value": float(explainer.expected_value),
        "contributions": dict(zip(columns, map(float, vec))),
    }
```

See [[API - FastAPI]] for routers, dependency injection, and OpenAPI. Serve the model bundle via [[ML — BentoML]] in production.

---

## Multiclass & Regression

```python
# Multiclass: shap_values shape (n_samples, n_features, n_classes)
from sklearn.ensemble import RandomForestClassifier

rf = RandomForestClassifier(n_estimators=200, random_state=42)
rf.fit(X_train, y_train)
explainer = shap.TreeExplainer(rf)
sv = explainer.shap_values(X_test)

for class_idx, class_name in enumerate(rf.classes_):
    shap.summary_plot(sv[class_idx], X_test, title=str(class_name))

# Regression: single output matrix
from sklearn.ensemble import GradientBoostingRegressor

reg = GradientBoostingRegressor(random_state=42)
reg.fit(X_train, y_train)
explainer = shap.TreeExplainer(reg)
shap_values = explainer.shap_values(X_test)
```

---

## MLflow Artifact Logging

```python
import mlflow
import matplotlib.pyplot as plt
import shap

with mlflow.start_run(run_name="shap-report"):
    mlflow.sklearn.log_model(model, "model")
    shap.summary_plot(sv, X_test, show=False)
    fig = plt.gcf()
    mlflow.log_figure(fig, "shap_summary.png")
    plt.close()
```

---

## Pitfalls

- **Background data** must reflect production distribution for `KernelExplainer` / `Explainer` maskers.
- **Correlated features** — SHAP splits credit across correlated inputs; interpret groups together.
- **Large datasets** — subsample for plots; `TreeExplainer` is fast but plotting millions of points is not.
- **Class handling** — verify which class index you plot for binary classifiers.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Trees | `shap.TreeExplainer(model).shap_values(X)` |
| Any model | `shap.KernelExplainer(f, background).shap_values(X)` |
| Unified API | `shap.Explainer(model, X_train)(X_test)` |
| Beeswarm | `shap.summary_plot(values, features)` |
| Waterfall | `shap.waterfall_plot(shap.Explanation(...))` |
| Sample background | `shap.sample(X_train, 100)` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — XGBoost]]
- [[ML — Optuna]]
- [[ML — MLflow]]
- [[ML — BentoML]]
- [[API - FastAPI]]

---

## Tags

#python #machine-learning #shap #explainability #xai #sklearn #xgboost #interpretability
