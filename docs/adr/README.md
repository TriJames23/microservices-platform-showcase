# Architecture Decision Records

This directory contains key architectural decisions that shaped the platform.

Architecture is a **build gate**, not a guideline (ADR-011).
All enforced ADRs have corresponding ArchUnit rules that fail the build on violation.

---

## Highlighted ADRs

| ADR | Title | Enforced By |
|---|---|---|
| [ADR-001](ADR-001-shared-kernel-purity.md) | Shared Kernel Purity | ArchUnit (5 rules) — 🔴 BLOCKER |
| [ADR-005](ADR-005-outbox-pattern.md) | Outbox Pattern for Integration Events | Code Review |
| [ADR-015](ADR-015-inbox-pattern.md) | Inbox Pattern (Idempotent Consumers) | Code Review + Integration Tests |
| [ADR-018](ADR-018-clean-boundary-enforcement.md) | Clean Architecture Boundary Enforcement | ArchUnit + Code Review |

---

## Full ADR Index

All 29 active ADRs cover decisions across:
- Kernel and infrastructure boundaries (ADR-001, 002, 006, 007, 018)
- Event-driven coordination (ADR-004, 005, 015, 016, 017, 020)
- CQRS and transaction semantics (ADR-012, 013)
- Saga and resilience (ADR-010, 014, 028)
- Security and authorization (ADR-003, 019, 023, 024)
- Observability (ADR-008)
- Contract versioning and communication standards (ADR-021, 026, 027)
- Idempotency (ADR-022)
- Operational concerns (ADR-025)
- Enforcement strategy (ADR-011)

> Source ADRs live in:
> `shared-services/shared-services-infrastructure-architecture-tests/architecture/decisions/active/`
