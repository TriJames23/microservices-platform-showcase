# ADR-007: Zero-Logic Starters

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: ArchUnit

---

## Context

"Starter" modules often become dumping grounds for shared default logic, making them heavy and opinionated.

---

## Decision

The `starter` module **MUST** contain **ONLY**:
- Dependency declarations (`pom.xml`)
- `AutoConfiguration` classes

It **MUST NOT** contain:
- Services
- Utilities
- Conditional business defaults
- Domain logic
 - Provider selection logic
 - Dependencies on concrete provider implementations

It **MAY**:
- Assemble abstractions
- Wire collections of implementations
- Enforce fail-fast guardrails

---

## Consequences

### Positive
✅ Lightweight consumers (minimal transitive dependencies)  
✅ Explicit opt-in (no hidden behavior)  
✅ Easy to understand what starter provides  
✅ Fast startup time

### Negative
⚠️ Cannot provide "smart defaults"  
⚠️ Consumers must configure more explicitly

---

## Enforcement

**ArchUnit Rule**:
```java
starterModule_mustNotContainDomainLogic()
```

**Severity**: 🟡 MEDIUM

---

## Example

### ❌ WRONG
```java
// shared-services-infrastructure-starter/src/main/java/.../DefaultOrderValidator.java
@Component
public class DefaultOrderValidator { // ← VIOLATION: Business logic in starter
    public boolean validate(Order order) {
        return order.getTotal() > 0 && order.getItems().size() > 0;
    }
}

// AutoConfiguration
@Configuration
public class StarterAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public OrderValidator orderValidator() {
        return new DefaultOrderValidator(); // ← VIOLATION: Opinionated default
    }
}
```

### ✅ CORRECT
```java
// shared-services-infrastructure-starter/pom.xml
<dependencies>
    <dependency>
        <groupId>com.betatrem</groupId>
        <artifactId>shared-services-infrastructure-outbox</artifactId>
    </dependency>
    <dependency>
        <groupId>com.betatrem</groupId>
        <artifactId>shared-services-infrastructure-saga</artifactId>
    </dependency>
</dependencies>

// shared-services-infrastructure-starter/.../StarterAutoConfiguration.java
@Configuration
public class StarterAutoConfiguration {
    // ✅ Only wiring, no business logic
    @Bean
    @ConditionalOnMissingBean
    public OutboxPublisher outboxPublisher(OutboxRepository repository) {
        return new JpaOutboxPublisher(repository);
    }
}
```

---

## What Belongs Where

| Concern | Location | Example |
|:--------|:---------|:--------|
| Dependency aggregation | ✅ Starter | `pom.xml` with all infrastructure modules |
| Bean wiring | ✅ Starter | `@Configuration` classes |
| Default configuration values | ✅ Starter | `application.yml` defaults |
| Business logic | ❌ NOT Starter | Validators, calculators, rules |
| Utilities | ❌ NOT Starter | String helpers, date formatters |
| Conditional defaults | ❌ NOT Starter | "If X then use Y" logic |

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Smart defaults in starter" | Becomes opinionated and heavy |
| "Opinionated fallback logic" | Hidden behavior, hard to override |
| "Utility classes in starter" | Violates single responsibility |

---

## Related ADRs

- [ADR-002](ADR-002-infrastructure-only-shared-services.md): Infrastructure-Only Shared Services
