# Kubernetes Basics Tutorial

Using [LiteLLM Proxy](https://docs.litellm.ai/docs/proxy/deploy) as a real-world example.

## Prerequisites

- `kubectl` installed
- `minikube` installed ([install guide](https://minikube.sigs.k8s.io/docs/start/))
- Docker (or another container runtime) installed and running

## Starting the cluster with Minikube

```bash
# Start a local single-node Kubernetes cluster
minikube start

# Verify the cluster is running
minikube status

# kubectl should now point to your minikube cluster
kubectl cluster-info

# Enable the metrics-server addon (needed for HPA in step 6)
minikube addons enable metrics-server

# (Optional) Open the Kubernetes dashboard in your browser
minikube dashboard
```

If you want more resources for the cluster:
```bash
minikube start --cpus=4 --memory=4096
```

## Concepts covered

| # | Concept | File | What it teaches |
|---|---------|------|-----------------|
| 1 | Namespace | `01-namespace.yaml` | Logical isolation |
| 2 | Secret | `02-secret.yaml` | Storing sensitive data |
| 3 | Deployment | `03-deployment.yaml` | Running pods, replicas, rolling updates |
| 4 | Service | `04-service.yaml` | Networking & exposing pods |
| 5 | HPA | `05-hpa.yaml` | Autoscaling based on CPU/memory |

## Walkthrough

### Step 1 — Create a Namespace

```bash
kubectl apply -f 01-namespace.yaml
kubectl get namespaces
```

A **Namespace** is like a folder — it groups related resources and keeps them isolated.

---

### Step 2 — Create a Secret

```bash
kubectl apply -f 02-secret.yaml
kubectl get secret -n litellm
```

A **Secret** is like a ConfigMap but for sensitive data. Values are base64-encoded (not encrypted by default — just obscured).

To encode your own value:
```bash
echo -n "my-secret-value" | base64
```

---

### Step 3 — Create a Deployment

```bash
kubectl apply -f 03-deployment.yaml
```

A **Deployment** manages a set of identical pods. Key things to explore:

```bash
# See the deployment
kubectl get deployment -n litellm

# See the pods it created
kubectl get pods -n litellm

# See detailed info (events, conditions)
kubectl describe deployment litellm -n litellm

# Check logs of a pod
kubectl logs -n litellm -l app=litellm --tail=50

# Scale manually (change replicas)
kubectl scale deployment litellm -n litellm --replicas=5
kubectl get pods -n litellm -w   # watch pods come up

# Scale back down
kubectl scale deployment litellm -n litellm --replicas=2
```

**Key concepts in the Deployment:**
- `replicas` — how many identical pods to run
- `selector.matchLabels` — how the deployment finds its pods
- `template` — the pod blueprint (container image, ports, env vars, volumes)
- `livenessProbe` — K8s restarts the pod if this fails
- `readinessProbe` — K8s stops sending traffic if this fails
- `resources` — CPU/memory requests and limits

---

### Step 4 — Create a Service

```bash
kubectl apply -f 04-service.yaml
kubectl get svc -n litellm
```

A **Service** gives pods a stable network identity. Pods come and go, but the service stays.

```bash
# If using minikube, get the URL:
minikube service litellm-service -n litellm --url

# Or port-forward to test locally:
kubectl port-forward svc/litellm-service -n litellm 4000:4000

# Then test:
curl http://localhost:4000/health/readiness
```

**Service types:**
- `ClusterIP` (default) — only reachable inside the cluster
- `NodePort` — exposes on each node's IP at a static port
- `LoadBalancer` — provisions a cloud load balancer (GKE/EKS/AKS)

---

### Step 5 — Autoscaling with HPA

```bash
kubectl apply -f 05-hpa.yaml
kubectl get hpa -n litellm
```

A **HorizontalPodAutoscaler** watches metrics and adjusts `replicas` automatically.

```bash
# Watch it in action
kubectl get hpa -n litellm -w

# Simulate load (from another terminal, after port-forwarding)
for i in $(seq 1 1000); do curl -s http://localhost:4000/health/readiness > /dev/null & done
```

---

## Useful debugging commands

```bash
# Get everything in the namespace
kubectl get all -n litellm

# Describe a pod (shows events, reasons for failures)
kubectl describe pod <pod-name> -n litellm

# Exec into a running pod
kubectl exec -it <pod-name> -n litellm -- /bin/sh

# Delete everything and start over
kubectl delete namespace litellm
```

## Cleanup

```bash
kubectl delete -f . -n litellm
# or simply:
kubectl delete namespace litellm
```
