# ADR-015: Inbox Pattern (Idempotent Consumers)

**Status**: ✅ Accepted  
**Date**: 2026-Q1  
**Enforced By**: Code Review + Integration Tests

---

## Context

Outbox ensures reliable publishing, but consumers still risk duplicate processing. Without a formal inbox standard, idempotent consumption is tribal knowledge.

---

## Decision

All inbound message processing MUST be idempotent and MUST use the Inbox Pattern:

- Persist message metadata **before** processing
- Deduplicate by unique message id (and consumer id)
- Mark processed **only after** handler success

---

## Constraints

- Inbox storage MUST be write-before-processing
- Message id MUST be unique per producer
- Failed processing MUST NOT mark as processed
- Cleanup MUST be time-based and configurable

---

## Consequences

### Positive
✅ Safe re-delivery — duplicates silently ignored  
✅ Reliable recovery after partial failures  
✅ Clear operational visibility into message state

### Negative
⚠️ Extra storage and maintenance  
⚠️ Requires cleanup jobs

---

## Cleanup Scheduler

- `platform.inbox.cleanup.enabled=true` by default
- Uses `DefaultSchedulerEngine` (periodic task model), not `DefaultWorkerLoopEngine`
- `platform.inbox.cleanup.retention-days` controls eligible age (default: 7 days)

---

## Related ADRs

- [ADR-005](ADR-005-outbox-pattern.md): Outbox Pattern
- ADR-022: Idempotency Key Standard
- ADR-028: Worker-Scheduler Engine Split
