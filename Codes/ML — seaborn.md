## What & When

**seaborn** is a **statistical visualization layer** on [[ML — matplotlib]] — high-level APIs for distributions, relationships, and correlation heatmaps on [[ML — pandas]] DataFrames. It accelerates ML **exploratory data analysis (EDA)** before feature engineering and [[ML — scikit-learn]] modeling.

Use seaborn when:

- Inspecting class balance, feature distributions, and outliers by target
- Correlation heatmaps to spot multicollinearity before linear models
- Pair plots for small feature subsets (≤ 6–8 columns)
- Comparing model score distributions across folds or segments
- Quick categorical vs numeric breakdowns (`boxenplot`, `violinplot`)

```python
import seaborn as sns
import matplotlib.pyplot as plt
import pandas as pd
```

EDA companion in [[Machine Learning]] — graduate to [[ML — matplotlib]] for custom evaluation charts.

---

## pip install

```bash
pip install seaborn
```

Pulls matplotlib automatically. For styled themes, no extra deps:

```python
import seaborn as sns
sns.set_theme(style="whitegrid", palette="muted")
```

---

## seaborn vs Alternatives

| Need | Prefer | Notes |
| --- | --- | --- |
| Fast DataFrame EDA | **seaborn** | Native pandas integration |
| Exact ROC/confusion layout | [[ML — matplotlib]] | seaborn lacks ROC helpers |
| Interactive exploration | Plotly Express | seaborn is static |
| One-line pandas plot | `df.plot()` | Less statistical defaults |
| Large pair plots (20+ cols) | Correlation heatmap only | Pairplot explodes combinatorially |
| Publication polish | seaborn + matplotlib tweaks | `fig.savefig` same as mpl |

---

## Distribution by Target — Class Separation

```python
import seaborn as sns
import matplotlib.pyplot as plt
import pandas as pd

df = pd.read_csv("churn.csv")   # columns: monthly_charges, tenure, churn, ...

g = sns.displot(
    data=df,
    x="monthly_charges",
    hue="churn",
    kind="kde",
    fill=True,
    common_norm=False,
    height=4,
    aspect=1.5,
)
g.set_axis_labels("Monthly Charges", "Density")
g.figure.suptitle("Charge Distribution by Churn", y=1.02)
plt.savefig("artifacts/charges_by_churn.png", dpi=150, bbox_inches="tight")
plt.close()
```

If distributions overlap heavily, the feature may be weak alone — combine in models or interactions.

---

## Box / Violin — Outliers & Segments

```python
fig, ax = plt.subplots(figsize=(8, 4))
sns.boxplot(data=df, x="contract_type", y="tenure", hue="churn", ax=ax)
ax.set_title("Tenure by Contract and Churn")
fig.tight_layout()
plt.close(fig)
```

```python
# Violin shows full density shape
sns.violinplot(data=df, x="region", y="monthly_charges", split=True, hue="churn")
```

Use before winsorizing/clipping in [[ML — pandas]] feature prep.

---

## Correlation Heatmap — Multicollinearity

```python
numeric = df.select_dtypes("number").drop(columns=["customer_id"], errors="ignore")
corr = numeric.corr()

fig, ax = plt.subplots(figsize=(10, 8))
sns.heatmap(
    corr,
    annot=False,
    cmap="coolwarm",
    center=0,
    vmin=-1,
    vmax=1,
    square=True,
    linewidths=0.5,
    ax=ax,
)
ax.set_title("Feature Correlation Matrix")
fig.tight_layout()
plt.savefig("artifacts/correlation.png", dpi=150)
plt.close(fig)
```

Pairs with |r| > 0.9 often warrant dropping one feature for linear/logistic models — tree models tolerate redundancy better.

---

## Pairplot — Small Feature Subsets

```python
subset = df[["tenure", "monthly_charges", "total_charges", "churn"]].copy()
subset["churn"] = subset["churn"].astype(str)

g = sns.pairplot(subset, hue="churn", corner=True, diag_kind="kde", height=2.2)
g.figure.suptitle("Pairwise Feature Relationships", y=1.02)
plt.savefig("artifacts/pairplot.png", dpi=150, bbox_inches="tight")
plt.close()
```

Limit to ≤ 6 features — runtime and readability degrade quickly.

---

## Model Diagnostics — CV Score Distribution

```python
import pandas as pd

cv_results = pd.DataFrame({
    "fold": [1, 2, 3, 4, 5],
    "roc_auc": [0.81, 0.79, 0.83, 0.80, 0.82],
    "model": ["logreg"] * 5,
})

fig, ax = plt.subplots(figsize=(5, 4))
sns.boxplot(data=cv_results, x="model", y="roc_auc", ax=ax)
sns.stripplot(data=cv_results, x="model", y="roc_auc", color="black", alpha=0.6, ax=ax)
ax.set_ylim(0.7, 0.9)
ax.set_title("CV ROC AUC Spread")
plt.close(fig)
```

Wide spread → unstable model or small validation folds.

---

## ML / Backend Patterns

### Theme once at notebook/script top

```python
sns.set_theme(context="notebook", style="ticks", font_scale=1.1)
PALETTE = {0: "#4C72B0", 1: "#DD8452"}   # consistent churn colors
```

### Facet by segment (backend monitoring)

```python
# Score drift across regions after deployment
monitor = pd.DataFrame({
    "region": ["EU", "EU", "US", "US"],
    "week": ["w1", "w2", "w1", "w2"],
    "mean_score": [0.22, 0.31, 0.20, 0.21],
})

g = sns.relplot(data=monitor, x="week", y="mean_score", hue="region", kind="line", marker="o")
g.figure.savefig("artifacts/score_drift.png", dpi=150)
plt.close()
```

### EDA checklist before training

| Plot | seaborn call | Look for |
| --- | --- | --- |
| Target balance | `countplot(hue=target)` | Severe imbalance → class weights |
| Numeric vs target | `displot(hue=target, kind="kde")` | Separable distributions |
| Correlations | `heatmap(corr)` | \|r\| > 0.9 pairs |
| Segment fairness | `barplot` rate by group | Disparate churn/approval rates |

Run EDA on **train split only** to avoid leakage — see [[ML — pandas]].

---

## Common Mistakes

| Mistake | Fix |
| --- | --- |
| Pairplot on 30 features | Subset or correlation heatmap only |
| KDE on tiny samples | Use `histplot` or `stripplot` |
| Ignoring `hue` order | Pass `hue_order=[0, 1]` for consistent colors |
| Not saving with `bbox_inches="tight"` | Clipped labels in reports |
| EDA on full data including test | Split first; EDA on train only to avoid leakage |

---

## Quick Reference

| Task | Code |
| --- | --- |
| Set theme | `sns.set_theme(style="whitegrid")` |
| Distribution | `sns.histplot` / `displot(kind="kde")` |
| Box by category | `sns.boxplot(data=df, x=cat, y=num, hue=target)` |
| Violin | `sns.violinplot(...)` |
| Correlation heatmap | `sns.heatmap(df.corr(), cmap="coolwarm", center=0)` |
| Pairwise grid | `sns.pairplot(df, hue=target, corner=True)` |
| Scatter + fit | `sns.regplot(data=df, x=..., y=...)` |
| Count categories | `sns.countplot(data=df, x=cat, hue=target)` |
| Line by group | `sns.relplot(kind="line", marker="o")` |
| Save figure | `plt.savefig(path, bbox_inches="tight")` |

---

## Tags

#machine-learning #seaborn #visualization #eda #pandas #matplotlib #python
