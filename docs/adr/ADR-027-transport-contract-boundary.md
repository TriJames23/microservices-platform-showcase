# ADR-027: Transport Contract Boundary

## Status
**ACCEPTED** - 2026-04-14

## Context
Service-owned contract modules currently risk mixing transport-agnostic business
contracts with transport-engine-specific assets such as gRPC `.proto` files and
generated protobuf stubs. When those concerns are colocated, service reactors
lose a clean boundary between business-facing contract truth and transport detail.

## Decision
### 1. Core Contract Boundary
- `*-contract` modules are the service-owned core contract boundary.
- They may contain:
  - transport-agnostic command/query DTOs
  - integration events
  - public business records shared across adapters
- They must not contain:
  - `.proto` files
  - generated protobuf or gRPC stub classes
  - transport-engine-specific schema assets

### 2. Transport Contract Boundary
- Transport-specific schemas must live in dedicated modules per transport.
- For gRPC, the module name is `*-contract-grpc`.
- `*-contract-grpc` owns:
  - `.proto` definitions
  - generated protobuf models
  - generated gRPC stubs

### 3. Dependency Direction
- `*-infrastructure-grpc` depends on `*-contract-grpc`.
- Domain and application layers must not depend on `*-contract-grpc`.
- `*-contract` remains transport-agnostic and may be consumed by messaging, HTTP
  adapters, and cross-service business integrations when the payload is not
  transport-engine-specific.

### 4. Scope
- This ADR formalizes the gRPC module boundary now.
- REST/OpenAPI transport modules may be introduced later if public shared REST
  schema reuse is required, but that is outside the current wave.

## Consequences
- Clearer contract ownership
- Reduced accidental transport leakage into business-facing contract modules
- Cleaner template scaffolding for new services

## References
- ADR-021: Contract Versioning Policy
- ADR-026: Synchronous Communication Standard (gRPC)
