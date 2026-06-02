## What & When

**PersistentVolume (PV)** is cluster storage; **PersistentVolumeClaim (PVC)** is a Pod's storage request; **StorageClass** provisions PVs dynamically (EBS, GCE PD, local-path on Minikube).

Use this note when:

- **StatefulSet** or Deployment needs durable disk
- Binding volumes in **Minikube** vs cloud
- Sizing storage for databases, ML artifact caches, or log buffers

CLI: [[Commands/K8S — kubectl & Minikube]]. Overview: [[K8S]].

---

## StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/gce-pd   # cloud-specific
parameters:
  type: pd-ssd
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Minikube default often `standard` — check:

```bash
kubectl get storageclass
```

---

## PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mlflow-artifacts
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: standard
  resources:
    requests:
      storage: 20Gi
```

| accessMode | Meaning |
| --- | --- |
| ReadWriteOnce (RWO) | One node read/write |
| ReadOnlyMany (ROX) | Many nodes read |
| ReadWriteMany (RWX) | Many nodes read/write (NFS, EFS) |

---

## Mount in Pod / Deployment

```yaml
spec:
  containers:
    - name: mlflow
      image: ghcr.io/mlflow/mlflow:latest
      volumeMounts:
        - name: artifacts
          mountPath: /mlflow/artifacts
  volumes:
    - name: artifacts
      persistentVolumeClaim:
        claimName: mlflow-artifacts
```

StatefulSet uses `volumeClaimTemplates` instead — see [[Codes/K8S — Workloads]].

---

## PV Lifecycle

```text
PVC created → StorageClass provisions PV → Bound
Pod uses PVC → Released on delete → Reclaim (Retain / Delete / Recycle)
```

```bash
kubectl get pvc,pv
kubectl describe pvc mlflow-artifacts
```

---

## EmptyDir (Non-Persistent)

Scratch space — lost when Pod dies:

```yaml
volumes:
  - name: cache
    emptyDir: {}
```

Use for temp files; not for databases.

---

## Config vs Storage

| Need | Use |
| --- | --- |
| Config files, non-secret env | [[Codes/K8S — Configuration & Security]] ConfigMap |
| Passwords | Secret |
| Database files, models, logs | PVC |

[[ML — MLflow]] artifact store: prefer **S3/MinIO** in production; PVC acceptable for Minikube demos.

---

## Cloud vs Local

| Environment | Typical StorageClass |
| --- | --- |
| Minikube | `standard`, `csi-hostpath` |
| EKS | `gp3` (EBS) |
| GKE | `pd-standard`, `pd-ssd` |

---

## Quick Reference

| Task | Object |
| --- | --- |
| Dynamic disk | StorageClass + PVC |
| Mount in Pod | `persistentVolumeClaim.claimName` |
| Per-replica disk | StatefulSet `volumeClaimTemplates` |
| Ephemeral scratch | `emptyDir` |
| Check binding | `kubectl get pvc` → STATUS Bound |

---

## Related Notes

- [[K8S]]
- [[Codes/K8S — Workloads]]
- [[Codes/K8S — Configuration & Security]]
- [[Commands/K8S — kubectl & Minikube]]
- [[ML — MLflow]]

---

## Tags

#kubernetes #k8s #pvc #storageclass #pv #storage #yaml
