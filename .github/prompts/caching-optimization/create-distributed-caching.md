# Create Distributed Caching

Implement Redis-based distributed caching for Spring Boot applications. Improve performance, reduce database load, and enable horizontal scaling with shared cache across multiple application instances.

## Requirements

Your distributed caching implementation should include:

- **Redis Integration**: Configure Spring Cache with Redis backend
- **Cache Abstraction**: Use Spring's declarative caching annotations
- **Serialization**: Efficient object serialization (JSON, Protocol Buffers)
- **TTL Configuration**: Set appropriate time-to-live for cached data
- **Cache Patterns**: Implement cache-aside, read-through, write-through patterns
- **Eviction Policies**: Configure cache eviction strategies (LRU, LFU)
- **Cache Warming**: Preload frequently accessed data
- **Multi-Level Caching**: Local cache (Caffeine) + distributed cache (Redis)
- **Monitoring**: Track cache hit rates and performance metrics
- **Cache Invalidation**: Strategies for keeping cache consistent

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x, Spring Cache, Redis
- **Serialization**: Jackson JSON, Kryo, or Protocol Buffers
- **Configuration**: YAML-based with connection pooling

## Template Example

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.CachePut;
import org.springframework.cache.annotation.Caching;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.CachingConfigurer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Repository;
import org.springframework.web.bind.annotation.*;
import com.github.benmanes.caffeine.cache.Caffeine;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import lombok.extern.slf4j.Slf4j;
import lombok.RequiredArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;
import lombok.Builder;
import jakarta.persistence.*;
import java.time.Duration;
import java.util.*;
import java.util.concurrent.TimeUnit;

/**
 * Distributed caching implementation using Redis and Spring Cache.
 * Provides multi-level caching with local (Caffeine) and distributed (Redis) layers.
 */
@Slf4j
public class DistributedCachingPatterns {

    /**
     * Redis and cache configuration.
     */
    @Configuration
    @EnableCaching
    @Slf4j
    public static class CacheConfiguration implements CachingConfigurer {

        /**
         * Redis connection factory using Lettuce.
         */
        @Bean
        public LettuceConnectionFactory redisConnectionFactory() {
            LettuceConnectionFactory factory = new LettuceConnectionFactory();
            factory.setHostName("localhost");
            factory.setPort(6379);
            factory.setDatabase(0);
            // Enable connection pooling for better performance
            factory.setShareNativeConnection(true);
            factory.setValidateConnection(true);
            return factory;
        }

        /**
         * RedisTemplate for low-level Redis operations.
         */
        @Bean
        public RedisTemplate<String, Object> redisTemplate(
                RedisConnectionFactory redisConnectionFactory) {

            RedisTemplate<String, Object> template = new RedisTemplate<>();
            template.setConnectionFactory(redisConnectionFactory);

            // Configure serializers
            StringRedisSerializer stringSerializer = new StringRedisSerializer();
            GenericJackson2JsonRedisSerializer jsonSerializer = new GenericJackson2JsonRedisSerializer(objectMapper());

            template.setKeySerializer(stringSerializer);
            template.setHashKeySerializer(stringSerializer);
            template.setValueSerializer(jsonSerializer);
            template.setHashValueSerializer(jsonSerializer);

            template.afterPropertiesSet();
            return template;
        }

        /**
         * Cache manager with multiple cache configurations.
         */
        @Bean
        public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
            // Default cache configuration
            RedisCacheConfiguration defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofMinutes(10))
                    .serializeKeysWith(RedisSerializationContext.SerializationPair
                            .fromSerializer(new StringRedisSerializer()))
                    .serializeValuesWith(RedisSerializationContext.SerializationPair
                            .fromSerializer(new GenericJackson2JsonRedisSerializer(objectMapper())))
                    .disableCachingNullValues();

            // Custom configurations for different caches
            Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();

            // User cache - 30 minutes TTL
            cacheConfigurations.put("users",
                    defaultConfig.entryTtl(Duration.ofMinutes(30)));

            // Product cache - 1 hour TTL
            cacheConfigurations.put("products",
                    defaultConfig.entryTtl(Duration.ofHours(1)));

            // Order cache - 5 minutes TTL (frequently changing data)
            cacheConfigurations.put("orders",
                    defaultConfig.entryTtl(Duration.ofMinutes(5)));

            // Session cache - 24 hours TTL
            cacheConfigurations.put("sessions",
                    defaultConfig.entryTtl(Duration.ofHours(24)));

            // Analytics cache - 1 day TTL
            cacheConfigurations.put("analytics",
                    defaultConfig.entryTtl(Duration.ofDays(1)));

            return RedisCacheManager.builder(redisConnectionFactory)
                    .cacheDefaults(defaultConfig)
                    .withInitialCacheConfigurations(cacheConfigurations)
                    .build();
        }

        /**
         * Local cache manager using Caffeine for hot data.
         */
        @Bean
        public org.springframework.cache.caffeine.CaffeineCacheManager caffeineCacheManager() {
            org.springframework.cache.caffeine.CaffeineCacheManager cacheManager =
                    new org.springframework.cache.caffeine.CaffeineCacheManager("local-users", "local-products");

            cacheManager.setCaffeine(Caffeine.newBuilder()
                    .maximumSize(1000)
                    .expireAfterWrite(5, TimeUnit.MINUTES)
                    .recordStats());

            return cacheManager;
        }

        /**
         * ObjectMapper with Java 8 date/time support.
         */
        @Bean
        public ObjectMapper objectMapper() {
            ObjectMapper mapper = new ObjectMapper();
            mapper.registerModule(new JavaTimeModule());
            mapper.findAndRegisterModules();
            return mapper;
        }
    }

    /**
     * User entity.
     */
    @Entity
    @Table(name = "users")
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class User implements java.io.Serializable {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String username;

        @Column(nullable = false)
        private String email;

        private String firstName;
        private String lastName;
        private String phoneNumber;
        private Boolean active;

        @Column(name = "created_at")
        private java.time.LocalDateTime createdAt;

        @Column(name = "updated_at")
        private java.time.LocalDateTime updatedAt;
    }

    /**
     * User repository.
     */
    @Repository
    public interface UserRepository extends org.springframework.data.jpa.repository.JpaRepository<User, Long> {
        Optional<User> findByUsername(String username);
        Optional<User> findByEmail(String email);
        List<User> findByActive(Boolean active);
    }

    /**
     * Cached user service demonstrating various caching patterns.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class CachedUserService {
        private final UserRepository userRepository;
        private final RedisTemplate<String, Object> redisTemplate;

        /**
         * Get user by ID with caching.
         * Cache-aside pattern: Check cache, load from DB if miss, populate cache.
         */
        @Cacheable(value = "users", key = "#userId", unless = "#result == null")
        public User getUserById(Long userId) {
            log.info("Loading user from database: {}", userId);
            return userRepository.findById(userId)
                    .orElseThrow(() -> new RuntimeException("User not found: " + userId));
        }

        /**
         * Get user by username with caching.
         * Custom cache key using username.
         */
        @Cacheable(value = "users", key = "'username:' + #username", unless = "#result == null")
        public User getUserByUsername(String username) {
            log.info("Loading user from database by username: {}", username);
            return userRepository.findByUsername(username)
                    .orElseThrow(() -> new RuntimeException("User not found: " + username));
        }

        /**
         * Update user and refresh cache.
         * Cache-put pattern: Update DB and cache simultaneously.
         */
        @CachePut(value = "users", key = "#user.id")
        public User updateUser(User user) {
            log.info("Updating user: {}", user.getId());
            user.setUpdatedAt(java.time.LocalDateTime.now());
            User updated = userRepository.save(user);

            // Evict username-based cache entry
            evictUserByUsername(updated.getUsername());

            return updated;
        }

        /**
         * Delete user and evict from cache.
         * Multiple cache evictions for different keys.
         */
        @Caching(evict = {
            @CacheEvict(value = "users", key = "#userId"),
            @CacheEvict(value = "users", key = "'username:' + #result.username")
        })
        public User deleteUser(Long userId) {
            log.info("Deleting user: {}", userId);
            User user = getUserById(userId);
            userRepository.deleteById(userId);
            return user;
        }

        /**
         * Get multiple users with batch caching.
         * Manually implement multi-get for better performance.
         */
        public List<User> getUsersByIds(List<Long> userIds) {
            List<User> users = new ArrayList<>();
            List<Long> uncachedIds = new ArrayList<>();

            // Check cache for each user
            for (Long userId : userIds) {
                String cacheKey = "users::" + userId;
                User cachedUser = (User) redisTemplate.opsForValue().get(cacheKey);

                if (cachedUser != null) {
                    log.debug("Cache hit for user: {}", userId);
                    users.add(cachedUser);
                } else {
                    log.debug("Cache miss for user: {}", userId);
                    uncachedIds.add(userId);
                }
            }

            // Batch load uncached users from database
            if (!uncachedIds.isEmpty()) {
                List<User> dbUsers = userRepository.findAllById(uncachedIds);

                // Cache loaded users
                for (User user : dbUsers) {
                    String cacheKey = "users::" + user.getId();
                    redisTemplate.opsForValue().set(cacheKey, user, Duration.ofMinutes(30));
                    users.add(user);
                }
            }

            return users;
        }

        /**
         * Get active users with conditional caching.
         * Cache only if result is not empty.
         */
        @Cacheable(value = "users", key = "'active:' + #active", unless = "#result.isEmpty()")
        public List<User> getActiveUsers(Boolean active) {
            log.info("Loading active users from database: {}", active);
            return userRepository.findByActive(active);
        }

        /**
         * Evict all user caches.
         */
        @CacheEvict(value = "users", allEntries = true)
        public void evictAllUserCaches() {
            log.info("Evicting all user caches");
        }

        /**
         * Evict specific user cache by username.
         */
        private void evictUserByUsername(String username) {
            String cacheKey = "users::username:" + username;
            redisTemplate.delete(cacheKey);
            log.debug("Evicted cache for username: {}", username);
        }

        /**
         * Cache warming: Preload frequently accessed users.
         */
        public void warmUserCache() {
            log.info("Warming user cache with active users");
            List<User> activeUsers = userRepository.findByActive(true);

            for (User user : activeUsers) {
                String cacheKey = "users::" + user.getId();
                redisTemplate.opsForValue().set(cacheKey, user, Duration.ofMinutes(30));
            }

            log.info("User cache warmed with {} users", activeUsers.size());
        }
    }

    /**
     * Product service with write-through caching.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class CachedProductService {
        private final RedisTemplate<String, Object> redisTemplate;

        /**
         * Get product with read-through caching.
         */
        @Cacheable(value = "products", key = "#productId")
        public Product getProduct(Long productId) {
            log.info("Loading product from database: {}", productId);

            // Simulate database load
            return Product.builder()
                    .id(productId)
                    .name("Product " + productId)
                    .price(99.99)
                    .stock(100)
                    .build();
        }

        /**
         * Update product with write-through caching.
         * Update both cache and database in single operation.
         */
        public Product updateProduct(Product product) {
            log.info("Updating product (write-through): {}", product.getId());

            // Update database
            Product updatedProduct = saveToDatabase(product);

            // Update cache
            String cacheKey = "products::" + product.getId();
            redisTemplate.opsForValue().set(cacheKey, updatedProduct, Duration.ofHours(1));

            return updatedProduct;
        }

        /**
         * Search products with caching.
         * Cache search results by query.
         */
        @Cacheable(value = "products", key = "'search:' + #query")
        public List<Product> searchProducts(String query) {
            log.info("Searching products: {}", query);

            // Simulate search
            return List.of(
                    Product.builder().id(1L).name("Product 1").price(99.99).stock(10).build(),
                    Product.builder().id(2L).name("Product 2").price(149.99).stock(5).build()
            );
        }

        /**
         * Get product inventory with short TTL.
         * Inventory changes frequently, use shorter cache duration.
         */
        public Integer getProductInventory(Long productId) {
            String cacheKey = "inventory::" + productId;
            Integer stock = (Integer) redisTemplate.opsForValue().get(cacheKey);

            if (stock == null) {
                log.info("Loading inventory from database: {}", productId);
                stock = loadInventoryFromDatabase(productId);
                // Short TTL for frequently changing data
                redisTemplate.opsForValue().set(cacheKey, stock, Duration.ofSeconds(30));
            }

            return stock;
        }

        private Product saveToDatabase(Product product) {
            // Simulate database save
            return product;
        }

        private Integer loadInventoryFromDatabase(Long productId) {
            // Simulate database load
            return 100;
        }
    }

    /**
     * Analytics service with long-lived caching.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class CachedAnalyticsService {
        private final RedisTemplate<String, Object> redisTemplate;

        /**
         * Get daily analytics with long cache duration.
         * Historical data doesn't change, safe to cache for extended periods.
         */
        @Cacheable(value = "analytics", key = "'daily:' + #date")
        public AnalyticsReport getDailyAnalytics(java.time.LocalDate date) {
            log.info("Computing daily analytics for: {}", date);

            // Expensive computation
            return AnalyticsReport.builder()
                    .date(date)
                    .totalRevenue(10000.0)
                    .totalOrders(250)
                    .activeUsers(1500)
                    .build();
        }

        /**
         * Get real-time analytics with no caching.
         * Real-time data should not be cached.
         */
        public AnalyticsReport getRealtimeAnalytics() {
            log.info("Computing real-time analytics");

            return AnalyticsReport.builder()
                    .date(java.time.LocalDate.now())
                    .totalRevenue(500.0)
                    .totalOrders(10)
                    .activeUsers(50)
                    .build();
        }

        /**
         * Invalidate analytics cache for date range.
         */
        public void invalidateAnalyticsCache(java.time.LocalDate startDate, java.time.LocalDate endDate) {
            log.info("Invalidating analytics cache from {} to {}", startDate, endDate);

            java.time.LocalDate date = startDate;
            while (!date.isAfter(endDate)) {
                String cacheKey = "analytics::daily:" + date;
                redisTemplate.delete(cacheKey);
                date = date.plusDays(1);
            }
        }
    }

    /**
     * Multi-level caching service (Local + Distributed).
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class MultiLevelCacheService {
        private final com.github.benmanes.caffeine.cache.Cache<String, User> localCache =
                Caffeine.newBuilder()
                        .maximumSize(100)
                        .expireAfterWrite(1, TimeUnit.MINUTES)
                        .recordStats()
                        .build();

        private final RedisTemplate<String, Object> redisTemplate;

        /**
         * Get user with multi-level caching.
         * Check local cache -> distributed cache -> database.
         */
        public User getUserMultiLevel(Long userId) {
            String cacheKey = "user:" + userId;

            // Level 1: Check local cache
            User user = localCache.getIfPresent(cacheKey);
            if (user != null) {
                log.debug("L1 cache hit for user: {}", userId);
                return user;
            }

            // Level 2: Check distributed cache
            user = (User) redisTemplate.opsForValue().get("users::" + userId);
            if (user != null) {
                log.debug("L2 cache hit for user: {}", userId);
                // Populate local cache
                localCache.put(cacheKey, user);
                return user;
            }

            // Level 3: Load from database
            log.info("Cache miss - loading from database: {}", userId);
            user = loadUserFromDatabase(userId);

            // Populate both cache levels
            redisTemplate.opsForValue().set("users::" + userId, user, Duration.ofMinutes(30));
            localCache.put(cacheKey, user);

            return user;
        }

        /**
         * Get local cache statistics.
         */
        public String getCacheStats() {
            com.github.benmanes.caffeine.cache.stats.CacheStats stats = localCache.stats();
            return String.format("Hits: %d, Misses: %d, Hit Rate: %.2f%%",
                    stats.hitCount(), stats.missCount(), stats.hitRate() * 100);
        }

        private User loadUserFromDatabase(Long userId) {
            return User.builder()
                    .id(userId)
                    .username("user" + userId)
                    .email("user" + userId + "@example.com")
                    .active(true)
                    .build();
        }
    }

    /**
     * REST controller for cached operations.
     */
    @RestController
    @RequestMapping("/api/v1")
    @Slf4j
    @RequiredArgsConstructor
    public static class CachedApiController {
        private final CachedUserService userService;
        private final CachedProductService productService;
        private final MultiLevelCacheService multiLevelCacheService;

        @GetMapping("/users/{id}")
        public org.springframework.http.ResponseEntity<User> getUser(@PathVariable Long id) {
            User user = userService.getUserById(id);
            return org.springframework.http.ResponseEntity.ok(user);
        }

        @GetMapping("/users/username/{username}")
        public org.springframework.http.ResponseEntity<User> getUserByUsername(@PathVariable String username) {
            User user = userService.getUserByUsername(username);
            return org.springframework.http.ResponseEntity.ok(user);
        }

        @GetMapping("/users/batch")
        public org.springframework.http.ResponseEntity<List<User>> getUsersBatch(
                @RequestParam List<Long> ids) {
            List<User> users = userService.getUsersByIds(ids);
            return org.springframework.http.ResponseEntity.ok(users);
        }

        @GetMapping("/products/{id}")
        public org.springframework.http.ResponseEntity<Product> getProduct(@PathVariable Long id) {
            Product product = productService.getProduct(id);
            return org.springframework.http.ResponseEntity.ok(product);
        }

        @PostMapping("/admin/cache/warm")
        public org.springframework.http.ResponseEntity<String> warmCache() {
            userService.warmUserCache();
            return org.springframework.http.ResponseEntity.ok("Cache warmed successfully");
        }

        @DeleteMapping("/admin/cache/evict")
        public org.springframework.http.ResponseEntity<String> evictCache() {
            userService.evictAllUserCaches();
            return org.springframework.http.ResponseEntity.ok("Cache evicted successfully");
        }

        @GetMapping("/admin/cache/stats")
        public org.springframework.http.ResponseEntity<String> getCacheStats() {
            String stats = multiLevelCacheService.getCacheStats();
            return org.springframework.http.ResponseEntity.ok(stats);
        }
    }

    /**
     * Domain models.
     */
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class Product implements java.io.Serializable {
        private Long id;
        private String name;
        private Double price;
        private Integer stock;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class AnalyticsReport implements java.io.Serializable {
        private java.time.LocalDate date;
        private Double totalRevenue;
        private Integer totalOrders;
        private Integer activeUsers;
    }
}
```

## Configuration Examples

```yaml
# application.yml

spring:
  # Redis configuration
  data:
    redis:
      host: localhost
      port: 6379
      password: ${REDIS_PASSWORD}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 10
          max-idle: 5
          min-idle: 2
          max-wait: 2000ms
      # Enable Redis repositories
      repositories:
        enabled: false

  # Cache configuration
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes default
      cache-null-values: false
      key-prefix: "myapp::"
      use-key-prefix: true

# JPA configuration
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: false
        generate_statistics: true

# Actuator for cache monitoring
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,caches
  endpoint:
    caches:
      enabled: true
  metrics:
    cache:
      instrument-cache: true
```

## Best Practices

1. **Choose Appropriate TTL Values**
   - User profiles: 30 minutes - 1 hour
   - Product catalog: 1-4 hours
   - Inventory: 30 seconds - 2 minutes
   - Analytics: 1 day - 1 week
   - Session data: 24 hours

2. **Use Cache Keys Wisely**
   - Include cache version in key prefix
   - Use composite keys for complex lookups
   - Avoid large keys (max 512 bytes)
   - Use consistent key naming conventions

3. **Implement Cache Warming**
   - Preload hot data at startup
   - Schedule periodic cache refresh
   - Warm cache after deployments
   - Monitor cache miss rates

4. **Handle Cache Failures Gracefully**
   - Don't fail requests on cache errors
   - Implement fallback to database
   - Log cache failures for monitoring
   - Use circuit breakers for Redis calls

5. **Monitor Cache Performance**
   - Track hit/miss ratios
   - Monitor memory usage
   - Alert on low hit rates
   - Measure cache operation latency

6. **Use Multi-Level Caching**
   - Local cache (Caffeine) for hot data
   - Distributed cache (Redis) for shared data
   - Balance between memory usage and latency
   - Implement cache coherence strategies

7. **Optimize Serialization**
   - Use efficient formats (JSON, Kryo, Protobuf)
   - Avoid serializing unnecessary fields
   - Consider compression for large objects
   - Test serialization performance

8. **Implement Proper Eviction**
   - Use @CacheEvict after updates
   - Clear related cache entries
   - Implement cache invalidation strategies
   - Consider event-driven invalidation

## Related Documentation

- [Spring Cache Abstraction](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache)
- [Spring Data Redis](https://spring.io/projects/spring-data-redis)
- [Redis Cache Patterns](https://redis.io/docs/manual/patterns/cache/)
- [Caffeine Cache](https://github.com/ben-manes/caffeine)
