# Kubernetes Runtime

**Cluster**: Docker Desktop Kubernetes (dev), production-equivalent topology.

---

## Service Deployment Model

Each service is an independent Kubernetes `Deployment`. No cross-service shared deployments.

```
Kubernetes Cluster
├── Namespace: default (services)
│   ├── identity-service        Deployment + Service + HPA
│   ├── auth-service            Deployment + Service + HPA
│   ├── catalog-service         Deployment + Service + HPA
│   ├── order-service           Deployment + Service + HPA
│   ├── inventory-service       Deployment + Service + HPA
│   ├── payment-service         Deployment + Service + HPA
│   ├── cart-service            Deployment + Service + HPA
│   ├── notification-service    Deployment + Service + HPA
│   └── shop-service            Deployment + Service + HPA
│
├── Namespace: argocd
│   └── argocd-server           GitOps controller
│
└── Namespace: monitoring
    ├── prometheus               Metric collection
    ├── alertmanager             Alert routing
    └── grafana                  Dashboard visualization
```

---

## Resource Configuration (per service)

Configured via Helm `values.yaml` per environment:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

---

## Horizontal Pod Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

HPA triggers when CPU utilization exceeds 70% of the requested value.

---

## Health Probes

All services expose Spring Actuator endpoints:

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 20
```

---

## Service Ports

| Service | HTTP Port | gRPC Port |
|---|---|---|
| identity-service | 8080 | 9090 |
| auth-service | 8081 | 9091 |
| catalog-service | 8082 | — |
| order-service | 8083 | — |
| inventory-service | 8084 | 9094 |
| payment-service | 8085 | — |
| notification-service | 8086 | 9096 |

---

## Shared Infrastructure

| Component | Namespace | Port-forward (dev) |
|---|---|---|
| ArgoCD | `argocd` | `8090:443` |
| Grafana | `monitoring` | `3000:80` |
| Prometheus | `monitoring` | `9090:9090` |

---

## Helm Chart

All services consume `platform-helm-charts`. Services supply only their `values.yaml` — they do not manage their own chart templates.

```bash
# ArgoCD watches and syncs automatically
# Manual sync:
argocd app sync identity-service-dev
```
