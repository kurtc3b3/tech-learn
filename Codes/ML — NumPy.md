## What & When

**NumPy** is the **numerical foundation** of the Python ML stack — N-dimensional arrays (`ndarray`), vectorized math, linear algebra, and random sampling. Every tabular ML library ([[ML — pandas]], [[ML — scikit-learn]], XGBoost) ultimately operates on NumPy arrays or array-compatible buffers.

Use NumPy when:

- Converting model inputs/outputs to/from sklearn, PyTorch, or API payloads
- Vectorized feature math (scaling, log transforms, distance matrices)
- Building train/validation splits, batching, and shuffling indices
- Linear algebra for custom metrics, PCA-style ops, or embedding math
- Memory-efficient numeric storage before pandas DataFrames

```python
import numpy as np
```

Part of the [[Machine Learning]] foundation stack. Pair with [[ML — pandas]] for tables and [[ML — scikit-learn]] for modeling.

---

## pip install

```bash
pip install numpy
```

Most ML stacks pin NumPy explicitly — check compatibility with [[ML — scikit-learn]] and PyTorch wheels:

```bash
pip install "numpy>=1.26,<3"
```

No separate runtime — import-only. Prefer **float64** for analysis, **float32** for large matrices and GPU handoff.

---

## NumPy vs Alternatives

| Need | Prefer | Notes |
| --- | --- | --- |
| Tabular columns + joins | [[ML — pandas]] | NumPy for raw matrices underneath |
| GPU tensors | PyTorch / CuPy | NumPy stays CPU |
| Sparse high-dim features | [[ML — scipy]] `sparse` | Dense NumPy wastes RAM |
| Python lists of numbers | NumPy | 10–100× faster, fixed dtype |
| sklearn `fit` / `predict` | NumPy 2D `float` arrays | `(n_samples, n_features)` contract |
| Symbolic math | SymPy | NumPy is numeric only |

---

## Array Basics — Shape & dtype

sklearn and most estimators expect **2D float arrays**: rows = samples, columns = features.

```python
import numpy as np

X = np.array([[1.0, 2.0], [3.0, 4.0], [5.0, 6.0]])   # (3, 2)
y = np.array([0, 1, 0])                                 # (3,)

X.shape          # (3, 2)
X.dtype          # float64
X.nbytes         # bytes in memory — watch for large feature matrices
```

```python
# From pandas without copy when possible
import pandas as pd

df = pd.read_csv("features.csv")
X = df[["age", "income"]].to_numpy(dtype=np.float32)
y = df["churn"].to_numpy()
```

---

## Vectorized Feature Transforms

Avoid Python loops over rows — broadcast operations across columns.

```python
# Standardize (manual — production uses sklearn StandardScaler)
mean = X.mean(axis=0)
std = X.std(axis=0, ddof=0)
X_scaled = (X - mean) / std

# Log1p for skewed counts
counts = np.array([0, 1, 10, 1000], dtype=float)
np.log1p(counts)

# Clip outliers before training
np.clip(X, a_min=-3, a_max=3)   # often after scaling
```

```python
# Boolean masks for filtering
mask = (y == 1) & (X[:, 0] > 30)
X_pos = X[mask]
```

---

## Random Splits & Reproducibility

```python
rng = np.random.default_rng(seed=42)

n = X.shape[0]
indices = rng.permutation(n)
test_size = int(0.2 * n)

test_idx = indices[:test_size]
train_idx = indices[test_size:]

X_train, X_test = X[train_idx], X[test_idx]
y_train, y_test = y[train_idx], y[test_idx]
```

Prefer `sklearn.model_selection.train_test_split` in pipelines — same idea, stratification support. See [[ML — scikit-learn]].

---

## Linear Algebra for ML

```python
# Pairwise squared Euclidean distances (n × m)
# ||a - b||² = ||a||² + ||b||² - 2 a·b
A = X_train
B = X_test
dists = (
    (A ** 2).sum(axis=1, keepdims=True)
    + (B ** 2).sum(axis=1)
    - 2 * A @ B.T
)
```

```python
# Weighted average prediction (simple ensemble)
probs = np.stack([model_a.predict_proba(X_test)[:, 1],
                  model_b.predict_proba(X_test)[:, 1]])
weights = np.array([0.6, 0.4])
ensemble = (probs.T @ weights).T
```

---

## ML / Backend Patterns

### API response → model input

```python
def features_from_request(payload: dict) -> np.ndarray:
    order = ["age", "tenure", "monthly_charges"]
    row = np.array([[payload[k] for k in order]], dtype=np.float32)
    return row   # shape (1, n_features) for sklearn predict
```

### NaN handling before sklearn

sklearn estimators reject NaNs unless explicitly supported.

```python
# Impute with column medians (prefer sklearn SimpleImputer in pipelines)
col_medians = np.nanmedian(X, axis=0)
inds = np.where(np.isnan(X))
X[inds] = np.take(col_medians, inds[1])
```

### Memory-conscious dtypes

```python
# Downcast float64 → float32 when precision allows
X = X.astype(np.float32)   # halves memory for large training sets
```

### Save/load numeric artifacts

```python
np.save("train_indices.npy", train_idx)
train_idx = np.load("train_indices.npy")
```

Pair `.npy` with MLflow or joblib for full model bundles — see [[Machine Learning]] MLOps section.

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| 1D `y` passed as `(n, 1)` | Use `y.ravel()` or `(n,)` shape |
| Integer labels as floats | Cast classification targets to int |
| `X[i, j] = val` on a view | Assign to copy or use `.copy()` |
| Mixing row/column order | Document feature column order in serving |
| Huge `float64` matrices | Use `float32` or sparse when possible |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Shape / dtype | `X.shape`, `X.dtype` |
| Column mean | `X.mean(axis=0)` |
| Row-wise sum | `X.sum(axis=1)` |
| Reshape | `X.reshape(n, -1)` |
| Concatenate | `np.concatenate([a, b], axis=0)` |
| Stack columns | `np.column_stack([a, b])` |
| Unique classes | `np.unique(y)` |
| Argmax predictions | `probs.argmax(axis=1)` |
| Set seed | `rng = np.random.default_rng(42)` |
| Shuffle indices | `rng.permutation(n)` |
| Save array | `np.save(path, arr)` |
| Matrix multiply | `A @ B` |

---

## Tags

#machine-learning #numpy #arrays #linear-algebra #sklearn #python #foundation
