# ADR-016: Dead Letter Queue (DLQ) and Error Handling

**Status**: Accepted  
**Date**: 2026-Q1  
**Enforced By**: Code Review + Operational Runbooks

---

## Context

Retry policies exist, but permanent failures need a consistent DLQ strategy. The platform must standardize how failed messages are parked and inspected.

---

## Decision

All workers MUST implement a DLQ strategy after max retries are exceeded.

Supported targets:
- Dead Topic (broker-based)
- Dead Table (database-based)

The target is selected by the transport implementation, but MUST preserve:
- Original payload
- Headers and correlation id
- Failure reason
- Retry count
- Timestamp

---

## Constraints

- A message MUST not be dropped silently
- DLQ write MUST be best-effort and observable
- DLQ records MUST be queryable for ops

---

## Consequences

### Positive
- Deterministic handling of poison messages
- Clear operational visibility

### Negative
- Additional storage and monitoring

---

## Enforcement

- Worker retry policy MUST route to DLQ
- Ops dashboard must expose DLQ metrics

---

## Related ADRs

- [ADR-010](ADR-010-resilience-scope.md): Resilience Scope
- [ADR-015](ADR-015-inbox-pattern.md): Inbox Pattern
- [ADR-022](ADR-022-idempotency-key-standard.md): Idempotency Key Standard
- [ADR-025](ADR-025-data-migration-and-backfill.md): Data Migration and Backfill
