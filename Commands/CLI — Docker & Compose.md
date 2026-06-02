## What & When

**Docker** packages apps into **images** and runs them as isolated **containers**. **Docker Compose** defines multi-container stacks (API + database + Redis) in a `compose.yaml` file — the standard local dev environment before [[K8S]].

Use when:

- **Local dev** for [[API - FastAPI]], [[Web — Flask]], workers ([[Processing — Celery]])
- **Reproducible** Postgres/Redis/RabbitMQ without host installs
- **Build images** pushed to registry for [[Codes/K8S — Workloads]] Deployments
- **ML serving** — containerize [[ML — BentoML]] / [[ML — Seldon]] predictors

Overview: [[CLI]].

---

## Install

```bash
# macOS — Docker Desktop includes compose plugin
brew install --cask docker

docker --version
docker compose version
```

Verify:

```bash
docker run hello-world
```

---

## Core Concepts

| Term | Meaning |
| --- | --- |
| **Image** | Immutable template (layers) |
| **Container** | Running instance of an image |
| **Volume** | Persistent storage |
| **Network** | Container DNS connectivity |
| **Registry** | Image store (Docker Hub, GHCR, ECR) |

---

## Dockerfile (FastAPI Example)

```dockerfile
FROM python:3.12-slim

WORKDIR /app
RUN pip install --no-cache-dir uv

COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

COPY . .
EXPOSE 8000
CMD ["uv", "run", "uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Build & run:

```bash
docker build -t myapi:local .
docker run --rm -p 8000:8000 --env-file .env myapi:local
curl localhost:8000/health
```

Pair with [[Python — python-dotenv]] locally; use Compose `env_file` or secrets in prod.

---

## docker compose — Stack

```yaml
# compose.yaml
services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      db:
        condition: service_healthy
    volumes:
      - ./app:/app/app   # dev hot-reload (optional)

  db:
    image: postgres:16
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

---

## Compose Commands

```bash
docker compose up -d              # start detached
docker compose ps
docker compose logs -f api
docker compose exec api bash
docker compose down               # stop & remove containers
docker compose down -v            # also remove volumes (data loss!)
docker compose build --no-cache
docker compose up -d --build
```

---

## Image Tag & Push

```bash
docker tag myapi:local ghcr.io/myorg/myapi:1.2.0
docker login ghcr.io
docker push ghcr.io/myorg/myapi:1.2.0
```

Use tagged image in [[Codes/K8S — Workloads]] Deployment `image:` field.

---

## docker CLI (Without Compose)

```bash
docker ps -a
docker images
docker stop <container>
docker rm <container>
docker rmi <image>
docker system df
docker system prune                 # careful — removes unused data
docker inspect <container>
```

---

## Volumes & Data

```bash
docker volume ls
docker volume inspect pgdata
```

Named volumes survive `docker compose down`; removed with `-v`. Back up Postgres before wiping volumes.

---

## Networks

Compose creates a default network — services resolve by name (`db`, `redis`):

```yaml
# api DATABASE_URL=postgresql://app:secret@db:5432/app
```

---

## Multi-Stage Builds (Smaller Images)

```dockerfile
FROM python:3.12-slim AS builder
WORKDIR /app
COPY . .
RUN pip wheel --no-cache-dir -w /wheels .

FROM python:3.12-slim
COPY --from=builder /wheels /wheels
RUN pip install --no-cache-dir /wheels/*.whl
COPY app ./app
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

---

## Docker vs Kubernetes

| | Docker Compose | [[K8S]] |
| --- | --- | --- |
| Scope | Single host | Cluster |
| Config | `compose.yaml` | YAML manifests |
| Scaling | `deploy.replicas` (limited) | Deployment replicas, HPA |
| Use | Local dev, demos | Production |

Typical path: **Compose locally → CI builds image → kubectl apply**.

---

## ML / Worker Patterns

| Workload | Compose service |
| --- | --- |
| FastAPI | `api` |
| Celery worker | `worker` + `redis` broker — [[Processing — Celery]] |
| MLflow tracking | `mlflow` + `db` — [[ML — MLflow]] |
| Bento model server | `model-server` — [[ML — BentoML]] |

---

## Troubleshooting

| Problem | Check |
| --- | --- |
| Port in use | Change host port mapping `8001:8000` |
| DB connection refused | `depends_on` + healthcheck; use service name `db` not `localhost` |
| Stale image | `docker compose build --no-cache` |
| Permission errors | Volume mount paths, user in Dockerfile |
| Out of disk | `docker system prune` |

---

## Quick Reference

```bash
docker compose up -d
docker compose logs -f api
docker compose exec db psql -U app -d app
docker build -t myapp:local .
docker push registry/myapp:tag
docker compose down
```

---

## Related Notes

- [[CLI]]
- [[K8S]]
- [[Codes/K8S — Workloads]]
- [[API - FastAPI]]
- [[Processing — Celery]]
- [[ML — BentoML]]
- [[Python — uv]]

---

## Tags

#cli #docker #docker-compose #containers #devops #local-dev #images
