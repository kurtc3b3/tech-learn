## What & When

This note is the **CLI reference** for **kubectl** (cluster control) and **Minikube** (local single-node cluster). For object model and YAML, see **Codes/K8S —** notes and [[K8S]].

Use when:

- **First-time setup** on a laptop
- **Day-2 debugging** — logs, exec, port-forward
- **Applying manifests** from [[Codes/K8S — Workloads]] and related notes

---

## Install

```bash
# macOS
brew install kubectl minikube

# Verify
kubectl version --client
minikube version
```

Optional: `kubectx` / `kubens` for fast context and namespace switching.

---

## Minikube — Local Cluster

```bash
minikube start
minikube start --cpus=4 --memory=8192 --driver=docker

minikube status
minikube stop
minikube delete
```

Useful addons:

```bash
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons list
```

Dashboard (optional):

```bash
minikube dashboard
```

### Access services locally

```bash
# Open tunnel for LoadBalancer Services
minikube tunnel

# Or port-forward a Service
kubectl port-forward svc/my-service 8080:80
```

Get Minikube IP:

```bash
minikube ip
```

---

## kubectl Context & Namespace

```bash
kubectl config current-context
kubectl config get-contexts
kubectl config use-context minikube

kubectl get ns
kubectl config set-context --current --namespace=dev
kubectl get pods -A          # all namespaces
```

---

## Discover Resources

```bash
kubectl get nodes
kubectl get all
kubectl get all -n ml-production

kubectl get deploy,rs,pods,svc,ingress
kubectl get cm,secret,sa,pvc,sc
kubectl get jobs,cronjobs,sts

kubectl api-resources
kubectl explain deployment.spec
```

---

## Apply & Delete

```bash
kubectl apply -f deployment.yaml
kubectl apply -f ./manifests/
kubectl delete -f deployment.yaml
kubectl delete deployment fastapi-api
```

Dry run:

```bash
kubectl apply -f deployment.yaml --dry-run=client
kubectl apply -f deployment.yaml --dry-run=server
```

---

## Inspect & Debug

```bash
kubectl describe pod <pod-name>
kubectl describe deployment fastapi-api
kubectl describe svc fastapi-api
kubectl describe ingress api-ingress

kubectl logs <pod-name>
kubectl logs deployment/fastapi-api
kubectl logs <pod-name> -c sidecar -f --tail=100

kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it deployment/fastapi-api -- bash
```

Ephemeral debug pod:

```bash
kubectl run curl --rm -it --restart=Never --image=curlimages/curl -- sh
# inside: curl http://fastapi-api.default.svc.cluster.local
```

---

## Rollouts

```bash
kubectl rollout status deployment/fastapi-api
kubectl rollout history deployment/fastapi-api
kubectl rollout undo deployment/fastapi-api
kubectl rollout undo deployment/fastapi-api --to-revision=2

kubectl set image deployment/fastapi-api api=myregistry/api:1.3.0
kubectl scale deployment/fastapi-api --replicas=5
```

---

## Port Forward & Proxy

```bash
kubectl port-forward pod/<pod-name> 8000:8000
kubectl port-forward svc/fastapi-api 8080:80
kubectl port-forward svc/iris-default 8000:8000 -n ml-production

kubectl proxy
# http://127.0.0.1:8001/api/v1/namespaces/default/pods/
```

---

## Secrets & Config (CLI)

```bash
kubectl create configmap api-config \
  --from-literal=LOG_LEVEL=debug \
  --from-file=settings.yaml

kubectl create secret generic api-secrets \
  --from-literal=DATABASE_URL='postgresql://...' \
  --from-env-file=.env

kubectl get secret api-secrets -o yaml
kubectl get configmap api-config -o yaml
```

Never log decoded secrets in shared terminals.

---

## Labels & Selectors

```bash
kubectl get pods -l app=fastapi-api
kubectl get pods -l 'env in (staging,dev)'
kubectl label pod my-pod tier=backend --overwrite
```

---

## Events & Top

```bash
kubectl get events --sort-by='.lastTimestamp'
kubectl top nodes
kubectl top pods
```

Requires metrics-server (`minikube addons enable metrics-server`).

---

## Common Troubleshooting Commands

| Problem | Commands |
| --- | --- |
| Pod not starting | `kubectl describe pod`, `kubectl logs --previous` |
| Service unreachable | `kubectl get endpoints`, label check |
| Ingress 404 | `kubectl describe ingress`, controller pods |
| PVC pending | `kubectl describe pvc`, `kubectl get sc` |
| Wrong cluster | `kubectl config current-context` |
| Image pull fail | `kubectl describe pod` → Events |

---

## Cheat Sheet (One Page)

```bash
# Lifecycle
minikube start && kubectl cluster-info

# Deploy
kubectl apply -f .
kubectl get pods -w
kubectl logs -f deploy/myapp

# Expose locally
kubectl port-forward svc/myapp 8080:80

# Fix rollout
kubectl rollout undo deploy/myapp

# Cleanup
kubectl delete -f .
minikube stop
```

---

## Related Notes

- [[K8S]]
- [[Codes/K8S — Cluster & Namespaces]]
- [[Codes/K8S — Workloads]]
- [[Codes/K8S — Networking]]
- [[Codes/K8S — Storage]]
- [[Codes/K8S — Configuration & Security]]
- [[ML — Seldon]]

---

## Tags

#kubernetes #k8s #kubectl #minikube #cli #commands #devops #debug
