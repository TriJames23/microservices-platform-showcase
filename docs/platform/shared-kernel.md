# Shared Kernel

**Repository**: `shared-kernel`

The framework-agnostic source of truth for domain, application, and contract abstractions shared by all services.

---

## What Lives Here

`shared-kernel` defines the contracts that all runtime lanes execute against. It contains no framework code.

### Domain Contracts
- `IAggregateRepository<T, TId>` — command-side aggregate persistence semantic base
- `IReadRepository<TProjection, TId>` — query/read-model access semantic base
- Value objects, `Result<T>`, domain events

### Application Contracts
- `ICommandBus`, `IQueryBus` — transport-neutral entrypoints for all use cases
- `IExecutionContext` — caller identity, tenant, correlation, trace
- `IStructuredLogger` — kernel-facing logging abstraction
- `ICommandDispatcher` — infrastructure abstraction for cross-service dispatch
- `IIntegrationCommandTypeRegistry` — receiving-side registry for integration commands

### Saga Contracts
- `ISagaDefinition` — saga state-machine
- `ISagaTransition` — next-step resolution
- `ISagaStepHandler` — local step execution
- `ISagaCorrelationResolver` — event-to-saga correlation
- `ISagaContext` — execution context for saga steps

### Resource Contracts
- `IResourceStore` — byte-level provider/storage ownership
- `IResourceAccessTokenService` — presigned access-token issuance
- `IResourcesService` — logical resource lifecycle

---

## What Is Forbidden Here

| Forbidden | Why |
|---|---|
| Spring / JPA / Jackson / Lombok | Framework leakage into domain |
| HTTP/gRPC/storage semantics | Transport coupling |
| Metrics, alert expressions, tracing runtime | Belongs to `shared-services` |
| Service-specific adapter code | Wrong layer |

Violations are enforced by ArchUnit — **build failure** on violation.

---

## Module Structure

```
shared-kernel/
├── shared-kernel-domain/       # Value objects, aggregates, domain events
├── shared-kernel-application/  # Ports, commands, queries, CQRS contracts
└── shared-kernel-contract/     # Integration event and command contracts
```
