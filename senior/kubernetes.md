# Kubernetes (K8s) — Developer Perspective

## What it is
Kubernetes is a container orchestration platform. It runs your Docker containers, keeps them alive, scales them up/down, and handles deployments. You describe what you want — K8s makes it happen.

**As a developer:** you don't manage K8s clusters (DevOps does), but you must understand what it does and design your app to work with it.

---

## The Problem It Solves

Without K8s, deploying 5 services means:
- Manually SSH into servers
- Run Docker containers by hand
- Restart crashed containers manually
- Scale by hand when traffic spikes

With K8s:
- Declare "I want 3 instances of Products API" → K8s runs them
- One crashes → K8s restarts it automatically
- Traffic spikes → K8s scales to 10 instances automatically
- Deploy new version → K8s does a rolling update (zero downtime)

---

## Core Concepts

### Pod
The smallest deployable unit. Usually wraps one Docker container.

```yaml
# A Pod running your Products API
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: products-api
      image: yourrepo/products-api:1.0.0
      ports:
        - containerPort: 8080
```

You rarely create Pods directly — you use Deployments.

---

### Deployment
Manages a set of identical Pods. Handles rolling updates, rollbacks.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: products-api
spec:
  replicas: 3              # run 3 instances
  selector:
    matchLabels:
      app: products-api
  template:
    spec:
      containers:
        - name: products-api
          image: yourrepo/products-api:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: connectionString
          livenessProbe:        # K8s calls this to check if container is alive
            httpGet:
              path: /health/live
              port: 8080
          readinessProbe:       # K8s calls this before routing traffic
            httpGet:
              path: /health/ready
              port: 8080
```

This is why your `/health/live` and `/health/ready` endpoints matter — K8s calls them.

---

### Service
A stable network address for a set of Pods. Pods come and go, their IPs change — a Service gives a fixed DNS name.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: products-api
spec:
  selector:
    app: products-api      # routes to all pods with this label
  ports:
    - port: 80
      targetPort: 8080
```

Other services call `http://products-api/api/products` — K8s routes to one of the 3 pods.

---

### Ingress
The entry point from the internet into the cluster. Works like an API Gateway / Load Balancer at the edge.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
spec:
  rules:
    - host: api.yourapp.com
      http:
        paths:
          - path: /api/products
            backend:
              service:
                name: products-api
                port:
                  number: 80
          - path: /api/orders
            backend:
              service:
                name: orders-api
                port:
                  number: 80
```

---

### ConfigMap & Secret
Inject configuration and secrets into containers without hardcoding.

```yaml
# ConfigMap — non-sensitive config
apiVersion: v1
kind: ConfigMap
data:
  ASPNETCORE_ENVIRONMENT: "Production"
  Redis__Host: "redis-service"

# Secret — sensitive values (base64 encoded)
apiVersion: v1
kind: Secret
data:
  ConnectionString: base64encodedvalue==
```

---

## What Kubernetes Gives You Automatically

| Feature | How |
|---|---|
| **Self-healing** | Crashed pod → K8s restarts it |
| **Scaling** | `kubectl scale deployment products-api --replicas=10` |
| **Zero-downtime deploys** | Rolling update — new pods start before old ones stop |
| **Rollback** | `kubectl rollout undo deployment/products-api` |
| **Service discovery** | Services addressable by name within cluster |
| **Health monitoring** | liveness + readiness probes |
| **Secret management** | Secrets injected as env vars, never in code |

---

## What Your App Must Do for K8s Compatibility

1. **Stateless** — no in-memory state, sessions in Redis not server memory
2. **Health endpoints** — `/health/live` and `/health/ready`
3. **Config via environment variables** — not hardcoded, not only appsettings.json
4. **Graceful shutdown** — handle `SIGTERM`, finish in-flight requests before stopping
5. **Structured logs to stdout** — K8s collects stdout, not file logs
6. **Dockerized** — needs a `Dockerfile`

```csharp
// Graceful shutdown in .NET — handle in-flight requests
builder.Services.Configure<HostOptions>(options =>
    options.ShutdownTimeout = TimeSpan.FromSeconds(30));
```

---

## Basic kubectl Commands (you'll use these)

```bash
kubectl get pods                          # list running pods
kubectl get deployments                   # list deployments
kubectl logs products-api-abc123          # view pod logs
kubectl describe pod products-api-abc123  # debug a pod
kubectl scale deployment products-api --replicas=5
kubectl rollout status deployment/products-api
kubectl rollout undo deployment/products-api  # rollback
kubectl apply -f deployment.yaml          # deploy/update
kubectl delete -f deployment.yaml         # tear down
```

---

## Local Development with K8s

**Docker Compose** — simpler, used for local dev:
```yaml
# docker-compose.yml
services:
  products-api:
    build: ./Products.Api
    ports: ["8080:8080"]
  orders-api:
    build: ./Orders.Api
    ports: ["8081:8080"]
  redis:
    image: redis:alpine
  sqlserver:
    image: mcr.microsoft.com/mssql/server:2022-latest
```

**minikube / k3d** — run a real K8s cluster locally for testing K8s-specific features.

For most development: Docker Compose is enough. Use K8s config for staging/production.

---

## Common Interview Questions

1. What problem does Kubernetes solve?
2. What is the difference between a Pod, Deployment, and Service?
3. What is a liveness probe vs readiness probe?
4. Why must apps be stateless to run on Kubernetes?
5. How does K8s do zero-downtime deployments?
6. What is the difference between a ConfigMap and a Secret?

---

## Common Mistakes

- Storing session state in memory (breaks when K8s routes to different pod)
- Not implementing health endpoints (K8s can't detect unhealthy pods)
- Hardcoding connection strings (use env vars / Secrets)
- Writing logs to files instead of stdout
- Not handling graceful shutdown (in-flight requests killed mid-processing)

---

## How It Connects

- Your `/health/live` and `/health/ready` endpoints (from observability.md) are called by K8s probes
- Docker (from docker.md) is the foundation — K8s runs Docker containers
- Stateless design (from system-design.md) is required for K8s scaling
- ConfigMaps + Secrets replace `appsettings.json` for environment-specific config
- CI/CD pipeline (from ci-cd.md) builds Docker image → pushes to registry → `kubectl apply`

---

## My Confidence Level
- `[ ]` Pod vs Deployment vs Service vs Ingress
- `[ ]` liveness vs readiness probes
- `[ ]` Stateless app requirements for K8s
- `[ ]` ConfigMap and Secret for configuration
- `[ ]` Rolling updates and rollbacks
- `[ ]` Basic kubectl commands
- `[ ]` Docker Compose for local dev vs K8s for production

## My Notes
<!-- Personal notes -->
