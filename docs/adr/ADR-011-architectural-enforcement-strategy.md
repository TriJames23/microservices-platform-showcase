# ADR-011: Architectural Enforcement Strategy

**Status**: Accepted  
**Scope**: Shared Kernel & Shared Services  
**Effective From**: 2026-Q1

---

## Context

As the system grows, architectural decay does not happen through intentional violations, but through:

- **Convenience shortcuts** under delivery pressure
- **Team rotation and onboarding gaps**
- **"Temporary" exceptions** that become permanent
- **Missing feedback loops** between architecture intent and implementation

**Architecture that exists only in documentation inevitably diverges from code.**

---

## Decision

Architecture **SHALL** be enforced as executable rules, not guidelines.

We adopt a **3-layer enforcement model**:

### 1. ADR (Intent Layer)
- Defines **why** a rule exists
- Describes **failure modes** if violated

### 2. Guardrails (Rule Layer)
- Formalizes ADRs into explicit constraints
- Written in human-readable markdown ([`architectural_guardrails.md`](file:///C:/Users/My%20Lap/.gemini/antigravity/brain/edc74793-c703-411e-9edf-ddfb90888fda/architectural_guardrails.md))

### 3. Automated Enforcement (Execution Layer)
- Implemented via ArchUnit tests
- Enforced at build time (CI gate)

**No architectural rule is considered valid unless it exists in all three layers.**

---

## Enforcement Principles

### 1. Architecture Is a Build Gate

- Any architectural violation **fails the build**
- No "temporary ignore" without ADR change
- No waiver via code review comments

### 2. Rules Must Be Mapped to ADRs

Every ArchUnit rule **MUST**:
- Reference exactly one ADR
- Describe the failure mode it prevents

**Example**:
```java
.because("ADR-010: @Transactional in Application layer causes retry storms")
```

### 3. Rules Are Versioned, Not Deleted

When architecture evolves:
- Rules are **deprecated**, not removed
- Deprecation includes:
  - Sunset timeline
  - Migration metrics
  - Successor rule (if any)

### 4. Exceptions Require Explicit Design

If a rule must be violated:
- A new ADR **MUST** be proposed
- Scope, duration, and impact **MUST** be documented
- Rule downgrade (ERROR → WARN) is temporary and tracked

---

## Consequences

### Positive
✅ Architecture becomes enforceable, not aspirational  
✅ Architectural drift is detected early  
✅ Senior architectural knowledge becomes institutional

### Negative
⚠️ Initial friction for developers  
⚠️ Requires discipline in rule evolution

**These tradeoffs are intentional.**

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Architecture guidelines in Confluence" | Not enforceable |
| "Rely on code review" | Not scalable |
| "Allow exceptions via annotations" | Leads to silent erosion |

---

## Related ADRs

- [ADR-001](file:///C:/Users/My%20Lap/.gemini/antigravity/brain/edc74793-c703-411e-9edf-ddfb90888fda/architecture_decision_records.md#adr-001-shared-kernel-purity): Shared Kernel Purity
- [ADR-002](file:///C:/Users/My%20Lap/.gemini/antigravity/brain/edc74793-c703-411e-9edf-ddfb90888fda/architecture_decision_records.md#adr-002-infrastructure-only-shared-services): Infrastructure-Only Shared Services
- [ADR-004](file:///C:/Users/My%20Lap/.gemini/antigravity/brain/edc74793-c703-411e-9edf-ddfb90888fda/architecture_decision_records.md#adr-004-event-driven-saga-orchestration): Event-Driven Saga Orchestration
- [ADR-010](file:///C:/Users/My%20Lap/.gemini/antigravity/brain/edc74793-c703-411e-9edf-ddfb90888fda/architecture_decision_records.md#adr-010-resilience-scope): Resilience Scope
