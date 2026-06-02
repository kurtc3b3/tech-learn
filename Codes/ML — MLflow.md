## What & When

**MLflow** is an open platform for the **ML lifecycle** — experiment tracking, model packaging, a **model registry**, and deployment hooks. It is the default way in this vault to record params, metrics, and artifacts before serving with [[ML — BentoML]] or [[ML — Seldon]].

Use MLflow when:

- Comparing runs from [[ML — Optuna]] or manual [[ML — scikit-learn]] training
- Versioning models and promoting **Staging → Production**
- Logging datasets, plots ([[ML — SHAP]] summaries), and environment specs
- Team needs a **central tracking server** (SQLite locally, Postgres + S3 in prod)

```bash
pip install mlflow
# sklearn / xgboost flavors ship with mlflow; extras: pip install mlflow[extras]
```

Expose inference via [[API - FastAPI]] or MLflow's built-in scoring server. Overview: [[Machine Learning]].

---

## MLflow vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Experiment tracking + registry | **MLflow** | `mlflow.start_run`, Model Registry |
| Hyperparameter search | [[ML — Optuna]] | Log `best_params` to MLflow |
| Feature store | [[ML — Feast]] | Complementary; not a tracker |
| Packaging for K8s | [[ML — Seldon]] | Often loads MLflow model URI |
| Python REST serving | [[ML — BentoML]] | Can import MLflow models |
| Explainability artifacts | [[ML — SHAP]] | `log_artifact` / `log_figure` |

---

## Local Tracking — Quickstart

```bash
export MLFLOW_TRACKING_URI=sqlite:///mlflow.db
# UI: mlflow ui --backend-store-uri sqlite:///mlflow.db
```

```python
import mlflow
import mlflow.sklearn
from sklearn.datasets import load_iris
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score
from sklearn.metrics import accuracy_score

mlflow.set_experiment("iris-baseline")

X, y = load_iris(return_X_y=True)

with mlflow.start_run(run_name="rf-200-trees") as run:
    params = {"n_estimators": 200, "max_depth": 6, "random_state": 42}
    mlflow.log_params(params)

    clf = RandomForestClassifier(**params)
    auc_scores = cross_val_score(clf, X, y, cv=5, scoring="accuracy")
    mlflow.log_metric("cv_accuracy_mean", auc_scores.mean())
    mlflow.log_metric("cv_accuracy_std", auc_scores.std())

    clf.fit(X, y)
    mlflow.sklearn.log_model(
        clf,
        artifact_path="model",
        input_example=X[:1],
        registered_model_name="iris-rf",   # creates registry entry if permissions allow
    )

    print("run_id:", run.info.run_id)
```

---

## Autologging

```python
import mlflow

mlflow.sklearn.autolog(log_input_examples=True, log_models=True)

from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import train_test_split
from sklearn.datasets import load_breast_cancer

X, y = load_breast_cancer(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

with mlflow.start_run():
    clf = LogisticRegression(max_iter=2000)
    clf.fit(X_train, y_train)   # params, metrics, model logged automatically
```

```python
# XGBoost
mlflow.xgboost.autolog()
```

---

## Model Registry — Stages

```python
from mlflow.tracking import MlflowClient

client = MlflowClient()

# Latest version of registered model
versions = client.search_model_versions("name='iris-rf'")
version = max(int(v.version) for v in versions)

client.transition_model_version(
    name="iris-rf",
    version=version,
    stage="Staging",
)

# Production promotion (after validation)
client.transition_model_version(
    name="iris-rf",
    version=version,
    stage="Production",
    archive_existing_versions=True,
)
```

Load by stage in serving code:

```python
model_uri = "models:/iris-rf/Production"
loaded = mlflow.sklearn.load_model(model_uri)
```

---

## Artifacts & Datasets

```python
import matplotlib.pyplot as plt
import pandas as pd

with mlflow.start_run():
    df = pd.read_csv("data/metrics.csv")
    df.to_csv("summary.csv", index=False)
    mlflow.log_artifact("summary.csv")

    fig, ax = plt.subplots()
    df.plot(x="epoch", y="loss", ax=ax)
    mlflow.log_figure(fig, "loss_curve.png")
    plt.close()

    # MLflow 2.x datasets (optional)
    mlflow.log_input(mlflow.data.from_pandas(df, source="metrics.csv"), context="training")
```

Store [[ML — SHAP]] plots and [[ML — Boruta]] feature JSON the same way.

---

## Remote Tracking Server

```python
import mlflow

mlflow.set_tracking_uri("https://mlflow.example.com")
mlflow.set_experiment("team-churn")

with mlflow.start_run():
    mlflow.log_param("solver", "lbfgs")
```

Typical stack: **Postgres** (backend store) + **S3/MinIO** (artifact store). Set via `mlflow server` env vars or Helm chart.

---

## Load Model in FastAPI

```python
from fastapi import FastAPI
from pydantic import BaseModel
import mlflow.sklearn
import numpy as np

app = FastAPI()
model = mlflow.sklearn.load_model("models:/iris-rf/Production")

class Features(BaseModel):
    values: list[float]

@app.post("/predict")
def predict(body: Features):
    X = np.array([body.values])
    pred = model.predict(X)
    return {"class": int(pred[0])}
```

See [[API - FastAPI]] for validation, auth, and lifespan loading. For high-throughput serving, export to [[ML — BentoML]] or deploy on [[ML — Seldon]] with the MLflow model URI.

---

## Optuna + MLflow Pattern

```python
def objective(trial):
    params = {"C": trial.suggest_float("C", 1e-3, 1e2, log=True)}
    with mlflow.start_run(nested=True):
        mlflow.log_params(params)
        # train & log metrics
        return score

with mlflow.start_run(run_name="optuna-parent"):
    study.optimize(objective, n_trials=50)
    mlflow.log_params(study.best_params)
    mlflow.log_metric("best_score", study.best_value)
```

---

## Pitfalls

- **Relative paths** — `log_artifact` needs the file to exist at log time.
- **Registry permissions** — `registered_model_name` requires server config.
- **Environment drift** — log `conda.yaml` / `pip freeze` or use MLflow models with `mlflow.pyfunc`.
- **Secrets** — never log API keys; use env vars in deployment.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Set experiment | `mlflow.set_experiment("name")` |
| Run | `with mlflow.start_run():` |
| Log param/metric | `mlflow.log_param`, `mlflow.log_metric` |
| Log sklearn model | `mlflow.sklearn.log_model(model, "model")` |
| Load model | `mlflow.sklearn.load_model("runs:/<id>/model")` |
| Registry URI | `models:/name/Production` |
| UI | `mlflow ui` |
| Autolog | `mlflow.sklearn.autolog()` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — Optuna]]
- [[ML — SHAP]]
- [[ML — BentoML]]
- [[ML — Seldon]]
- [[ML — Feast]]
- [[API - FastAPI]]

---

## Tags

#python #machine-learning #mlflow #mlops #experiment-tracking #model-registry #sklearn
