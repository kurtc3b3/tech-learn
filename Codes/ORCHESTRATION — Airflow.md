## What & When

**Apache Airflow** orchestrates workflows as **DAGs** (directed acyclic graphs) — Python-defined tasks with dependencies, schedules, retries, and a web UI. Use for **ETL**, **batch ops**, **backfills**, and coordinating services (BigQuery loads on [[GCP]], [[Processing — Celery]] triggers, [[ORM - SQLAlchemy]] exports).

Use Airflow when:

- Steps have **explicit dependencies** (not just a task queue)
- You need **scheduling**, **backfill**, and **run history**
- Ops/data teams share **DAG visibility** in the UI
- Running on **Cloud Composer** ([[GCP]]) or self-hosted workers

```bash
pip install apache-airflow
# Constraint file recommended — see airflow.apache.org install docs
# Local: airflow standalone  (dev all-in-one)
```

Overview: [[ORCHESTRATION]]. For single jobs use [[Processing — Celery]].

---

## Airflow vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| DAG + schedule + UI | **Airflow** | This note |
| ML pipeline on K8s | [[ORCHESTRATION — Kubeflow Pipelines]] | Container components |
| Task queue from API | [[Processing — Celery]] | `.delay()` |
| Parallel Python cluster | [[Processing — Ray]] | Not DAG-centric |
| LLM chains | [[AI — LangGraph]] | Different domain |
| GCP managed Airflow | Composer — [[GCP]] | Same DAG model |

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **DAG** | Workflow definition (no cycles) |
| **Task** | Single unit of work |
| **Operator** | Task template (Python, Bash, SQL, …) |
| **Scheduler** | Enqueues DAG runs by timetable |
| **Executor** | Runs tasks (Local, Celery, Kubernetes, …) |
| **XCom** | Small cross-task data exchange |
| **Connection / Variable** | Secrets and config in metadata DB |

---

## TaskFlow API (Airflow 2.x — Preferred)

```python
from datetime import datetime, timedelta
from airflow.decorators import dag, task

@dag(
    start_date=datetime(2025, 1, 1),
    schedule="@daily",
    catchup=False,
    tags=["etl"],
)
def daily_sales_etl():
    @task
    def extract():
        return {"rows": [{"id": 1, "amount": 42.0}]}

    @task
    def transform(data: dict):
        data["rows"][0]["amount"] *= 1.1
        return data

    @task
    def load(data: dict):
        # write to warehouse / DB
        print(f"Loaded {len(data['rows'])} rows")

    raw = extract()
    cleaned = transform(raw)
    load(cleaned)

daily_sales_etl()
```

Place DAG files in `dags/` folder; scheduler parses on interval.

---

## Classic PythonOperator

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def extract(**context):
    context["ti"].xcom_push(key="count", value=100)

def load(**context):
    count = context["ti"].xcom_pull(task_ids="extract", key="count")
    print(count)

with DAG("classic_etl", start_date=datetime(2025, 1, 1), schedule="@daily", catchup=False) as dag:
    t1 = PythonOperator(task_id="extract", python_callable=extract)
    t2 = PythonOperator(task_id="load", python_callable=load)
    t1 >> t2
```

Prefer `@task` decorator for typed data flow between steps.

---

## Dependencies

```python
a >> b >> c          # linear
[a, b] >> c          # fan-in
a >> [b, c]          # fan-out
```

---

## Scheduling

| Expression | Meaning |
| --- | --- |
| `@daily` | Once per day |
| `@hourly` | Hourly |
| `0 2 * * *` | Cron: 02:00 daily |
| `None` | Manual trigger only |

`catchup=False` — skip historical runs when enabling a DAG (common in prod).

---

## Retries & Alerts

```python
default_args = {
    "retries": 2,
    "retry_delay": timedelta(minutes=5),
    "email_on_failure": False,
}

@dag(default_args=default_args, ...)
def my_dag(): ...
```

Integrate Slack/PagerDuty via callbacks or providers.

---

## Trigger Celery / External Systems

```python
from airflow.providers.celery.operators.celery import CeleryOperator

# Or @task calling your app:
@task
def enqueue_scoring():
    from myapp.tasks import score_batch
    score_batch.delay(batch_id="2025-01-15")
```

See [[Processing — Celery]], [[Browser Automation — Scrapy]] for work executed outside Airflow.

---

## Local Development

```bash
export AIRFLOW_HOME=~/airflow
airflow db migrate
airflow standalone          # UI http://localhost:8080
# Copy DAG to $AIRFLOW_HOME/dags/
```

Use **Docker Compose** official quick-start for team parity — [[Commands/CLI — Docker & Compose]].

---

## Production Notes

| Topic | Guidance |
| --- | --- |
| Metadata DB | PostgreSQL (not SQLite) |
| Executor | CeleryExecutor or KubernetesExecutor at scale |
| DAG idempotency | Tasks safe to re-run (backfill) |
| Secrets | Connections / Secret Manager — [[GCP]] |
| Don't put heavy ML | Trigger [[ORCHESTRATION — Kubeflow Pipelines]] or Ray instead |

---

## Composer (GCP)

Same DAG Python — Composer manages Airflow + GCS DAG bucket + IAM. See [[GCP]], [[ORCHESTRATION]].

---

## Quick Reference

| Task | Pattern |
| --- | --- |
| Define DAG | `@dag` + `@task` |
| Linear chain | `a >> b >> c` |
| Pass data | TaskFlow return values / XCom |
| Manual run | Airflow UI → Trigger DAG |
| Pause DAG | Toggle in UI |
| CLI test task | `airflow tasks test dag_id task_id 2025-01-01` |

---

## Related Notes

- [[ORCHESTRATION]]
- [[ORCHESTRATION — Kubeflow Pipelines]]
- [[Processing — Celery]]
- [[GCP]]
- [[ML — MLflow]]
- [[Python Development]]

---

## Tags

#orchestration #airflow #dag #etl #python #composer #workflow #data-engineering
