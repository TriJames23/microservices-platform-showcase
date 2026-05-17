# GitOps Flow

**Tool**: ArgoCD  
**Config Repo**: `micro-config-repo`  
**Chart Repo**: `platform-helm-charts`

---

## Core Principle

Config repo is the **single source of truth** for what runs in each environment.

- No manual `kubectl apply`
- No ad-hoc Helm installs
- Every deployed state is traceable to a git commit in `micro-config-repo`

---

## Config Repo Structure

```
micro-config-repo/
└── environments/
    ├── dev/
    │   ├── identity-service/
    │   │   └── values.yaml
    │   ├── auth-service/
    │   │   └── values.yaml
    │   ├── catalog-service/
    │   │   └── values.yaml
    │   ├── order-service/
    │   │   └── values.yaml
    │   ├── inventory-service/
    │   │   └── values.yaml
    │   ├── payment-service/
    │   │   └── values.yaml
    │   ├── cart-service/
    │   │   └── values.yaml
    │   ├── notification-service/
    │   │   └── values.yaml
    │   └── shop-service/
    │       └── values.yaml
    ├── stg/
    │   └── ...
    └── prod/
        └── ...
```

One folder = one service per environment. Cross-service overrides are **forbidden**.

---

## ArgoCD Application Model

One ArgoCD application per service per environment:

```yaml
# argocd/dev/identity-service.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: identity-service-dev
  namespace: argocd
spec:
  sources:
    - repoURL: https://github.com/org/platform-helm-charts
      chart: microservice
      targetRevision: 1.0.0
    - repoURL: https://github.com/org/micro-config-repo
      targetRevision: HEAD
      ref: values
  helm:
    valueFiles:
      - $values/environments/dev/identity-service/values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## Values File Structure

```yaml
# environments/dev/identity-service/values.yaml
image:
  repository: registry.example.com/identity-service
  tag: "abc1234"    # ← updated by Jenkins on each build

replicaCount: 1

service:
  port: 8080

autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 3

resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi

env:
  SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/identity
  SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
  PLATFORM_OUTBOX_ENABLED: "true"
  PLATFORM_INBOX_ENABLED: "true"
```

---

## Sync Flow

```
Jenkins pushes new image tag to values.yaml
       │
       ▼
micro-config-repo receives commit
       │
       ▼
ArgoCD detects out-of-sync state (polls config repo every 3 minutes)
       │
       ▼
ArgoCD syncs:
  - pulls shared chart from platform-helm-charts
  - applies service-specific values from micro-config-repo
  - renders Helm templates
  - applies to Kubernetes cluster
       │
       ▼
Kubernetes performs rolling update
       │
       ▼
New pods pass readiness probe (/actuator/health)
       │
       ▼
Old pods terminated
```

---

## Health Verification

ArgoCD monitors application health via:
- Kubernetes Deployment rollout status
- Pod readiness (Spring Actuator `/actuator/health/readiness`)
- Prometheus scrape availability

Degraded health triggers ArgoCD alert and blocks further automated rollout.

---

## Rollback

Because every state is a git commit:

```bash
# Rollback by reverting the config-repo commit
git revert HEAD --no-edit
git push

# ArgoCD auto-syncs to the reverted state
```

No manual intervention required — ArgoCD self-heals to the config repo state.
