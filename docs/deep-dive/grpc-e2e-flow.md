# gRPC E2E Flow

> **Deep Dive** — Full technical sequence for gRPC request handling.

Source: `book/04-grpc-e2e-flow.md`

---

## Key Principle

gRPC is **not** a shortcut for cross-service state changes. It is limited to:
- Pre-check semantics (read-only queries before local commit)
- Availability/price checks before starting a saga

Cross-service **state mutations** always go through Outbox → Inbox.

---

## Full Sequence

```
1. gRPC request arrives with metadata (JWT, correlation, tenant)

2. ContextPropagationServerInterceptor
   → Extracts execution context from gRPC metadata
   → Propagates to ExecutionContextHolder

3. PassiveIdempotencyLoggingServerInterceptor
   → Logs idempotency metadata passively
   ⚠️  This is LOGGING ONLY — NOT business idempotency enforcement

4. gRPC service adapter
   → Maps protobuf request → feature request DTO
   → Wraps in command/query marker
   → Calls ICommandBus or IQueryBus (SAME bus as HTTP)

5. Pipeline runs: validation → authorization → handler
   (identical to HTTP — transport is irrelevant at application layer)

6. Handler returns Result<T>

7. TemplateGrpcErrorMapper
   → Maps Result<T> failures → StatusRuntimeException
   → Maps unexpected exceptions → StatusRuntimeException

8. gRPC response sent
```

---

## Module Boundaries

| Concern | Module |
|---|---|
| `.proto` files and generated stubs | `*-contract-grpc` only |
| Core business contracts | `*-contract` (transport-agnostic) |
| gRPC adapter wiring | `template-infrastructure-grpc` |
| Server platform wiring | `GrpcPlatformAutoConfiguration` |
| Cross-service command dispatch | `GrpcCommandDispatchAutoConfiguration` |

`.proto` files MUST NOT coexist with core contract modules.

---

## Authorization

gRPC adapters must not bypass `@IAuthorize` by performing ad-hoc role checks from `IExecutionContext`.

Authorization is declared on the **marker** — `AuthorizationBehavior` enforces it in the pipeline.

---

## Configuration

```yaml
platform:
  grpc:
    server:
      enabled: true       # false in local sandbox
    security:
      jwt:
        enabled: true
```
