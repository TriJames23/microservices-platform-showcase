# ADR-005: Outbox Pattern for Integration Events

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: Code Review

---

## Context

Data consistency between database and message broker is critical. Publishing events directly to Kafka/RabbitMQ from application code can lead to:
- Lost messages (DB committed, broker failed)
- Duplicate messages (broker succeeded, DB rollback)
- Coupling domain to message infrastructure

---

## Decision

Use the **Transactional Outbox Pattern** for reliable event publishing.

### Constraints

- **ONLY Integration Events** are persisted in the `outbox` table
- **Domain Events** are in-memory only
- Domain Events **MUST** be mapped by Application layer to Integration Events before persistence
- Outbox publisher runs in separate transaction (polling or CDC)

---

## Consequences

### Positive
✅ Guaranteed message delivery (at-least-once)  
✅ Domain models remain pure (no broker dependencies)  
✅ Message contract decoupled from internal state  
✅ Can replay/reprocess events

### Negative
⚠️ Eventual consistency (slight delay)  
⚠️ Requires outbox polling/CDC infrastructure  
⚠️ Potential duplicate messages (need idempotent consumers)

---

## Architecture

```
┌─────────────┐
│   Domain    │
│   Events    │ (in-memory)
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Application    │
│  Layer          │ Maps to Integration Events
└──────┬──────────┘
       │
       ▼
┌─────────────────┐     ┌──────────────┐
│  Outbox Table   │────▶│   Outbox     │
│  (Transactional)│     │  Publisher   │
└─────────────────┘     └──────┬───────┘
                               │
                               ▼
                        ┌──────────────┐
                        │ Kafka/RabbitMQ│
                        └──────────────┘
```

---

## Example

### ❌ WRONG
```java
// Domain
public class Order {
    public void confirm() {
        this.status = OrderStatus.CONFIRMED;
        // ← VIOLATION: Domain publishing to broker
        kafkaTemplate.send("orders", new OrderConfirmedEvent(this.id));
    }
}
```

### ✅ CORRECT
```java
// Domain (Pure, in-memory events)
public class Order extends AggregateRoot {
    public void confirm() {
        this.status = OrderStatus.CONFIRMED;
        // ✅ Domain Event (in-memory)
        this.registerEvent(new OrderConfirmedDomainEvent(this.id));
    }
}

// Application (Maps to Integration Event)
@Service
public class OrderService {
    public void confirmOrder(OrderId id) {
        Order order = repository.findById(id);
        order.confirm();
        repository.save(order); // Saves order + outbox entry in same TX
        
        // Map domain event to integration event
        order.getDomainEvents().forEach(event -> {
            if (event instanceof OrderConfirmedDomainEvent) {
                outboxRepository.save(new OutboxEntry(
                    new OrderConfirmedIntegrationEvent(event.getOrderId())
                ));
            }
        });
    }
}

// Infrastructure (Outbox Publisher - separate process)
@Scheduled(fixedDelay = 1000)
public void publishOutboxEvents() {
    List<OutboxEntry> pending = outboxRepository.findPending();
    pending.forEach(entry -> {
        kafkaTemplate.send("orders", entry.getPayload());
        outboxRepository.markPublished(entry.getId());
    });
}
```

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Persist Domain Events directly" | Couples domain to persistence format |
| "Publish to Kafka from Application code" | No transactional guarantee |
| "Two-phase commit (2PC)" | Too complex, poor performance |

---

## Cleanup Scheduler Lifecycle

- `OutboxSchedulerAutoConfiguration` owns the `OutboxCleanupProcessor` and `DefaultSchedulerEngine` bean for periodic outbox table cleanup.
- Cleanup activates when `platform.outbox.cleanup.enabled=true`. Default in sandbox is `false` (opt-in).
- Cleanup uses `DefaultSchedulerEngine` (periodic task model), not `DefaultWorkerLoopEngine` (message-processing model). See ADR-028 for the engine split rationale.
- `platform.outbox.cleanup.interval-seconds` controls how often cleanup runs (default: 300s).
- `platform.outbox.cleanup.retention-days` controls how old a processed row must be before it is eligible for deletion (default: 7 days).
- `platform.outbox.cleanup.resilience.policy-name` when blank falls back to `worker.executor.resilience.policy-name`.
- Template YAML must expose the full `platform.outbox.cleanup` block including the `resilience.policy-name` field.

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-004](ADR-004-event-driven-saga-orchestration.md): Event-Driven Saga Orchestration
- [ADR-028](ADR-028-worker-scheduler-engine-split.md): Worker-Scheduler Engine Split
