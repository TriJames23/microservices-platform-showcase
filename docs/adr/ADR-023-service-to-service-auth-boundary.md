# ADR-023: Service-to-Service Auth Boundary

## Status
**ACCEPTED** - 2026-03-17

## Context
Internal services need consistent authentication and authorization rules.
Without a boundary, services implement custom logic and drift over time.

## Decision
### 1. Identity Types
Support two identities:
- **User identity** (end-user request)
- **Service identity** (machine-to-machine)

### 2. Token Propagation
- Incoming user tokens are propagated to downstream services.
- Service tokens are used only for background jobs and internal workers.

### 3. Authorization Rules
- Each service enforces its own authorization.
- No service should rely on upstream enforcement alone.

### 4. Claims Contract
Standard claims:
- `sub` (subject)
- `tenant_id`
- `roles` or `scopes`
- `iss`, `aud`, `exp`

### 5. Boundary Enforcement
Infrastructure layer validates tokens and extracts principal context.
Application layer only receives the principal abstraction, not JWT details.

## Consequences
- Consistent security across services
- Clear separation between auth concerns and business logic

## References
- ADR-003: Execution Context Ownership
