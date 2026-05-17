# Observability Architecture

## Design Principle

Observability is a **passive infrastructure concern**. Services emit intent; the platform records it.

- Services never configure Prometheus or Alertmanager
- Services never define raw metric names
- Services never send alerts directly to Slack

---

## Metrics

Services emit metrics through **semantic interfaces**, not raw recorder calls:

| Interface | Lane |
|---|---|
| `IOutboxMetrics` | Outbox publish backlog, success, failure |
| `IInboxMetrics` | Inbox ingestion, processing, failure |
| `ISagaMetrics` | Saga started, completed, failed, compensated, duration |
| `IGrpcMetrics` | gRPC request latency, error rate |

Raw metric names are centralized in `MetricNames`. Alert expressions are centralized in `AlertDefinitions`.

**`IMetricsRecorder`** is the generic adapter boundary — it lives inside `shared-services-infrastructure-observability` and must not be injected into application or domain code.

---

## Tagging

Low-cardinality tags are applied via `withCommonTags(...)`:

| Tag | Value |
|---|---|
| `service` | Service name |
| `tenantId` | Tenant identifier |

High-cardinality values (`correlationId`, `traceId`) are for **tracing/logging only** — not default metric tags.

---

## Tracing

`ITracer` and `ITraceSpan` instrument:
- Saga execution steps
- Outbox publish
- Inbox processing
- gRPC server/client boundaries

Recommended span names for the commerce flow:

```
checkout.start → catalog.price.check → inventory.availability.check
→ order.create → inventory.reserve → payment.process
→ inventory.release → order.complete / order.fail
```

---

## Alert Delivery

Alert delivery is mandatory and must not terminate inside the observability stack.

```
Service emits IOutboxMetrics / IInboxMetrics / ISagaMetrics / IGrpcMetrics
        │
        ▼
Prometheus evaluates alert rules (AlertDefinitions)
        │
        ▼
Alertmanager routes by: service · severity · environment
        │
        ▼
Slack channels (mandatory human response interface)

  #svc-<service>    → service-level alerts
  #incident-prod    → production incidents
  #deployments      → CI/CD notifications
  #infra-observability → platform alerts
```

---

## Stack

| Component | Role |
|---|---|
| Prometheus | Metric collection and alert evaluation |
| Alertmanager | Alert routing and deduplication |
| Grafana | Dashboard visualization |
| Micrometer | JVM bridge to Prometheus |
| Spring Actuator | `/actuator/health`, `/actuator/info`, `/actuator/prometheus` |

The observability stack is **shared platform runtime** — not deployed per service.
