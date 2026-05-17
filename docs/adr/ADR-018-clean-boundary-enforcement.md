# ADR-018: Clean Architecture Boundary Enforcement by Module

**Status**: ✅ Accepted  
**Date**: 2026-Q1  
**Enforced By**: ArchUnit + Code Review

---

## Context

The platform spans multiple modules (shared-kernel, shared-services, service modules). Without a strict boundary rule set, abstractions leak and architecture drifts silently.

---

## Decision

Define and enforce module boundaries:

| Module | Allowed |
|---|---|
| `shared-kernel` | Domain primitives and base abstractions only — no infrastructure |
| `shared-services` | Infrastructure-only implementations — no domain rules |
| `service-application` | Use cases, ports, policies, CQRS handlers only |
| `service-domain` | Aggregates, entities, domain events only |
| `service-infrastructure` | Adapters, wiring, configuration only |

---

## Constraints

- Application MUST NOT depend on infrastructure packages
- Domain MUST NOT depend on Spring or external libraries
- Infrastructure MUST NOT contain business rules
- Shared-kernel MUST remain framework-agnostic

---

## Enforcement

**ArchUnit rules** verify package boundaries at build time.  
**Code review checklist** covers cross-module dependencies.

A violation here is a CI failure — not a review comment.

---

## Consequences

### Positive
✅ Predictable layering across all services  
✅ Safer refactors — changes are bounded  
✅ Clear ownership boundaries

### Negative
⚠️ More discipline required per PR  
⚠️ Some mapping duplication across layers

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- ADR-002: Infrastructure-Only Shared Services
