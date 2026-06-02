
## What & When

**H2O.ai** provides a JVM-backed machine learning platform with a Python client (`h2o` package). It excels at **AutoML**, **distributed** training on clusters, and a consistent API across GLM, GBM, DRF, Deep Learning, and stacked ensembles — without hand-tuning every algorithm.

Use H2O when:

- You want **H2O AutoML** to search algorithms and ensembles automatically
- Data lives on **HDFS / S3** and you train on an H2O cluster
- Teams prefer a **managed ML** workflow over hand-picked [[ML — XGBoost]] params
- You need MOJO export for low-latency Java/scoring pipelines

For single-machine Python-first workflows, start with [[ML — scikit-learn]] or [[ML — LightGBM]] ([[Machine Learning]]). Track winners with [[ML — MLflow]]; refine top models with [[ML — Optuna]] on exported hyperparameters.

```bash
pip install h2o
# starts embedded local cluster; production uses remote h2o.connect()
```

---

## H2O vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Python-native tabular | [[ML — scikit-learn]], [[ML — XGBoost]] | Lighter footprint |
| Fast local boosting | [[ML — LightGBM]] | No JVM |
| AutoML (Python) | H2O AutoML | Also consider external tuners + sklearn |
| Deep learning research | [[ML — PyTorch]] | Flexible nn modules |
| Experiment tracking | [[ML — MLflow]] | Log AutoML leaderboard |
| Fine-grained tuning | [[ML — Optuna]] | After AutoML shortlist |

---

## Start Cluster & Load Data

```python
import h2o
from h2o.automl import H2OAutoML

h2o.init()  # local JVM; use h2o.connect(url="http://cluster:54321") in prod

# From pandas
import pandas as pd
df = pd.read_csv("data/churn.csv")
hf = h2o.H2OFrame(df)

# Or path directly
# hf = h2o.import_file("s3://bucket/churn.csv")
```

Always `h2o.cluster().shutdown()` in scripts when done (or rely on process exit).

---

## Frame Basics

```python
print(hf.describe())
print(hf.dim)
print(hf.names)
print(hf.types)

# Train / test split (H2OFrame method)
train, valid, test = hf.split_frame(ratios=[0.7, 0.15], seed=42)

x = hf.columns
x.remove("churn")
y = "churn"

# Factorize target for classification
train[y] = train[y].asfactor()
valid[y] = valid[y].asfactor()
test[y] = test[y].asfactor()
```

---

## H2O AutoML

```python
aml = H2OAutoML(
    max_models=20,
    max_runtime_secs=600,
    seed=42,
    project_name="churn_automl",
    sort_metric="AUC",
    stopping_metric="AUC",
    stopping_tolerance=0.001,
    stopping_rounds=3,
    balance_classes=True,       # imbalance
    nfolds=5,
    include_algos=["GBM", "XGBoost", "DRF", "GLM", "DeepLearning"],
)

aml.train(x=x, y=y, training_frame=train, validation_frame=valid)

lb = aml.leaderboard
print(lb.head(10))

leader = aml.leader
print(leader.model_id, leader.auc(valid=True))
```

AutoML trains multiple models, stacks ensembles, and ranks by the chosen metric.

---

## Predict & Evaluate

```python
preds = leader.predict(test)
perf = leader.model_performance(test)

print(perf.auc())
print(perf.confusion_matrix())
print(preds.head())
```

Download predictions as pandas:

```python
pred_df = preds.as_data_frame()
```

---

## Single Algorithm (GBM Example)

```python
from h2o.estimators.gbm import H2OGradientBoostingEstimator

gbm = H2OGradientBoostingEstimator(
    ntrees=500,
    learn_rate=0.05,
    max_depth=6,
    sample_rate=0.8,
    col_sample_rate=0.8,
    stopping_rounds=5,
    stopping_tolerance=0.001,
    stopping_metric="AUC",
    seed=42,
)
gbm.train(x=x, y=y, training_frame=train, validation_frame=valid)
print(gbm.auc(valid=True))
```

---

## Variable Importance

```python
vi = leader.varimp(use_pandas=True)
print(vi.head(15))
```

---

## MOJO Export (Production Scoring)

```python
model_path = h2o.save_model(model=leader, path="models/", force=True)
mojo_path = leader.download_mojo(path="models/", get_genmodel_jar=True)
# MOJO loads in H2O scoring, Sparkling Water, or dedicated scoring apps
```

Pair artifact paths with [[ML — MLflow]] `log_artifact` for versioning.

---

## MLflow Integration

```python
import mlflow

with mlflow.start_run(run_name="h2o-automl-churn"):
    mlflow.log_param("max_models", 20)
    mlflow.log_metric("leader_auc", leader.auc(valid=True))
    mlflow.log_text(lb.as_data_frame().to_csv(), "leaderboard.csv")
    # Save MOJO / model dir as artifacts
    mlflow.log_artifact(mojo_path)
```

There is no first-class `mlflow.h2o` flavor in all versions — treat MOJO + leaderboard CSV as artifacts, or export to sklearn-compatible formats when available.

---

## Optuna After AutoML

Typical workflow:

1. Run AutoML to identify algorithm family (e.g. GBM wins).
2. Export best hyperparameter ranges from leaderboard.
3. Use [[ML — Optuna]] on a smaller Python model ([[ML — XGBoost]] / [[ML — LightGBM]]) for faster iteration, or narrow H2O grids with `hyper_parameters` in `H2OGridSearch`.

```python
from h2o.grid.grid_search import H2OGridSearch

hyperparams = {
    "learn_rate": [0.01, 0.05, 0.1],
    "max_depth": [4, 6, 8],
    "ntrees": [300, 500],
}
grid = H2OGridSearch(model=H2OGradientBoostingEstimator, hyper_params=hyperparams)
grid.train(x=x, y=y, training_frame=train, validation_frame=valid)
print(grid.get_grid(sort_by="auc", decreasing=True))
```

---

## Shutdown

```python
h2o.cluster().shutdown(prompt=False)
```

---

## Pitfalls

| Issue | Fix |
| --- | --- |
| JVM memory | `h2o.init(max_mem_size="16G")` |
| Leakage | Split frames before AutoML; no target in `x` |
| ID columns | Exclude high-cardinality IDs from `x` |
| Python 3.12+ | Check wheel compatibility for your H2O version |
| Local vs cluster | `h2o.connect` for shared cluster resources |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Init | `h2o.init()` / `h2o.connect(url=...)` |
| Load | `h2o.H2OFrame(df)` / `h2o.import_file(path)` |
| Split | `hf.split_frame(ratios=[0.7, 0.15], seed=42)` |
| AutoML | `H2OAutoML(...); aml.train(x, y, training_frame=train)` |
| Leader | `aml.leader` |
| Leaderboard | `aml.leaderboard` |
| Predict | `model.predict(test)` |
| Save MOJO | `leader.download_mojo(path="models/")` |
| Shutdown | `h2o.cluster().shutdown()` |

---

## Related Notes

- [[Machine Learning]] — AutoML in the stack map
- [[ML — scikit-learn]], [[ML — XGBoost]], [[ML — LightGBM]] — Python-native alternatives
- [[ML — MLflow]], [[ML — Optuna]]
- [[Processing]] — distributed jobs (Ray, Celery) when H2O cluster is separate

---

## Tags

#python #h2o #automl #machine-learning #tabular #distributed #mojo #mlflow #optuna
