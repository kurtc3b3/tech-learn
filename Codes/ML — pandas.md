## What & When

**pandas** is the **tabular data workhorse** for ML — load CSV/Parquet/SQL, clean, join, aggregate, and engineer features before handing numeric matrices to [[ML — scikit-learn]]. It sits between raw data sources and [[ML — NumPy]] arrays in most classical pipelines.

Use pandas when:

- Exploring and cleaning training datasets (missing values, outliers, dtypes)
- Feature engineering with groupby, pivot, window, and merge operations
- Point-in-time joins for leakage-safe train sets
- Exporting processed splits to Parquet for reproducible training jobs
- Lightweight batch scoring prep in backend services (small tables)

```python
import pandas as pd
import numpy as np
```

Core [[Machine Learning]] foundation — see learning path in [[Machine Learning]].

---

## pip install

```bash
pip install pandas
```

With Parquet I/O (common in ML pipelines):

```bash
pip install pandas pyarrow
# or: pip install pandas fastparquet
```

For large out-of-core ETL, consider Polars or DuckDB — pandas remains the sklearn ecosystem default.

---

## pandas vs Alternatives

| Need | Prefer | Notes |
| --- | --- | --- |
| sklearn-ready tabular ML | **pandas** | Best docs + examples |
| 100M+ row ETL | Polars / DuckDB | pandas loads all into RAM |
| Pure numeric matrix | [[ML — NumPy]] | After feature work is done |
| SQL-heavy feature store | SQL + [[ML — Feast]] | pandas for ad hoc pulls |
| Time series forecasting | pandas + [[ML — Prophet]] | Datetime index essential |
| Quick plots | [[ML — seaborn]] | Accepts DataFrames natively |

---

## Load & Inspect — ML EDA Start

```python
import pandas as pd

df = pd.read_csv("churn.csv", parse_dates=["signup_date"])
# df = pd.read_parquet("s3://bucket/features/partition=2024-01/")

df.shape
df.dtypes
df.isna().mean().sort_values(ascending=False).head(10)
df["target"].value_counts(normalize=True)   # class balance
```

```python
# Separate features / target early
TARGET = "churn"
feature_cols = [c for c in df.columns if c not in (TARGET, "customer_id")]
X_df = df[feature_cols]
y = df[TARGET]
```

---

## dtypes & Memory

Wrong dtypes inflate RAM and break encoders.

```python
# Downcast numerics
for col in X_df.select_dtypes("float").columns:
    X_df[col] = pd.to_numeric(X_df[col], downcast="float")
for col in X_df.select_dtypes("integer").columns:
    X_df[col] = pd.to_numeric(X_df[col], downcast="integer")

# Low-cardinality strings → category
for col in X_df.select_dtypes("object").columns:
    if X_df[col].nunique() < 50:
        X_df[col] = X_df[col].astype("category")
```

---

## Missing Values & Outliers

```python
# Drop rows missing target only
df = df.dropna(subset=[TARGET])

# Flag missingness as features (often helps tree models)
for col in ["income", "tenure"]:
    df[f"{col}_was_missing"] = df[col].isna().astype(int)
    df[col] = df[col].fillna(df[col].median())
```

```python
# Winsorize extreme values (manual cap)
cap = df["monthly_charges"].quantile(0.99)
df["monthly_charges"] = df["monthly_charges"].clip(upper=cap)
```

In production pipelines, wrap imputers in [[ML — scikit-learn]] `Pipeline` — fit on train only.

---

## Feature Engineering

```python
# Datetime features for tree/linear models
df["signup_year"] = df["signup_date"].dt.year
df["signup_month"] = df["signup_date"].dt.month
df["days_since_signup"] = (pd.Timestamp("today") - df["signup_date"]).dt.days
```

```python
# Aggregated customer history (watch leakage — use cutoff date)
cutoff = pd.Timestamp("2024-01-01")
history = df[df["event_date"] < cutoff]
agg = history.groupby("customer_id").agg(
    n_events=("event_id", "count"),
    total_spend=("amount", "sum"),
    avg_spend=("amount", "mean"),
).reset_index()

features = df.merge(agg, on="customer_id", how="left")
features[["n_events", "total_spend"]] = features[["n_events", "total_spend"]].fillna(0)
```

## Train / Test Split with pandas

```python
from sklearn.model_selection import train_test_split

train_df, test_df = train_test_split(
    df,
    test_size=0.2,
    random_state=42,
    stratify=df[TARGET],
)

X_train = train_df[feature_cols]
y_train = train_df[TARGET]
FEATURE_ORDER = list(X_train.columns)   # serving contract
X_train_np = X_train.to_numpy(dtype=np.float32)
```

```python
# Time-based split — no shuffle for temporal data
df = df.sort_values("event_date")
split_at = int(0.8 * len(df))
train_df, test_df = df.iloc[:split_at], df.iloc[split_at:]
```

---

## ML / Backend Patterns

### Parquet train snapshots

```python
train_df.to_parquet("artifacts/train_2024-06-02.parquet", index=False)
train_df = pd.read_parquet("artifacts/train_2024-06-02.parquet")
```

### Batch scoring from API queue

```python
def score_batch(rows: list[dict], pipeline, feature_order: list[str]) -> pd.DataFrame:
    batch = pd.DataFrame(rows)[feature_order]
    batch["score"] = pipeline.predict_proba(batch)[:, 1]
    return batch
```

### Data validation gate

```python
REQUIRED = {"age", "tenure", "monthly_charges"}
assert REQUIRED <= set(df.columns), f"Missing: {REQUIRED - set(df.columns)}"
assert df["age"].between(0, 120).all(), "Invalid age range"
```

### Leakage checklist

| Risk | pandas fix |
| --- | --- |
| Future data in aggregates | Filter `event_date < cutoff` before groupby |
| Target in feature list | Drop `TARGET` from `feature_cols` |
| Duplicate customer rows | `drop_duplicates(subset=["customer_id"])` |
| Test stats in imputation | Fit imputer on `train_df` only |

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| `SettingWithCopyWarning` | Use `.loc` or `.copy()` |
| Shuffled time-series split | Sort by date; split chronologically |
| `merge` row explosion | Validate row counts; use `validate="m:1"` |
| Categorical `NaN` as string | Explicit `fillna("missing")` before encoding |
| Saving CSV for large data | Prefer Parquet + pyarrow |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Read CSV | `pd.read_csv(path, parse_dates=[...])` |
| Missing rate | `df.isna().mean()` |
| Class balance | `df[target].value_counts(normalize=True)` |
| Select dtypes | `df.select_dtypes("number")` |
| Group aggregate | `df.groupby("id").agg(...)` |
| Merge | `left.merge(right, on="key", how="left")` |
| Sort + split time | `df.sort_values("date"); df.iloc[:n]` |
| To NumPy | `df[cols].to_numpy(dtype=np.float32)` |
| Parquet I/O | `to_parquet` / `read_parquet` |
| Duplicate check | `df.duplicated(subset=["id"]).sum()` |

---

## Tags

#machine-learning #pandas #feature-engineering #tabular #etl #sklearn #python
