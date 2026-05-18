# ADR-008: Passive Observability

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: Code Review

---

## Context

Logging and metrics inside domain logic obscures business intent and couples domain to observability libraries (SLF4J, Micrometer).

---

## Decision

Observability is a **Passive Infrastructure Concern**.

### Constraints

- Domain **MUST NOT** depend on logging or metrics APIs directly
- If observability signals are needed, they must be expressed as:
  - **Domain Events** (for business-significant occurrences)
  - **Returned values** (for state changes)
  - NOT side effects (logging, metrics)
- Infrastructure layer handles all observability concerns
- Metrics taxonomy, alert contracts, and tracing runtime helpers live in `shared-services`, not `shared-kernel`
- Runtime lanes outside observability must use semantic metrics interfaces instead of generic recorder calls

---

## Consequences

### Positive
✅ Pure business logic (no logging noise)  
✅ Domain remains framework-agnostic  
✅ Easy to change observability stack  
✅ Business events are explicit, not hidden in logs

### Negative
⚠️ Cannot log directly from domain (less convenient)  
⚠️ Requires discipline to use events instead of logging

---

## Example

### ❌ WRONG
```java
// Domain
public class Order {
    private static final Logger log = LoggerFactory.getLogger(Order.class); // ← VIOLATION
    
    public void confirm() {
        this.status = OrderStatus.CONFIRMED;
        log.info("Order {} confirmed", this.id); // ← VIOLATION: Logging in domain
        metrics.counter("orders.confirmed").increment(); // ← VIOLATION: Metrics in domain
    }
}
```

### ✅ CORRECT
```java
// Domain (Pure, uses events)
public class Order extends AggregateRoot {
    public void confirm() {
        this.status = OrderStatus.CONFIRMED;
        // ✅ Express as Domain Event
        this.registerEvent(new OrderConfirmedEvent(this.id, this.customerId));
    }
}

// Infrastructure (Observability via Event Listener)
@Component
public class OrderEventLogger {
    private static final Logger log = LoggerFactory.getLogger(OrderEventLogger.class);
    
    @EventListener
    public void on(OrderConfirmedEvent event) {
        // ✅ Logging in infrastructure
        log.info("Order {} confirmed by customer {}", 
            event.getOrderId(), event.getCustomerId());
        
        // ✅ Metrics in infrastructure
        metrics.counter("orders.confirmed",
            "customer", event.getCustomerId().toString()
        ).increment();
    }
}
```

---

## Observability Patterns

| Need | ❌ Wrong (Domain) | ✅ Correct (Infrastructure) |
|:-----|:-----------------|:---------------------------|
| Log business event | `log.info("Order confirmed")` | Publish `OrderConfirmedEvent`, log in listener |
| Track metric | `metrics.counter().increment()` | Publish event, count in listener |
| Trace execution | `span.start()` | AOP/Interceptor at infrastructure boundary |
| Debug state | `log.debug("state={}", state)` | Return state, log in application/infrastructure |

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Logger injection into Domain" | Couples domain to logging framework |
| "Metrics counters in business logic" | Obscures business intent |
| "Conditional logging (if debug enabled)" | Still couples to logging API |

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-005](ADR-005-outbox-pattern.md): Outbox Pattern (uses events)
- `book/14-observability-metrics-alerting-and-tracing-flow.md`: platform metrics, alert, and tracing flow
