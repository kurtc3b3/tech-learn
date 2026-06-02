## What & When

**Service** gives Pods a **stable virtual IP and DNS name** inside the cluster. **Ingress** routes external HTTP(S) traffic to Services by host and path — TLS termination, canary rules (with controllers like nginx-ingress).

Use this note when:

- Exposing a **Deployment** internally (ClusterIP) or externally (LoadBalancer / NodePort)
- Terminating **HTTPS** at the edge with Ingress
- Wiring **[[API - FastAPI]]** or [[ML — Seldon]] predictors to a public URL
- Debugging **connection refused** between services

CLI: [[Commands/K8S — kubectl & Minikube]]. Overview: [[K8S]].

---

## Service Types

| Type | Use |
| --- | --- |
| **ClusterIP** | Default — internal only |
| **NodePort** | Open port on every node (dev / legacy) |
| **LoadBalancer** | Cloud LB provisions external IP |
| **ExternalName** | CNAME to external DNS |

---

## ClusterIP Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-api
  labels:
    app: fastapi-api
spec:
  type: ClusterIP
  selector:
    app: fastapi-api
  ports:
    - name: http
      port: 80
      targetPort: 8000
      protocol: TCP
```

DNS inside cluster:

```text
fastapi-api.default.svc.cluster.local:80
```

Short names work within same namespace: `http://fastapi-api`.

---

## Headless Service (StatefulSet)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
```

Returns Pod IPs directly — needed for StatefulSet stable identity.

---

## Ingress

Requires an **Ingress controller** (nginx-ingress, traefik, etc.). Minikube: `minikube addons enable ingress`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: fastapi-api
                port:
                  number: 80
```

Path-based routing (same host, multiple services):

```yaml
paths:
  - path: /api
    pathType: Prefix
    backend:
      service:
        name: fastapi-api
        port:
          number: 80
  - path: /ml
    pathType: Prefix
    backend:
      service:
        name: seldon-iris-default
        port:
          number: 8000
```

---

## End-to-End Flow

```text
Internet → Ingress (TLS) → Service:80 → Pod:8000 (Uvicorn/FastAPI)
```

Local dev without Ingress:

```bash
kubectl port-forward svc/fastapi-api 8080:80
# curl localhost:8080/health
```

See [[Commands/K8S — kubectl & Minikube]].

---

## Service ↔ Deployment Wiring

1. Deployment Pod template labels: `app: fastapi-api`
2. Service `selector` matches same labels
3. Endpoints object auto-populated from ready Pods (readiness probe matters)

```bash
kubectl get endpoints fastapi-api
kubectl describe svc fastapi-api
```

---

## Network Policies (Brief)

Optional firewall between Pods — default allow-all. Restrict egress to DB namespace in production (advanced; not required for learning path).

---

## ML Inference Exposure

[[ML — Seldon]] creates Services per predictor:

```bash
kubectl get svc -n ml-production
kubectl port-forward svc/iris-default 8000:8000 -n ml-production
```

Production: Ingress or service mesh instead of port-forward.

---

## Troubleshooting Checklist

| Symptom | Check |
| --- | --- |
| 502 from Ingress | Pod readiness, Service port vs `targetPort` |
| No endpoints | Label mismatch Deployment ↔ Service |
| Works via port-forward, not Ingress | Ingress host, class, controller running |
| TLS error | Secret `tls.crt` / `tls.key` in namespace |

---

## Quick Reference

| Task | Object |
| --- | --- |
| Internal API | ClusterIP Service |
| Public HTTP | Ingress → Service |
| Stable Pod DNS | Headless Service + StatefulSet |
| Dev access | `kubectl port-forward svc/...` |
| TLS cert | Ingress `tls` + Secret |

---

## Related Notes

- [[K8S]]
- [[Codes/K8S — Workloads]]
- [[Codes/K8S — Configuration & Security]]
- [[Commands/K8S — kubectl & Minikube]]
- [[API - FastAPI]]
- [[ML — Seldon]]

---

## Tags

#kubernetes #k8s #service #ingress #networking #yaml #tls
