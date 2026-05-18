# ADR-013: CQRS Implementation Strategy

## Status
**ACCEPTED** - 2026-03-17

## Context
We enforce strict CQRS boundaries across all services so that:
- Command side protects invariants and uses domain models.
- Query side serves read models optimized for reads.
- Application layer remains framework-agnostic and clean.

This ADR defines the core CQRS rules. Detailed guidance for policy usage and async strategy is now separated into:
- ADR-019: Policy Usage Boundaries
- ADR-020: Async Strategy

## Decision

### 1. Strict CQRS Separation
- **Query side** uses Read Models (`IReadModel`) only.
- **Command side** uses Domain Models only.
- Do not mix Read Models in command handlers or domain aggregates in query handlers.

### 2. Core Contracts and Markers
Shared-kernel defines:
- `ICommand<R>` and `ICommandHandler<C, R>`
- `IQuery<R extends IReadModel>` and `IQueryHandler<Q, R>`
- `IReadModel`

These are **behavioral contracts**, not tooling.

### 2.1 Command Bus vs Command Dispatcher (Boundary Clarification)
To preserve transaction boundaries and loose coupling:

- **ICommandBus (Application Port)**:
  - Used only by inbound adapters (Controllers, gRPC servers).
  - Executes handlers directly and returns business results (`CompletableFuture<R>`).

- **ICommandDispatcher (Infrastructure Abstraction)**:
  - Used only by infrastructure processes (Sagas, Workers).
  - Enqueues commands to Outbox/Broker (fire-and-forget: `CompletableFuture<Void>`).

Sagas **MUST NOT** call `ICommandBus` directly.

### 3. Handler Visibility and Mediator Access
All application handlers must be **package-private**.
- Handlers are invoked only through the Mediator/CommandBus/QueryBus.
- No direct access from controllers or other adapters.

### 4. Application Layer Must Be Framework-Agnostic
Application classes must NOT use:
- `@Service`, `@Component`, `@Transactional`
- Any framework annotations or lifecycle hooks

Infrastructure config wires handlers via `@Bean`.

### 5. Policy Usage Boundaries (Summary)
Policies are application-level invariants and are called **only by handlers**.
Controllers and facades must never call policies directly.

See: **ADR-019** for the full policy usage specification and examples.

### 6. Async Strategy (Summary)
Handlers may return `CompletableFuture` to support non-blocking flows.
Application must not expose reactive types or infrastructure-specific async tools.

See: **ADR-020** for the complete async strategy.

### 7. QueryBuilder Abstraction
Application MUST NOT use `IQueryBuilder` or similar query construction tooling.
- Application defines **intent** (ports).
- Infrastructure defines **mechanics** (SQL/JPQL/DSL).

Query repositories expose intent-based methods such as:
- `Optional<UserProfileView> findByUserId(UUID id)`
- `List<UserProfileView> findActiveUsers()`

### 8. Controller and Facade Boundaries
Controllers and facades are transport adapters only:
- Map transport DTOs to commands/queries
- Call Mediator/CommandBus/QueryBus
- Never call repositories or policies

### 9. Result DTO Boundaries
- Command handlers return command-specific result DTOs.
- Query handlers return read models only.
- Never reuse query read models as command results.

### 10. Read Model Ownership
**10.1 Default**: Read Models live in `application.query` and are owned by the service.

**10.2 Shared Read Models**: If a Read Model is shared with other services via API or events,
move it to the **contract module** to avoid cross-service dependency on the Application module.

### 11. Repository Role Clarification (Optional but Recommended)
Write repositories MUST:
- operate on aggregates
- not expose query-specific optimizations

Read repositories MUST:
- operate on read models
- be optimized for query performance
- not enforce domain invariants

### 12. Enforcement
ArchUnit tests enforce:
- Query handlers not using domain models
- Application not using query builder tooling
- Handler visibility rules
- Controllers not calling repositories or policies

## Consequences
- Clear ownership between command and query sides
- Clean application boundary, no infrastructure leakage
- Consistent usage patterns across services

## References
- ADR-019: Policy Usage Boundaries
- ADR-020: Async Strategy
- ADR-013 Addendum: QueryBuilder Ownership and Relocation
- ADR-001: Shared Kernel Purity
