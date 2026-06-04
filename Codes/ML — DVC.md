## What & When

**DVC** versions **datasets**, **models**, and **ML pipelines** next to Git. Use it when data is too large for Git, you need **`dvc repro`** to rebuild only what changed, and teams share artifacts via **S3/GCS/SSH remotes**.

```bash
pip install dvc
# Cloud remotes (pick one or more):
pip install "dvc[s3]"
pip install "dvc[gs]"
pip install "dvc[ssh]"
```

Overview: [[DVC]]. Compare [[ML — MLflow]] for experiment tracking.

---

## DVC vs MLflow

| | DVC | MLflow |
| --- | --- | --- |
| Primary focus | Data + pipeline reproducibility | Experiment metrics + model registry |
| Version control | Git + `.dvc` + `dvc.lock` | Tracking server / DB |
| Large artifacts | Cache + remote | Artifact store |
| Pipeline DAG | `dvc.yaml` | Optional MLflow projects |

Use **both**: DVC pins `train.csv` and `model.pkl`; MLflow logs `accuracy` and registers models.

---

## Prerequisites

```bash
git init my-ml-project && cd my-ml-project
dvc init
```

Commit `.dvc/` and `.dvcignore` after init.

---

## Track a Dataset (`dvc add`)

```bash
mkdir -p data
# copy or download raw.csv into data/

dvc add data/raw.csv
git add data/raw.csv.dvc data/.gitignore .gitignore
git commit -m "Track raw data with DVC"
```

`data/raw.csv` moves to cache; Git only tracks `data/raw.csv.dvc`.

---

## Remote Storage

### Amazon S3

```bash
dvc remote add -d storage s3://my-bucket/dvc-store
# credentials: AWS CLI env / ~/.aws/credentials

dvc push
dvc pull   # on another machine after git clone
```

### Google Cloud Storage

```bash
dvc remote add -d storage gs://my-bucket/dvc-store
# auth: gcloud auth application-default login

dvc push
dvc pull
```

See [[GCP]] for bucket IAM and service accounts in CI.

---

## Minimal Pipeline Project

### `params.yaml`

```yaml
prepare:
  seed: 42

train:
  n_estimators: 200
  max_depth: 6
  random_state: 42
```

### `src/prepare.py`

```python
import pandas as pd
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
import yaml
from pathlib import Path

params = yaml.safe_load(Path("params.yaml").read_text())["prepare"]
X, y = load_iris(return_X_y=True)
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=params["seed"]
)
out = Path("data/prepared")
out.mkdir(parents=True, exist_ok=True)
pd.DataFrame(X_train).assign(target=y_train).to_csv(out / "train.csv", index=False)
pd.DataFrame(X_test).assign(target=y_test).to_csv(out / "test.csv", index=False)
```

### `src/train.py`

```python
import json
import pickle
from pathlib import Path

import pandas as pd
import yaml
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

params = yaml.safe_load(Path("params.yaml").read_text())["train"]
train = pd.read_csv("data/prepared/train.csv")
test = pd.read_csv("data/prepared/test.csv")

X_train, y_train = train.drop(columns=["target"]), train["target"]
X_test, y_test = test.drop(columns=["target"]), test["target"]

clf = RandomForestClassifier(
    n_estimators=params["n_estimators"],
    max_depth=params["max_depth"],
    random_state=params["random_state"],
)
clf.fit(X_train, y_train)
acc = accuracy_score(y_test, clf.predict(X_test))

Path("models").mkdir(exist_ok=True)
with open("models/model.pkl", "wb") as f:
    pickle.dump(clf, f)

Path("metrics").mkdir(exist_ok=True)
with open("metrics/accuracy.json", "w") as f:
    json.dump({"accuracy": acc}, f)
print(f"accuracy={acc:.4f}")
```

### Create stages with CLI

```bash
dvc stage add -n prepare \
  -d src/prepare.py -d params.yaml \
  -o data/prepared/train.csv -o data/prepared/test.csv \
  python src/prepare.py

dvc stage add -n train \
  -d src/train.py -d data/prepared/train.csv -d data/prepared/test.csv \
  -p train.n_estimators -p train.max_depth -p train.random_state \
  -o models/model.pkl \
  -M metrics/accuracy.json \
  python src/train.py
```

### Equivalent `dvc.yaml` (for reference)

```yaml
stages:
  prepare:
    cmd: python src/prepare.py
    deps:
      - src/prepare.py
      - params.yaml
    params:
      - prepare.seed
    outs:
      - data/prepared/train.csv
      - data/prepared/test.csv

  train:
    cmd: python src/train.py
    deps:
      - src/train.py
      - data/prepared/train.csv
      - data/prepared/test.csv
      - params.yaml
    params:
      - train.n_estimators
      - train.max_depth
      - train.random_state
    outs:
      - models/model.pkl
    metrics:
      - metrics/accuracy.json:
          cache: false
```

---

## Run and Reproduce

```bash
dvc repro
```

- First run: executes all stages in order.
- Change `params.yaml` → only affected stages rerun.
- Outputs recorded in `dvc.lock` — **commit `dvc.yaml` + `dvc.lock`**.

```bash
git add dvc.yaml dvc.lock params.yaml src/ metrics/.gitkeep
git commit -m "Add DVC pipeline"
dvc push
```

---

## Metrics

```bash
dvc metrics show
dvc metrics diff   # compare to previous Git rev
```

`metrics/accuracy.json` stays small in Git when `cache: false` under `metrics:` in `dvc.yaml`.

---

## Checkout Data for an Old Git Commit

```bash
git checkout v1.0
dvc checkout
```

Restores workspace data to match that revision’s `dvc.lock`.

---

## Optional: MLflow Inside `train.py`

```python
import mlflow

with mlflow.start_run():
    mlflow.log_params(params)
    # ... train ...
    mlflow.log_metric("accuracy", acc)
    mlflow.sklearn.log_model(clf, "model")
```

Set `MLFLOW_TRACKING_URI` — see [[ML — MLflow]]. DVC still owns `models/model.pkl` on disk.

---

## `dvc exp` (Experiments — Brief)

Queue parameter sweeps without manual copies:

```bash
dvc exp run -S train.n_estimators=100,300,500
dvc exp show
```

Fits between ad-hoc runs and full [[ML — Optuna]] studies.

---

## CI Sketch (GitHub Actions)

```yaml
- uses: actions/checkout@v4
- run: pip install dvc "dvc[gs]"
- run: dvc pull
- run: dvc repro
- run: dvc metrics show
```

Provide cloud credentials via secrets; cache remote for speed.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Forgot `dvc.lock` in Git | Always commit after `dvc repro` |
| `dvc add` on pipeline `outs` | Let stages declare `outs` — avoid double-tracking |
| No remote — teammates missing data | `dvc push` after commit |
| Not in Git repo | `dvc init` requires Git |
| Huge metrics in cache | `cache: false` for metric files |

---

## Quick Reference

```bash
pip install "dvc[gs]"
dvc init
dvc add path/to/data
dvc remote add -d storage gs://bucket/path
dvc stage add -n train -d train.py -o model.pkl python train.py
dvc repro
dvc push && dvc pull
dvc metrics show
```

---

## Related Notes

- [[DVC]]
- [[Machine Learning]]
- [[ML — MLflow]]
- [[ML — scikit-learn]]
- [[Commands/CLI — Git & GitHub]]
- [[Commands/CLI — Docker & Compose]]
- [[GCP]]

---

## Tags

#dvc #mlops #pipeline #dvc-yaml #s3 #gcs #reproducibility #python #sklearn
