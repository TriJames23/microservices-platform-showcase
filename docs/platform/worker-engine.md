# Worker Engine

**Module**: `shared-services-infrastructure-worker`

The background execution runtime for the platform. Provides two distinct engine types for different execution patterns.

---

## Engine Types

| Engine | Class | Purpose |
|---|---|---|
| Message-processing loop | `DefaultWorkerLoopEngine` | Processes outbox/inbox message queues |
| Periodic task scheduler | `DefaultSchedulerEngine` | Runs time-based maintenance tasks |

These engines are **not interchangeable**. Using the wrong engine for a task is an architecture violation (ADR-028).

---

## Worker Loop Engine

`DefaultWorkerLoopEngine` — used by `OutboxWorkerAutoConfiguration` and `InboxWorkerAutoConfiguration`.

Polling model: the engine continuously loops, picks up pending messages, processes them via `IProcessingResult`, and handles retry/DLQ routing.

**Used by:**
- Outbox publisher worker
- Inbox processor worker

---

## Scheduler Engine

`DefaultSchedulerEngine` — used by `OutboxSchedulerAutoConfiguration` and `InboxSchedulerAutoConfiguration`.

Fixed-delay periodic model: runs a task on a schedule, gets back `ITaskResult`, and the task itself controls retry interval on failure.

**Used by:**
- Outbox cleanup processor
- Inbox cleanup processor

---

## Execution Pipeline

Background jobs go through the **same application pipeline** as interactive flows:

```
Worker triggers job
       │
       ▼
ICommandBus / IQueryBus (same bus as HTTP/gRPC)
       │
       ▼
Pipeline: validation → authorization → idempotency → resilience → audit
       │
       ▼
Handler executes
```

Repository access directly from a job loop is **forbidden**. Jobs must go through the bus.

---

## Resilience Configuration

```yaml
worker:
  executor:
    core-pool-size: 4
    max-pool-size: 8
    queue-capacity: 100
    resilience:
      policy-name: worker   # Resilience4j named instance (fallback for all lanes)
```

---

## Observability

Worker metrics and tracing are emitted by the runtime lane — not by ad-hoc instrumentation inside job logic. Worker-side metrics flow through `IOutboxMetrics` / `IInboxMetrics` for their respective lanes.

→ See [Scheduler Engine deep-dive](../deep-dive/scheduler-engine.md) for engine split details.
