# Caching & Optimization Library

Comprehensive prompts for implementing distributed caching and cache invalidation strategies to reduce latency, improve throughput, and minimize database load in high-performance microservices architectures.

## Available Prompts

### 1. Create Distributed Caching
Implement distributed caching with Redis and Caffeine for multi-layer caching strategy.

| Feature | Description |
|---------|-------------|
| Redis Integration | Distributed cache shared across instances |
| Caffeine Cache | High-performance local in-memory cache |
| Multi-Layer Caching | L1 (local) + L2 (distributed) cache hierarchy |
| Spring Cache Abstraction | Declarative caching with annotations |
| Cache-Aside Pattern | Load-through caching with fallback |
| TTL Configuration | Time-based expiration policies |
| Cache Serialization | JSON and Java serialization support |
| Conditional Caching | Cache based on method parameters |
| Cache Statistics | Hit rate, miss rate, eviction tracking |
| Performance | Sub-millisecond cache lookups |

### 2. Create Cache Invalidation
Implement intelligent cache invalidation strategies to maintain data consistency.

| Feature | Description |
|---------|-------------|
| Time-Based Invalidation | TTL and absolute expiration |
| Event-Based Invalidation | Invalidate on data changes |
| Pattern-Based Invalidation | Clear multiple cache keys by pattern |
| Write-Through Cache | Update cache on data modification |
| Write-Behind Cache | Async cache updates with batching |
| Cache Eviction Policies | LRU, LFU, FIFO eviction strategies |
| Pub/Sub Invalidation | Cross-instance cache invalidation |
| Versioned Caching | Cache versioning for schema changes |
| Cache Warming | Pre-populate cache on startup |
| Monitoring | Track invalidation events and patterns |

## Quick Start

### 1. Setup Distributed Caching

```
@workspace Use .github/prompts/caching-optimization/create-distributed-caching.md

Configure Redis distributed cache with Caffeine local cache for product catalog
```

### 2. Implement Cache Invalidation

```
@workspace Use .github/prompts/caching-optimization/create-cache-invalidation.md

Add event-based cache invalidation when products are updated or deleted
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Request                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────┐
            │  L1 Cache      │ ← Caffeine (Local, In-Memory)
            │  (Caffeine)    │   - 10,000 entries max
            │                │   - 5 minute TTL
            │  Hit: ~1ms     │   - 95% hit rate
            └────────┬───────┘
                     │ Miss
                     ▼
            ┌────────────────┐
            │  L2 Cache      │ ← Redis (Distributed, Network)
            │  (Redis)       │   - Shared across instances
            │                │   - 1 hour TTL
            │  Hit: ~5ms     │   - 85% hit rate
            └────────┬───────┘
                     │ Miss
                     ▼
            ┌────────────────┐
            │  Database      │ ← PostgreSQL
            │  (PostgreSQL)  │   - Source of truth
            │                │   - 100-500ms query time
            │  Query: ~100ms │   - 5% miss rate
            └────────────────┘

Cache Invalidation Flow:

    ┌─────────────┐
    │   Update    │ ← User updates product
    │   Product   │
    └──────┬──────┘
           │
           ▼
    ┌──────────────────┐
    │   Application    │
    │   1. Update DB   │
    │   2. Invalidate  │
    └──────┬───────────┘
           │
           ├──────────────────────┐
           │                      │
           ▼                      ▼
    ┌────────────┐        ┌────────────┐
    │  L1 Cache  │        │  L2 Cache  │
    │  Evict     │        │  Evict     │
    │  product:  │        │  product:  │
    │  123       │        │  123       │
    └────────────┘        └────────────┘
           │                      │
           │  Redis Pub/Sub       │
           └──────────┬───────────┘
                      │
                      ▼
           ┌──────────────────────┐
           │  Other Instances     │
           │  Evict Local Cache   │
           └──────────────────────┘

Cache Warming (Startup):

    Application Start
           │
           ▼
    Load Top 1000 Products
           │
           ├─────────────────┬─────────────────┐
           │                 │                 │
           ▼                 ▼                 ▼
    [Product 1] ──→  [Product 2] ──→  [Product N]
           │                 │                 │
           ▼                 ▼                 ▼
    L1 + L2 Cache     L1 + L2 Cache     L1 + L2 Cache
           │                 │                 │
           └─────────────────┴─────────────────┘
                             │
                             ▼
                   Ready to Serve Traffic
```

## Integration Patterns

### Spring Boot Service with Multi-Layer Caching

```java
@Configuration
@EnableCaching
public class CacheConfig {

    /**
     * Two-level cache manager: Caffeine (L1) + Redis (L2).
     */
    @Bean
    public CacheManager cacheManager(
            RedisConnectionFactory connectionFactory) {

        // L1: Caffeine local cache
        CaffeineCacheManager localCacheManager = new CaffeineCacheManager();
        localCacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .recordStats()
        );

        // L2: Redis distributed cache
        RedisCacheConfiguration redisCacheConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

        RedisCacheManager redisCacheManager = RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(redisCacheConfig)
            .build();

        // Combine into multi-layer cache
        return new MultiLayerCacheManager(localCacheManager, redisCacheManager);
    }
}

@Service
@RequiredArgsConstructor
public class ProductService {

    private final ProductRepository productRepository;
    private final CacheManager cacheManager;

    /**
     * Get product with automatic caching.
     */
    @Cacheable(value = "products", key = "#productId")
    public Product getProduct(String productId) {
        log.info("Cache miss - loading product from database: {}", productId);
        return productRepository.findById(productId)
            .orElseThrow(() -> new ProductNotFoundException(productId));
    }

    /**
     * Update product and invalidate cache.
     */
    @CachePut(value = "products", key = "#product.id")
    public Product updateProduct(Product product) {
        Product updated = productRepository.save(product);

        // Publish invalidation event to other instances
        publishCacheInvalidationEvent(product.getId());

        return updated;
    }

    /**
     * Delete product and evict from cache.
     */
    @CacheEvict(value = "products", key = "#productId")
    public void deleteProduct(String productId) {
        productRepository.deleteById(productId);

        // Publish invalidation event
        publishCacheInvalidationEvent(productId);
    }

    /**
     * Invalidate all products in category.
     */
    @CacheEvict(value = "products", allEntries = true)
    public void invalidateProductsInCategory(String categoryId) {
        log.info("Invalidating all products in category: {}", categoryId);
    }
}
```

### Event-Based Cache Invalidation with Redis Pub/Sub

```java
@Configuration
public class CacheInvalidationConfig {

    /**
     * Redis message listener for cache invalidation events.
     */
    @Bean
    public RedisMessageListenerContainer redisMessageListenerContainer(
            RedisConnectionFactory connectionFactory,
            CacheInvalidationListener listener) {

        RedisMessageListenerContainer container = new RedisMessageListenerContainer();
        container.setConnectionFactory(connectionFactory);
        container.addMessageListener(listener,
            new PatternTopic("cache:invalidation:*"));

        return container;
    }
}

@Component
@RequiredArgsConstructor
@Slf4j
public class CacheInvalidationListener implements MessageListener {

    private final CacheManager cacheManager;

    @Override
    public void onMessage(Message message, byte[] pattern) {
        String channel = new String(message.getChannel());
        String cacheKey = new String(message.getBody());

        log.info("Received cache invalidation: channel={}, key={}",
            channel, cacheKey);

        // Extract cache name and key
        String[] parts = cacheKey.split(":", 2);
        String cacheName = parts[0];
        String key = parts[1];

        // Evict from local cache
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.evict(key);
            log.info("Evicted from local cache: {}:{}", cacheName, key);
        }
    }
}

@Service
@RequiredArgsConstructor
public class CacheInvalidationService {

    private final RedisTemplate<String, String> redisTemplate;

    /**
     * Publish cache invalidation event to all instances.
     */
    public void publishInvalidation(String cacheName, String key) {
        String channel = "cache:invalidation:" + cacheName;
        String message = cacheName + ":" + key;

        redisTemplate.convertAndSend(channel, message);

        log.info("Published cache invalidation: {}:{}", cacheName, key);
    }

    /**
     * Invalidate by pattern (e.g., all products in category).
     */
    public void invalidateByPattern(String pattern) {
        Set<String> keys = redisTemplate.keys(pattern);

        if (keys != null && !keys.isEmpty()) {
            redisTemplate.delete(keys);
            log.info("Invalidated {} keys matching pattern: {}",
                keys.size(), pattern);
        }
    }
}
```

### Cache Warming on Application Startup

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheWarmingService {

    private final ProductService productService;
    private final CacheManager cacheManager;

    /**
     * Warm cache on application startup.
     */
    @EventListener(ApplicationReadyEvent.class)
    public void warmCache() {
        log.info("Starting cache warming...");

        long startTime = System.currentTimeMillis();

        // Load top 1000 popular products
        List<String> popularProductIds = getPopularProductIds(1000);

        int warmedCount = 0;
        for (String productId : popularProductIds) {
            try {
                productService.getProduct(productId);
                warmedCount++;
            } catch (Exception e) {
                log.warn("Failed to warm cache for product: {}", productId, e);
            }
        }

        long duration = System.currentTimeMillis() - startTime;

        log.info("Cache warming completed: {} products loaded in {}ms",
            warmedCount, duration);
    }

    private List<String> getPopularProductIds(int limit) {
        // Load from database or external service
        return productRepository.findPopularProducts(limit)
            .stream()
            .map(Product::getId)
            .collect(Collectors.toList());
    }
}
```

### Reactive Caching with Project Reactor

```java
@Service
@RequiredArgsConstructor
public class ReactiveProductService {

    private final ReactiveRedisTemplate<String, Product> redisTemplate;
    private final ReactiveProductRepository productRepository;

    /**
     * Get product with reactive caching.
     */
    public Mono<Product> getProduct(String productId) {
        String cacheKey = "product:" + productId;

        return redisTemplate.opsForValue().get(cacheKey)
            .switchIfEmpty(
                productRepository.findById(productId)
                    .flatMap(product ->
                        redisTemplate.opsForValue()
                            .set(cacheKey, product, Duration.ofHours(1))
                            .thenReturn(product)
                    )
            )
            .doOnNext(product ->
                log.debug("Cache hit for product: {}", productId)
            );
    }

    /**
     * Update product and invalidate cache.
     */
    public Mono<Product> updateProduct(Product product) {
        String cacheKey = "product:" + product.getId();

        return productRepository.save(product)
            .flatMap(updated ->
                redisTemplate.opsForValue()
                    .set(cacheKey, updated, Duration.ofHours(1))
                    .thenReturn(updated)
            );
    }

    /**
     * Delete product and evict from cache.
     */
    public Mono<Void> deleteProduct(String productId) {
        String cacheKey = "product:" + productId;

        return productRepository.deleteById(productId)
            .then(redisTemplate.delete(cacheKey))
            .then();
    }
}
```

## Key Concepts

### Multi-Layer Caching
Hierarchical cache strategy with local (L1) and distributed (L2) caches. L1 provides ultra-low latency, L2 provides consistency across instances.

### Cache-Aside Pattern
Application code explicitly manages cache: check cache first, load from database on miss, populate cache.

### Write-Through Cache
Updates go to both cache and database synchronously. Ensures cache consistency at the cost of write latency.

### Write-Behind Cache
Updates go to cache immediately, database writes are batched and async. Improves write performance but risks data loss.

### Cache Invalidation
Process of removing stale data from cache. Strategies include TTL, event-based, and pattern-based invalidation.

### Cache Warming
Pre-populating cache with frequently accessed data on application startup to avoid cold start performance issues.

## Best Practices

1. **Use Multi-Layer Caching**
   - L1 (Caffeine): 5-10 minute TTL, 10K entries
   - L2 (Redis): 1 hour TTL, shared across instances
   - Database: source of truth

2. **Set Appropriate TTLs**
   - Hot data (products): 1-6 hours
   - Reference data (categories): 24 hours
   - User sessions: 30 minutes
   - Analytics: 5-15 minutes

3. **Implement Cache Invalidation**
   - Invalidate on data updates
   - Use Redis Pub/Sub for cross-instance invalidation
   - Pattern-based invalidation for bulk operations
   - Monitor stale cache rate

4. **Avoid Cache Stampede**
   - Use distributed locks for cache loading
   - Implement probabilistic early expiration
   - Pre-warm cache before expiration

5. **Monitor Cache Performance**
   - Track hit rate (target: >80%)
   - Track miss rate
   - Monitor eviction rate
   - Measure cache latency

6. **Handle Cache Failures Gracefully**
   - Fallback to database on cache miss
   - Log cache errors but don't fail requests
   - Implement circuit breaker for cache

7. **Optimize Cache Key Design**
   - Use consistent naming: "products:123"
   - Include version in key for schema changes
   - Keep keys short to save memory
   - Avoid high cardinality in keys

8. **Use Conditional Caching**
   - Cache only cacheable responses (200 OK)
   - Skip caching for personalized data
   - Use cache conditionally based on parameters

## Configuration

```yaml
# application.yml
spring:
  cache:
    type: caffeine
    cache-names:
      - products
      - categories
      - users
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=5m

  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    password: ${REDIS_PASSWORD:}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5

cache:
  redis:
    time-to-live: 1h
    cache-null-values: false
  caffeine:
    initial-capacity: 100
    maximum-size: 10000
    expire-after-write: 5m
    record-stats: true
  warming:
    enabled: true
    popular-products-count: 1000
```

## Troubleshooting

### Low Cache Hit Rate
1. Check cache TTL is appropriate
2. Verify cache warming is working
3. Review cache key design
4. Check if cache is being invalidated too frequently

### Cache Stampede
1. Implement distributed locking
2. Use probabilistic early expiration
3. Increase cache TTL
4. Pre-warm cache

### Stale Data in Cache
1. Verify invalidation events are published
2. Check Redis Pub/Sub is working
3. Review TTL configuration
4. Implement versioned caching

### High Redis Memory Usage
1. Review cache sizes and TTLs
2. Implement eviction policies
3. Clear unused cache keys
4. Consider cache partitioning

## Related Documentation

- [Spring Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)
- [Redis Caching](https://redis.io/docs/manual/client-side-caching/)
- [Cache Patterns](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html)

## Library Navigation

This library focuses on caching and optimization. For related capabilities:
- See **resilience-operations** library for circuit breakers around cache failures
- See **observability** library for cache metrics and monitoring
- See **data-quality-validation** library for cache data validation
