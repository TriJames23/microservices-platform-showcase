# ADR-004: Event-Driven Saga Orchestration

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: ArchUnit (Rule C)

---

## Context

Distributed transactions require coordination across multiple services. Explicit step-calls create tight coupling between Saga and Application Services.

---

## Decision

`SagaManager` reacts **ONLY to Events** and invokes `SagaStep` abstractions.

### Constraints

- Saga **NEVER** calls Application Services or Infrastructure APIs directly
- Saga **MUST NOT** manage transactions
- Saga **MUST NOT** hold Domain Entities (only IDs, enums, value objects)
- Saga **MUST** be stateless orchestrator, not domain aggregate

---

## Consequences

### Positive
✅ Loose coupling between Saga and Services
✅ Saga logic remains orchestration-focused
✅ Easy to add new steps without changing Saga
✅ No stale state issues

### Negative
⚠️ More complex than direct method calls
⚠️ Requires event infrastructure
⚠️ Debugging can be harder (async flow)

---

## Enforcement

**ArchUnit Rule C**:
```java
ruleC_saga_mustNotHold_DomainEntities()
```

**Severity**: 🟠 HIGH

**What it prevents**:
- Stale state (Saga holds outdated entity snapshot)
- Concurrency violations (Saga state conflicts with aggregate version)
- Serialization bloat (Domain entities with lazy-loaded collections)
- Transactional boundaries blur (Saga becomes pseudo-aggregate)

---

## Example

### ❌ WRONG
```java
public class OrderSaga {
    private Order order; // ← VIOLATION - Domain entity
    private Payment payment; // ← VIOLATION - Domain entity
    
    public void handle(OrderPlacedEvent event) {
        // Stale state: order may have been modified by another process
        order.confirm(); // Concurrency violation
        
        // Direct call to Application Service
        orderService.confirmOrder(order.getId()); // ← VIOLATION
    }
}
```

### ✅ CORRECT
```java
public class OrderSaga {
    private UUID orderId; // ✅ Primitive
    private UUID paymentId; // ✅ Primitive
    private OrderStatus status; // ✅ Enum/Value Object
    
    @SagaEventHandler
    public void on(OrderPlacedEvent event) {
        this.orderId = event.getOrderId();
        this.status = OrderStatus.PENDING_PAYMENT;
        
        // Publish command/event, don't call service directly
        commandGateway.send(new RequestPaymentCommand(orderId));
    }
    
    @SagaEventHandler
    public void on(PaymentCompletedEvent event) {
        // Load fresh aggregate from repository if needed
        Order order = orderRepository.findById(orderId);
        order.confirm();
        orderRepository.save(order);
        
        this.status = OrderStatus.CONFIRMED;
    }
}
```

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Saga calling ApplicationService methods" | Creates tight coupling |
| "Saga mutating Domain state directly" | Violates DDD boundaries |
| "Saga holding Domain Entities" | Causes stale state and concurrency issues |

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-005](ADR-005-outbox-pattern.md): Outbox Pattern for Integration Events
- [ADR-010](ADR-010-resilience-scope.md): Resilience Scope
