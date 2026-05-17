# Outbox / Inbox Flow

> **Deep Dive** — Full technical sequence for reliable cross-service command delivery.

Source: `book/05-outbox-inbox-integration-command-flow.md`

---

## Full Sequence

```
Producing Service
─────────────────
1. Use case executes inside local transaction
2. Business state mutated on aggregate
3. Integration event/command written to outbox table (SAME transaction)
4. Transaction commits — both business state and outbox row are durable

Outbox Worker (separate process)
─────────────────────────────────
5. OutboxAutoConfiguration polls pending outbox rows
6. Publishes to Kafka topic (or gRPC dispatch for command lanes)
7. Marks outbox row as published

Kafka
──────
8. Message sits in topic with at-least-once delivery guarantee

Consuming Service
──────────────────
9. KafkaIntegrationEventConsumer receives message
10. ExecutionContextFactory builds context from Kafka headers
11. Inbox row written BEFORE processing (write-before-processing)
12. Deduplication check — if already processed, skip silently
13. InboxAutoConfiguration invokes ICommandBus
14. Standard pipeline runs: validation → authorization → handler
15. Handler executes business logic
16. Inbox row marked as processed (only on success)
17. ExecutionContextHolder.clear() in finally block
```

---

## Cleanup Lifecycle

Both outbox and inbox maintain cleanup schedulers using `DefaultSchedulerEngine` (periodic model):

| Setting | Outbox | Inbox |
|---|---|---|
| Enabled by default | ❌ (opt-in) | ✅ |
| Config key | `platform.outbox.cleanup.enabled` | `platform.inbox.cleanup.enabled` |
| Retention | `platform.outbox.cleanup.retention-days` | `platform.inbox.cleanup.retention-days` |

**Note:** Cleanup uses `DefaultSchedulerEngine`, NOT `DefaultWorkerLoopEngine`. These two engines serve different purposes and must not be confused (ADR-028).

---

## Observability During Flow

| Step | Metric |
|---|---|
| Outbox publish | `IOutboxMetrics` — backlog size, publish success/failure |
| Inbox ingestion | `IInboxMetrics` — ingestion count, processing success/failure |
| Tracing | `ITracer` — span at outbox publish and inbox processing boundaries |

---

## Business Idempotency

Transport-level deduplication is provided by inbox. But the receiving service still owns **business idempotency** at the command handler boundary.

Even when inbox has already deduplicated by message id, the command handler should be safe to receive the same business intent twice.
