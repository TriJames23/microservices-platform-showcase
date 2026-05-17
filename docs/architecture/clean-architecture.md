# Clean Architecture

## The Boundary Model

Every service is structured in strict, concentric layers. Framework leakage across these boundaries is a **build failure**, not a review comment.

```
┌──────────────────────────────────────────────────┐
│                  Transport Layer                  │
│        HTTP Controllers · gRPC Adapters          │
│        (thin — calls ICommandBus/IQueryBus only) │
└─────────────────────┬────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────┐
│                Application Layer                  │
│  UseCase Handlers · Validators · Policy Classes  │
│  (orchestrates domain, owns transaction boundary)│
└─────────────────────┬────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────┐
│                  Domain Layer                     │
│   Aggregates · Value Objects · Domain Events     │
│   (zero framework, pure business logic)          │
└─────────────────────┬────────────────────────────┘
                      │
┌─────────────────────▼────────────────────────────┐
│              Infrastructure Layer                 │
│  Persistence · Messaging · gRPC · Resource · ... │
│  (implements ports defined by inner layers)      │
└──────────────────────────────────────────────────┘
```

---

## Kernel Purity

`shared-kernel` is the framework-agnostic source of truth. It must never contain:

- Spring, JPA, or transport types
- HTTP/gRPC/storage semantics
- Metrics, alert expressions, or tracing runtime helpers
- Service-specific runtime policy or adapter code

**Allowed in kernel:**
- Value objects, `Result<T>`, ports, commands, queries, events
- `IExecutionContext`, `IStructuredLogger`
- `ISagaDefinition`, `ICommandDispatcher`, `IIntegrationCommandTypeRegistry`
- `IAggregateRepository<T, TId>`, `IReadRepository<TProjection, TId>`

---

## Feature Shape Contract

Every command and every query feature must expose exactly **six classes**:

| Class | Role |
|---|---|
| `marker` | Application entrypoint, carries `@IAuthorize` |
| `request` | DTO-in: input payload |
| `result` | DTO-out: output payload |
| `validator` | First-class validation dependency |
| `handler` | Orchestration only — package-private |
| `factory` | Registration wiring |

> Query side is **not** allowed to skip `request` or `validator` because the payload is small.

---

## Authorization Model

Authorization intent is **declared on the marker** with `@IAuthorize` and enforced by `AuthorizationBehavior` in the pipeline.

```
✅ CORRECT
@IAuthorize(roles = {"ADMIN"})
public record CreateOrderCommand implements ICommand { ... }

❌ FORBIDDEN
handler.execute() {
    if (context.getRoles().contains("ADMIN")) { ... } // manual check
}
```

Handlers must not manually inspect roles through `IExecutionContext`.

---

## Transport Contract Boundary

`.proto` files, generated protobuf classes, and gRPC stubs have a hard module boundary:

| Module | Contains |
|---|---|
| `*-contract` | Transport-agnostic core contracts only |
| `*-contract-grpc` | Protobuf transport contracts only |

Core and gRPC contracts must **never** be colocated.

---

## Common Violations (Enforced by ArchUnit)

| Violation | Rule |
|---|---|
| Spring/JPA types in `shared-kernel` | Kernel must stay framework-agnostic |
| Repository access from controller | Transport layer calls bus only |
| Manual role checks in handler | Use `@IAuthorize` on the marker |
| `ValidationResult.withErrors(...)` | Use `new ValidationResult(errors)` |
| `.proto` files in `*-contract` | Must live in `*-contract-grpc` |
| Inline handler construction in config | Use root factory + sub-factories |
