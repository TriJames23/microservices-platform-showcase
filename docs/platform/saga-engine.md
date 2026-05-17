# Saga Engine

**Module**: `shared-services-infrastructure-saga`

The platform's distributed workflow coordination engine, built on event-driven integration commands.

---

## Design

Saga orchestration is **event-driven and command-dispatch based** — never direct application-service invocation across service boundaries.

```
Saga starts (integration event or application event)
       │
       ▼
ISagaDefinition resolves → ISagaTransition
       │
       ▼
ISagaStepHandler executes via ISagaContext
       │
       ├── Local step → raise local event / command via service bus
       │
       └── Cross-service step → ICommandDispatcher → Outbox → Kafka → Inbox → ICommandBus
```

---

## Kernel Contracts

| Contract | Role |
|---|---|
| `ISagaDefinition` | State-machine definition |
| `ISagaTransition` | Next-step resolution and handler binding |
| `ISagaStepHandler` | Local step execution against `ISagaContext` |
| `ISagaCorrelationResolver` | Event-to-saga instance correlation |
| `ICommandDispatcher` | Cross-service dispatch boundary |

---

## Commerce Example

```
order-service saga orchestrates:

1. gRPC: catalog.price.check (pre-check, synchronous)
2. gRPC: inventory.availability.check (pre-check, synchronous)
3. order.create (local)
4. → ReserveInventoryIntegrationCommand (outbox → inbox → inventory-service)
5. → ProcessPaymentIntegrationCommand (outbox → inbox → payment-service)

On payment failure:
6. → ReleaseInventoryIntegrationCommand (compensation)
```

---

## Observability

Saga steps emit through `ISagaMetrics`:
- `saga.started`
- `saga.step.completed`
- `saga.step.failed`
- `saga.compensated`
- `saga.duration`

---

## Configuration

```yaml
platform:
  saga:
    enabled: true   # false in local sandbox
```

If `platform.inbox.commands.enabled=true`, the service must register `IIntegrationCommandTypeRegistry`.  
If `platform.outbox.commands.enabled=true`, the service must register a real `ICommandDispatcher` route.
