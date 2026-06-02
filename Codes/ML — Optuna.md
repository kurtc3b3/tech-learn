## What & When

**Optuna** is a **hyperparameter optimization** framework with define-by-run APIs, efficient samplers (TPE, CMA-ES), and optional **pruning** of unpromising trials. It integrates cleanly with [[ML — scikit-learn]], boosting libraries, and PyTorch training loops.

Use Optuna when:

- Grid search is too expensive or too coarse
- You need **early stopping** of bad trials (MedianPruner, HyperbandPruner)
- Tracking many experiments — pair with [[ML — MLflow]] or Optuna Dashboard
- Tuning after [[ML — Boruta]] feature selection or inside nested CV

```bash
pip install optuna
# Visualization: pip install optuna-dashboard plotly
```

Serve the winning model via [[ML — BentoML]] / [[ML — Seldon]] behind [[API - FastAPI]]. Overview: [[Machine Learning]].

---

## Optuna vs Related Tools

| Need | Use | Notes |
| --- | --- | --- |
| Bayesian / TPE search | **Optuna** | `study.optimize(objective, n_trials=100)` |
| Exhaustive small grids | `GridSearchCV` | sklearn built-in |
| Distributed Ray Tune | Ray | Cluster-scale; heavier ops |
| Neural arch search | Optuna + PyTorch | Same `trial` API |
| Experiment registry | [[ML — MLflow]] | Log `study.best_params` per run |
| Explain best model | [[ML — SHAP]] | After re-fit with best params |

---

## Minimal Objective (sklearn)

```python
import optuna
from sklearn.datasets import load_breast_cancer
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import cross_val_score

X, y = load_breast_cancer(return_X_y=True)

def objective(trial: optuna.Trial) -> float:
    params = {
        "n_estimators": trial.suggest_int("n_estimators", 50, 300),
        "max_depth": trial.suggest_int("max_depth", 2, 16),
        "min_samples_split": trial.suggest_float("min_samples_split", 0.01, 0.3),
        "class_weight": trial.suggest_categorical("class_weight", [None, "balanced"]),
    }
    clf = RandomForestClassifier(**params, random_state=42, n_jobs=-1)
    scores = cross_val_score(clf, X, y, cv=5, scoring="roc_auc")
    return scores.mean()

study = optuna.create_study(direction="maximize", study_name="rf-breast")
study.optimize(objective, n_trials=50, show_progress_bar=True)

print(study.best_value, study.best_params)
best_clf = RandomForestClassifier(**study.best_params, random_state=42)
best_clf.fit(X, y)
```

---

## Suggest API Cheat Sheet

```python
def objective(trial: optuna.Trial) -> float:
    # Int / float / log-uniform
    layers = trial.suggest_int("n_layers", 1, 4)
    lr = trial.suggest_float("lr", 1e-5, 1e-1, log=True)
    wd = trial.suggest_float("weight_decay", 1e-6, 1e-2, log=True)

    # Categorical & discrete uniform
    optimizer = trial.suggest_categorical("optimizer", ["adam", "sgd"])
    step = trial.suggest_discrete_uniform("step", 0.1, 0.9, 0.1)

    # Conditional parameters (define-by-run)
    if optimizer == "sgd":
        momentum = trial.suggest_float("momentum", 0.0, 0.99)
    else:
        momentum = 0.0

    # User attributes (for logging, not optimized)
    trial.set_user_attr("dataset_version", "v3")
    return 0.0  # placeholder
```

---

## Pruning — Stop Bad Trials Early

```python
from sklearn.model_selection import StratifiedKFold
from sklearn.metrics import log_loss

def objective_with_pruning(trial: optuna.Trial) -> float:
    params = {
        "max_depth": trial.suggest_int("max_depth", 3, 12),
        "learning_rate": trial.suggest_float("learning_rate", 0.01, 0.3, log=True),
    }
    import xgboost as xgb
    model = xgb.XGBClassifier(**params, n_estimators=500, tree_method="hist")

    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
    losses = []
    for fold, (train_idx, val_idx) in enumerate(cv.split(X, y)):
        model.fit(
            X[train_idx], y[train_idx],
            eval_set=[(X[val_idx], y[val_idx])],
            verbose=False,
        )
        proba = model.predict_proba(X[val_idx])
        loss = log_loss(y[val_idx], proba)
        losses.append(loss)
        trial.report(loss, step=fold)
        if trial.should_prune():
            raise optuna.TrialPruned()
    return sum(losses) / len(losses)

study = optuna.create_study(
    direction="minimize",
    pruner=optuna.pruners.MedianPruner(n_startup_trials=5, n_warmup_steps=1),
)
study.optimize(objective_with_pruning, n_trials=100)
```

---

## Storage, Resume & Parallel

```python
# SQLite persistence (local dev)
storage = "sqlite:///optuna_rf.db"
study = optuna.create_study(
    study_name="rf-production",
    storage=storage,
    direction="maximize",
    load_if_exists=True,
)

# Parallel workers (separate processes, same storage URL)
# terminal 1: study.optimize(objective, n_trials=50)
# terminal 2: study.optimize(objective, n_trials=50)
```

```python
# PostgreSQL for team shared studies
storage = "postgresql://user:pass@localhost:5432/optuna"
study = optuna.create_study(study_name="team-lgbm", storage=storage, direction="maximize")
```

---

## MLflow Integration

```python
import mlflow
import optuna

def objective(trial: optuna.Trial) -> float:
    params = {
        "C": trial.suggest_float("C", 1e-3, 1e2, log=True),
        "penalty": trial.suggest_categorical("penalty", ["l2"]),
    }
    from sklearn.linear_model import LogisticRegression
    from sklearn.model_selection import cross_val_score

    with mlflow.start_run(nested=True):
        mlflow.log_params(params)
        clf = LogisticRegression(**params, max_iter=2000)
        auc = cross_val_score(clf, X, y, cv=5, scoring="roc_auc").mean()
        mlflow.log_metric("auc", auc)
    return auc

with mlflow.start_run(run_name="optuna-sweep"):
    study = optuna.create_study(direction="maximize")
    study.optimize(objective, n_trials=30)
    mlflow.log_params(study.best_params)
    mlflow.log_metric("best_auc", study.best_value)
```

---

## sklearn `OptunaSearchCV` (Experimental)

```python
from optuna.integration import OptunaSearchCV
from sklearn.svm import SVC

search = OptunaSearchCV(
    estimator=SVC(),
    param_distributions={
        "C": (1e-3, 1e2),
        "gamma": (1e-4, 1e-1),
    },
    n_trials=40,
    cv=5,
    scoring="accuracy",
    random_state=42,
)
search.fit(X, y)
print(search.best_params_, search.best_score_)
```

---

## Visualization & Dashboard

```python
import optuna.visualization as vis

fig = vis.plot_optimization_history(study)
fig = vis.plot_param_importances(study)
fig = vis.plot_parallel_coordinate(study)

# CLI dashboard
# optuna-dashboard sqlite:///optuna_rf.db
```

---

## Production Pattern

1. **Search** offline with Optuna + [[ML — MLflow]].
2. **Re-train** final model on full training data with `study.best_params`.
3. **Register** artifact in MLflow Model Registry.
4. **Deploy** with [[ML — BentoML]] or [[ML — Seldon]]; expose HTTP via [[API - FastAPI]] gateway if needed.

```python
final_params = study.best_params
# Re-fit on train+val, log once, promote registry stage → Production
```

---

## Quick Reference

| Task | Code |
| --- | --- |
| Create study | `optuna.create_study(direction="maximize")` |
| Optimize | `study.optimize(objective, n_trials=100)` |
| Best | `study.best_params`, `study.best_value` |
| Prune | `trial.report(val, step=n); trial.should_prune()` |
| Persist | `storage="sqlite:///study.db", load_if_exists=True` |
| Int suggest | `trial.suggest_int("x", low, high)` |
| Log suggest | `trial.suggest_float("lr", 1e-5, 1e-1, log=True)` |

---

## Related Notes

- [[Machine Learning]]
- [[ML — scikit-learn]]
- [[ML — MLflow]]
- [[ML — Boruta]]
- [[ML — SHAP]]
- [[ML — XGBoost]]
- [[API - FastAPI]]

---

## Tags

#python #machine-learning #optuna #hyperparameter-tuning #sklearn #mlops #tpe
