# Deployment & GitOps

## Delivery Model

Every service deploys independently. No shared deployment pipelines, no coordinated releases.

```
  Developer pushes → GitHub
         │
         ▼
  Jenkins (per-service CI)
  ├── Build & test
  ├── Architecture tests (ArchUnit)
  ├── Docker build (multi-stage, Java 25)
  ├── Image push → Registry (tagged by git-sha)
  └── Update config-repo values.yaml (one service only)
         │
         ▼
  ArgoCD detects config-repo change
         │
         ▼
  Kubernetes Deployment (one application per service per environment)
         │
         ▼
  /actuator/health · /actuator/prometheus → Prometheus scrape
```

---

## Config Repo Structure

All Helm values are centralized in `micro-config-repo`, isolated per service and per environment:

```
micro-config-repo/
└── environments/
    ├── dev/
    │   ├── identity-service/
    │   │   └── values.yaml
    │   ├── order-service/
    │   │   └── values.yaml
    │   └── ...
    └── prod/
        └── ...
```

Jenkins updates **exactly one** `values.yaml` per build — the service that changed. No cross-service coupling.

---

## ArgoCD Application Model

One ArgoCD application per service per environment:

```yaml
# argocd/dev/identity-service.yaml
sources:
  - repoURL: <platform-helm-charts>    # shared chart
    chart: microservice
  - repoURL: <micro-config-repo>       # service-specific values
    path: environments/dev/identity-service
```

ArgoCD syncs automatically when the config-repo changes. Each service has its own sync policy, health checks, and rollback window.

---

## Kubernetes Runtime

| Resource | Configuration |
|---|---|
| Deployment | Rolling update strategy |
| HPA | CPU-based autoscaling (min 1, max N replicas) |
| Resource limits | Per-service `requests` and `limits` |
| Health probes | `/actuator/health` — liveness + readiness |
| Metrics endpoint | `/actuator/prometheus` — Prometheus scrape |

---

## Local Sandbox Defaults

Services run in standalone mode locally with safe defaults:

```yaml
platform:
  grpc.server.enabled: false
  saga.enabled: false
  outbox.enabled: true
  inbox.enabled: true
  outbox.commands.enabled: false
  inbox.commands.enabled: false
  idempotency.provider: persistence

redis.enabled: false
app.storage.provider: LOCAL
messaging.broker: KAFKA
```

---

## Helm Chart

`platform-helm-charts` provides the shared Helm chart consumed by all services. Services do not manage their own chart — they supply only a `values.yaml` override.
