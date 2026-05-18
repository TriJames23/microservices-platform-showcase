# ADR-002: Infrastructure-Only Shared Services

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: ArchUnit (Rule B)

---

## Context

Developers were mixing utility logic and business logic in the `shared-services` modules, creating hidden coupling and architectural drift.

---

## Decision

`shared-services` modules must contain **ONLY Infrastructure concerns** (IO, Security, Serialization, Networking).

### Constraints

- All business rules **MUST** reside in `shared-kernel` or the specific Service's Domain
- `shared-services` **MUST NOT** expose reusable business APIs
- All access must happen via Application Ports or runtime wiring
- Infrastructure **MAY** depend on `..application.ports..` but **MUST NOT** depend on `..application.services..`

---

## Consequences

### Positive
✅ Clear separation of concerns  
✅ `shared-services` can be upgraded/replaced without breaking business rules  
✅ Prevents monolith creep in microservices architecture  
✅ Testability - Infrastructure can be mocked via Ports

### Negative
⚠️ Requires discipline to not put "convenient" utilities in Infrastructure  
⚠️ Developers must create Port interfaces for Infrastructure→Application communication

---

## Enforcement

**ArchUnit Rule B**:
```java
ruleB_infrastructure_mustNotDependOn_ApplicationServices()
```

**Severity**: 🟠 HIGH

**What it prevents**:
- Hidden coupling (Infrastructure controlling business flow)
- Circular dependencies (Application → Infrastructure → Application)
- Deployment rigidity (Cannot deploy infrastructure changes independently)

---

## Example

### ❌ WRONG
```java
@Component
class JpaOutboxPublisher {
    @Autowired
    private OrderApplicationService orderService; // ← VIOLATION
    
    public void publishOutboxEvents() {
        orderService.compensateOrder(...); // Infrastructure controlling application flow
    }
}
```

### ✅ CORRECT
```java
// 1. Create Port in Application layer
package com.example.application.ports;

public interface OrderCompensationPort {
    void compensate(OrderId orderId);
}

// 2. Application Service implements Port
@Service
class OrderApplicationService implements OrderCompensationPort {
    @Override
    public void compensate(OrderId orderId) {
        // Application logic
    }
}

// 3. Infrastructure depends on Port
@Component
class JpaOutboxPublisher {
    @Autowired
    private OrderCompensationPort compensationPort; // ✅ Interface
    
    public void publishOutboxEvents() {
        compensationPort.compensate(...); // Decoupled
    }
}
```

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Allow Infrastructure→Service for convenience" | Leads to monolith creep |
| "Use events for everything" | Over-engineering for simple cases |
| "Shared business utilities module" | Violates single responsibility |

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-011](ADR-011-architectural-enforcement-strategy.md): Enforcement Strategy
