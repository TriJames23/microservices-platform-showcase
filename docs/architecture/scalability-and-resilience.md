# Scalability & Resilience

## Horizontal Pod Autoscaling

Services are configured for CPU-based HPA in Kubernetes:

```yaml
autoscaling:
  enabled: true
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 70
```

HPA is configured via Helm values in `micro-config-repo` per service per environment.

---

## Resilience4j Policy Model

Every runtime lane has a named Resilience4j policy bound to it. Policies are declared in `values.yaml` and wired by `shared-services-infrastructure-resilience`.

| Lane | Policy binding |
|---|---|
| HTTP pipeline | `pipeline` |
| Worker loop | `worker` |
| Outbox publisher | `outbox` |
| Inbox processor | `inbox` |
| Saga execution | `saga` |
| gRPC client | `grpc` |
| Kafka consumer | `messaging` |

**Blank policy name falls back** to default Resilience4j behavior — this is intentional for local sandbox.

---

## Worker Architecture

Two distinct engine types, never mixed (ADR-028):

| Engine | Pattern | Use case |
|---|---|---|
| `DefaultWorkerLoopEngine` | Message-processing loop | Outbox publisher, Inbox processor, Saga steps |
| `DefaultSchedulerEngine` | Fixed-delay periodic | Outbox cleanup, Inbox cleanup |

---

## DLQ and Error Handling

- Outbox workers retry on failure; exhausted messages route to DLQ
- Inbox processors provide transport-level deduplication
- Business idempotency is owned by the receiving service at the command boundary
- Saga compensation triggers on step failure — no inline direct rollback

See [ADR-016](../adr/ADR-016-dlq-and-error-handling.md) for DLQ policy details.

---

## Resource Limits

All services define explicit Kubernetes resource requests and limits:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 500m
    memory: 1Gi
```

Limits are enforced per environment via Helm values — production runs tighter constraints than dev.
