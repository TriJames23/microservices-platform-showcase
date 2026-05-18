# ADR-001: Shared Kernel Purity

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: ArchUnit (5 rules)

---

## Context

We need a core business logic layer that is reusable across multiple microservices without imposing heavy framework dependencies or version conflicts.

---

## Decision

The `shared-kernel` (domain/application) will be **Pure Java**.

### Constraints

- **MUST NOT** depend on Spring, Hibernate, or any IO libraries
- **MUST NOT** depend on framework or code-generation libraries (Jackson, Lombok, Jakarta/JavaEE)
- **MUST** remain framework-agnostic

### Allowed Dependencies

- JDK standard library only
- Pure Java libraries (e.g., Vavr for functional programming)

---

## Consequences

### Positive
✅ High testability (sub-millisecond unit tests)  
✅ Zero classpath pollution for consumers  
✅ Framework independence (can migrate Spring → Micronaut without touching domain)

### Negative
⚠️ Requires mapping logic in Infrastructure layer  
⚠️ Cannot use convenient framework annotations in domain

---

## Enforcement

**ArchUnit Rules**:
- `sharedKernelDomain_mustNotDependOn_Spring()`
- `sharedKernelDomain_mustNotDependOn_Jakarta()`
- `sharedKernelDomain_mustNotDependOn_Jackson()`
- `sharedKernelDomain_mustNotDependOn_ApplicationLayer()`
- `sharedKernelDomain_mustNotDependOn_Infrastructure()`

**Severity**: 🔴 BLOCKER

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Spring Data Entities in Domain" | Couples domain to JPA lifecycle |
| "Lombok in Domain" | Code generation obscures business logic |
| "Jackson annotations in Domain" | Couples domain to serialization format |

---

## Related ADRs

- [ADR-002](ADR-002-infrastructure-only-shared-services.md): Infrastructure-Only Shared Services
- [ADR-008](ADR-008-passive-observability.md): Passive Observability
