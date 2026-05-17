# Messaging Platform

**Module**: `shared-services-infrastructure-messaging`

The Kafka integration layer. Owns broker-specific consumer/producer wiring and routes inbound integration events into the application pipeline.

---

## Design Principle

The messaging module translates **external signals into local intent**. It must not contain business logic, aggregate mutations, or direct repository access.

```
Kafka topic (integration event)
       │
       ▼
KafkaIntegrationEventConsumer
  → ExecutionContextFactory.build(headers)   ← explicit context at transport boundary
  → ExecutionContextHolder.set(context)
  → route into InboxProcessor
       │
       ▼
InboxProcessor (@Transactional)
  → IIntegrationEventHandler maps event → command
  → ICommandBus executes command
  → inbox row marked PROCESSED (in same TX)
  → if failed: transaction rolls back, failure captured in REQUIRES_NEW TX
       │
       ▼
  ExecutionContextHolder.clear()  ← always in finally block
```

---

## Key Contracts

| Contract | Role |
|---|---|
| `IIntegrationEventHandler` | Application-facing event reaction boundary |
| `ICommandBus` | Path back into use-case execution after event mapping |
| `KafkaIntegrationEventConsumer` | Owns context boundary control |
| `InboxAutoConfiguration` | Provides atomic transactional inbox processing |

---

## Auto-configuration

| Config | Role |
|---|---|
| `KafkaMessagingAutoConfiguration` | Kafka consumer/producer wiring |
| `RabbitMessagingAutoConfiguration` | RabbitMQ consumer/producer wiring (alternative) |

---

## Configuration

```yaml
messaging:
  broker: KAFKA
  bootstrap-servers: localhost:9092

platform:
  messaging:
    consumer:
      resilience:
        policy-name: messaging
  inbox:
    enabled: true
    commands:
      enabled: false    # true only if service owns IIntegrationCommandTypeRegistry
```

---

## Forbidden Patterns

| Pattern | Why |
|---|---|
| Direct aggregate mutation in message handler | Must go through `ICommandBus` |
| `JpaRepository` injected in messaging adapter | Persistence belongs in persistence module |
| Remote orchestration hidden inside broker adapter | Must use saga/outbox boundaries |
| Event handler skipping inbox/idempotency semantics | Inbox is mandatory for all inbound processing |
| Business authorization logic in messaging handler | Handler must not inline policy checks |

---

## Observability

- `IInboxMetrics` — ingestion count, processing success/failure
- `ITracer` — trace span at inbox processing boundary
- `ExecutionContext` carries `correlationId` and `tenantId` for logging continuity
