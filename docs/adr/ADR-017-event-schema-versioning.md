# ADR-017: Event Schema Versioning and Compatibility

**Status**: Accepted  
**Date**: 2026-Q1  
**Enforced By**: Code Review + Contract Tests

---

## Context

ADR-013 mentions `SCHEMA_VERSION`, but does not define compatibility rules. Without a policy, event evolution becomes unsafe and breaks consumers.

---

## Decision

All integration events MUST include a schema version and follow backward compatibility rules.

Rules:
- Additive changes are backward compatible
- Removing or renaming fields is breaking
- Breaking changes require a new major version or a new event type

Schema registry (or equivalent) MUST be used to validate compatibility on publish.

---

## Constraints

- Producers MUST publish schema version in headers
- Consumers MUST accept at least the last two minor versions
- Breaking changes MUST NOT be deployed without consumer migration plan

---

## Consequences

### Positive
- Safer event evolution
- Predictable consumer upgrades

### Negative
- Requires schema governance
- More work for breaking changes

---

## Enforcement

- Contract tests for version compatibility
- Release checklist requires schema review

---

## Related ADRs

- [ADR-005](ADR-005-outbox-pattern.md): Outbox Pattern
- [ADR-013](ADR-013-CQRS-Implementation-Strategy.md): CQRS Implementation Strategy
- [ADR-021](ADR-021-contract-versioning-policy.md): Contract Versioning Policy
