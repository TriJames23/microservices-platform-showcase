# ADR-019: Policy Usage Boundaries

## Status
**ACCEPTED** - 2026-03-17

## Context
Policies encode application-level invariants. Without strict boundaries,
policies leak into controllers or facades and violate Clean Architecture.

This ADR extracts the policy guidance from ADR-013 to keep CQRS rules focused.

## Decision
### 1. Ownership
Policies live in the **Application** layer, under `application.policy`.

### 2. Allowed Callers
Policies may be called **only by application handlers**:
- `ICommandHandler` implementations
- `IQueryHandler` implementations (rare, when validation of read is needed)

### 3. Forbidden Callers
The following MUST NOT call policies:
- Controllers / REST adapters
- Facades / orchestration adapters
- Infrastructure components

### 4. Rationale
- Controllers are transport adapters only.
- Policies represent business invariants and belong to use cases.
- Keeping policies in handlers preserves testability and boundary purity.

### 5. Enforcement
ArchUnit rules in `CQRSArchitectureTest` enforce:
- Controllers must not depend on `application.policy`
- Facades must not depend on `application.policy`

## Consequences
- Clean separation between transport and business logic
- Policies remain consistent and testable
- Reduced coupling across layers

## References
- ADR-013: CQRS Implementation Strategy
- ADR-001: Shared Kernel Purity
