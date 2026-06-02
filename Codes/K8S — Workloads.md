## What & When

**Workloads** are Kubernetes objects that run containers: **Pod** (one or more containers), **ReplicaSet** (Pod replica count — usually managed by Deployment), **Deployment** (declarative rolling updates), **StatefulSet** (stable identity + storage), **Job** (run-to-completion), **CronJob** (scheduled Job).

Use this note when:

- Running **stateless APIs** ([[API - FastAPI]], [[Web — Flask]] images) — **Deployment**
- Need **ordered pods + persistent disk** — **StatefulSet**
- **One-off** or **cron** batch — Job / CronJob
- Debugging **CrashLoopBackOff** or rollout failures

CLI: [[Commands/K8S — kubectl & Minikube]]. Overview: [[K8S]].

---

## Pod

Smallest schedulable unit — one or more containers sharing network/storage.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: debug-pod
  labels:
    app: debug
spec:
  containers:
    - name: app
      image: nginx:1.27
      ports:
        - containerPort: 80
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          memory: 256Mi
```

> [!tip] Prefer Deployment over bare Pods Pods are ephemeral; Deployments recreate them on failure and manage rollouts.

---

## ReplicaSet

Maintains a stable Pod replica count. Rarely authored directly — **Deployment** owns ReplicaSets.

```bash
kubectl get rs
kubectl describe rs <name>
```

---

## Deployment (Primary Pattern)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-api
  labels:
    app: fastapi-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: fastapi-api
    spec:
      containers:
        - name: api
          image: myregistry/fastapi-api:1.2.0
          ports:
            - containerPort: 8000
          envFrom:
            - configMapRef:
                name: api-config
            - secretRef:
                name: api-secrets
          livenessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 10
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            initialDelaySeconds: 5
```

Rollout:

```bash
kubectl rollout status deployment/fastapi-api
kubectl rollout history deployment/fastapi-api
kubectl rollout undo deployment/fastapi-api
```

Pair with [[Codes/K8S — Networking]] Service + Ingress.

---

## StatefulSet

Stable network ID (`pod-0`, `pod-1`) and **PersistentVolumeClaims** per replica.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 10Gi
```

> [!warning] Managed databases often beat self-hosted StatefulSets For production Postgres/MySQL, use cloud RDS unless you own DBA ops.

See [[Codes/K8S — Storage]].

---

## Job

Runs Pods until containers exit successfully.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: myapp/migrate:1.0
          command: ["alembic", "upgrade", "head"]
```

```bash
kubectl wait --for=condition=complete job/db-migrate --timeout=120s
kubectl logs job/db-migrate
```

---

## CronJob

Scheduled Jobs — like cron on a node, but cluster-native.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-etl
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: etl
              image: myapp/etl:latest
              args: ["run", "--date=yesterday"]
```

Alternative for complex pipelines: [[Processing — Celery]] Beat off-cluster or in-cluster workers.

---

## Workload Comparison

| Kind | Use | Stable name | Storage |
| --- | --- | --- | --- |
| Deployment | Stateless HTTP, workers | No | Optional shared PVC rare |
| StatefulSet | DB, Kafka, clustered apps | Yes | Per-pod PVC typical |
| Job | Migrate, batch once | No | Optional |
| CronJob | Scheduled batch | No | Optional |
| Pod (bare) | Debug only | No | — |

---

## Probes

| Probe | Purpose |
| --- | --- |
| **liveness** | Restart if deadlocked |
| **readiness** | Remove from Service endpoints until ready |
| **startup** | Slow-start apps (optional) |

FastAPI health endpoint aligns with [[API - FastAPI]] `/health` patterns.

---

## ML Serving Example

[[ML — Seldon]] uses custom resources; underlying predictors are Deployments:

```text
SeldonDeployment → Deployment (predictor) → Pod(s)
```

See [[ML — Seldon]] for CRD YAML; workload mechanics here still apply.

---

## Quick Reference

| Task | Kind |
| --- | --- |
| REST API replicas | Deployment |
| Rolling update | Deployment `strategy` |
| Ordered replicas + disk | StatefulSet |
| Run script once | Job |
| Cron schedule | CronJob |
| Debug shell | [[Commands/K8S — kubectl & Minikube]] |

---

## Related Notes

- [[K8S]]
- [[Codes/K8S — Cluster & Namespaces]]
- [[Codes/K8S — Networking]]
- [[Codes/K8S — Storage]]
- [[Codes/K8S — Configuration & Security]]
- [[Commands/K8S — kubectl & Minikube]]
- [[ML — Seldon]]

---

## Tags

#kubernetes #k8s #deployment #pod #statefulset #job #cronjob #yaml #workloads
