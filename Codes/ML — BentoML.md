## What & When

**BentoML** packages trained models into **Bentos** — versioned, container-ready bundles with a **Python service API** for batch and online inference. It is the vault's preferred path from [[ML — MLflow]] / [[ML — scikit-learn]] to a production HTTP service, often fronted by [[API - FastAPI]] or Bento's own server.

Use BentoML when:

- You want **one command** to build a Docker image from a sklearn/XGBoost/PyTorch model
- Serving must scale with **adaptive batching** and multiple runners
- Importing models from **MLflow registry** URIs
- Simpler ops than full [[ML — Seldon]] on Kubernetes for a single team service

```bash
pip install bentoml
# Optional: pip install bentoml[grpc] xgboost
```

Track source runs in [[ML — MLflow]]; fetch online features from [[ML — Feast]] inside the service. Overview: [[Machine Learning]].

---

## BentoML vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Python-native model packaging | **BentoML** | `bentoml.models`, `@bentoml.service` |
| K8s-native multi-model mesh | [[ML — Seldon]] | SeldonDeployment CRD |
| Experiment tracking | [[ML — MLflow]] | `bentoml.mlflow.import_model` |
| Custom API layer | [[API - FastAPI]] | Gateway or separate app calling Bento |
| Cloud vendor endpoints | SageMaker / Vertex | Export from Bento containers |
| Explainability | [[ML — SHAP]] | Compute in service `predict` path |

---

## Save a sklearn Model

```python
import bentoml
from sklearn.ensemble import RandomForestClassifier
from sklearn.datasets import load_iris

X, y = load_iris(return_X_y=True)
clf = RandomForestClassifier(n_estimators=100, random_state=42)
clf.fit(X, y)

tag = bentoml.sklearn.save_model("iris_rf", clf, signatures={
    "predict": {"batchable": True},
    "predict_proba": {"batchable": True},
})
print(tag)   # Model("iris_rf", ...)
```

```python
# Load in another process
model = bentoml.sklearn.load_model("iris_rf:latest")
pred = model.predict(X[:5])
```

---

## Service Definition (BentoML 1.2+)

```python
# service.py
from __future__ import annotations
import numpy as np
import bentoml
from bentoml.io import NumpyNdarray

rf_runner = bentoml.models.get("iris_rf:latest").to_runner()

svc = bentoml.Service("iris_classifier", runners=[rf_runner])

@svc.api(input=NumpyNdarray(), output=NumpyNdarray())
async def classify(input_arr: np.ndarray) -> np.ndarray:
    return await rf_runner.predict.async_run(input_arr)
```

```bash
bentoml serve service.py:svc --reload
# curl -X POST http://127.0.0.1:3000/classify -H "Content-Type: application/json" -d '{"ndarray": [[5.1,3.5,1.4,0.2]]}'
```

---

## Build Bento & Container

```python
# bentofile.yaml (optional) or Python API
import bentoml

bentoml.bentos.build_service(
    "service.py:svc",
    name="iris_svc",
    version="v1",
    labels={"team": "ml-platform"},
)
```

```bash
bentoml containerize iris_svc:v1 -t iris-classifier:latest
docker run -p 3000:3000 iris-classifier:latest
```

---

## Import from MLflow

```python
import bentoml

bentoml.mlflow.import_model(
    "iris-rf",
    model_uri="models:/iris-rf/Production",
    signatures={"predict": {"batchable": True}},
)
```

Keeps **registry stage** as source of truth; rebuild Bento when Production version changes.

---

## Custom Logic — Pre/Post Processing

```python
import bentoml
import numpy as np
from bentoml.io import JSON

runner = bentoml.models.get("iris_rf:latest").to_runner()
svc = bentoml.Service("iris_json", runners=[runner])

@svc.api(input=JSON(), output=JSON())
async def predict(payload: dict) -> dict:
    features = np.array([payload["features"]], dtype=np.float64)
    proba = await runner.predict_proba.async_run(features)
    return {"probabilities": proba[0].tolist(), "class": int(proba[0].argmax())}
```

Combine with [[ML — Feast]] by loading features inside this handler before calling the runner.

---

## FastAPI + Bento (Gateway Pattern)

Run Bento as the model worker; expose public API with [[API - FastAPI]] for auth, rate limits, and OpenAPI.

```python
from fastapi import FastAPI
import httpx

app = FastAPI()
BENTO_URL = "http://127.0.0.1:3000"

@app.post("/v1/iris")
async def iris_proxy(features: list[float]):
    async with httpx.AsyncClient() as client:
        r = await client.post(
            f"{BENTO_URL}/predict",
            json={"ndarray": [features]},
            timeout=5.0,
        )
        r.raise_for_status()
        return r.json()
```

For a single service, Bento alone is enough; add FastAPI when you need unified API gateway semantics.

---

## Adaptive Batching

```python
@bentoml.service(
    resources={"cpu": "2"},
    traffic={"timeout": 60},
)
class IrisService:
    model = bentoml.models.get("iris_rf:latest")

    @bentoml.api(batchable=True, batch_dim=0, max_batch_size=32, max_latency_ms=20)
    def predict(self, inputs: np.ndarray) -> np.ndarray:
        return self.model.predict(inputs)
```

Batching increases throughput for GPU/CPU-bound models under concurrent load.

---

## Monitoring Hooks

```python
import bentoml
from prometheus_client import Counter

PREDICTIONS = Counter("bento_predictions_total", "Total predictions")

@svc.api(input=NumpyNdarray(), output=NumpyNdarray())
async def classify(x: np.ndarray) -> np.ndarray:
    result = await rf_runner.predict.async_run(x)
    PREDICTIONS.inc(len(x))
    return result
```

Pair with Grafana; log training metrics in [[ML — MLflow]] separately.

---

## SHAP in Service (Optional)

```python
import shap

@svc.api(input=JSON(), output=JSON())
async def explain(payload: dict) -> dict:
    x = np.array([payload["features"]])
    model = bentoml.sklearn.load_model("iris_rf:latest")
    explainer = shap.TreeExplainer(model)
    sv = explainer.shap_values(x)
    return {"shap": sv[0].tolist()}
```

Cache explainers at startup for latency. See [[ML — SHAP]].

---

## Deployment Checklist

1. Train → log [[ML — MLflow]] → register Production.
2. `bentoml.mlflow.import_model` or `save_model`.
3. Implement `@bentoml.service` with preprocessing.
4. `bentoml build` / `containerize` → push image.
5. Deploy to K8s / ECS; optional [[ML — Seldon]] wrapper for mesh features.

---

## Pitfalls

- **Signature mismatch** — input dtype/shape must match training.
- **Sync vs async** — use `async_run` inside async APIs.
- **Cold start** — large models: increase container memory, warmup probes.
- **Version tags** — pin `model:latest` carefully in production; prefer immutable tags.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Save sklearn | `bentoml.sklearn.save_model("name", model)` |
| Load | `bentoml.sklearn.load_model("name:latest")` |
| Service | `@bentoml.service` / `bentoml.Service(...)` |
| Serve | `bentoml serve service.py:svc` |
| Build | `bentoml.bentos.build_service("service.py:svc")` |
| Docker | `bentoml containerize bento_tag` |
| MLflow import | `bentoml.mlflow.import_model(name, model_uri=...)` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — MLflow]]
- [[ML — Seldon]]
- [[ML — Feast]]
- [[ML — SHAP]]
- [[API - FastAPI]]

---

## Tags

#python #machine-learning #bentoml #model-serving #mlops #docker #sklearn #inference
