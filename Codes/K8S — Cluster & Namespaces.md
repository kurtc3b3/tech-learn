## What & When

**Cluster** is the top-level Kubernetes boundary — control plane plus worker **nodes** that run **Pods**. **Namespaces** partition objects inside one cluster (teams, environments, blast radius). Understand both before applying workloads.

Use this note when:

- Designing **dev / staging / prod** isolation on one cluster
- Choosing **labels and selectors** for Services and Deployments
- Explaining how the **API server** and **controllers** fit together
- Setting **ResourceQuota** / **LimitRange** per namespace (ops)

CLI: [[Commands/K8S — kubectl & Minikube]]. Overview: [[K8S]].

---

## Cluster vs Namespace

| | Cluster | Namespace |
| --- | --- | --- |
| Scope | Entire K8s installation | Slice inside cluster |
| Examples | EKS, GKE, Minikube | `default`, `staging`, `ml-production` |
| Objects | Nodes, PersistentVolumes (cluster-scoped) | Deployments, Services, most workloads |
| DNS | — | `svc.ns.svc.cluster.local` |

Most YAML sets `metadata.namespace`; omitting uses `default`.

---

## Namespace Manifest

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ml-production
  labels:
    env: production
    team: ml
```

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=ml-production
```

---

## Labels & Selectors

Labels are key/value tags; **selectors** tie controllers to Pods and Services to Pod endpoints.

```yaml
# Deployment excerpt
metadata:
  labels:
    app: api
    tier: backend
spec:
  selector:
    matchLabels:
      app: api
      tier: backend
  template:
    metadata:
      labels:
        app: api
        tier: backend
```

Service must match Pod labels:

```yaml
spec:
  selector:
    app: api
    tier: backend
```

Query with labels:

```bash
kubectl get pods -l app=api
kubectl get all -l env=staging
```

---

## Contexts (Multi-Cluster)

```bash
kubectl config get-contexts
kubectl config use-context minikube
kubectl config set-context --current --namespace=dev
```

`~/.kube/config` holds clusters, users, and contexts — treat like credentials.

---

## Resource Quota (Optional)

Limit namespace consumption:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: staging
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.memory: 16Gi
```

---

## Node Overview

```bash
kubectl get nodes -o wide
kubectl describe node minikube
```

| Node condition | Meaning |
| --- | --- |
| Ready | kubelet healthy |
| MemoryPressure / DiskPressure | Scheduling constrained |

Pods land on nodes via **scheduler**; you can pin with `nodeSelector`, `affinity`, or **taints/tolerations** (advanced).

---

## Architecture Recap

```text
kubectl → API server → etcd (state)
                    → controllers (Deployment, etc.)
                    → scheduler → assign Pod to Node
Node: kubelet starts Pod containers
```

---

## Patterns in This Vault

| Pattern | Namespace example |
| --- | --- |
| [[ML — Seldon]] inference | `ml-production` |
| [[ML — MLflow]] tracking server | `ml-platform` |
| [[API - FastAPI]] staging API | `staging` |
| [[Processing — Celery]] workers | `workers` (or off-cluster) |

---

## Quick Reference

| Task | YAML / action |
| --- | --- |
| Create namespace | `kind: Namespace` |
| Set default ns | `kubectl config set-context --current --namespace=...` |
| Label pod template | `template.metadata.labels` |
| Select pods | `spec.selector.matchLabels` |
| List by label | `kubectl get pods -l key=value` |

Commands cheat sheet: [[Commands/K8S — kubectl & Minikube]].

---

## Related Notes

- [[K8S]]
- [[Codes/K8S — Workloads]]
- [[Codes/K8S — Networking]]
- [[Commands/K8S — kubectl & Minikube]]
- [[ML — Seldon]]

---

## Tags

#kubernetes #k8s #namespace #cluster #labels #yaml
