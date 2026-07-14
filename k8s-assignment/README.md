# Kubernetes Microservices Monitoring Assignment

A queue-based, CPU-intensive Node.js microservices system deployed on
Kubernetes, autoscaled with HPA and monitored with Prometheus + Grafana.

## Architecture

| Service | Role | Exposure |
|---|---|---|
| **Service A** — Job Submitter | REST API (`/submit`, `/status/:id`); pushes jobs into Redis queue | Ingress + LoadBalancer |
| **Service B** — Worker | Consumes jobs from Redis; runs CPU-intensive work (primes, bcrypt, sort); exposes `/metrics` | ClusterIP (scaled by HPA, 2→10 pods) |
| **Service C** — Stats/Aggregator | Exposes `/stats` and `/metrics` (job counts, avg time, queue length) | ClusterIP |
| **Redis** | Job queue + result store | ClusterIP |

## Project Structure

```
k8s-assignment/
├── services/             # Node.js source
│   ├── service-a/
│   ├── service-b/
│   └── service-c/
├── k8s/                  # Kubernetes manifests (the deliverable)
│   ├── configmap.yaml
│   ├── redis/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── service-a/
│   ├── service-b/
│   ├── service-c/
│   └── monitoring/
├── scratch/               # throwaway test manifests, not part of the deliverable
└── README.md
```

## Prerequisites

- `kubectl`
- A local cluster: Minikube or Kind
- `helm` (for Prometheus/Grafana later)

## Configuration

All services read shared config from the `app-config` ConfigMap
(`k8s/configmap.yaml`):

| Key | Value | Used for |
|---|---|---|
| `REDIS_HOST` | `redis-svc` | Hostname services use to reach Redis |
| `REDIS_PORT` | `6379` | Redis port |
| `QUEUE_NAME` | `jobs` | Name of the Redis queue/list |

> ConfigMap values are copied into a pod's environment only at pod
> creation — they are not live-linked. After editing `configmap.yaml`
> and re-applying, existing pods must be recycled to pick up the
> change:
> ```bash
> kubectl rollout restart deployment <deployment-name>
> ```

## Deployment

Apply manifests in this order.

### 1. Shared config

```bash
kubectl apply -f k8s/configmap.yaml
```

### 2. Redis

```bash
kubectl apply -f k8s/redis/
kubectl get pods
kubectl get svc redis-svc
kubectl get endpoints redis-svc
```

Verify Redis is reachable by its cluster DNS name:

```bash
kubectl run redis-test --image=redis:7-alpine --restart=Never --rm -it -- redis-cli -h redis-svc ping

# expect: PONG
```

### 3. Service A / B / C

_TBD — added once the Node.js services and their manifests are built._

### 4. Horizontal Pod Autoscaler (Service B)

_TBD — target: scale 2 → 10 pods when CPU > 70%._

### 5. Prometheus + Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install prometheus prometheus-community/kube-prometheus-stack
```

_TBD — scrape config for Service B/C `/metrics`, Grafana dashboards._

## Stress Testing

_TBD — once Service A is deployed and exposed via Ingress:_

```bash
ab -n 5000 -c 200 http://<ingress-ip>/submit
```

Watch: Redis queue backlog, HPA scaling Service B, Grafana dashboards
(CPU, job counts, queue length), backlog draining as replicas increase.

## Observations on Scaling

_TBD — filled in after stress testing._

## Troubleshooting Notes

- **ConfigMap edit not reflected in a running pod** — pods only read
  ConfigMap values at creation; run
  `kubectl rollout restart deployment <name>` to pick up changes.
- **Service exists but traffic goes nowhere** — check
  `kubectl get endpoints <service-name>`. An empty result means the
  Service's `selector` doesn't match any pod's labels (e.g. a typo);
  `kubectl apply` succeeds either way since the YAML itself is valid,
  it just isn't finding pods to route to.
