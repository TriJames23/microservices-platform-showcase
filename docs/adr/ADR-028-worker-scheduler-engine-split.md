# ADR-028: Worker-Scheduler Engine Split

**Status**: ✅ Accepted  
**Date**: 2026-Q2  
**Enforced By**: ArchUnit + Code Review

---

## Context

The original `DefaultWorkerLoopEngine` served two distinct runtime responsibilities:

1. **Message-processing loop** — continuously polls outbox/inbox tables, dispatches work to handlers, marks results.
2. **Periodic task execution** — runs cleanup, maintenance, or housekeeping jobs on a fixed interval.

When combined in one engine, the poll-based assumptions (batch size, idle delay, error delay, in-flight limits) bled into periodic tasks, and the periodic task timing model (fixed delay, `ITaskResult`-driven next interval) bled into message pipelines. Retry semantics, thread sizing, and observability became ambiguous.

A large refactor wave in 2026-Q2 separated these into two concrete engine types.

---

## Decision

The platform provides **two distinct engine types**:

| Engine | Class | Purpose | Task Contract |
|--------|-------|---------|---------------|
| Message-processing loop | `DefaultWorkerLoopEngine` | Poll-based outbox/inbox message processing | `IProcessingResult` |
| Periodic task scheduler | `DefaultSchedulerEngine` | Fixed-delay periodic task execution | `ITaskResult` |

### `DefaultWorkerLoopEngine`

- Owned by `OutboxWorkerAutoConfiguration` and `InboxWorkerAutoConfiguration`.
- Reads `platform.outbox.worker.*` and `platform.inbox.worker.*` for batch size, idle/error delay, rate limit, and in-flight limits.
- Returns `IProcessingResult` from each processing cycle.
- Resilience wraps individual message dispatch, not the polling loop itself.

### `DefaultSchedulerEngine`

- Owned by `OutboxSchedulerAutoConfiguration` and `InboxSchedulerAutoConfiguration`.
- Reads `platform.outbox.cleanup.*` and `platform.inbox.cleanup.*` for retention, batch size, and interval.
- Returns `ITaskResult` from each task execution.
- `ITaskResult.success()` → next run at `interval-seconds`.
- `ITaskResult.failure(Duration)` → next run after the duration returned by the task.
- The scheduler does **not** own retry loops internally; the task drives its own rescheduling through `ITaskResult`.

---

## Constraints

- **`DefaultWorkerLoopEngine` must not be used for periodic maintenance tasks.**
- **`DefaultSchedulerEngine` must not be used for message-processing pipelines.**
- Both engines share the `worker.executor` thread pool.
- Both engines resolve resilience policy through the shared fallback chain:
  - worker-specific `resilience.policy-name` → `worker.executor.resilience.policy-name`.
- Service-owned periodic jobs must use `DefaultSchedulerEngine` and implement `ITaskResult`.
- Service-owned message-processing jobs must use `DefaultWorkerLoopEngine` and implement `IProcessingResult`.

---

## Consequences

### Positive
✅ Clear separation of polling semantics from scheduling semantics.  
✅ `ITaskResult` gives tasks explicit control over failure rescheduling intervals.  
✅ `IProcessingResult` gives message loops explicit control over marking and retry within the pipeline.  
✅ Observability surfaces are distinct per engine type.  
✅ Throughput sizing (batch size, idle delay, in-flight) only applies to `DefaultWorkerLoopEngine`.

### Negative
⚠️ Service teams must learn which engine to pick when adding background jobs.  
⚠️ Existing code that used the combined engine must be migrated.

---

## Enforcement

**ArchUnit Rules**:
- Periodic cleanup beans must resolve through `DefaultSchedulerEngine`.
- `ITaskResult` usage must not appear in classes wired through `DefaultWorkerLoopEngine`.
- `IProcessingResult` usage must not appear in classes wired through `DefaultSchedulerEngine`.

**Severity**: 🔴 CRITICAL

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Single engine with mode flag" | Mode flags create implicit coupling and ambiguous observability |
| "Thread-per-task model for periodic jobs" | Bypasses shared executor sizing and resilience policy |
| "Use Spring `@Scheduled` directly" | Bypasses worker executor, resilience, and platform observability |

---

## Related ADRs

- [ADR-005](ADR-005-outbox-pattern.md): Outbox Pattern — cleanup lifecycle lives here
- [ADR-010](ADR-010-resilience-scope.md): Resilience Scope — shared policy fallback chain
- [ADR-015](ADR-015-inbox-pattern.md): Inbox Pattern — cleanup lifecycle lives here
- [ADR-020](ADR-020-async-strategy.md): Async Strategy — executor sizing and threading model
