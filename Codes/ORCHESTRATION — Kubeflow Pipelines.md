## What & When

**Kubeflow Pipelines (KFP)** defines **ML workflows** as Python DAGs that run as **containers on [[K8S]]** — each step is a component with inputs/outputs, enabling reproducible train/eval/deploy pipelines. Use when ML stages need **isolated environments**, **artifact lineage**, and **cluster execution** (not lightweight cron).

Use KFP when:

- **Train → evaluate → register** models on Kubernetes
- Steps need **different Docker images** (sklearn vs PyTorch)
- Integrating with **Vertex AI Pipelines** or self-hosted Kubeflow
- Pairing with [[ML — MLflow]] tracking and [[ML — Seldon]] deployment

```bash
pip install kfp
# Cluster: Kubeflow on GKE or Vertex AI Pipelines endpoint
```

Overview: [[ORCHESTRATION]]. General ETL DAGs → [[ORCHESTRATION — Airflow]].

---

## KFP vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| ML container pipeline on K8s | **KFP** | This note |
| General data DAG | [[ORCHESTRATION — Airflow]] | Broader operators |
| Experiment tracking | [[ML — MLflow]] | Log inside components |
| Hyperparameter search | [[ML — Optuna]], [[Processing — Ray]] | Can be a pipeline step |
| Model deploy | [[ML — Seldon]], [[ML — BentoML]] | Final component |
| Task queue | [[Processing — Celery]] | Not pipeline DAG |

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **Component** | Reusable containerized function |
| **Pipeline** | DAG of component calls |
| **Run** | One execution instance |
| **Artifact** | Datasets, models passed between steps |
| **Experiment** | Groups runs for comparison |

---

## Python Component (KFP SDK v2)

```python
from kfp import dsl

@dsl.component(base_image="python:3.12-slim", packages_to_install=["pandas", "scikit-learn"])
def preprocess(raw_path: str, out_path: dsl.OutputPath("Dataset")):
    import pandas as pd
    df = pd.read_csv(raw_path)
    df = df.dropna()
    df.to_csv(out_path, index=False)

@dsl.component(base_image="python:3.12-slim", packages_to_install=["pandas", "scikit-learn", "joblib"])
def train(data_path: dsl.InputPath("Dataset"), model_path: dsl.OutputPath("Model")):
    import pandas as pd
    from sklearn.ensemble import RandomForestClassifier
    import joblib

    df = pd.read_csv(data_path)
    X, y = df.drop("target", axis=1), df["target"]
    clf = RandomForestClassifier(n_estimators=100)
    clf.fit(X, y)
    joblib.dump(clf, model_path)

@dsl.pipeline(name="iris-training", description="Train sklearn model")
def iris_pipeline(raw_data: str = "gs://my-bucket/iris.csv"):
    prep = preprocess(raw_path=raw_data)
    train(data_path=prep.outputs["out_path"])
```

Compile and submit:

```python
from kfp import compiler

compiler.Compiler().compile(iris_pipeline, "iris_pipeline.yaml")
# Upload YAML to Kubeflow UI or:
# client = kfp.Client(host="https://kubeflow.example.com/pipeline")
# client.create_run_from_pipeline_package("iris_pipeline.yaml", arguments={...})
```

---

## MLflow Inside a Component

```python
@dsl.component(packages_to_install=["mlflow", "scikit-learn"])
def train_with_tracking(data_path: dsl.InputPath("Dataset"), run_name: str):
    import mlflow
    import pandas as pd
    from sklearn.model_selection import train_test_split
    from sklearn.metrics import accuracy_score
    from sklearn.ensemble import RandomForestClassifier

    df = pd.read_csv(data_path)
    X, y = df.drop("target", axis=1), df["target"]
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)

    with mlflow.start_run(run_name=run_name):
        clf = RandomForestClassifier()
        clf.fit(X_train, y_train)
        acc = accuracy_score(y_test, clf.predict(X_test))
        mlflow.log_metric("accuracy", acc)
        mlflow.sklearn.log_model(clf, "model")
```

See [[ML — MLflow]].

---

## Artifacts & GCS

KFP v2 uses **artifact URIs** — often **GCS** ([[GCP]]) or MinIO on cluster:

```python
raw_data: str = "gs://my-ml-bucket/datasets/churn.csv"
```

Mount secrets for cloud access via K8s service accounts — [[Codes/K8S — Configuration & Security]].

---

## Pipeline Parameters

```python
@dsl.pipeline
def training_pipeline(
    learning_rate: float = 0.01,
    epochs: int = 10,
    dataset_uri: str = "gs://bucket/data",
):
    ...
```

Trigger runs with different parameters from UI or SDK — supports A/B training configs.

---

## Deploy After Train

Final component pattern:

```text
train → evaluate (gate on metric) → build Bento image → apply SeldonDeployment
```

Links: [[ML — BentoML]], [[ML — Seldon]], [[Codes/K8S — Workloads]].

---

## Vertex AI Pipelines

Same KFP SDK — compile pipeline, submit to **Vertex** managed backend instead of self-hosted Kubeflow. See [[GCP]] (Vertex AI).

---

## Airflow Triggers KFP

```python
# Airflow @task pseudocode
@task
def launch_kfp_run():
    import kfp
    client = kfp.Client(host=os.environ["KFP_HOST"])
    client.create_run_from_pipeline_package(
        "iris_pipeline.yaml",
        arguments={"raw_data": "gs://..."},
    )
```

Orchestration layers: Airflow for calendar; KFP for ML execution — [[ORCHESTRATION — Airflow]].

---

## Local / Dev Tips

| Approach | Use |
| --- | --- |
| Compile only | Validate YAML without cluster |
| Kind / Minikube + Kubeflow | Full local K8s — [[Commands/K8S — kubectl & Minikube]] |
| Vertex AI | Managed, less cluster ops |
| Heavy iteration | Run train step alone in notebook first — [[ML — scikit-learn]] |

---

## Quick Reference

| Task | API |
| --- | --- |
| Component | `@dsl.component` |
| Pipeline | `@dsl.pipeline` |
| Compile | `Compiler().compile(...)` |
| Input/output paths | `dsl.InputPath`, `dsl.OutputPath` |
| Submit run | `kfp.Client().create_run_from_pipeline_package` |
| View runs | Kubeflow / Vertex Pipelines UI |

---

## Related Notes

- [[ORCHESTRATION]]
- [[ORCHESTRATION — Airflow]]
- [[K8S]]
- [[GCP]]
- [[Machine Learning]]
- [[ML — MLflow]]
- [[ML — BentoML]]
- [[ML — Seldon]]
- [[Processing — Ray]]

---

## Tags

#orchestration #kubeflow #kfp #mlops #kubernetes #python #pipeline #ml #training
