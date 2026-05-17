# Execution Context & Auditability

## `IExecutionContext`

The single read abstraction for caller identity, tenant, correlation, and trace continuity across all runtime lanes.

```java
interface IExecutionContext {
    String getSubject();       // authenticated caller identity
    String getTenantId();      // tenant scope
    String getCorrelationId(); // request correlation
    String getTraceId();       // distributed trace
}
```

`IExecutionContext` is a kernel contract. It must not be used as a manual authorization engine inside handlers.

---

## Header Propagation

```
Inbound HTTP Request
  Headers: X-Tenant-Id · X-Correlation-Id · X-Trace-Id
        │
        ▼
  ContextHttpFilter
  (builds base execution context from headers)
        │
        ▼
  SecurityIdentityFilter
  (enriches trusted identity from verified JWT — does NOT trust spoofable headers)
        │
        ▼
  ExecutionContextHolder (thread-local)
        │
        ▼
  Available to: handlers · audit · observability · messaging adapters
```

---

## Kafka Boundary

Kafka listeners build execution context explicitly at the transport boundary:

```
KafkaIntegrationEventConsumer
  → ExecutionContextFactory.build(headers)
  → ExecutionContextHolder.set(context)
  → [process inbox / business logic]
  → ExecutionContextHolder.clear()  ← always in finally block
```

Context is set and cleared explicitly — not delegated to Spring messaging interceptors.

---

## Audit Trail

`AuditAutoConfiguration` and `ObservabilityAutoConfiguration` consume `IExecutionContext` for:
- Audit record emission (who, what, when, tenant, correlation)
- Structured log enrichment (`IStructuredLogger`)
- Observability tag propagation (`service`, `tenantId`)

---

## Key Rules

| Rule | Enforcement |
|---|---|
| Handlers use `IExecutionContext` for context access only | ArchUnit |
| Manual role checks through context forbidden | ArchUnit |
| Identity enriched from verified credentials, not headers | Security filter chain |
| Context cleared in `finally` at Kafka boundary | Code contract |
