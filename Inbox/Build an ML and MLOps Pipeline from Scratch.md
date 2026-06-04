> **Expanded roadmap:** See [[Machine Learning]], [[DVC]], [[Processing]], and [[ORCHESTRATION]]. This note is a short checklist.

- **Environment:** Reproducible Python project with `uv` or `poetry` [[Python — uv]] [[Python — Poetry]] [[Linting]] [[Unit Testing - pytest]]
- **Explore & prepare data:** Tabular EDA and transforms [[ML — pandas]] [[ML — NumPy]] [[ML — scipy]] [[ML — matplotlib]] [[ML — seaborn]]
- **Version datasets & pipelines:** Git + DVC remotes for large artifacts [[DVC]] [[ML — DVC]]
- **Feature work:** Selection and engineering [[ML — Boruta]] [[ML — scikit-learn]]
- **Train baselines:** Classical models first [[ML — scikit-learn]] then [[ML — XGBoost]] [[ML — LightGBM]] [[ML — H2O]]
- **Tune & explain:** Hyperparameters and interpretability [[ML — Optuna]] [[ML — SHAP]]
- **Track experiments:** Params, metrics, registry [[ML — MLflow]]
- **Deep learning & time series:** When tabular is not enough [[ML — PyTorch]] [[ML — Prophet]]
- **Shared features (optional):** Online/offline feature store [[ML — Feast]]
- **Distributed training / batch:** [[Processing — Ray]] for scale-out compute
- **Orchestrate ML DAGs on K8s:** [[ORCHESTRATION — Kubeflow Pipelines]] with [[K8S]]
- **Serve models:** REST containers then cluster deploy [[ML — BentoML]] [[ML — Seldon]] [[API - FastAPI]]
- **NLP side path:** Text-specific stack [[NLP]] [[NLP — spaCy]] [[NLP — NLTK]] [[NLP — Gensim]]
