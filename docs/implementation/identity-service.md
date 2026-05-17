# Identity Service

**Repository**: `identity-service`

User identity management — registration, profile, and credential lifecycle.

---

## Architecture

Clean Architecture applied strictly. Boundary violations are caught by ArchUnit at build time.

```
identity-service/
├── identity-domain/              # Aggregates, value objects, domain events
├── identity-application/         # Use cases, ports, CQRS handlers
├── identity-contract/            # Transport-agnostic API contracts
├── identity-contract-grpc/       # Protobuf contracts (gRPC only)
├── identity-infrastructure/      # Root wiring
│   ├── *-persistence/            # JPA adapters, mappers, reconstitution
│   ├── *-web/                    # HTTP controllers (thin)
│   ├── *-grpc/                   # gRPC adapters (thin)
│   ├── *-messaging/              # Integration event handlers
│   └── *-security/               # Security adapter
└── identity-boot/                # Spring Boot application entry point
```

---

## Domain Model

Key aggregate: **`User`** — owns identity credentials, verification state, and profile data.

```java
// Aggregate extends AggregateRootBase
// Restore path MUST call:
user.setVersionForReconstitution(version);
user.setDeletedForReconstitution(deleted);
```

---

## CQRS Feature Shape

Every command and query follows the 6-class contract:

```
RegisterUserCommand:
  marker   → RegisterUserMarker (@IAuthorize declared here)
  request  → RegisterUserRequest
  result   → RegisterUserResult
  validator→ RegisterUserValidator
  handler  → RegisterUserHandler (package-private, orchestration only)
  factory  → RegisterUserCommandFactory
```

---

## Transport

| Transport | Role |
|---|---|
| HTTP REST | External client API — calls `ICommandBus` / `IQueryBus` only |
| gRPC | Inter-service identity verification (pre-check semantics) |

Controllers and gRPC adapters are thin — zero business logic at transport layer.

---

## Integration

| Event | Direction |
|---|---|
| `UserRegisteredIntegrationEvent` | Emitted via Outbox → Kafka on successful registration |
| `UserVerifiedIntegrationEvent` | Emitted on email/phone verification |

Cross-service callers consume integration events — not direct API calls.

---

## Platform Features Used

| Feature | Config |
|---|---|
| Outbox | `platform.outbox.enabled=true` |
| Inbox | `platform.inbox.enabled=true` |
| Idempotency | `platform.idempotency.provider=persistence` |
| Observability | `/actuator/health`, `/actuator/prometheus` |
| gRPC (prod) | `platform.grpc.server.enabled=true` |

---

## Deployment

- One `values.yaml` per environment: `environments/{env}/identity-service/values.yaml`
- One ArgoCD application per environment
- HPA enabled — CPU-based autoscaling
