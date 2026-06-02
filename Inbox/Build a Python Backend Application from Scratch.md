> **Expanded roadmap:** See [[Python Development]] for flow, design patterns, and full references. This note remains a short checklist.

- **Templating:** Create a reusable python template project using `copier` [[Python — Copier]]
- **Dependency Manager:** Start with `uv` `poetry` [[Python — uv]] [[Python — Poetry]]
- **Code Quality:** Setup linting with `pre-commit` and `ruff` [[Linting]] [[Linting — Ruff]] [[Linting — pre-commit]] [[Linting — mypy]]
- **Testing:** Setup tests with `pytest` and `unittest` [[Unit Testing - pytest]]
- **Logging:** Setup logging and progress with `logging` and `tqdm` [[Python — logging]] [[Python — tqdm]]
- **Configuration:** Setup settings using `pydantic-settings` or `python-dotenv` [[Python — Pydantic]] [[Python — python-dotenv]]
- **Framework:** Create base classes with `abc`, develop `decorators` [[Python — abc]] [[Python — functools]]
- **Web Scraping and Automation:** Create base scrapers with `httpx`, `playwright`, `scrapy` with `tenacity`, extract data with `bs4`, run with `asyncio`. Transform html to markdown with `markdownify`. For AI based use `crawl4ai` or `scrapegraphai` [[Browser Automation]] [[Browser Automation — Playwright]] [[Browser Automation — Scrapy]] [[Browser Automation — crawl4ai]] [[Browser Automation — ScrapeGraphAI]] [[Browser Automation — Scrapling]]
- **API:** Setup api project with `fastapi`, serve with `uvicorn`. Serve models with `bentoml` and `seldon` [[API - FastAPI]] [[ML — BentoML]] [[ML — Seldon]]
- **Web (HTML / monolith):** See [[Web]] — [[Web — Flask]], [[Web — Django]], [[Web — Tornado]]
- **Data Models:** Template queries with `jinja2`, use file operations with `pathlib`, create data models with `pydantic` and `dataclasses`, create ORMs with `sqlalchemy` [[Python — Jinja2 Package]] [[Python — pathlib]] [[Python — Pydantic]] [[ORM - SQLAlchemy]]
- **CLI:** Create CLIs with `click`, `rich`, `argparse` and `typer` [[CLI]] [[Python — Click & Rich]] [[Python — Typer]] [[Python — argparse]]. Operator tools: `git`, `gh`, `docker`, `newman` [[Commands/CLI — Git & GitHub]] [[Commands/CLI — Docker & Compose]] [[Commands/CLI — Newman & Postman]]
- **Data platform:** See [[DB]] — [[DB — Redis]], [[DB — Kafka]], [[DB — RabbitMQ]], [[DB — MongoDB]], [[DB — Neo4j]], [[DB — ELK]], [[DB — InfluxDB]], [[DB — Prometheus & Grafana]]. Relational: [[ORM - SQLAlchemy]]
- **AI:** See [[AI]] — [[AI — LangChain]], [[AI — LangGraph]], [[AI — CrewAI]], [[AI — Agno]], [[AI — MCP]], [[AI — ACP]], [[AI — Docling]]. ML/MLOps: [[Machine Learning]] [[ML — MLflow]] [[ML — Feast]]. Jobs: [[Processing]] [[Processing — Celery]] [[Processing — Ray]]
- **Anti-bot Protection Bypassing:** For anti-bot protection issues, use `camoufox`, `botright`, `botasaurus`. For residential proxy use `brightdata` [TODO: Concept and Codes]

- [[Python — functools]], [[Python — itertools]], [[Python — collections]], [[Python — contextlib]]
