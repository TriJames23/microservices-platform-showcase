# ADR-015: Inbox Pattern (Idempotent Consumers)

**Status**: Accepted  
**Date**: 2026-Q1  
**Enforced By**: Code Review + Integration Tests

---

## Context

Outbox ensures reliable publishing, but consumers still risk duplicate processing. The Event Flow mentions Inbox, but there is no formal standard to enforce idempotent consumption.

---

## Decision

All inbound message processing MUST be idempotent and MUST use the Inbox Pattern:

- Persist message metadata before processing
- Deduplicate by unique message id (and consumer id)
- Mark processed only after handler success

---

## Constraints

- Inbox storage MUST be write-before-processing
- Message id MUST be unique per producer
- Failed processing MUST not mark as processed
- Cleanup MUST be time-based and configurable

---

## Consequences

### Positive
- Safe re-delivery
- Reliable recovery
- Clear operational visibility

### Negative
- Extra storage and maintenance
- Requires cleanup jobs

---

## Enforcement

- Inbox table or storage is mandatory for all consumers
- Integration tests must verify duplicate delivery is ignored

---

## Cleanup Scheduler Lifecycle

- `InboxSchedulerAutoConfiguration` owns the `InboxCleanupProcessor` and `DefaultSchedulerEngine` bean for periodic inbox table cleanup.
- Cleanup is enabled by default (`platform.inbox.cleanup.enabled=true`). Disable explicitly only if the service manages inbox retention externally.
- Cleanup uses `DefaultSchedulerEngine` (periodic task model), not `DefaultWorkerLoopEngine` (message-processing model). See ADR-028 for the engine split rationale.
- `platform.inbox.cleanup.interval-seconds` controls how often cleanup runs (default: 300s).
- `platform.inbox.cleanup.retention-days` controls how old a processed row must be before deletion (default: 7 days).
- `platform.inbox.cleanup.resilience.policy-name` when blank falls back to `worker.executor.resilience.policy-name`.
- Template YAML must expose the full `platform.inbox.cleanup` block including the `resilience.policy-name` field.

---

## Related ADRs

- [ADR-005](ADR-005-outbox-pattern.md): Outbox Pattern
- [ADR-011](ADR-011-architectural-enforcement-strategy.md): Enforcement Strategy
- [ADR-022](ADR-022-idempotency-key-standard.md): Idempotency Key Standard
- [ADR-028](ADR-028-worker-scheduler-engine-split.md): Worker-Scheduler Engine Split
