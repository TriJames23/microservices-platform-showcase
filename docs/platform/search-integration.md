# Search Integration

**Module**: `shared-services-infrastructure-search`

The Elasticsearch adapter layer. Provides document mapping and query execution for read-side search use cases.

---

## Design Principle

Search is a **projection concern** — not a write-side decision source.

```
❌ FORBIDDEN: checkout, pricing, stock, payment decisions querying Elasticsearch
✅ ALLOWED:   browse, discovery, full-text search, faceted filtering
```

Elasticsearch is eventual-consistent by design. Near real-time is an operational target — not a guarantee.

---

## E2E Flow

```
Write-side use case commits + emits integration event (Outbox → Kafka)
       │
       ▼
Read-side projector reacts to integration event
       │
       ▼
Service-owned search adapter writes/updates Elasticsearch document
       │
       ▼
Query use case returns IReadModel backed by search index
```

---

## Service Ownership

Each service owns its own search documents:

| Service | Owns |
|---|---|
| `catalog-service` | Catalog/product search documents |
| `order-service` | Order read/search documents |

Services must not share or cross-write search document ownership.

---

## Configuration

```yaml
search:
  enabled: true

spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: ${ES_PASSWORD}
```

`SearchAutoConfiguration` activates only when `search.enabled=true` and an `ElasticsearchClient` is present.

---

## Module Structure

```
*-infrastructure-search/
├── documents/     # Elasticsearch document classes
├── repositories/  # Elasticsearch repository adapters
├── projectors/    # Integration event → document update handlers
└── config/        # SearchAutoConfiguration wiring
```

---

## Key Rules

| ✅ Correct | ❌ Forbidden |
|---|---|
| Search used for browse/discovery queries | Elasticsearch as source of truth for write-side |
| Document types owned by the service | Elasticsearch document types leaking into domain |
| Projector reacts to integration events | Aggregate reconstitution inside search module |
| Near real-time lag acknowledged | Search assumed to be synchronously consistent |
| `spring.elasticsearch` configured in values.yaml | Hardcoded connection strings |

---

## Related Flows

- [Event-Driven Architecture](../architecture/event-driven-architecture.md) — how projection events flow from outbox to search
- [deep-dive/aggregate-reconstitution.md](../deep-dive/aggregate-reconstitution.md) — persistence vs search separation
