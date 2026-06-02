## What & When

**ConfigMap** holds non-sensitive configuration (URLs, feature flags, file snippets). **Secret** holds sensitive data (passwords, tokens, TLS keys) — base64-encoded at rest, not encryption by default. **ServiceAccount** identifies Pods to the Kubernetes API and cloud IAM (via workload identity).

Use this note when:

- Injecting env vars from config — mirror [[Python — python-dotenv]] locally, K8s in prod
- Mounting `config.yaml` or `.env`-style files without rebaking images
- Granting a Pod permission to call AWS/GCP APIs or in-cluster resources

CLI: [[Commands/K8S — kubectl & Minikube]]. Overview: [[K8S]].

---

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: api-config
data:
  LOG_LEVEL: info
  APP_ENV: production
  settings.yaml: |
    cors_origins:
      - https://app.example.com
```

### Env from ConfigMap

```yaml
spec:
  containers:
    - name: api
      image: myapp/api:1.0
      envFrom:
        - configMapRef:
            name: api-config
      env:
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: api-config
              key: LOG_LEVEL
```

### Mount as file

```yaml
volumeMounts:
  - name: config
    mountPath: /etc/app/config
    readOnly: true
volumes:
  - name: config
    configMap:
      name: api-config
      items:
        - key: settings.yaml
          path: settings.yaml
```

---

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-secrets
type: Opaque
stringData:   # plain text in manifest; encoded on apply
  DATABASE_URL: postgresql://user:pass@db:5432/app
  JWT_SECRET: change-me-in-prod
```

> [!warning] Secrets in Git Use Sealed Secrets, External Secrets Operator, or cloud secret managers for production — do not commit raw Secret YAML.

### Env from Secret

```yaml
envFrom:
  - secretRef:
      name: api-secrets
```

TLS Secret for Ingress:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-tls
type: kubernetes.io/tls
data:
  tls.crt: <base64-cert>
  tls.key: <base64-key>
```

Create from files:

```bash
kubectl create secret generic api-secrets \
  --from-literal=DATABASE_URL='postgresql://...' \
  --from-env-file=.env.production
```

---

## ServiceAccount

Every Pod runs with a ServiceAccount (default: `default` in namespace).

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-sa
  namespace: default
```

Attach to Pod:

```yaml
spec:
  serviceAccountName: api-sa
  containers: [...]
```

### RBAC (Role + RoleBinding)

Grant read ConfigMaps in namespace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: config-reader
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: api-config-reader
subjects:
  - kind: ServiceAccount
    name: api-sa
roleRef:
  kind: Role
  name: config-reader
  apiGroup: rbac.authorization.k8s.io
```

Cloud **workload identity** maps ServiceAccount → IAM role (EKS IRSA, GKE WI) — platform-specific setup.

---

## ConfigMap vs Secret vs Image

| Source | When |
| --- | --- |
| ConfigMap | Non-secret config |
| Secret | Credentials, TLS |
| Image env | Defaults only — override with ConfigMap/Secret |
| [[Python — python-dotenv]] | Local dev outside cluster |

---

## FastAPI / ML Patterns

| App | Typical secrets |
| --- | --- |
| [[API - FastAPI]] | `DATABASE_URL`, `JWT_SECRET`, API keys |
| [[ML — MLflow]] | S3 credentials, backend store URI |
| [[ML — Seldon]] | Pull secrets for private registry (`imagePullSecrets`) |

Registry pull secret:

```bash
kubectl create secret docker-registry regcred \
  --docker-server=... --docker-username=... --docker-password=...
```

```yaml
spec:
  imagePullSecrets:
    - name: regcred
```

---

## Quick Reference

| Task | Object |
| --- | --- |
| Non-secret env | ConfigMap + `envFrom` |
| Passwords / keys | Secret + `secretRef` |
| Config file mount | ConfigMap volume |
| Ingress TLS | `type: kubernetes.io/tls` Secret |
| Pod identity | ServiceAccount |
| API permissions | Role + RoleBinding |

---

## Related Notes

- [[K8S]]
- [[Codes/K8S — Workloads]]
- [[Codes/K8S — Networking]]
- [[Codes/K8S — Storage]]
- [[Commands/K8S — kubectl & Minikube]]
- [[Python — python-dotenv]]
- [[API - FastAPI]]

---

## Tags

#kubernetes #k8s #configmap #secret #serviceaccount #rbac #security #yaml
