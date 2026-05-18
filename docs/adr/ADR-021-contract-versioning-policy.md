# ADR-021: Contract Versioning Policy (API + Event)

## Status
**ACCEPTED** - 2026-03-17

## Context
Multiple services depend on shared API and event contracts. Without a versioning
policy, changes risk breaking consumers and blocking deployment.

## Decision
### 1. Semantic Compatibility Rules
- **Backward compatible** changes are allowed without version bump:
  - Add optional fields (nullable or with defaults)
  - Add new endpoints (non-breaking)
- **Breaking** changes require a new major version:
  - Remove or rename fields
  - Change field types or meaning
  - Change event names or routing keys

### 2. Versioning for API Contracts
- API contracts use **semantic versioning**: `MAJOR.MINOR`.
- Major version in URL path or header:
  - `/v1/...`
  - Or `X-Api-Version: 1`
- Transport-specific schemas are versioned independently from transport-agnostic core
  contract records. A `*-contract-grpc` module may evolve protobuf transport details without
  changing a service's core `*-contract` payloads when the semantic contract is unchanged.

### 3. Versioning for Event Contracts
- Event schema uses `SCHEMA_VERSION` in headers.
- Use **schema registry** or versioned topic names if needed.
- New versions must preserve backward compatibility unless a hard break is planned.

### 4. Deprecation Policy
- Breaking changes require a deprecation window.
- Minimum: 2 release cycles or 60 days (whichever is longer).
- Deprecation announcements must be published to all consumers.

### 5. Enforcement
Contract changes must include:
- Version bump (if breaking)
- Compatibility notes
- Migration plan
- Explicit module-boundary review when `.proto` files, generated stubs, or other
  transport-specific assets are touched

## Consequences
- Predictable changes for consumers
- Reduced downtime during upgrades

## References
- ADR-017: Event Schema Versioning
- ADR-026: Synchronous Communication Standard (gRPC)
- ADR-027: Transport Contract Boundary
