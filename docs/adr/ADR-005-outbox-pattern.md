# ADR-005: Outbox Pattern for Integration Events

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: Code Review

---

## Context

Data consistency between database and message broker is critical. Publishing events directly to Kafka from application code can lead to:
- Lost messages (DB committed, broker failed)
- Duplicate messages (broker succeeded, DB rollback)
- Coupling domain to message infrastructure

---

## Decision

Use the **Transactional Outbox Pattern** for reliable event publishing.

### Constraints

- **ONLY Integration Events** are persisted in the `outbox` table
- **Domain Events** are in-memory only
- Domain Events **MUST** be mapped by Application layer to Integration Events before persistence
- Outbox publisher runs in separate transaction (polling)

---

## Flow

```
Domain Event (in-memory)
      │
      ▼ Application Layer maps to Integration Event
      │
      ▼
Outbox Table  ←── written in SAME transaction as business mutation
      │
      ▼ Outbox Worker (polls, separate TX)
      │
      ▼
Kafka
```

---

## Consequences

### Positive
✅ Guaranteed message delivery (at-least-once)  
✅ Domain models remain pure (no broker dependencies)  
✅ Message contract decoupled from internal state  
✅ Can replay/reprocess events

### Negative
⚠️ Eventual consistency (slight delay)  
⚠️ Requires outbox polling infrastructure  
⚠️ Potential duplicate messages (need idempotent consumers — see ADR-015)

---

## Cleanup Scheduler

- `platform.outbox.cleanup.enabled=false` by default — opt-in for production
- Cleanup uses `DefaultSchedulerEngine` (periodic), not `DefaultWorkerLoopEngine` (message loop)
- `platform.outbox.cleanup.retention-days` controls eligible age (default: 7 days)

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-015](ADR-015-inbox-pattern.md): Inbox Pattern
- ADR-028: Worker-Scheduler Engine Split
