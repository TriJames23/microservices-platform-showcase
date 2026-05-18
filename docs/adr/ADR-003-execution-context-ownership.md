# ADR-003: Execution Context Ownership

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: ArchUnit

---

## Context

We need to propagate Identity, Tenant, and Trace IDs across layers without polluting domain signatures with infrastructure concerns.

---

## Decision

`ExecutionContext` is **owned exclusively by infrastructure-context**.

### Constraints

- **Created and populated ONLY** by infrastructure (Filters, Interceptors, TaskDecorators)
- **Read-only** for Application and Domain layers
- **NOT injectable** into Domain objects
- Application layer **MAY** read from ExecutionContext only via abstraction
- **NEVER** pass ExecutionContext as method parameter

### Infrastructure Integration Requirement

All infrastructure modules that:
- perform IO operations
- generate external access artifacts (e.g., signed URLs, tokens)
- participate in distributed workflows

MUST integrate with ExecutionContext to ensure:
- trace continuity
- auditability
- consistent identity propagation

---

## Consequences

### Positive
✅ Clean domain signatures (no infrastructure leakage)  
✅ Automatic propagation across async boundaries  
✅ Centralized context management  
✅ Easy to add new context fields without changing signatures

### Negative
⚠️ Requires ThreadLocal management (careful with async)  
⚠️ Cannot see context in method signatures (less explicit)

---

## Enforcement

**ArchUnit Rule**:
```java
domain_mustNotDependOn_ExecutionContext()
```

**Severity**: 🟡 MEDIUM

---

## Example

### ❌ WRONG
```java
// Domain
public class Order {
    public void confirm(ExecutionContext context) { // ← VIOLATION
        this.confirmedBy = context.getUserId();
    }
}

// Application
@Service
public class OrderService {
    public void confirmOrder(OrderId id, ExecutionContext context) { // ← VIOLATION
        Order order = repository.findById(id);
        order.confirm(context);
    }
}
```

### ✅ CORRECT
```java
// Domain (Pure)
public class Order {
    public void confirm(UserId confirmedBy) { // ✅ Explicit parameter
        this.confirmedBy = confirmedBy;
    }
}

// Application (Reads context via abstraction)
@Service
public class OrderService {
    private final IExecutionContext executionContext;
    
    public void confirmOrder(OrderId id) { // ✅ Clean signature
        UserId currentUser = executionContext.getUserId();
        Order order = repository.findById(id);
        order.confirm(currentUser);
    }
}

// Infrastructure (Populates context)
@Component
public class ExecutionContextFilter implements Filter {
    public void doFilter(ServletRequest request, ...) {
        ExecutionContext.set(new ExecutionContext(
            extractUserId(request),
            extractTenantId(request),
            extractTraceId(request)
        ));
    }
}
```

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Pass ExecutionContext through method signatures" | Pollutes domain with infrastructure |
| "Mutate context inside Application logic" | Breaks single responsibility |
| "Inject ExecutionContext into Domain" | Violates ADR-001 (Domain Purity) |

---

## Related ADRs

- [ADR-001](ADR-001-shared-kernel-purity.md): Shared Kernel Purity
- [ADR-008](ADR-008-passive-observability.md): Passive Observability
