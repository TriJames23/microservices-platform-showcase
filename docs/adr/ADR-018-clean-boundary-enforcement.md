# ADR-018: Clean Architecture Boundary Enforcement by Module

**Status**: Accepted  
**Date**: 2026-Q1  
**Enforced By**: ArchUnit + Code Review

---

## Context

The platform spans multiple modules (shared-kernel, shared-services, service modules). Without a strict boundary rule set, abstractions leak and architecture drifts.

---

## Decision

Define and enforce module boundaries:

- **shared-kernel**: domain primitives and base abstractions only, no infrastructure
- **shared-services**: infrastructure-only implementations, no domain rules
- **service-application**: use cases, ports, policies, CQRS handlers only
- **service-domain**: aggregates, entities, domain events only
- **service-infrastructure**: adapters, wiring, configuration only

---

## Constraints

- Application MUST NOT depend on infrastructure packages
- Domain MUST NOT depend on Spring or external libraries
- Infrastructure MUST NOT contain business rules
- Shared-kernel MUST remain framework-agnostic

---

## Consequences

### Positive
- Predictable layering
- Safer refactors
- Clear ownership boundaries

### Negative
- More discipline required
- Some duplication across services

---

## Enforcement

- ArchUnit rules for package boundaries
- Code review checklist for cross-module dependencies

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-002](ADR-002-infrastructure-only-shared-services.md): Shared Services Scope

