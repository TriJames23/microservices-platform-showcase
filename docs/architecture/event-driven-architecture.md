# Event-Driven Architecture

## Why Event-Driven?

Synchronous cross-service calls create coupling, failure cascades, and distributed locking problems.

This platform uses **event-driven coordination** for all cross-service **state changes**:
- Synchronous gRPC is limited to **pre-check semantics** (read-only, availability checks)
- All writes that cross service boundaries go through **Outbox → Kafka → Inbox**

---

## The Outbox / Inbox Pattern

Reliable message delivery without distributed transactions.

```
┌─────────────────────────────────────────────────────────────────┐
│  Producing Service                                               │
│                                                                  │
│  ┌────────────┐    write in      ┌──────────┐                   │
│  │  Use Case  │ ─── same TX ──► │  Outbox  │ (DB table)        │
│  └────────────┘                 └─────┬────┘                   │
│                                       │                          │
│                              Outbox Worker polls                 │
│                                       │                          │
│                                       ▼                          │
│                               ┌──────────────┐                  │
│                               │    Kafka     │                  │
│                               └──────┬───────┘                  │
└──────────────────────────────────────┼──────────────────────────┘
                                       │
┌──────────────────────────────────────┼──────────────────────────┐
│  Consuming Service                   │                          │
│                                      ▼                          │
│                               ┌──────────────┐                  │
│                               │    Inbox     │ (DB table)       │
│                               └──────┬───────┘                  │
│                                      │                          │
│                              Inbox Worker executes               │
│                                      │                          │
│                                      ▼                          │
│                              ICommandBus → Handler              │
└─────────────────────────────────────────────────────────────────┘
```

**Key properties:**
- Write to outbox in the **same local transaction** as the business mutation
- Kafka delivery happens asynchronously by outbox worker
- Receiving service persists to inbox **before** executing — idempotency by default
- Business idempotency is still owned by the receiving service at the command boundary

---

## Saga Orchestration

Multi-step, cross-service workflows are coordinated by a **Saga** using integration commands.

```
order-service (saga orchestrator)
       │
       ├──► ReserveInventoryIntegrationCommand ──► inventory-service
       │
       ├──► ProcessPaymentIntegrationCommand ──► payment-service
       │
       └──► (on failure) ReleaseInventoryIntegrationCommand ──► inventory-service (compensation)
```

**Saga contracts (from `shared-kernel`):**
- `ISagaDefinition` — state-machine contract
- `ISagaTransition` — next-step resolution
- `ISagaStepHandler` — local step execution via `ISagaContext`
- `ICommandDispatcher` — cross-service dispatch boundary
- `ISagaCorrelationResolver` — event-to-saga correlation

**Runtime (from `shared-services`):**
- `SagaAutoConfiguration` discovers and wires saga definitions
- `OutboxCommandsAutoConfiguration` / `InboxCommandsAutoConfiguration` own command lanes
- Compensation triggers when a saga step fails — no inline direct rollback

---

## Integration Events vs Integration Commands

| | Integration Event | Integration Command |
|---|---|---|
| **Direction** | Broadcast (1-to-many) | Targeted (1-to-1) |
| **Intent** | "This happened" | "Do this" |
| **Transport** | Kafka topic | Outbox → Kafka → Inbox |
| **Idempotency** | Consumer-owned | Inbox enforces + service-owned |
| **Example** | `OrderCreatedIntegrationEvent` | `ReserveInventoryIntegrationCommand` |

---

## Observability Hooks

Every outbox/inbox/saga step emits semantic metrics through:
- `IOutboxMetrics` — backlog, publish success/failure
- `IInboxMetrics` — ingestion, processing success/failure
- `ISagaMetrics` — started, completed, failed, compensated, duration

Raw metric names are centralized in `MetricNames`. Runtime lanes never reference them directly.
