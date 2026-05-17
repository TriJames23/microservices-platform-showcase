# Communication Patterns

## Decision Framework

| Question | Answer |
|---|---|
| Need a read or availability pre-check? | **gRPC** (synchronous, pre-commit only) |
| Need to trigger a cross-service state change? | **Outbox → Inbox** (async, reliable) |
| Need multi-step coordination with compensation? | **Saga via integration command** |
| Publishing a domain event to multiple consumers? | **Kafka integration event** |

---

## HTTP REST

Used for **client-facing** request/response interactions.

```
Client → HTTP Controller (thin)
            │
            ▼
        ICommandBus / IQueryBus
            │
            ▼
        Pipeline (validation · authorization · behaviors)
            │
            ▼
        Handler → Result<T>
            │
            ▼
        HTTP Response (mapped from Result<T>)
```

- Controllers call bus only — zero business logic at transport layer
- `Result<T>` is always the handler return type
- Error mapping from `Result<T>` happens at the transport boundary

---

## gRPC

Used for **synchronous inter-service** pre-checks — **not** for state changes.

```
Service A → GrpcCommandDispatch → Service B gRPC server
                                       │
                                  Interceptors:
                                  ContextPropagationServerInterceptor
                                  PassiveIdempotencyLoggingServerInterceptor
                                       │
                                  ICommandBus / IQueryBus
                                       │
                                  TemplateGrpcErrorMapper → StatusRuntimeException
```

**Rules:**
- Synchronous gRPC limited to **read-only / pre-check** semantics before local commit
- Cross-service state changes (inventory reserve, payment process) go through Outbox/Inbox
- `PassiveIdempotencyLoggingServerInterceptor` is **passive logging only** — not enforcement
- `.proto` files live in `*-contract-grpc`, never in `*-contract`

---

## Outbox → Kafka → Inbox

Reliable, transactional message delivery for all cross-service state mutations.

| Property | Behavior |
|---|---|
| Delivery | At-least-once (inbox handles deduplication) |
| Transaction | Write to outbox in same TX as business mutation |
| Failure | Outbox worker retries; DLQ on exhaustion |
| Idempotency | Inbox provides transport-level; service owns business-level |

---

## Integration Commands vs Integration Events

```
Integration Event   →  broadcast, "this happened", 1-to-many
Integration Command →  targeted, "do this", 1-to-1 via outbox/inbox
```

Saga uses **integration commands** — not events — to coordinate state changes across service boundaries.
