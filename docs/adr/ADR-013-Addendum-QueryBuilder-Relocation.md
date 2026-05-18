# ADR-013 Addendum: QueryBuilder Ownership and Relocation

## Status
**ACCEPTED** - 2026-02-11

## Context

The system currently defines `IQueryBuilder`, `IQueryBuilderFactory`, and `IQueryResult` in `shared-kernel-application`. However, this placement creates architectural ambiguity:

1. **Ownership Confusion**: Is QueryBuilder an application abstraction or infrastructure tooling?
2. **CQRS Violation Risk**: Application layer building SQL-like queries violates intent-based design
3. **Shared-Kernel Pollution**: Shared-kernel should define behavioral contracts, not persistence tooling
4. **Clean Architecture Breach**: Application knowing about query construction mechanics

## Decision

### 1. QueryBuilder is Infrastructure Tooling

**RULE**: `IQueryBuilder` and all related interfaces belong **exclusively** to the JPA infrastructure module.

**New Location**:
```
shared-services-infrastructure-jpa/
└── src/main/java/com/betatrem/shared_services_infrastructure_jpa/
    └── query/
        ├── IQueryBuilder.java
        ├── IQueryBuilderFactory.java
        ├── IQueryResult.java
        ├── JpaQueryBuilder.java           // Implementation
        └── JpaQueryExecutor.java          // Execution engine
```

**Rationale**:
- QueryBuilder expresses **HOW** to query (mechanism)
- Application should only express **WHAT** to query (intent)
- Infrastructure owns all persistence mechanics

---

### 2. Shared-Kernel Clarification

**RULE**: `shared-kernel-application` MUST NOT expose query construction abstractions.

**What Belongs in Shared-Kernel**:
- ✅ `IQuery<R extends IReadModel>` - Query intent marker
- ✅ `IQueryHandler<Q, R>` - Query handler contract
- ✅ `IReadModel` - Read model marker
- ✅ `ICommand<R>` - Command marker
- ✅ `ICommandHandler<C, R>` - Command handler contract

**What Does NOT Belong in Shared-Kernel**:
- ❌ `IQueryBuilder` - Query construction mechanics
- ❌ `IQueryBuilderFactory` - Query builder factory
- ❌ `IQueryResult` - Query result with SQL/JPQL
- ❌ Any SQL/JPQL/DSL construction tools

**WHY**: Shared-kernel defines **behavioral contracts** (what a use case is), NOT **persistence tooling** (how data is accessed).

---

### 3. Application Layer Responsibilities (Query Side)

**RULE**: Application defines query **intent**, not query **structure**.

#### Application Defines Queries Using:

**1. Query Objects** (Intent Markers):
```java
public record GetUserProfileQuery(UUID userId)
        implements IQuery<UserProfileView> {}
```

**2. Intent-Based Query Repositories**:
```java
public interface IUserProfileQueryRepository {
    Optional<UserProfileView> findByUserId(UUID userId);
    List<UserProfileView> findActiveUsers();
    List<UserProfileView> findByRole(String role);
}
```

#### Application MUST NOT:
- ❌ Build SQL / JPQL / DSL queries
- ❌ Depend on `IQueryBuilder`
- ❌ Pass criteria/specification objects
- ❌ Know about database schema

**Principle**: Application expresses **"what data is needed"**, not **"how it is retrieved"**.

---

### 4. Infrastructure Responsibilities (JPA Module)

**RULE**: Infrastructure owns ALL query construction mechanics.

#### Infrastructure Owns:
- ✅ `IQueryBuilder` interface and implementation
- ✅ SQL / JPQL / native query construction
- ✅ Query optimization strategies
- ✅ Schema knowledge
- ✅ Join strategies
- ✅ Projection mapping

#### Example Responsibility Split:

**Application Layer** (Intent):
```java
// Port definition
public interface IUserProfileQueryRepository {
    Optional<UserProfileView> findByUserId(UUID userId);
}
```

**Infrastructure Layer** (Mechanics):
```java
// Adapter implementation
class JpaUserProfileQueryRepository implements IUserProfileQueryRepository {
    private final IQueryBuilderFactory queryBuilderFactory;
    
    @Override
    public Optional<UserProfileView> findByUserId(UUID userId) {
        // Infrastructure decides:
        // - Which table/view to query
        // - Which joins to use
        // - Which columns to project
        // - How to map to UserProfileView
        return queryBuilderFactory
            .select("u.id, u.username, u.email, u.status, u.created_at")
            .from("users u")
            .where("u.id = :userId")
            .param("userId", userId)
            .getSingleResult(UserProfileView.class);
    }
}
```

**Infrastructure MAY**:
- Use `IQueryBuilder` for dynamic queries
- Use native SQL or JPQL
- Optimize queries freely (caching, indexing hints)
- Change query strategy without touching Application

**Infrastructure MUST**:
- Match application-defined intent exactly
- Never leak `QueryBuilder` outside infrastructure
- Return only Read Models (DTOs implementing `IReadModel`)

---

### 5. Prohibited Patterns

**The following are FORBIDDEN**:

❌ **Injecting IQueryBuilder into Application or Use Case**
```java
// ❌ WRONG
class GetUserProfileQueryHandler {
    private final IQueryBuilderFactory queryBuilderFactory; // ❌ FORBIDDEN
}
```

❌ **Returning IQueryBuilder or IQueryResult via Application Ports**
```java
// ❌ WRONG
public interface IUserProfileQueryRepository {
    IQueryResult buildUserQuery(UUID userId); // ❌ FORBIDDEN
}
```

❌ **Defining Query Criteria/Specification Objects in Application**
```java
// ❌ WRONG
public class UserQueryCriteria {
    private String username;
    private String email;
    private List<String> roles;
    // This becomes pseudo-SQL
}
```

❌ **Re-exporting QueryBuilder via Shared-Kernel**
```java
// ❌ WRONG - in shared-kernel-application
public interface IQueryBuilder { ... } // ❌ FORBIDDEN
```

❌ **Letting Handlers Assemble Query Fragments**
```java
// ❌ WRONG
class GetUserProfileQueryHandler {
    public UserProfileView handle(GetUserProfileQuery query) {
        var queryBuilder = factory.create();
        queryBuilder.select("u.id, u.username"); // ❌ Handler knows schema
        queryBuilder.from("users u");
        // ...
    }
}
```

**WHY FORBIDDEN**: These patterns convert Application into a pseudo-DAO layer and violate CQRS intent-based design.

---

### 6. Architectural Rationale

#### Why QueryBuilder Belongs to JPA Infrastructure:

1. **It is a Mechanism, Not a Policy**
   - QueryBuilder is "how to build queries"
   - Application should only define "what to query"

2. **It Encodes Persistence Details**
   - Knows about tables, columns, joins
   - Couples to database schema
   - Changes when DB refactors

3. **It Must Be Free to Evolve**
   - Infrastructure can optimize queries
   - Can switch from JPQL to native SQL
   - Can add caching, query hints
   - **Without touching Application layer**

#### Why Application Must Not See QueryBuilder:

1. **Prevents SQL-Shaped Thinking**
   - Handlers should orchestrate, not build queries
   - Keeps use cases focused on business logic

2. **Protects Intent-Based CQRS**
   - Application defines "find user by ID"
   - Infrastructure decides "SELECT ... FROM users WHERE id = ?"

3. **Enables DB Refactoring**
   - Can change schema without touching use cases
   - Can denormalize read models
   - Can switch to read replicas

4. **Maintains Clean Architecture**
   - Application depends on abstractions (ports)
   - Infrastructure depends on details (QueryBuilder)
   - Dependency points inward

---

### 7. Enforcement Rules

**Mandatory ArchUnit Tests**:

```java
@ArchTest
static final ArchRule applicationMustNotDependOnQueryBuilder =
    noClasses()
        .that().resideInAPackage("..application..")
        .should().dependOnClassesThat()
        .resideInAPackage("..infrastructure.jpa.query..")
        .because("""
                ADR-013 Addendum [BLOCKER]: Application MUST NOT depend on QueryBuilder.
                
                QueryBuilder is infrastructure tooling, not application abstraction.
                Application defines query intent, Infrastructure defines query mechanics.
                
                Fix:
                - Create intent-based Query Repository Port in application.port.out
                - Move QueryBuilder usage to Infrastructure adapter
                
                See: ADR-013 Addendum - QueryBuilder Ownership
                """);

@ArchTest
static final ArchRule sharedKernelMustNotExposeQueryBuilder =
    noClasses()
        .that().resideInAPackage("..shared_kernel_application..")
        .should().dependOnClassesThat()
        .haveSimpleName("IQueryBuilder")
        .orShould().dependOnClassesThat()
        .haveSimpleName("IQueryBuilderFactory")
        .because("""
                ADR-013 Addendum [BLOCKER]: Shared-Kernel MUST NOT expose QueryBuilder.
                
                Shared-Kernel defines behavioral contracts (IQuery, ICommand),
                NOT persistence tooling (IQueryBuilder).
                
                Fix:
                - Move IQueryBuilder to shared-services-infrastructure-jpa
                - Remove from shared-kernel-application
                
                See: ADR-013 Addendum - Shared-Kernel Clarification
                """);
```

---

### 8. Migration Plan

#### Phase 1: Create Infrastructure Module
1. Create `shared-services-infrastructure-jpa/query/` package
2. Move `IQueryBuilder`, `IQueryBuilderFactory`, `IQueryResult` from `shared-kernel-application`
3. Implement `JpaQueryBuilder` and `JpaQueryExecutor`

#### Phase 2: Update Infrastructure Adapters
1. Update all JPA query repositories to import from infrastructure module
2. Verify no application layer imports `IQueryBuilder`

#### Phase 3: Remove from Shared-Kernel
1. Delete `IQueryBuilder` from `shared-kernel-application`
2. Delete `IQueryBuilderFactory` from `shared-kernel-application`
3. Delete `IQueryResult` from `shared-kernel-application`

#### Phase 4: Verification
1. Run ArchUnit tests
2. Verify no compilation errors
3. Verify application layer has zero infrastructure dependencies

---

## Consequences

### Positive
- ✅ Clear ownership: QueryBuilder is infrastructure tooling
- ✅ Strong CQRS: Application defines intent, Infrastructure defines mechanics
- ✅ Clean Architecture: Application depends on abstractions, not details
- ✅ Refactoring Freedom: Infrastructure can optimize without touching Application
- ✅ Shared-Kernel Purity: Only behavioral contracts, no tooling

### Negative
- ⚠️ Less flexibility for "dynamic query building" at use-case level
- ⚠️ Requires more explicit repository methods
- ⚠️ Migration effort to move existing code

### Trade-Off Justification
The trade-off is **intentional** and aligned with Clean Architecture:
- We sacrifice "convenience" (building queries in handlers)
- We gain "maintainability" (intent-based, refactorable, testable)

---

## Summary (One-Line Rule)

> **Application defines query intent. Infrastructure defines query mechanics. QueryBuilder is mechanics — therefore Infrastructure only.**

---

## References
- ADR-013: CQRS Implementation Strategy
- ADR-001: Clean Architecture Layers
- Martin Fowler: CQRS Pattern
- Robert C. Martin: Clean Architecture

---

## Notes
- This addendum supersedes any previous informal QueryBuilder placement
- All existing code using `IQueryBuilder` in Application must be refactored
- New features must follow this pattern from day 1
