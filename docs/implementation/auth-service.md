# Auth Service

**Repository**: `auth-service`

Token lifecycle management — JWT issuance, refresh, revocation, and authorization enforcement.

---

## Architecture

Clean Architecture with strict layer boundaries enforced by ArchUnit.

```
auth-service/
├── auth-domain/              # Token aggregates, auth domain events
├── auth-application/         # Token use cases, authorization policies
├── auth-contract/            # Transport-agnostic auth contracts
├── auth-contract-grpc/       # gRPC protobuf contracts
├── auth-infrastructure/      # Root wiring
│   ├── *-persistence/        # Token store adapters
│   ├── *-security/           # JWT adapter, security filter chain
│   ├── *-web/                # HTTP endpoints (thin)
│   └── *-grpc/               # gRPC verification service (thin)
└── auth-boot/                # Spring Boot application entry point
```

---

## Key Responsibilities

| Responsibility | Mechanism |
|---|---|
| JWT issuance | Application use case → domain → JWT adapter |
| Token refresh | Stateless rotation via secure refresh token |
| Token revocation | Persistent revocation list (Redis-backed) |
| Authorization enforcement | `AuthorizationBehavior` in shared pipeline |
| Service-to-service auth | gRPC JWT verification — pre-check semantics |

---

## Authorization Model

Authorization intent is declared on feature markers:

```java
@IAuthorize(roles = {"USER"})
public record RefreshTokenMarker implements ICommand { ... }
```

`AuthorizationBehavior` enforces it in the pipeline. Handlers must not manually inspect roles.

---

## Security Filter Chain

```
HTTP Request → JWT filter
                  → SecurityIdentityFilter (enriches IExecutionContext from verified JWT)
                  → Pipeline (AuthorizationBehavior enforces @IAuthorize)
                  → Handler
```

`IExecutionContext` carries subject, tenant, correlation from verified credentials — not from spoofable headers.

---

## gRPC Verification

Other services call `auth-service` via gRPC for token verification (read-only, pre-check semantics).

```yaml
# Other services call auth-service gRPC for verification
platform:
  security:
    grpc:
      jwt:
        enabled: true
```

---

## Integration

| Event | Direction |
|---|---|
| `TokenIssuedIntegrationEvent` | Emitted on successful token issuance |
| `TokenRevokedIntegrationEvent` | Emitted on revocation — consumers clear local caches |

---

## Platform Features Used

| Feature | Config |
|---|---|
| Redis | Token revocation store — `redis.enabled=true` |
| Outbox | `platform.outbox.enabled=true` |
| Idempotency | `platform.idempotency.provider=redis` |
| gRPC server | `platform.grpc.server.enabled=true` |
| Observability | `/actuator/health`, `/actuator/prometheus` |
