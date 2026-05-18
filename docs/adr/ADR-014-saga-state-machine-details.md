# ADR-014: Saga State Machine Details

**Status**: Accepted  
**Date**: 2026-Q1  
**Enforced By**: Code Review + Integration Tests

---

## Context

Saga orchestration is the most complex part of the platform. ADR-004 defines event-driven orchestration, but does not specify how the state machine, timeouts, idempotency, and compensation are implemented.

---

## Decision

Every Saga MUST be implemented as an explicit state machine with:

- Deterministic transitions by event type
- Explicit step names
- Timeout handling as first-class events
- Idempotent transitions
- Compensation logic for failed steps

---

## Constraints

- Transitions MUST be defined via `ISagaTransition<E>` and resolved by `ISagaDefinition<E>`
- Every step handler MUST be idempotent
- Timeout events MUST be produced by a scheduler or worker (never by direct sleeps)
- Compensation MUST be explicit and reversible
- Saga state MUST be persisted and rebuilt from `SagaInstance` only

---

## Consequences

### Positive
- Predictable saga flow
- Safe retries and replay
- Clear auditability of failure and compensation

### Negative
- More boilerplate for each saga
- Requires worker infrastructure for timeouts

---

## Enforcement

- Unit tests for each transition
- Integration tests for timeout and compensation
- Code review: no implicit transitions or hidden side effects

---

## Example

### REQUIRED
```
START --(OrderPlaced)--> WAIT_PAYMENT --(PaymentSucceeded)--> COMPLETED
WAIT_PAYMENT --(PaymentTimeout)--> COMPENSATE --(CompensationDone)--> FAILED
```

---

## Related ADRs

- [ADR-004](ADR-004-event-driven-saga-orchestration.md): Event-Driven Saga Orchestration
- [ADR-012](ADR-012-transaction-boundary-unitofwork-ownership.md): Transaction Boundary Ownership
- [ADR-022](ADR-022-idempotency-key-standard.md): Idempotency Key Standard
