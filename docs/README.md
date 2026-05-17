# Documentation

This documentation is organized into two tiers.

---

## Diagrams (Start Here)

Five focused diagrams — each tells one part of the system story.

| Diagram | Story |
|---|---|
| [System Overview](../diagrams/system-overview.png) | Platform at a glance: clients → services → shared platform → Kafka → Kubernetes → Observability |
| [Deployment Flow](../diagrams/deployment-flow.png) | GitHub → Jenkins → Docker → Config Repo → ArgoCD → Kubernetes |
| [Event-Driven Flow](../diagrams/event-driven-flow.png) | Outbox → Kafka → Inbox — reliable cross-service messaging |
| [Kubernetes Runtime](../diagrams/kubernetes-runtime.png) | 9 services × independent Deployment + HPA, monitored by Prometheus |
| [Observability Architecture](../diagrams/observability-architecture.png) | Services → Prometheus → Alertmanager → Grafana |

---

## Tier 1 — Architecture Docs (Start Here)

High-signal documents covering how the platform is designed and why.
Intended for: engineers, tech leads, hiring managers, interviewers.

### Architecture
| Document | Description |
|---|---|
| [System Overview](architecture/system-overview.md) | Platform scope, services, and integration model |
| [Clean Architecture](architecture/clean-architecture.md) | Layer boundaries, kernel purity, and boundary enforcement |
| [Event-Driven Architecture](architecture/event-driven-architecture.md) | Outbox, Inbox, Kafka, and integration event model |
| [Communication Patterns](architecture/communication-patterns.md) | HTTP, gRPC, async command dispatch — when to use each |
| [Execution Context & Auditability](architecture/execution-context-and-auditability.md) | `IExecutionContext`, header propagation, audit trail |
| [Observability Architecture](architecture/observability-architecture.md) | Metrics, alerting, tracing, and alert delivery |
| [Deployment & GitOps](architecture/deployment-and-gitops.md) | Jenkins → Docker → ArgoCD → Kubernetes delivery model |
| [Scalability & Resilience](architecture/scalability-and-resilience.md) | HPA, Resilience4j, retry/DLQ policy model |

### Platform Infrastructure
| Document | Description |
|---|---|
| [Shared Kernel](platform/shared-kernel.md) | Framework-agnostic contracts: ports, commands, events, results |
| [Shared Services](platform/shared-services.md) | Spring adapter implementations for all kernel contracts |
| [Worker Engine](platform/worker-engine.md) | Background processing model: `DefaultWorkerLoopEngine` |
| [Saga Engine](platform/saga-engine.md) | `ISagaDefinition`, state machine, compensation |
| [Messaging Platform](platform/messaging-platform.md) | Kafka integration, consumer resilience, DLQ |
| [Resource Management](platform/resource-management.md) | `IResourceStore`, S3/Azure/Local storage |
| [Search Integration](platform/search-integration.md) | Elasticsearch adapter and search flow |

### Implementation Examples
| Document | Description |
|---|---|
| [Identity Service](implementation/identity-service.md) | User identity, Clean Architecture implementation |
| [Auth Service](implementation/auth-service.md) | Token lifecycle and authorization |
| [Kubernetes Runtime](implementation/kubernetes-runtime.md) | Deployment config, HPA, resource limits |
| [CI/CD Pipeline](implementation/ci-cd-pipeline.md) | Jenkins pipeline stages and artifact flow |
| [GitOps Flow](implementation/gitops-flow.md) | Config-repo structure and ArgoCD sync model |

---

## Tier 2 — Deep Dive (Optional)

For engineers who want to understand specific platform mechanics in detail.

| Document | Description |
|---|---|
| [Outbox / Inbox Flow](deep-dive/outbox-inbox-flow.md) | Full outbox-to-inbox command dispatch sequence |
| [gRPC E2E Flow](deep-dive/grpc-e2e-flow.md) | Interceptors, dispatch, error mapping |
| [Aggregate Reconstitution](deep-dive/aggregate-reconstitution.md) | Persistence lane and business state restore |
| [WebSocket Streaming](deep-dive/websocket-streaming.md) | Notification delivery via WebSocket + gRPC streaming |
| [Scheduler Engine](deep-dive/scheduler-engine.md) | `DefaultSchedulerEngine` for cleanup and periodic tasks |

---

## Architecture Decision Records

Key decisions that shaped the platform. Full index: [`adr/README.md`](adr/README.md)

| ADR | Decision |
|---|---|
| [ADR-001](adr/ADR-001-shared-kernel-purity.md) | Shared Kernel Purity |
| [ADR-005](adr/ADR-005-outbox-pattern.md) | Outbox Pattern |
| [ADR-015](adr/ADR-015-inbox-pattern.md) | Inbox Pattern |
| [ADR-018](adr/ADR-018-clean-boundary-enforcement.md) | Clean Boundary Enforcement |
