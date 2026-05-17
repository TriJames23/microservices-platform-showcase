# System Overview

## What This Platform Is

A production-grade **microservices commerce platform** built on Clean Architecture, Event-Driven Design, and GitOps delivery.

The platform is not a prototype. Architecture is enforced at build time via ArchUnit — boundary violations are CI failures, not code review opinions.

---

## Services

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Layer                             │
│                  (HTTP REST  /  gRPC)                           │
└─────────────┬───────────────────────────────────────┬───────────┘
              │                                       │
    ┌─────────▼──────────┐                 ┌──────────▼──────────┐
    │   identity-service │                 │    auth-service      │
    │   (User identity)  │                 │ (Token / AuthZ)      │
    └────────────────────┘                 └─────────────────────┘

┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│catalog-service│  │  cart-service│  │  shop-service│  │order-service  │
│(search/ES)   │  │(cart mgmt)   │  │(multi-shop)  │  │(saga orchestr)│
└──────────────┘  └──────────────┘  └──────────────┘  └──────┬───────┘
                                                              │ Saga
                                         ┌────────────────────┼──────────────────┐
                                         │                    │                  │
                               ┌─────────▼────┐  ┌───────────▼──┐  ┌────────────▼───┐
                               │inventory-svc  │  │payment-svc   │  │notification-svc│
                               │(reserve/rel.) │  │(process/ref.)│  │(WS + gRPC str.)│
                               └──────────────┘  └──────────────┘  └────────────────┘

─────────────────────────── Shared Platform ─────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────────┐
│  shared-services  │  Outbox · Inbox · Saga · gRPC · Messaging · Observability  │
│  shared-kernel    │  Domain · Application · Contracts (framework-agnostic)     │
└─────────────────────────────────────────────────────────────────────────────────┘

─────────────────────────── Infrastructure ─────────────────────────────────────────

  PostgreSQL · Kafka · Elasticsearch · Redis · Prometheus · Grafana · Kubernetes
```

---

## Integration Model

| Concern | Mechanism |
|---|---|
| Synchronous query | HTTP REST or gRPC (read-only / pre-check only) |
| Cross-service state change | Outbox → Kafka → Inbox → `ICommandBus` |
| Multi-step coordination | Saga via integration commands |
| Search | Elasticsearch via `shared-services-infrastructure-search` |
| Notifications | WebSocket + gRPC streaming via `notification-service` |
| Identity propagation | `IExecutionContext` — tenant, correlation, trace across all lanes |

---

## Shared Platform Libraries

**`shared-kernel`** — The framework-agnostic source of truth:
- `ICommandBus`, `IQueryBus` — transport-neutral entrypoints
- `Result<T>` — explicit application outcome envelope
- `IExecutionContext` — caller identity, tenant, correlation, trace
- `ISagaDefinition`, `ISagaStepHandler` — saga state-machine contracts
- `IAggregateRepository`, `IReadRepository` — persistence semantic bases
- `ICommandDispatcher`, `IIntegrationCommandTypeRegistry` — integration boundaries

**`shared-services`** — Spring adapter implementations:
- All kernel ports implemented as Spring auto-configuration modules
- Outbox, Inbox, Saga, gRPC, Worker, Observability, Redis, Resource, Search, Resilience
- `shared-services-infrastructure-starter` wires all modules
- `shared-services-infrastructure-architecture-tests` enforces the platform contract at build time

---

## Architectural Enforcement

Architecture is a **build gate**, not a guideline.

```
3-layer enforcement model (ADR-011):

  ADRs         →  Intent: why rules exist and what failure modes look like
  Guardrails   →  Rules: explicit constraints in human-readable form
  ArchUnit     →  Execution: automated enforcement at CI build time
```

Run all architectural tests:
```bash
mvn test -pl shared-services-infrastructure-architecture-tests
```
