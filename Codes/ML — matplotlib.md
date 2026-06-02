## What & When

**matplotlib** is Python's **low-level plotting library** — full control over figures for ML diagnostics that [[ML — seaborn]] abstracts away. Use it for learning curves, confusion matrices, calibration plots, custom dashboards, and saving publication-ready PNG/SVG for MLflow artifacts.

Use matplotlib when:

- Plotting sklearn `learning_curve`, validation curves, and metric trends
- Custom confusion matrices and ROC/PR curves with exact styling
- Embedding static charts in reports, MLflow runs, or FastAPI-generated PDFs
- Overriding seaborn defaults for production notebook → script pipelines
- Animating training loss (optional) in custom PyTorch loops

```python
import matplotlib.pyplot as plt
import numpy as np
```

Visualization layer in [[Machine Learning]] — pair [[ML — seaborn]] for fast EDA, matplotlib for precise control.

---

## pip install

```bash
pip install matplotlib
```

Headless servers (CI, Docker training jobs):

```python
import matplotlib
matplotlib.use("Agg")   # non-interactive backend — before pyplot import
import matplotlib.pyplot as plt
```

Jupyter: `%matplotlib inline` still common; modern IPython often auto-detects.

---

## matplotlib vs Alternatives

| Need | Prefer | Notes |
| --- | --- | --- |
| Quick EDA on DataFrames | [[ML — seaborn]] | Built on matplotlib |
| Interactive dashboards | Plotly / Altair | matplotlib is static-first |
| sklearn metric plots | **matplotlib** | Full control over axes |
| Large scatter (1M+ points) | datashader / hexbin | Raw scatter slows down |
| Notebook one-liners | pandas `.plot()` | Wraps matplotlib |
| ML report automation | **matplotlib** + `savefig` | Reliable in CI |

---

## pyplot Basics — ML Figure Pattern

```python
import matplotlib.pyplot as plt
import numpy as np

fig, ax = plt.subplots(figsize=(8, 5))

epochs = np.arange(1, 21)
train_loss = 0.8 * np.exp(-0.15 * epochs) + 0.05
val_loss = 0.9 * np.exp(-0.12 * epochs) + 0.12

ax.plot(epochs, train_loss, label="train", marker="o", markersize=3)
ax.plot(epochs, val_loss, label="validation", marker="o", markersize=3)
ax.set_xlabel("Epoch")
ax.set_ylabel("Loss")
ax.set_title("Training vs Validation Loss")
ax.legend()
ax.grid(True, alpha=0.3)

fig.tight_layout()
fig.savefig("artifacts/loss_curve.png", dpi=150)
plt.close(fig)   # free memory in batch jobs
```

Always `close()` figures in training loops to avoid memory leaks.

---

## Confusion Matrix Heatmap

```python
from sklearn.metrics import confusion_matrix

y_true = np.array([0, 1, 0, 1, 1, 0, 1, 0])
y_pred = np.array([0, 1, 0, 0, 1, 0, 1, 1])

cm = confusion_matrix(y_true, y_pred, labels=[0, 1])

fig, ax = plt.subplots(figsize=(5, 4))
im = ax.imshow(cm, cmap="Blues")

ax.set_xticks([0, 1], labels=["Pred 0", "Pred 1"])
ax.set_yticks([0, 1], labels=["True 0", "True 1"])
ax.set_xlabel("Predicted")
ax.set_ylabel("Actual")

for i in range(cm.shape[0]):
    for j in range(cm.shape[1]):
        ax.text(j, i, cm[i, j], ha="center", va="center", color="black")

fig.colorbar(im, ax=ax, fraction=0.046)
fig.tight_layout()
fig.savefig("artifacts/confusion_matrix.png", dpi=150)
plt.close(fig)
```

For quick EDA heatmaps, [[ML — seaborn]] `heatmap` is less verbose.

---

## ROC & Precision-Recall Curves

```python
from sklearn.metrics import roc_curve, auc, precision_recall_curve, average_precision_score

y_true = np.array([0, 0, 1, 1, 1, 0, 1, 0])
y_score = np.array([0.1, 0.4, 0.35, 0.8, 0.9, 0.2, 0.75, 0.05])

fpr, tpr, _ = roc_curve(y_true, y_score)
roc_auc = auc(fpr, tpr)

precision, recall, _ = precision_recall_curve(y_true, y_score)
ap = average_precision_score(y_true, y_score)

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(10, 4))

ax1.plot(fpr, tpr, label=f"AUC = {roc_auc:.3f}")
ax1.plot([0, 1], [0, 1], "k--", alpha=0.4)
ax1.set_xlabel("False Positive Rate")
ax1.set_ylabel("True Positive Rate")
ax1.set_title("ROC")
ax1.legend()

ax2.plot(recall, precision, label=f"AP = {ap:.3f}")
ax2.set_xlabel("Recall")
ax2.set_ylabel("Precision")
ax2.set_title("Precision-Recall")
ax2.legend()

fig.tight_layout()
fig.savefig("artifacts/roc_pr.png", dpi=150)
plt.close(fig)
```

Log these figures to MLflow with `mlflow.log_figure(fig, "roc.png")`.

---

## Learning Curve (sklearn)

```python
from sklearn.model_selection import learning_curve
from sklearn.linear_model import LogisticRegression

train_sizes, train_scores, val_scores = learning_curve(
    LogisticRegression(max_iter=1000), X, y, cv=5,
    train_sizes=np.linspace(0.1, 1.0, 5), scoring="roc_auc",
)
fig, ax = plt.subplots(figsize=(7, 4))
ax.plot(train_sizes, train_scores.mean(1), "o-", label="Train")
ax.plot(train_sizes, val_scores.mean(1), "o-", label="CV")
ax.set(xlabel="Training samples", ylabel="ROC AUC", title="Learning Curve")
ax.legend()
fig.tight_layout()
plt.close(fig)
```

See [[ML — scikit-learn]] for `validation_curve` and grid search.

---

## ML / Backend Patterns

### Reusable style for reports

```python
plt.style.use("seaborn-v0_8-whitegrid")   # or ggplot, fivethirtyeight
plt.rcParams.update({"figure.dpi": 120, "font.size": 10})
```

### Log to MLflow

```python
import mlflow

fig, ax = plt.subplots()
ax.plot([1, 2, 3], [1, 4, 2])
mlflow.log_figure(fig, "metrics/trend.png")
plt.close(fig)
```

### Feature importance bar chart

```python
importances.sort_values().plot(kind="barh", ax=ax)   # tree feature_importances_ or [[ML — SHAP]]
fig.savefig("artifacts/feature_importance.png", dpi=150)
```

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Forgetting `plt.close()` in loops | Leaks memory in long training |
| `plt.show()` in headless CI | Use `Agg` backend + `savefig` only |
| Overplotting millions of points | `alpha=0.1`, hexbin, or sample |
| Unlabeled axes in shared reports | Always set title + x/y labels |
| Modifying global rcParams everywhere | Set style once at script entry |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Figure + axes | `fig, ax = plt.subplots(figsize=(w, h))` |
| Line plot | `ax.plot(x, y, label="...")` |
| Bar chart | `ax.bar(x, height)` / `Series.plot.barh()` |
| Heatmap (manual) | `ax.imshow(matrix, cmap="Blues")` |
| Labels / title | `ax.set_xlabel`, `set_ylabel`, `set_title` |
| Legend | `ax.legend()` |
| Grid | `ax.grid(True, alpha=0.3)` |
| Tight layout | `fig.tight_layout()` |
| Save PNG | `fig.savefig(path, dpi=150)` |
| Close | `plt.close(fig)` |
| Headless | `matplotlib.use("Agg")` before pyplot |

---

## Tags

#machine-learning #matplotlib #visualization #plots #sklearn #mlflow #python
