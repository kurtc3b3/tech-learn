## What & When

**SciPy** fills the gaps between [[ML — NumPy]] and domain ML libraries — **statistics**, **sparse linear algebra**, **optimization**, **distance metrics**, and **signal processing**. sklearn builds on SciPy internally; you reach for it directly when pipelines need statistical tests, custom distances, or sparse feature matrices.

Use SciPy when:

- Statistical hypothesis tests for feature screening or A/B analysis
- Sparse matrices for text/count features before [[ML — scikit-learn]]
- `scipy.optimize` for custom loss minimization or calibration
- Distance/similarity functions (`cdist`, `pdist`) for clustering or k-NN prototypes
- `scipy.stats` for distribution fitting, z-scores, and sampling

```python
import scipy
import numpy as np
from scipy import stats, sparse, optimize, spatial
```

Part of the [[Machine Learning]] foundation — especially prep and evaluation, not primary modeling.

---

## pip install

```bash
pip install scipy
```

Usually installed as a dependency of [[ML — scikit-learn]]. Pin alongside NumPy:

```bash
pip install "numpy>=1.26" "scipy>=1.11"
```

---

## SciPy vs Alternatives

| Need | Prefer | Notes |
| --- | --- | --- |
| sklearn built-in metrics/tests | [[ML — scikit-learn]] | Wraps SciPy in many places |
| Rich statistical modeling | statsmodels | SciPy for low-level distributions |
| Sparse + sklearn | **SciPy sparse** | CSR/CSC → `LinearSVC`, `LogisticRegression` |
| Heavy optimization | Optuna / PyTorch | SciPy for small custom problems |
| Fast array math | [[ML — NumPy]] | SciPy adds algorithms on top |
| EDA plots | [[ML — seaborn]] | Uses SciPy stats under the hood |

---

## scipy.stats — Distributions & Tests

### Feature screening with hypothesis tests

```python
import numpy as np
import pandas as pd
from scipy import stats

df = pd.read_csv("features.csv")
target = "churn"

numeric_cols = df.select_dtypes("number").columns.drop(target)
results = []

for col in numeric_cols:
    a = df.loc[df[target] == 0, col].dropna()
    b = df.loc[df[target] == 1, col].dropna()
    stat, p = stats.mannwhitneyu(a, b, alternative="two-sided")
    results.append({"feature": col, "p_value": p})

screen = pd.DataFrame(results).sort_values("p_value")
# Low p → likely informative; still validate with CV + [[ML — Boruta]]
```

### Z-scores and outlier flags

```python
from scipy.stats import zscore

z = zscore(df["monthly_charges"].dropna())
outlier_mask = np.abs(z) > 3
```

### Confidence intervals for model metric

```python
# Bootstrap AUC CI (illustrative)
from sklearn.metrics import roc_auc_score

rng = np.random.default_rng(42)
y_true = np.array([0, 1, 0, 1, 1, 0])
y_score = np.array([0.1, 0.9, 0.4, 0.8, 0.7, 0.2])

boots = []
for _ in range(1000):
    idx = rng.integers(0, len(y_true), len(y_true))
    if len(np.unique(y_true[idx])) < 2:
        continue
    boots.append(roc_auc_score(y_true[idx], y_score[idx]))

ci_low, ci_high = np.percentile(boots, [2.5, 97.5])
```

---

## scipy.sparse — Text & High-Dim Features

```python
from scipy.sparse import csr_matrix, hstack
from sklearn.feature_extraction.text import TfidfVectorizer

texts_train = ["fast ml pipeline", "slow batch job"]
texts_test = ["ml pipeline batch"]

vec = TfidfVectorizer(max_features=10_000)
X_train = vec.fit_transform(texts_train)    # scipy.sparse.csr_matrix
X_test = vec.transform(texts_test)

X_train.shape       # (2, n_features)
X_train.nnz         # non-zero entries — memory efficient
type(X_train)       # csr_matrix
```

```python
# Combine sparse text + dense numeric (common in hybrid models)
dense = csr_matrix([[1.0, 0.0], [0.0, 1.0]])
X_hybrid = hstack([X_train, dense], format="csr")
```

Pass sparse matrices directly to [[ML — scikit-learn]] linear models (`LinearSVC`, `LogisticRegression(solver="saga")`).

---

## scipy.spatial — Distances

```python
from scipy.spatial.distance import cdist, pdist, squareform

X = np.random.randn(100, 5).astype(np.float32)
centroids = np.random.randn(3, 5).astype(np.float32)

# Each sample → distance to each centroid (100 × 3)
D = cdist(X, centroids, metric="euclidean")

# Assign cluster by nearest centroid (k-means assignment step)
labels = D.argmin(axis=1)
```

Useful for clustering prototypes, similarity dashboards, and sanity-checking embedding spaces (with [[ML — NumPy]] vectors).

---

## scipy.optimize — Custom Calibration

```python
from scipy.optimize import minimize

# Fit scalar temperature T on validation logits (simple calibration)
logits = np.array([-2.0, -0.5, 0.3, 1.5])
y_val = np.array([0, 0, 1, 1])

def nll(T):
    T = max(T[0], 1e-6)
    probs = 1 / (1 + np.exp(-logits / T))
    probs = np.clip(probs, 1e-9, 1 - 1e-9)
    return -np.sum(y_val * np.log(probs) + (1 - y_val) * np.log(1 - probs))

result = minimize(nll, x0=[1.0], bounds=[(0.01, 10.0)])
T_opt = result.x[0]
```

For hyperparameter search at scale, use [[ML — Optuna]] instead.

---

## ML / Backend Patterns

### A/B test for model rollout

```python
from scipy.stats import chi2_contingency

# conversion: [[control_conv, control_no], [treatment_conv, treatment_no]]
table = np.array([[120, 880], [145, 855]])
chi2, p, dof, expected = chi2_contingency(table)
# p < 0.05 → significant lift; pair with business metrics
```

### Sparse inference memory budget

```python
def sparse_row_bytes(matrix: sparse.csr_matrix) -> int:
    return matrix.data.nbytes + matrix.indices.nbytes + matrix.indptr.nbytes
```

### Distribution check before log transform

```python
from scipy.stats import skew
# |skew| > 1 → consider log1p in [[ML — pandas]] feature step
skew_vals = {col: skew(df[col].dropna()) for col in numeric_cols}
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Dense `np.array` for TF-IDF | Keep CSR from vectorizer |
| p-hacking many features | Adjust for multiple testing (Bonferroni/FDR) |
| `cdist` on huge matrices | Batch or approximate (FAISS for ANN) |
| Ignoring sparse dtype | Use `float32` in `.astype(np.float32)` |
| SciPy optimize for 100+ params | Use dedicated tuners |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Mann-Whitney U test | `stats.mannwhitneyu(a, b)` |
| t-test | `stats.ttest_ind(a, b)` |
| Chi-square | `stats.chi2_contingency(table)` |
| Z-score | `stats.zscore(x)` |
| Sample from dist | `stats.norm.rvs(loc=0, scale=1, size=100)` |
| CSR matrix | `sparse.csr_matrix(dense)` |
| Stack sparse | `sparse.hstack([a, b], format="csr")` |
| Pairwise distances | `spatial.distance.pdist(X, metric="cosine")` |
| Cross-distance | `spatial.distance.cdist(A, B)` |
| Minimize scalar/vector | `optimize.minimize(fn, x0, bounds=...)` |

---

## Tags

#machine-learning #scipy #statistics #sparse #optimization #sklearn #python
