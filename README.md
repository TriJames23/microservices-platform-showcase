# Microservices Platform Showcase

A production-grade microservices platform built on **Clean Architecture**, **Event-Driven Design**, and **GitOps delivery** — engineered for correctness, observability, and operational autonomy.

---

## What This Is ?

This platform implements a full-stack commerce backend across multiple independently deployable services, unified by a shared architectural contract enforced at build time.

**Core principles:**
- Domain purity enforced via ArchUnit — framework leakage is a build failure
- Event-driven cross-service coordination via Outbox / Inbox / Saga
- GitOps-driven delivery with ArgoCD and per-service Helm values
- Observability-first: every runtime lane emits semantic metrics to Prometheus + Grafana

---

## Platform at a Glance

| Layer | Technology |
|---|---|
| Language | Java 25, Spring Boot |
| Transport | HTTP REST, gRPC |
| Messaging | Apache Kafka |
| Persistence | PostgreSQL + JPA (EntityManager) |
| Search | Elasticsearch |
| Caching | Redis |
| Saga / Coordination | Custom Saga Engine (outbox/inbox-based) |
| Observability | Prometheus · Grafana · Micrometer |
| Delivery | Jenkins → Docker → ArgoCD → Kubernetes |
| Architecture Tests | ArchUnit (enforced at CI) |

---

## Repositories

| Repository | Role |
|---|---|
| `shared-kernel` | Framework-agnostic domain, application, and contract abstractions |
| `shared-services` | Spring infrastructure adapters: messaging, gRPC, saga, outbox, inbox, search, worker, observability |
| `identity-service` | User identity and authentication |
| `auth-service` | Token issuing and authorization |
| `catalog-service` | Product catalog with Elasticsearch-backed search |
| `order-service` | Order lifecycle and saga orchestration |
| `inventory-service` | Inventory reservation via integration commands |
| `payment-service` | Payment processing |
| `cart-service` | Cart management |
| `notification-service` | WebSocket + gRPC streaming notifications |
| `shop-service` | Multi-shop domain |
| `micro-config-repo` | GitOps config: per-service Helm values per environment |
| `platform-helm-charts` | Shared Helm chart consumed by all services |

---

## Recommended Reading Order

> Start here. Each doc is written for signal density, not volume.

1. **[System Overview](docs/architecture/system-overview.md)** — what the platform does and how it hangs together
2. **[Deployment & GitOps](docs/architecture/deployment-and-gitops.md)** — Jenkins → Docker → ArgoCD → Kubernetes
3. **[Event-Driven Architecture](docs/architecture/event-driven-architecture.md)** — Outbox, Inbox, Saga, Kafka
4. **[Observability Architecture](docs/architecture/observability-architecture.md)** — metrics, alerting, tracing
5. **[Clean Architecture](docs/architecture/clean-architecture.md)** — the boundary model and why it's enforced

---

## Architecture Decision Records

Key decisions that shaped this platform are documented in [`/docs/adr`](docs/adr/README.md).

Highlighted ADRs:
- [ADR-001 — Shared Kernel Purity](docs/adr/ADR-001-shared-kernel-purity.md)
- [ADR-005 — Outbox Pattern](docs/adr/ADR-005-outbox-pattern.md)
- [ADR-015 — Inbox Pattern](docs/adr/ADR-015-inbox-pattern.md)
- [ADR-018 — Clean Boundary Enforcement](docs/adr/ADR-018-clean-boundary-enforcement.md)

---

## Diagrams

| Diagram | Description |
|---|---|
| [System Overview](diagrams/system-overview.png) | Services, transports, and dependencies |
| [Kafka Flow](diagrams/kafka-flow.png) | Outbox → Kafka → Inbox event routing |
| [Outbox Flow](diagrams/outbox-flow.png) | Reliable command/event egress pattern |
| [Deployment Flow](diagrams/deployment-flow.png) | CI/CD and GitOps delivery pipeline |
| [Kubernetes Architecture](diagrams/kubernetes-architecture.png) | Cluster topology |
| [Observability Flow](diagrams/observability-flow.png) | Metrics → Prometheus → Alertmanager → Slack |

---

## Deep Dive (Optional)

For those who want to go deeper into specific platform mechanics:

→ [`docs/deep-dive/`](docs/deep-dive/)
