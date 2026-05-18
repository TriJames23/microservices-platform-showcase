# ADR-020: Async Strategy

## Status
**ACCEPTED** - 2026-03-17

## Context
Async behavior impacts reliability, testability, and clean boundaries.
We need a consistent rule set for where async is allowed and which types are exposed.

This ADR extracts async guidance from ADR-013 to keep CQRS rules focused.

## Decision
### 1. Application-Level Async Type
Application handlers may return `CompletableFuture<R>` to support non-blocking flows.
- Application must remain framework-agnostic.
- Do not expose reactive types (`Mono`, `Flux`) in application contracts.

### 2. Where Async Is Implemented
Async execution details live in **Infrastructure**:
- Thread pools
- Schedulers
- `@Async` or framework execution

Application defines **intent** only.

### 3. Command and Query Buses
CommandBus/QueryBus/Mediator may execute handlers asynchronously,
but the handler contract remains stable and framework-agnostic.

### 4. Saga and Messaging
Long-running workflows use:
- Saga state machines
- Outbox/Inbox patterns
- Worker executors

These are infrastructure concerns and must not leak into application APIs.

### 5. Error Handling
Async errors must be translated to domain-appropriate results or propagated
through the bus layer. Infrastructure handles retries, DLQ, and backoff.

## Consequences
- Consistent async model across services
- Stable application contracts
- Clear responsibility split between application and infrastructure

## References
- ADR-013: CQRS Implementation Strategy
- ADR-016: DLQ and Error Handling
- ADR-014: Saga State Machine Details
