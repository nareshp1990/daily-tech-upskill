# Day 18 — Caching Strategies
**Date:** 2026-03-25
**Phase:** 2 — System Design
**Time Target:** 1–2 hours

---

## 1. Today's Focus Topic
**Caching Strategies — Redis, Cache Patterns, Invalidation, and Spring Boot Integration**

Caching is the single most impactful performance optimisation you can apply. Today you'll go beyond "put it in Redis" and learn the trade-offs between caching patterns, understand cache invalidation strategies, and implement production-grade caching in Spring Boot. You'll also see where caching fails — and why "cache everything" is a trap.

---

## 2. Core Concepts

### 2.1 Why Cache?

```
Without cache:                      With cache:
Client → Service → Database         Client → Service → Cache (HIT) → Response
         ~50ms                                         ~2ms

                                    Client → Service → Cache (MISS) → Database → Cache (SET) → Response
                                                                       ~50ms (first time only)
```

**Cache hit ratio** is the key metric: `hits / (hits + misses)`. A 95% hit ratio means 95% of requests are served in microseconds instead of milliseconds.

### 2.2 Cache Levels

```
┌─────────────────────────────────┐
│  L1: In-Process (Caffeine)      │  ~100ns, per-instance, lost on restart
├─────────────────────────────────┤
│  L2: Distributed (Redis)        │  ~1ms, shared across instances
├─────────────────────────────────┤
│  L3: CDN (Cloudflare, Azure CDN)│  ~10ms, global, static/semi-static content
├─────────────────────────────────┤
│  L4: Database Query Cache       │  Built-in, limited, often disabled in MySQL 8+
└─────────────────────────────────┘
```

**Use L1 for:** hot data that's OK to be slightly stale per-instance (config, feature flags).
**Use L2 for:** shared data that must be consistent across instances (sessions, user profiles).
**Use L3 for:** static assets, API responses that rarely change.

### 2.3 Caching Patterns

#### Cache-Aside (Lazy Loading) — Most Common

```
Read path:
1. Check cache → HIT? Return.
2. MISS → Read from DB → Write to cache → Return.

Write path:
1. Write to DB.
2. Invalidate (delete) cache entry.
```

```java
@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository repo;
    private final RedisTemplate<String, Product> redis;

    public Product getProduct(Long id) {
        String key = "product:" + id;
        Product cached = redis.opsForValue().get(key);
        if (cached != null) return cached;              // HIT

        Product product = repo.findById(id)             // MISS
            .orElseThrow(() -> new NotFoundException(id));
        redis.opsForValue().set(key, product, Duration.ofMinutes(30));
        return product;
    }

    @Transactional
    public Product updateProduct(Long id, ProductUpdate update) {
        Product product = repo.findById(id).orElseThrow();
        product.apply(update);
        repo.save(product);
        redis.delete("product:" + id);                  // Invalidate
        return product;
    }
}
```

**Pros:** Only caches data that's actually requested. Simple.
**Cons:** Cache miss = slow first request. Stale data window between DB write and cache delete.

#### Write-Through

```
Write path:
1. Write to cache AND DB synchronously.
2. Read always hits cache.
```

```java
@Transactional
public Product updateProduct(Long id, ProductUpdate update) {
    Product product = repo.findById(id).orElseThrow();
    product.apply(update);
    repo.save(product);
    redis.opsForValue().set("product:" + id, product, Duration.ofMinutes(30));
    return product;
}
```

**Pros:** Cache is always fresh. No stale reads.
**Cons:** Write latency increases (DB + cache). Caches data that may never be read.

#### Write-Behind (Write-Back)

```
Write path:
1. Write to cache immediately (fast return).
2. Async: flush cache to DB in batches.
```

**Pros:** Fastest writes. Batching reduces DB load.
**Cons:** Data loss risk if cache crashes before flush. Complex to implement correctly.

#### Read-Through

```
Read path:
1. Application asks cache.
2. Cache itself loads from DB on miss (cache is the data access layer).
```

The cache library handles the loading — you configure a `CacheLoader`.

### 2.4 Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

| Strategy | How | When |
|----------|-----|------|
| TTL (Time-to-Live) | Entry expires after N seconds | When slight staleness is OK |
| Event-driven | Invalidate on write event (Kafka, DB trigger) | When freshness matters |
| Version-based | Key includes version: `product:42:v3` | When you can track versions |
| Manual purge | Admin API to flush specific keys | Emergency fixes |

**TTL is your safety net** — even with event-driven invalidation, always set a TTL so stale data eventually expires if an event is lost.

### 2.5 Spring Boot Cache Abstraction

```java
@Configuration
@EnableCaching
public class CacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeValuesWith(
                SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("products",
                config.entryTtl(Duration.ofHours(1)))       // per-cache TTL
            .withCacheConfiguration("user-sessions",
                config.entryTtl(Duration.ofMinutes(15)))
            .build();
    }
}

@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product getProduct(Long id) {
        return repo.findById(id).orElseThrow();     // only called on cache miss
    }

    @CachePut(value = "products", key = "#id")
    public Product updateProduct(Long id, ProductUpdate update) {
        Product product = repo.findById(id).orElseThrow();
        product.apply(update);
        return repo.save(product);                   // updates cache with return value
    }

    @CacheEvict(value = "products", key = "#id")
    public void deleteProduct(Long id) {
        repo.deleteById(id);                         // removes cache entry
    }
}
```

---

## 3. Cache Failure Modes

### 3.1 Cache Stampede (Thundering Herd)

A popular key expires → hundreds of requests simultaneously hit the DB.

**Solutions:**
```java
// 1. Probabilistic early expiration (jitter)
long ttl = 1800 + ThreadLocalRandom.current().nextInt(300); // 30min ± 5min jitter

// 2. Locking — only one thread refreshes
public Product getProductWithLock(Long id) {
    String key = "product:" + id;
    Product cached = redis.opsForValue().get(key);
    if (cached != null) return cached;

    String lockKey = "lock:product:" + id;
    Boolean acquired = redis.opsForValue().setIfAbsent(lockKey, "1", Duration.ofSeconds(5));
    if (Boolean.TRUE.equals(acquired)) {
        try {
            Product product = repo.findById(id).orElseThrow();
            redis.opsForValue().set(key, product, Duration.ofMinutes(30));
            return product;
        } finally {
            redis.delete(lockKey);
        }
    }
    // Other threads wait briefly and retry
    Thread.sleep(50);
    return redis.opsForValue().get(key);
}
```

### 3.2 Cache Penetration

Requests for non-existent keys always miss the cache and hit the DB.

**Solution:** Cache null results with a short TTL:
```java
// Cache a "null marker" for missing products
redis.opsForValue().set("product:" + id, Product.NULL_MARKER, Duration.ofMinutes(5));
```

### 3.3 Hot Key

A single key gets disproportionate traffic (e.g., viral product page).

**Solutions:**
- L1 (in-process) cache in front of Redis for hot keys.
- Replicate the key across multiple Redis slots: `product:42:r1`, `product:42:r2`, random read.

---

## 4. Hands-On Exercise

### Task: Add multi-layer caching to a Spring Boot service

1. **Set up Caffeine as L1** (in-process, 100 entries, 5-min TTL).
2. **Set up Redis as L2** (shared, 30-min TTL).
3. **Implement cache-aside** with the read path checking L1 → L2 → DB.
4. **Add cache invalidation** on writes that clears both L1 and L2.
5. **Add a `/cache/stats` actuator endpoint** that exposes hit ratio, size, and eviction count.

### Spring Dependencies:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

---

## 5. Key Takeaways

1. **Cache-aside is the default pattern** — simple, effective, and hard to get wrong.
2. **Always set a TTL** — it's your safety net against stale data from missed invalidation events.
3. **Guard against stampedes** — use jitter on TTLs and locking on popular keys.
4. **L1 + L2 layering** — Caffeine for hot data, Redis for shared state, CDN for static content.
5. **Cache what's read often, changes rarely** — caching frequently-changing data adds complexity with little benefit.

---

## 6. Connection to Previous Days

In Day 13 (Production LLM Architecture), you built a semantic cache for LLM responses using vector similarity. That's a specialised cache — same pattern (check cache before expensive operation), same invalidation challenges (when does a cached LLM response go stale?). Today's patterns apply directly to caching embedding results and RAG retrievals too.

---

## 7. Resources

- **[Redis Best Practices](https://redis.io/docs/management/optimization/)** — Official performance tuning guide.
- **[Spring Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)** — Annotation-based caching reference.
- **[Designing Data-Intensive Applications, Ch. 5](https://dataintensive.net/)** — Replication and consistency models.

---

*Day 18. The fastest request is the one you don't make.*
