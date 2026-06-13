## What & When

**dbt (data build tool)** transforms data **inside your warehouse** with version-controlled SQL — the **T** in **ELT** (extract → load raw data, then transform). In ML pipelines it builds **clean feature tables** and training datasets in **BigQuery** (or Snowflake, Postgres) before [[ML — pandas]], [[ML — scikit-learn]], or [[ML — Feast]] consume them.

Use dbt when:

- Analytics or ML features live in a **cloud warehouse**, not app OLTP ([[ORM - SQLAlchemy]])
- You want **modular SQL**, dependency graphs, tests, and docs like application code
- **BigQuery** on [[GCP]] is your data platform
- [[ORCHESTRATION — Airflow]] or dbt Cloud schedules daily feature refreshes

```bash
pip install dbt-core dbt-bigquery
# Local learning: pip install dbt-core dbt-duckdb
```

Overview: [[Machine Learning]]. Orchestration: [[ORCHESTRATION — Airflow]], [[GCP]] (BigQuery, Composer).

---

## dbt vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Warehouse SQL transforms | **dbt** | Models as `.sql` files |
| App CRUD / API DB | [[ORM - SQLAlchemy]] | Not dbt |
| Batch ETL DAG | [[ORCHESTRATION — Airflow]] | Can trigger `dbt run` |
| Dataset versioning | [[ML — DVC]] | Git for files; dbt for warehouse tables |
| Feature store | [[ML — Feast]] | Offline store often BigQuery tables from dbt |
| In-notebook prep only | [[ML — pandas]] | Skip dbt for tiny one-offs |

---

## Core Concepts

| Concept | Meaning |
| --- | --- |
| **Model** | One `.sql` file → table or view in warehouse |
| **`ref('model')`** | Depends on another dbt model (builds DAG) |
| **`source('raw', 'table')`** | Declares raw loaded data |
| **`dbt run`** | Execute models in dependency order |
| **`dbt test`** | Assert unique, not_null, relationships |
| **dbt Core** | Open-source CLI |
| **dbt Cloud** | Hosted IDE, jobs, scheduling |

---

## Project Layout (Minimal)

```text
my_project/
  dbt_project.yml
  models/
    staging/
      stg_orders.sql
    marts/
      fct_customer_orders.sql
  models/schema.yml      # tests & docs
  profiles.yml           # connection (usually ~/.dbt/)
```

---

## Basics — Hello World

```bash
dbt init my_project
cd my_project
```

`models/hello_world.sql`:

```sql
SELECT 'Hello, World!' AS message
```

```bash
dbt run    # creates view/table hello_world in warehouse
dbt test
dbt docs generate && dbt docs serve
```

---

## Models & Dependencies

`models/stg_orders.sql`:

```sql
SELECT
    id          AS order_id,
    customer_id,
    amount,
    created_at
FROM {{ source('raw', 'orders') }}
```

`models/total_by_customer.sql`:

```sql
SELECT
    customer_id,
    COUNT(*)    AS num_orders,
    SUM(amount) AS total_spent
FROM {{ ref('stg_orders') }}
GROUP BY customer_id
```

dbt runs `stg_orders` first, then `total_by_customer` — **DAG from SQL alone**.

`models/schema.yml`:

```yaml
models:
  - name: total_by_customer
    columns:
      - name: customer_id
        tests: [unique, not_null]
```

---

## BigQuery Setup

**Install:**

```bash
pip install dbt-core dbt-bigquery
```

**`~/.dbt/profiles.yml`:**

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: bigquery
      method: oauth              # or service-account
      project: my-gcp-project
      dataset: analytics_dev
      threads: 4
      location: US
      # method: service-account
      # keyfile: /path/to/service-account.json
```

Set project via `gcloud auth application-default login` for OAuth, or use a service account JSON for CI.

**Run against BigQuery:**

```bash
dbt debug    # test connection
dbt run
dbt test
```

See [[GCP]] for BigQuery, Composer, and IAM.

---

## BigQuery-Specific Patterns

### Partitioned table (cost & latency)

```sql
{{ config(
    materialized='table',
    partition_by={
      "field": "created_at",
      "data_type": "timestamp"
    }
) }}

SELECT
    order_id,
    customer_id,
    amount,
    created_at
FROM {{ ref('stg_orders') }}
```

Filter on partition column with **literal** dates in downstream queries for partition pruning.

### Incremental models

```sql
{{ config(
    materialized='incremental',
    unique_key='order_id',
    partition_by={"field": "created_at", "data_type": "timestamp"},
) }}

SELECT * FROM {{ ref('stg_orders') }}
{% if is_incremental() %}
WHERE created_at > (SELECT MAX(created_at) FROM {{ this }})
{% endif %}
```

On BigQuery, dbt uses **`MERGE`** for incremental refreshes.

### Python models (optional)

dbt-bigquery can run **Python models** via BigQuery DataFrames / Colab Enterprise — use when transforms are easier in Python than SQL; most ML feature prep stays SQL + export to [[ML — pandas]].

---

## ML Pipeline Fit

```text
Raw ingest (Airflow / Fivetran) → BigQuery raw dataset
    → dbt staging + mart models (feature tables)
    → export / query → pandas / sklearn training
    → [[ML — MLflow]] log → [[ML — BentoML]] serve
```

[[ML — Feast]] can treat dbt-built BigQuery tables as **offline store** sources.

---

## Common Commands

| Command | Purpose |
| --- | --- |
| `dbt run` | Build all models |
| `dbt run -s stg_orders+` | Model and downstream |
| `dbt test` | Run schema tests |
| `dbt build` | Run + test (dbt ≥1.0) |
| `dbt compile` | Render SQL without executing |
| `dbt debug` | Validate profile connection |

Schedule in CI or [[ORCHESTRATION — Airflow]]:

```bash
dbt run --target prod && dbt test --target prod
```

---

## Quick Reference

| Task | Approach |
| --- | --- |
| New model | Add `models/*.sql`, use `ref()` |
| Raw table | Declare in `sources.yml`, `source()` |
| BigQuery connect | `dbt-bigquery` in `profiles.yml` |
| Partition large facts | `partition_by` in `config()` |
| Nightly refresh | Airflow / Composer + `dbt run` |
| ML training input | Query mart table → pandas read_gbq |

---

## Related Notes

- [[Machine Learning]]
- [[GCP]]
- [[ORCHESTRATION — Airflow]]
- [[ML — Feast]]
- [[ML — MLflow]]
- [[ML — DVC]]
- [[ML — pandas]]
- [[ORM - SQLAlchemy]]

---

## Tags

#dbt #bigquery #sql #elt #analytics #mlops #feature-engineering #gcp #python #warehouse
