# Aggregate Reconstitution

> **Deep Dive** — Full technical sequence for persistence and aggregate restore from database state.

Source: `book/13-persistence-and-aggregate-reconstitution-flow.md`

---

## The Problem

When an aggregate is loaded from the database, it must be **fully restored** — not just its business fields, but also its infrastructure identity state. Missing `version` or `deleted` breaks optimistic concurrency and soft-delete semantics.

---

## Mandatory Restore Contract

Every aggregate restore path MUST call:

```java
aggregate.setVersionForReconstitution(version);
aggregate.setDeletedForReconstitution(deleted);
```

`AggregateRootBase` owns both fields as part of aggregate runtime identity. Architecture tests fail if either call is missing.

---

## Persistence Lane Structure

```
*-infrastructure-persistence/
├── entities/       # JPA entity classes
├── repositories/   # Spring Data JpaRepository (internal only)
├── mappers/        # toDomain() / toEntity() — owns reconstitution
├── adapters/       # JpaAggregateRepository / JpaReadRepository impls
└── config/         # @EntityScan, @EnableJpaRepositories
```

**Top-level module name**: `*-infrastructure-persistence`  
**NEVER**: `*-infrastructure-jpa` (jpa is an implementation detail package, not a module identity)

---

## Write Side — Full Sequence

```
1. Handler depends on service port:
   ISampleRepository extends IAggregateRepository<SampleAggregate, SampleId>

2. Persistence adapter:
   JpaSampleRepositoryAdapter extends JpaAggregateRepository<SampleAggregate, SampleJpaEntity, SampleId, UUID>

3. Adapter composes:
   - ISampleSpringDataRepository extends JpaRepository<SampleJpaEntity, UUID>
   - SampleDomainEntityMapper

4. Mapper.toDomain(entity):
   - restore business fields
   - aggregate.setVersionForReconstitution(entity.getVersion())   ← MANDATORY
   - aggregate.setDeletedForReconstitution(entity.isDeleted())    ← MANDATORY

5. On save:
   - existing rows are UPDATED, not treated as brand-new
   - version must continue to represent aggregate lifecycle
   - optimistic concurrency semantics must be preserved
```

---

## Read Side

```
ISampleQueryRepository extends IReadRepository<SampleView, SampleId>
       ↓
JpaSampleQueryRepositoryAdapter extends JpaReadRepository
       ↓
JPQL query → SampleView projection (NOT a domain aggregate)
```

**Rules:**
- Query adapters return read models only — **never** domain aggregates
- Query adapters must not rebuild aggregates just to project them back into DTOs
- Constructor aliases and scalar aliases must line up explicitly

---

## Spring Data Repository Boundary

Spring Data `JpaRepository` is **not** an application port.

| ✅ Allowed | ❌ Forbidden |
|---|---|
| Used inside persistence adapter | Injected into handlers |
| Used inside mapper | Injected into policies |
| Internal to persistence module | Exposed to transport adapters |

`JpaRepository` is a service-specific persistence convenience — not the shared platform default. Platform default is `EntityManager + @Bean`.

---

## Architecture Test Failures

ArchUnit fails the build when:
- `setVersionForReconstitution(version)` is missing from aggregate restore
- `setDeletedForReconstitution(deleted)` is missing from aggregate restore
- Top-level module is named `*-infrastructure-jpa` instead of `*-infrastructure-persistence`
- `JpaRepository` is treated as the shared platform default
- Save paths lose optimistic concurrency semantics
