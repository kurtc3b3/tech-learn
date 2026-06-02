## What & When

**Ray** is a **distributed runtime** for Python: a cluster scheduler, shared **object store**, and libraries for data (**Ray Data**), tuning (**Ray Tune**), and serving (**Ray Serve**). Tasks are `@ray.remote` functions; **actors** hold mutable state across calls.

Use Ray when:

- You need **parallel CPU/GPU** work across cores or machines (not a message-queue mental model — see [[Processing]])
- Batch scoring, simulation, or ETL maps over large datasets
- Hyperparameter search at scale — complements [[ML — Optuna]]; log winners to [[ML — MLflow]]
- Distributed training around [[ML — PyTorch]] (Ray Train)
- Model deployment with **Ray Serve** behind [[API - FastAPI]]

```bash
pip install "ray[default]"
# Optional: pip install "ray[data,tune,serve,train]"
```

Local laptop: `ray.init()` starts a cluster in-process. Production: K8s / Anyscale. Overview: [[Processing]].

---

## Ray vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Task queue + cron | [[Processing — Celery]] | Broker, Beat, `.delay()` |
| Async I/O | [[Python — asyncio]] | Single-process concurrency |
| One-machine cores | [[Python — multiprocessing]] | No distributed object store |
| **Parallel map / actors / ML** | **Ray** | `ray.get`, Tune, Serve |
| Small HPO on one machine | [[ML — Optuna]] | Lighter; Ray Tune for cluster |
| Experiment tracking | [[ML — MLflow]] | Log Tune trials / Serve versions |
| HTTP gateway | [[API - FastAPI]] | Proxy to Ray Serve deployments |

---

## `ray.init` & Cluster Basics

```python
import ray

# Local: all cores on laptop
ray.init(ignore_reinit_error=True)

# Connect to existing cluster
# ray.init(address="ray://head:10001")

# Shutdown when script ends (optional in notebooks)
# ray.shutdown()
```

Cluster CLI: `ray start --head` then `ray start --address='head:6379'`. GPUs: `ray.init(num_gpus=1)` and `@ray.remote(num_gpus=0.25)`.

---

## Tasks — `@ray.remote` & `ray.get`

```python
import ray
import time

ray.init()

@ray.remote
def compute(x: int) -> int:
    time.sleep(1)
    return x * x

# Fire many tasks; returns ObjectRefs (futures)
refs = [compute.remote(i) for i in range(10)]

# Block until all complete
results = ray.get(refs)  # [0, 1, 4, 9, ...]

# Single ref
first = ray.get(compute.remote(5))  # 25
```

Partial results: `ready, _ = ray.wait(refs, num_returns=3)` then `ray.get(ready)`. Object refs use the shared store (zero-copy on-node for numpy).

---

## Actors (Stateful Workers)

```python
@ray.remote
class Counter:
    def __init__(self):
        self.value = 0

    def inc(self) -> int:
        self.value += 1
        return self.value

    def get(self) -> int:
        return self.value

counter = Counter.remote()
ray.get(counter.inc.remote())   # 1
ray.get(counter.inc.remote())   # 2
ray.get(counter.get.remote())   # 2
```

Use actors for parameter servers, connection pools, or a GPU model loaded once with many `predict.remote()` calls.

---

## Ray Data (Overview)

**Ray Data** scales tabular/file pipelines: read → transform → write, backed by the object store.

```python
import ray

ds = ray.data.read_parquet("s3://bucket/events/")

# Transform (runs distributed)
ds = ds.filter(lambda row: row["country"] == "US")
ds = ds.map_batches(
    lambda batch: {"score": batch["value"] * 2},
    batch_format="pandas",
)

# Materialize or stream
ds.write_parquet("/tmp/out")
# df = ds.take(100)  # sample rows
```

Readers: `read_parquet`, `read_csv`, `read_json`, `from_pandas`. Use `map_batches` for [[ML — PyTorch]] / sklearn batch inference.

---

## Ray Tune (Brief)

Distributed hyperparameter search; integrates with [[ML — Optuna]] search spaces or built-in samplers.

```python
from ray import tune
from ray.tune.schedulers import ASHAScheduler

def trainable(config):
    # train model with config["lr"], config["layers"]
    loss = ...
    tune.report({"loss": loss})

scheduler = ASHAScheduler(metric="loss", mode="min")

analysis = tune.run(
    trainable,
    config={
        "lr": tune.loguniform(1e-4, 1e-1),
        "layers": tune.choice([2, 3, 4]),
    },
    num_samples=50,
    scheduler=scheduler,
    resources_per_trial={"cpu": 2},
)

best = analysis.get_best_config(metric="loss", mode="min")
```

Log trials to [[ML — MLflow]] inside `trainable` for a single experiment UI. For small single-node studies, [[ML — Optuna]] may be simpler; use Tune when trials need a **cluster**.

---

## Ray Serve (Brief)

**Ray Serve** deploys Python callables / models as HTTP (or gRPC) **deployments**, with autoscaling and multi-model composition.

```python
from ray import serve

serve.start(detached=True)

@serve.deployment(num_replicas=2, ray_actor_options={"num_cpus": 1})
class IrisModel:
    def __init__(self):
        import joblib
        self.model = joblib.load("iris.pkl")

    async def __call__(self, request):
        body = await request.json()
        import numpy as np
        x = np.array([body["features"]])
        return {"prediction": int(self.model.predict(x)[0])}

IrisModel.deploy()
# curl http://127.0.0.1:8000/IrisModel -d '{"features": [5.1,3.5,1.4,0.2]}'
```

`serve.run(IrisModel.options(route_prefix="/v1/predict").bind())`. Expose via [[API - FastAPI]] gateway or private Serve URL.

---

## PyTorch + MLflow

Inside `@ray.remote(num_gpus=1)` workers: train with [[ML — PyTorch]], `mlflow.log_params` / `log_metric` / `log_artifact` ([[ML — MLflow]]), then deploy via Ray Serve or [[ML — BentoML]].

---

## FastAPI Integration

Init Ray once at startup ([[API - FastAPI — Lifespan]]), not per request. **Inference** → Ray Serve; **ad-hoc jobs** → `remote` + poll `ray.get(ref, timeout=0)`.

```python
import httpx
from fastapi import FastAPI
import ray

ray.init(address="auto", ignore_reinit_error=True)
app = FastAPI()
SERVE_URL = "http://127.0.0.1:8000/IrisModel"

@ray.remote
def heavy_report(user_id: str) -> dict:
    return {"user_id": user_id, "rows": 1_000_000}

@app.post("/reports/{user_id}", status_code=202)
async def start_report(user_id: str):
    ref = heavy_report.remote(user_id)
    return {"ref_id": ref.hex()}

@app.get("/reports/result/{ref_hex}")
async def report_result(ref_hex: str):
    ref = ray.ObjectRef(bytes.fromhex(ref_hex))
    try:
        return ray.get(ref, timeout=0)
    except ray.exceptions.GetTimeoutError:
        return {"status": "running"}

@app.post("/v1/iris")
async def proxy(features: list[float]):
    async with httpx.AsyncClient() as client:
        r = await client.post(SERVE_URL, json={"features": features}, timeout=10.0)
        r.raise_for_status()
        return r.json()
```

---

## Backend Patterns

### When Ray vs Celery

```text
Celery:  API → broker → worker (discrete jobs, cron, retries)
Ray:     driver → scheduler → workers (parallel refs, actors, ML libs)
```

Use **Celery** for operational job queues; **Ray** for parallel compute and ML stacks. They can coexist (Celery task kicks off a Ray job script on a GPU node).

**Production:** KubeRay or managed runtime; object spilling on disk; `serve.start(detached=True)`; pin Ray version across nodes; Dashboard at `:8265`.

---

## Pitfalls

- **Calling `ray.init()` in every request** — init once at app startup ([[API - FastAPI — Lifespan]]).
- **Huge objects in arguments** — put data in Ray Data or object store once, pass refs.
- **Blocking `ray.get` in async route** — offload to thread pool or use async Serve handlers.
- **Tune without early stopping** — wasted GPU; use ASHA / HyperBand schedulers.
- **Mixing Ray and fork** — avoid forking after `ray.init()` in the same process.

---

## Quick Reference

| Task | Code |
| --- | --- |
| Start local | `ray.init()` |
| Remote fn | `@ray.remote` + `f.remote(args)` |
| Collect | `ray.get(ref_or_list)` |
| Actor | `@ray.remote class` + `.remote()` |
| Partial wait | `ray.wait(refs, num_returns=n)` |
| Data read | `ray.data.read_parquet(path)` |
| Tune | `tune.run(trainable, config={...})` |
| Serve deploy | `@serve.deployment` + `serve.run` |
| Shutdown | `ray.shutdown()` |

---

## Related Notes

- [[Processing]]
- [[Processing — Celery]]
- [[API - FastAPI]]
- [[API - FastAPI — Lifespan]]
- [[ML — Optuna]]
- [[ML — MLflow]]
- [[ML — PyTorch]]
- [[Python — multiprocessing]]
- [[Python Development]]

---

## Tags

#python #ray #distributed #parallel #machine-learning #ray-tune #ray-serve #ray-data #fastapi #mlflow #pytorch #processing #hpo
