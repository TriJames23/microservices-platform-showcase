# ADR-010: Resilience Scope

**Status**: Accepted  
**Date**: 2026-Q1  
**Enforced By**: ArchUnit (Rule A) + Starter Wiring + E2E Tests

---

## Context

Retries wrapping transactions cause massive contention and rollback issues. When `@Transactional` is placed on
Application Services and wrapped by resilience patterns (retry, circuit breaker), it creates:

1. Transaction contention: each retry opens a new transaction, holding DB locks longer.
2. Rollback storms: circuit breaker half-open triggers repeated rollbacks.
3. Resource exhaustion: connection pool depletion from abandoned/retried work.
4. Distributed transaction coupling: saga compensation conflicts with local transaction semantics.

---

## Decision

Resilience mechanisms (Retry/CircuitBreaker/Timeout/Bulkhead) MUST NOT wrap database transactions or UnitOfWork boundaries.

### Constraints

* Application layer MUST NOT use `@Transactional`.
* Transaction boundaries are owned by Infrastructure adapters and/or Infrastructure pipeline behaviors.
* Resilience should be applied at:
  * edges (external IO calls)
  * saga step execution (where retry is meaningful and must not reopen a DB transaction incorrectly)

---

## Where Resilience Is Applied (Current Platform)

| Boundary | Mechanism | Policy Key | Notes |
|---|---|---|---|
| REST/gRPC use-cases | Mediator pipeline (`ResilienceBehavior`) | `platform.pipeline.resilience.policy-name` | Ensures resilience wraps handler execution (outside transaction behavior). |
| Workers (Outbox/Inbox) | `DefaultWorkerLoopEngine` (optional) | `platform.outbox.worker.resilience.policy-name` / `platform.inbox.worker.resilience.policy-name` (fallback to `worker.executor.resilience.policy-name`) | Per-message wrapping; controlled by starter wiring. |
| Messaging consumer (Kafka listener) | Consumer wraps receiver (optional) | `platform.messaging.consumer.resilience.policy-name` | Keeps inbound transport resilient without leaking into application. |
| Saga step execution | `SagaExecutionEngine` (optional) | `platform.saga.resilience.policy-name` | Saga must run even when resilience capability is disabled. |

---

## Enforcement

* ArchUnit: Application must not use `@Transactional` (Rule A).
* Guardrail: transport adapters should only call use-cases via Bus/Mediator (so pipeline always applies).

---

## Example

### WRONG

```java
@Service
public class OrderApplicationService {
  @Transactional // VIOLATION
  @Retry(name = "pipeline-behavior")
  public void placeOrder(PlaceOrderCommand cmd) { ... }
}
```

### CORRECT

```java
class PlaceOrderCommandHandler { ... } // no framework annotations

@Configuration
class OrderWiring {
  @Bean PlaceOrderCommandHandler handler(...) { ... }
}
```

Transaction boundary is applied by infrastructure (TransactionBehavior) and persistence adapters.

---

## Related ADRs

* ADR-012: Transaction boundary / UnitOfWork ownership
* ADR-023: Service-to-service auth boundary (context enrichment and anti-spoofing)
* ADR-026: Synchronous communication standard (gRPC)

