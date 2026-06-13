## What & When

**Feast** (Feature Store for Machine Learning) is an open-source **feature store** for defining, versioning, and serving **training and online features** with **point-in-time correct** joins. It bridges batch pipelines ([[ML — pandas]], Spark) and low-latency inference behind [[API - FastAPI]].

Use Feast when:

- Multiple teams reuse the same **feature definitions**
- You need **historical features** for training without label leakage
- Online serving must match offline training values
- Complementing [[ML — MLflow]] (models) with centralized **feature registry**

```bash
pip install feast
# Optional offline stores: pip install feast[aws,redis,postgres]
feast init feature_repo   # scaffold repo
```

Train models with [[ML — scikit-learn]] on materialized feature tables; track runs in [[ML — MLflow]]. Overview: [[Machine Learning]].

---

## Feast vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Feature definitions + PIT joins | **Feast** | `FeatureView`, `entity_df` |
| Experiment tracking | [[ML — MLflow]] | Model + metrics, not features |
| Ad-hoc EDA | [[ML — pandas]] | Before promoting to Feast |
| Real-time-only cache | Redis alone | No registry / versioning |
| Batch feature scripts | [[ML — dbt]] / Spark | Feast consumes outputs |
| Model serving | [[ML — BentoML]], [[ML — Seldon]] | Load features at request time |

---

## Core Concepts

| Concept | Role |
| --- | --- |
| **Entity** | Join key (e.g. `customer_id`) |
| **Feature view** | Group of features from a data source |
| **Feature service** | Named bundle of views for training or serving |
| **Offline store** | Historical training data (Parquet, BigQuery, …) |
| **Online store** | Low-latency key-value (Redis, DynamoDB, SQLite) |
| **Registry** | Metadata (local `registry.db` or remote) |

---

## feature_store.yaml (Minimal)

```yaml
project: churn_features
registry: data/registry.db
provider: local
offline_store:
  type: file
online_store:
  type: sqlite
  path: data/online_store.db
entity_key_serialization_version: 2
```

---

## Define Entity & Feature View

```python
# features.py (inside feature_repo/)
from datetime import timedelta
from feast import Entity, FeatureView, Field, FileSource, ValueType
from feast.types import Float32, Int64

customer = Entity(
    name="customer_id",
    join_keys=["customer_id"],
    value_type=ValueType.INT64,
)

customer_stats_source = FileSource(
    path="data/customer_stats.parquet",
    timestamp_field="event_timestamp",
)

customer_stats_fv = FeatureView(
    name="customer_stats",
    entities=[customer],
    ttl=timedelta(days=90),
    schema=[
        Field(name="total_purchases", dtype=Int64),
        Field(name="avg_order_value", dtype=Float32),
        Field(name="days_since_last_order", dtype=Int64),
    ],
    source=customer_stats_source,
    online=True,
)
```

Apply definitions:

```bash
cd feature_repo
feast apply
```

---

## Materialize — Offline → Online

```python
# After batch job writes parquet with event_timestamp column
from datetime import datetime
from feast import FeatureStore

store = FeatureStore(repo_path="feature_repo/")

store.materialize(
    start_date=datetime(2024, 1, 1),
    end_date=datetime(2025, 6, 1),
)
```

Schedule `feast materialize-incremental` in [[ORCHESTRATION — Airflow]] / Cron for fresh online features.

---

## Historical Features (Training)

Point-in-time join prevents **future leakage**.

```python
import pandas as pd
from feast import FeatureStore

store = FeatureStore(repo_path="feature_repo/")

entity_df = pd.DataFrame({
    "customer_id": [101, 102, 103],
    "event_timestamp": pd.to_datetime([
        "2024-06-01",
        "2024-06-01",
        "2024-06-02",
    ]),
    "churn": [0, 1, 0],   # labels — only used after join for training
})

training_df = store.get_historical_features(
    entity_df=entity_df,
    features=[
        "customer_stats:total_purchases",
        "customer_stats:avg_order_value",
        "customer_stats:days_since_last_order",
    ],
).to_df()

# training_df → [[ML — scikit-learn]] / [[ML — XGBoost]]
```

---

## Online Retrieval (Inference)

```python
from feast import FeatureStore

store = FeatureStore(repo_path="feature_repo/")

features = store.get_online_features(
    features=[
        "customer_stats:total_purchases",
        "customer_stats:avg_order_value",
        "customer_stats:days_since_last_order",
    ],
    entity_rows=[{"customer_id": 101}],
).to_dict()

# features["total_purchases"] → list aligned with entity_rows
```

---

## Feature Service

```python
from feast import FeatureService

churn_fs = FeatureService(
    name="churn_v1",
    features=[customer_stats_fv],
)

# feature_services.py — register in repo, feast apply
```

```python
training_df = store.get_historical_features(
    entity_df=entity_df,
    features=store.get_feature_service("churn_v1"),
).to_df()
```

---

## FastAPI Inference Pattern

```python
from fastapi import FastAPI
from feast import FeatureStore
import mlflow.sklearn
import pandas as pd

app = FastAPI()
store = FeatureStore(repo_path="feature_repo/")
model = mlflow.sklearn.load_model("models:/churn-model/Production")

@app.get("/score/{customer_id}")
def score(customer_id: int):
    online = store.get_online_features(
        features=store.get_feature_service("churn_v1"),
        entity_rows=[{"customer_id": customer_id}],
    ).to_df()

    X = online.drop(columns=["customer_id"], errors="ignore")
    prob = model.predict_proba(X)[0, 1]
    return {"customer_id": customer_id, "churn_probability": float(prob)}
```

Wire auth and validation per [[API - FastAPI]]. Log prediction events separately from Feast materialization jobs.

---

## MLflow Workflow

1. Materialize features → `get_historical_features` → train [[ML — scikit-learn]].
2. Log model + **feature service name** + Feast commit hash in [[ML — MLflow]].
3. Deploy; at request time call `get_online_features` with same feature service.

```python
with mlflow.start_run():
    mlflow.log_param("feature_service", "churn_v1")
    mlflow.sklearn.log_model(clf, "model")
```

---

## Pitfalls

- **Timestamp column** must exist on sources; timezone-aware consistency matters.
- **TTL** — expired online features return null; set `ttl` to business horizon.
- **Schema drift** — change `FeatureView` schema deliberately; version views (`customer_stats_v2`).
- **Not a model server** — Feast serves features only; use [[ML — BentoML]] for model HTTP.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Init repo | `feast init feature_repo` |
| Apply defs | `feast apply` |
| Materialize | `store.materialize(start, end)` |
| Historical | `store.get_historical_features(entity_df, features=[...])` |
| Online | `store.get_online_features(features, entity_rows)` |
| Feature service | `get_feature_service("name")` |
| Store handle | `FeatureStore(repo_path="feature_repo/")` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — MLflow]]
- [[ML — pandas]]
- [[ML — BentoML]]
- [[API - FastAPI]]

---

## Tags

#python #machine-learning #feast #feature-store #mlops #point-in-time #online-features
