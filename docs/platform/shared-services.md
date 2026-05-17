# Shared Services

**Repository**: `shared-services`

The Spring infrastructure adapter layer. Implements all `shared-kernel` contracts as Spring auto-configuration modules.

---

## What Lives Here

`shared-services` owns all runtime adapters. Services consume these modules through the starter — they do not reimplement them.

| Module | Role |
|---|---|
| `infrastructure-starter` | Auto-wires all selected modules |
| `infrastructure-context` | `ExecutionContextFactory`, `ContextHttpFilter`, `SecurityIdentityFilter` |
| `infrastructure-messaging` | Kafka integration event consumer/producer |
| `infrastructure-outbox` | Outbox table, worker, cleanup scheduler |
| `infrastructure-inbox` | Inbox table, worker, cleanup scheduler |
| `infrastructure-saga` | `SagaAutoConfiguration`, step dispatch |
| `infrastructure-grpc` | gRPC server wiring, interceptors, dispatcher |
| `infrastructure-worker` | `DefaultWorkerLoopEngine`, `DefaultSchedulerEngine` |
| `infrastructure-observability` | `IMetricsRecorder`, `ITracer`, `AlertDefinitions`, `MetricNames` |
| `infrastructure-resilience` | Resilience4j policy bindings |
| `infrastructure-redis` | Redis adapter |
| `infrastructure-jpa` | `EntityManager` base — default persistence pattern |
| `infrastructure-resource` | `IResourceStore` implementations (LOCAL, S3, Azure) |
| `infrastructure-search` | Elasticsearch adapter |
| `infrastructure-idempotency` | Idempotency behavior (persistence or Redis backed) |
| `infrastructure-audit` | Audit emission |
| `infrastructure-mediator` | Mediator/pipeline execution |
| `infrastructure-security` | Security adapter |
| `infrastructure-architecture-tests` | ArchUnit rules, ADRs, guardrails, scaffolding |

---

## Key Rules

- `shared-services` implements kernel ports — it does not redefine kernel semantics
- `JpaRepository` is a service-specific exception — platform default is `EntityManager + @Bean`
- Services consume modules through starter; they must not bypass bus/pipeline boundaries
- Configuration roots: `platform.*`, `messaging`, `app.storage`, `worker.executor`, `search`, `observability`, `resilience`, `redis`

---

## Architecture Enforcement

```bash
# Run all platform architecture tests
mvn test -pl shared-services-infrastructure-architecture-tests
```

All enforced rules must pass. Missing template modules, missing YAML roots, and missing documentation are also policy violations — not just Java package boundaries.
