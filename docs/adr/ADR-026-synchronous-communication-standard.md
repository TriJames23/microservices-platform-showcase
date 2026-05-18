# ADR-026: Synchronous Communication Standard (gRPC)

## Status
**ACCEPTED** - 2026-03-18

## Context
Services require reliable, low-latency request-response communication. Without a single
standard, synchronous calls tend to leak transport-specific classes into Application or
Domain layers and violate clean boundaries.

## Decision
### 1. Transport Standard
All synchronous service-to-service communication **must use gRPC**.

### 2. Boundary Rules
- **Application layer defines intent-only Ports** (interfaces).
- **Infrastructure layer implements Ports** using gRPC stubs.
- **gRPC stubs or Protobuf classes must never appear in Domain/Application**.
- **Service-specific .proto files belong to a dedicated transport contract module**.
  Example: `identity-contract-grpc/src/main/proto/identity.proto`.
- **Core service contract modules stay transport-agnostic** and must not contain `.proto` files,
  generated protobuf classes, or transport-engine-specific DTOs.

### 3. Shared Infrastructure Module
Provide an infrastructure-only module:
`shared-services-infrastructure-grpc`, containing only technical concerns:
- gRPC server/client engine configuration
- Interceptors to propagate `correlationId`, `traceId`, `userId`
- Standardized error mapping from gRPC status to platform `ResultError`

### 4. Context Propagation
- Every outbound gRPC call must include:
  - `X-Correlation-ID`
  - `X-Trace-ID`
  - `X-User-ID` (if available)
  - `X-Tenant-ID` (if available)

### 5. Resilience Placement
All gRPC calls from Application Ports must be wrapped in the standard resilience executor
(see ADR-010), but **retry/circuit-breaker must not wrap active transactions**.

### 6. Error Handling
gRPC errors must be mapped into platform `ResultError` (or domain-friendly errors)
at the infrastructure boundary. Do not leak gRPC `StatusRuntimeException` into Application.

## Consequences
- Clean separation between business logic and transport technology
- Simplified testing (ports can be mocked without gRPC server)
- Safe future transport swap (gRPC to RSocket/HTTP2) with no domain changes
- Explicit module boundary: `*-contract` for transport-agnostic business contracts,
  `*-contract-grpc` for protobuf transport contracts

## References
- ADR-001: Shared Kernel Purity
- ADR-002: Infrastructure-only Shared Services
- ADR-010: Resilience Scope
- ADR-021: Contract Versioning Policy
- ADR-027: Transport Contract Boundary
