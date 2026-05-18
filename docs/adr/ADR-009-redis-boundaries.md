# ADR-009: Redis Boundaries

**Status**: ✅ Accepted  
**Date**: 2025-Q4  
**Enforced By**: Code Review

---

## Context

Redis allows multiple data patterns (cache, lock, pub/sub, store). Without clear boundaries, teams misuse Redis as a primary database or mix concerns.

---

## Decision

Define clear boundaries for Redis usage patterns.

### Allowed Use Cases

1. **Caching**: Use `@Cacheable` abstraction
2. **Locking**: Use `Redisson` distributed lock abstractions
3. **Idempotency**: Use separate key prefixes for idempotency tokens

### Constraints

- Application and Domain layers **MUST NOT** depend on Redis clients or APIs directly
- Redis access is encapsulated in Infrastructure only
- **Strict Rule**: Redis is **Ephemeral** - No persistent data (System of Record)
- Redis **MUST NOT** be the source of truth for business data

---

## Consequences

### Positive
✅ Clear separation of concerns  
✅ Redis is distinct from primary database  
✅ Easy to replace Redis implementation  
✅ No data loss risk (Redis is cache, not store)

### Negative
⚠️ Cannot use Redis for everything  
⚠️ Requires discipline to not store critical data in Redis

---

## Usage Patterns

### 1. Caching (✅ Allowed)
```java
// Application
@Service
public class ProductService {
    @Cacheable(value = "products", key = "#id")
    public Product findById(ProductId id) {
        return repository.findById(id);
    }
}

// Infrastructure
@Configuration
@EnableCaching
public class CacheConfiguration {
    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory).build();
    }
}
```

### 2. Distributed Locking (✅ Allowed)
```java
// Infrastructure
@Component
public class DistributedLockAdapter implements LockPort {
    private final RedissonClient redisson;
    
    public void withLock(String key, Runnable action) {
        RLock lock = redisson.getLock("lock:" + key);
        try {
            lock.lock();
            action.run();
        } finally {
            lock.unlock();
        }
    }
}
```

### 3. Idempotency (✅ Allowed)
```java
// Infrastructure
@Component
public class RedisIdempotencyStore implements IdempotencyPort {
    private final StringRedisTemplate redis;
    
    public boolean isProcessed(String requestId) {
        return redis.hasKey("idempotency:" + requestId);
    }
    
    public void markProcessed(String requestId) {
        redis.opsForValue().set("idempotency:" + requestId, "1", 24, TimeUnit.HOURS);
    }
}
```

### 4. Primary Data Store (❌ FORBIDDEN)
```java
// ❌ WRONG: Using Redis as System of Record
@Component
public class OrderRepository {
    private final RedisTemplate<String, Order> redis;
    
    public void save(Order order) {
        redis.opsForValue().set("order:" + order.getId(), order); // ← VIOLATION
        // If Redis crashes, data is LOST!
    }
}
```

---

## Key Prefixes

To avoid key collisions, use consistent prefixes:

| Use Case | Prefix | TTL | Example |
|:---------|:-------|:----|:--------|
| Cache | `cache:` | 1-24h | `cache:product:123` |
| Lock | `lock:` | 30s | `lock:order:456` |
| Idempotency | `idempotency:` | 24h | `idempotency:req-789` |
| Session | `session:` | 30m | `session:user-abc` |

---

## Rejected Alternatives

| Alternative | Reason for Rejection |
|:------------|:---------------------|
| "Use Redis as primary database" | Data loss risk (ephemeral) |
| "Mix cache and lock keys without prefixes" | Key collisions |
| "Allow Application to use Redis directly" | Violates layering |

---

## Related ADRs

- [ADR-002](ADR-002-infrastructure-only-shared-services.md): Infrastructure-Only Shared Services
