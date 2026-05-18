# ADR-024: Multi-Tenant Boundary

## Status
**ACCEPTED** - 2026-03-17

## Context
If the platform serves multiple tenants, boundaries for isolation and
partitioning must be explicit to avoid data leaks and scaling issues.

## Decision
### 1. Tenant Context
Tenant ID is part of the execution context and is required for all writes.

### 2. Data Isolation Strategy
Allowed strategies (choose per service):
- Shared database with tenant discriminator
- Separate schema per tenant
- Separate database per tenant

### 3. Enforcement
- All repository queries must include tenant scope
- Cross-tenant access is forbidden by default

### 4. Messaging
Tenant ID must be included in message headers and outbox records.

## Consequences
- Clear tenant boundaries
- Reduced risk of data leakage

## References
- ADR-003: Execution Context Ownership
