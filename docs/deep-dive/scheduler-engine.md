# Scheduler Engine

> **Deep Dive** — Worker engine types, task contracts, and the engine split rationale.

Source: `book/32-worker-module-implementation-guide.md`, ADR-028

---

## Two Engines, Two Purposes

The platform provides two distinct runtime engines. They are **not interchangeable** (ADR-028).

| Engine | Class | Purpose | Task Contract |
|---|---|---|---|
| Message-processing loop | `DefaultWorkerLoopEngine` | Polls and processes outbox/inbox messages | `IProcessingResult` |
| Periodic task scheduler | `DefaultSchedulerEngine` | Runs time-based cleanup or maintenance tasks | `ITaskResult` |

Using `DefaultSchedulerEngine` for message-processing or `DefaultWorkerLoopEngine` for periodic tasks is a **drift pattern** — caught by architecture tests.

---

## Who Uses Which Engine

| Auto-configuration | Engine Used |
|---|---|
| `OutboxWorkerAutoConfiguration` | `DefaultWorkerLoopEngine` |
| `InboxWorkerAutoConfiguration` | `DefaultWorkerLoopEngine` |
| `OutboxSchedulerAutoConfiguration` | `DefaultSchedulerEngine` |
| `InboxSchedulerAutoConfiguration` | `DefaultSchedulerEngine` |

---

## `ITaskResult` Contract

Tasks scheduled by `DefaultSchedulerEngine` return `ITaskResult`:

```java
// On clean completion
return ITaskResult.success();

// On error — task drives its own retry interval
return ITaskResult.failure(Duration.ofSeconds(30));
```

The scheduler does not implement retry loops internally. The task itself drives interval and backoff via `ITaskResult`.

---

## Worker E2E Sequence

```
1. Worker runtime triggers a scheduled or background job

2. Job resolves execution context (or inherits from platform)

3. Job calls ICommandBus or IQueryBus
   (same pipeline as HTTP/gRPC — no shortcuts)

4. Standard pipeline runs:
   validation → authorization → idempotency → resilience → audit → handler

5. Worker metrics and tracing emitted by runtime lane
   (not ad-hoc instrumentation in job logic)
```

---

## Resilience Policy Binding

Both engines share `worker.executor.resilience.policy-name` as the global fallback:

```yaml
worker:
  executor:
    resilience:
      policy-name: worker   # Resilience4j named instance
```

Cleanup workers can override with lane-specific policy names:
```yaml
platform:
  outbox:
    cleanup:
      resilience:
        policy-name: ""     # blank = fallback to worker.executor policy
  inbox:
    cleanup:
      resilience:
        policy-name: ""
```

---

## Forbidden Patterns

| Pattern | Why |
|---|---|
| Repository access directly from job loop | Must go through `ICommandBus` |
| Business retry logic outside resilience policy | Worker policy owns retry semantics |
| Bypassing application pipeline | Same behaviors apply to background work |
| Mixing engine types | `DefaultSchedulerEngine` ≠ `DefaultWorkerLoopEngine` |
| Background jobs calling remote services directly | Must go through saga/outbox boundaries |
