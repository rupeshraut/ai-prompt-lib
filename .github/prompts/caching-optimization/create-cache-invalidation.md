# Create Cache Invalidation

Implement cache invalidation strategies for Spring Boot applications. Ensure cache consistency through event-driven invalidation, time-based expiration, and coordinated updates across distributed cache instances.

## Requirements

Your cache invalidation implementation should include:

- **Declarative Invalidation**: Use @CacheEvict for automatic cache clearing
- **Event-Driven Invalidation**: Invalidate cache on domain events (create, update, delete)
- **Pattern-Based Invalidation**: Clear multiple related cache entries
- **Time-Based Expiration**: Implement TTL-based automatic invalidation
- **Conditional Invalidation**: Invalidate based on business rules
- **Cascading Invalidation**: Clear dependent cache entries
- **Bulk Invalidation**: Efficiently clear multiple cache entries
- **Cross-Service Invalidation**: Coordinate cache clearing across microservices
- **Monitoring**: Track invalidation events and cache consistency
- **Testing**: Verify cache invalidation behavior

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x, Spring Cache, Redis
- **Messaging**: Spring Events, Kafka for cross-service invalidation
- **Configuration**: YAML-based with flexible invalidation strategies

## Template Example

```java
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Caching;
import org.springframework.cache.CacheManager;
import org.springframework.cache.Cache;
import org.springframework.context.ApplicationEventPublisher;
import org.springframework.context.event.EventListener;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.Cursor;
import org.springframework.data.redis.core.ScanOptions;
import org.springframework.transaction.event.TransactionalEventListener;
import org.springframework.transaction.event.TransactionPhase;
import lombok.extern.slf4j.Slf4j;
import lombok.RequiredArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;
import lombok.Builder;
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Cache invalidation strategies for maintaining cache consistency.
 * Implements time-based, event-driven, and manual invalidation patterns.
 */
@Slf4j
public class CacheInvalidationPatterns {

    /**
     * Domain events for cache invalidation.
     */
    @Getter
    @AllArgsConstructor
    public static class UserCreatedEvent {
        private final Long userId;
        private final String username;
        private final LocalDateTime timestamp;
    }

    @Getter
    @AllArgsConstructor
    public static class UserUpdatedEvent {
        private final Long userId;
        private final String username;
        private final LocalDateTime timestamp;
        private final Set<String> modifiedFields;
    }

    @Getter
    @AllArgsConstructor
    public static class UserDeletedEvent {
        private final Long userId;
        private final String username;
        private final LocalDateTime timestamp;
    }

    @Getter
    @AllArgsConstructor
    public static class OrderCreatedEvent {
        private final Long orderId;
        private final Long userId;
        private final LocalDateTime timestamp;
    }

    @Getter
    @AllArgsConstructor
    public static class CacheInvalidationEvent {
        private final String cacheName;
        private final String cacheKey;
        private final String reason;
        private final LocalDateTime timestamp;
    }

    /**
     * User service with declarative cache invalidation.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class UserService {
        private final UserRepository userRepository;
        private final ApplicationEventPublisher eventPublisher;

        /**
         * Get user by ID (cached).
         */
        @org.springframework.cache.annotation.Cacheable(value = "users", key = "#userId")
        public User getUser(Long userId) {
            log.info("Loading user from database: {}", userId);
            return userRepository.findById(userId)
                    .orElseThrow(() -> new RuntimeException("User not found: " + userId));
        }

        /**
         * Create user and publish event for cache warming.
         */
        public User createUser(User user) {
            log.info("Creating user: {}", user.getUsername());
            User savedUser = userRepository.save(user);

            // Publish event for cache warming
            eventPublisher.publishEvent(new UserCreatedEvent(
                    savedUser.getId(),
                    savedUser.getUsername(),
                    LocalDateTime.now()));

            return savedUser;
        }

        /**
         * Update user and evict cache.
         * Evicts both ID-based and username-based cache entries.
         */
        @Caching(evict = {
            @CacheEvict(value = "users", key = "#user.id"),
            @CacheEvict(value = "users", key = "'username:' + #user.username"),
            @CacheEvict(value = "user-profiles", key = "#user.id")
        })
        public User updateUser(User user) {
            log.info("Updating user: {}", user.getId());
            User updatedUser = userRepository.save(user);

            // Publish event for cross-service invalidation
            eventPublisher.publishEvent(new UserUpdatedEvent(
                    updatedUser.getId(),
                    updatedUser.getUsername(),
                    LocalDateTime.now(),
                    Set.of("email", "profile")));

            return updatedUser;
        }

        /**
         * Delete user and evict all related caches.
         */
        @Caching(evict = {
            @CacheEvict(value = "users", key = "#userId"),
            @CacheEvict(value = "users", key = "'username:' + #result.username"),
            @CacheEvict(value = "user-profiles", key = "#userId"),
            @CacheEvict(value = "user-orders", key = "#userId"),
            @CacheEvict(value = "user-preferences", key = "#userId")
        })
        public User deleteUser(Long userId) {
            log.info("Deleting user: {}", userId);
            User user = getUser(userId);
            userRepository.deleteById(userId);

            // Publish event for cleanup
            eventPublisher.publishEvent(new UserDeletedEvent(
                    userId,
                    user.getUsername(),
                    LocalDateTime.now()));

            return user;
        }

        /**
         * Conditional cache eviction based on user status.
         */
        @CacheEvict(value = "users", key = "#userId", condition = "#result.active == false")
        public User deactivateUser(Long userId) {
            log.info("Deactivating user: {}", userId);
            User user = getUser(userId);
            user.setActive(false);
            return userRepository.save(user);
        }

        /**
         * Evict all user caches (admin operation).
         */
        @CacheEvict(value = "users", allEntries = true)
        public void evictAllUsers() {
            log.warn("Evicting all user caches");
        }
    }

    /**
     * Event-driven cache invalidation listener.
     */
    @Component
    @Slf4j
    @RequiredArgsConstructor
    public static class CacheInvalidationListener {
        private final CacheManager cacheManager;
        private final RedisTemplate<String, Object> redisTemplate;
        private final ApplicationEventPublisher eventPublisher;

        /**
         * Handle user created event - warm cache.
         */
        @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
        public void handleUserCreated(UserCreatedEvent event) {
            log.info("User created event received: {}", event.getUserId());
            // Cache warming logic here if needed
        }

        /**
         * Handle user updated event - invalidate related caches.
         */
        @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
        public void handleUserUpdated(UserUpdatedEvent event) {
            log.info("User updated event received: {}, modified fields: {}",
                    event.getUserId(), event.getModifiedFields());

            // Invalidate user-related caches based on modified fields
            if (event.getModifiedFields().contains("email")) {
                evictCache("users", "email:" + event.getUserId());
            }

            if (event.getModifiedFields().contains("profile")) {
                evictCache("user-profiles", String.valueOf(event.getUserId()));
            }

            // Publish cache invalidation event
            publishCacheInvalidationEvent("users", String.valueOf(event.getUserId()),
                    "User updated");
        }

        /**
         * Handle user deleted event - cascade cache invalidation.
         */
        @EventListener
        public void handleUserDeleted(UserDeletedEvent event) {
            log.info("User deleted event received: {}", event.getUserId());

            // Cascade invalidation of related caches
            evictCache("users", String.valueOf(event.getUserId()));
            evictCache("user-profiles", String.valueOf(event.getUserId()));
            evictCache("user-orders", String.valueOf(event.getUserId()));
            evictCache("user-preferences", String.valueOf(event.getUserId()));

            // Clear pattern-based caches
            evictCachesByPattern("user:" + event.getUserId() + ":*");
        }

        /**
         * Handle order created event - invalidate user's order cache.
         */
        @EventListener
        public void handleOrderCreated(OrderCreatedEvent event) {
            log.info("Order created event received for user: {}", event.getUserId());

            // Invalidate user's order cache
            evictCache("user-orders", String.valueOf(event.getUserId()));
            evictCache("order-stats", String.valueOf(event.getUserId()));
        }

        /**
         * Evict specific cache entry.
         */
        private void evictCache(String cacheName, String key) {
            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.evict(key);
                log.debug("Evicted cache: {} key: {}", cacheName, key);
            }
        }

        /**
         * Evict cache entries matching pattern.
         * Uses Redis SCAN for efficient pattern matching.
         */
        private void evictCachesByPattern(String pattern) {
            log.info("Evicting caches matching pattern: {}", pattern);

            Set<String> keys = scanKeys(pattern);
            if (!keys.isEmpty()) {
                redisTemplate.delete(keys);
                log.info("Evicted {} cache entries", keys.size());
            }
        }

        /**
         * Scan Redis keys matching pattern.
         */
        private Set<String> scanKeys(String pattern) {
            Set<String> keys = new HashSet<>();

            try {
                ScanOptions options = ScanOptions.scanOptions()
                        .match(pattern)
                        .count(100)
                        .build();

                Cursor<byte[]> cursor = redisTemplate.getConnectionFactory()
                        .getConnection()
                        .scan(options);

                while (cursor.hasNext()) {
                    keys.add(new String(cursor.next()));
                }

                cursor.close();
            } catch (Exception e) {
                log.error("Error scanning Redis keys: {}", e.getMessage());
            }

            return keys;
        }

        /**
         * Publish cache invalidation event for monitoring.
         */
        private void publishCacheInvalidationEvent(String cacheName, String key, String reason) {
            eventPublisher.publishEvent(new CacheInvalidationEvent(
                    cacheName, key, reason, LocalDateTime.now()));
        }
    }

    /**
     * Scheduled cache invalidation for time-based expiration.
     */
    @Component
    @Slf4j
    @RequiredArgsConstructor
    @EnableScheduling
    public static class ScheduledCacheInvalidation {
        private final CacheManager cacheManager;
        private final RedisTemplate<String, Object> redisTemplate;

        /**
         * Clear expired analytics cache daily at 2 AM.
         */
        @Scheduled(cron = "0 0 2 * * *")
        public void clearExpiredAnalytics() {
            log.info("Running scheduled analytics cache cleanup");

            Cache analyticsCache = cacheManager.getCache("analytics");
            if (analyticsCache != null) {
                analyticsCache.clear();
                log.info("Analytics cache cleared");
            }
        }

        /**
         * Clear inactive user caches hourly.
         */
        @Scheduled(fixedRate = 3600000) // 1 hour
        public void clearInactiveUserCaches() {
            log.info("Clearing inactive user caches");

            // Logic to identify and clear inactive user caches
            String pattern = "users::*";
            Set<String> keys = scanKeysWithTTL(pattern, Duration.ofHours(24));

            if (!keys.isEmpty()) {
                redisTemplate.delete(keys);
                log.info("Cleared {} inactive user cache entries", keys.size());
            }
        }

        /**
         * Clear session caches that haven't been accessed in 1 hour.
         */
        @Scheduled(fixedRate = 600000) // 10 minutes
        public void clearStaleSessions() {
            log.info("Clearing stale session caches");

            String pattern = "sessions::*";
            Set<String> keysToDelete = new HashSet<>();

            Set<String> keys = scanKeys(pattern);
            for (String key : keys) {
                Long ttl = redisTemplate.getExpire(key);
                if (ttl != null && ttl < 3600) { // Less than 1 hour remaining
                    keysToDelete.add(key);
                }
            }

            if (!keysToDelete.isEmpty()) {
                redisTemplate.delete(keysToDelete);
                log.info("Cleared {} stale session entries", keysToDelete.size());
            }
        }

        /**
         * Refresh product catalog cache every 6 hours.
         */
        @Scheduled(fixedRate = 21600000) // 6 hours
        public void refreshProductCatalog() {
            log.info("Refreshing product catalog cache");

            Cache productCache = cacheManager.getCache("products");
            if (productCache != null) {
                productCache.clear();
                log.info("Product catalog cache cleared for refresh");
            }
        }

        private Set<String> scanKeys(String pattern) {
            Set<String> keys = new HashSet<>();

            try {
                ScanOptions options = ScanOptions.scanOptions()
                        .match(pattern)
                        .count(100)
                        .build();

                Cursor<byte[]> cursor = redisTemplate.getConnectionFactory()
                        .getConnection()
                        .scan(options);

                while (cursor.hasNext()) {
                    keys.add(new String(cursor.next()));
                }

                cursor.close();
            } catch (Exception e) {
                log.error("Error scanning Redis keys: {}", e.getMessage());
            }

            return keys;
        }

        private Set<String> scanKeysWithTTL(String pattern, Duration minTTL) {
            Set<String> matchingKeys = new HashSet<>();
            Set<String> allKeys = scanKeys(pattern);

            for (String key : allKeys) {
                Long ttl = redisTemplate.getExpire(key);
                if (ttl != null && ttl > minTTL.getSeconds()) {
                    matchingKeys.add(key);
                }
            }

            return matchingKeys;
        }
    }

    /**
     * Manual cache invalidation service for admin operations.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class CacheInvalidationService {
        private final CacheManager cacheManager;
        private final RedisTemplate<String, Object> redisTemplate;
        private final ApplicationEventPublisher eventPublisher;

        /**
         * Evict specific cache entry.
         */
        public void evictCache(String cacheName, String key) {
            log.info("Evicting cache: {} key: {}", cacheName, key);

            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.evict(key);
                publishInvalidationEvent(cacheName, key, "Manual eviction");
            } else {
                log.warn("Cache not found: {}", cacheName);
            }
        }

        /**
         * Clear all entries in cache.
         */
        public void clearCache(String cacheName) {
            log.info("Clearing all entries in cache: {}", cacheName);

            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.clear();
                publishInvalidationEvent(cacheName, "ALL", "Cache cleared");
            }
        }

        /**
         * Evict multiple cache entries by keys.
         */
        public void evictMultiple(String cacheName, List<String> keys) {
            log.info("Evicting {} entries from cache: {}", keys.size(), cacheName);

            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                keys.forEach(cache::evict);
                publishInvalidationEvent(cacheName,
                        "BULK:" + keys.size(), "Bulk eviction");
            }
        }

        /**
         * Evict cache entries by pattern using Redis SCAN.
         */
        public int evictByPattern(String pattern) {
            log.info("Evicting cache entries matching pattern: {}", pattern);

            Set<String> keys = scanKeys(pattern);
            if (!keys.isEmpty()) {
                redisTemplate.delete(keys);
                publishInvalidationEvent("redis", pattern, "Pattern-based eviction");
            }

            return keys.size();
        }

        /**
         * Evict all caches (nuclear option).
         */
        public void evictAllCaches() {
            log.warn("Evicting ALL caches - nuclear option");

            Collection<String> cacheNames = cacheManager.getCacheNames();
            for (String cacheName : cacheNames) {
                Cache cache = cacheManager.getCache(cacheName);
                if (cache != null) {
                    cache.clear();
                }
            }

            publishInvalidationEvent("ALL", "ALL", "All caches cleared");
        }

        /**
         * Get cache statistics.
         */
        public Map<String, CacheStatistics> getCacheStatistics() {
            Map<String, CacheStatistics> stats = new HashMap<>();

            Collection<String> cacheNames = cacheManager.getCacheNames();
            for (String cacheName : cacheNames) {
                String pattern = cacheName + "::*";
                Set<String> keys = scanKeys(pattern);

                long totalMemory = 0;
                for (String key : keys) {
                    Long size = redisTemplate.opsForValue().size(key);
                    if (size != null) {
                        totalMemory += size;
                    }
                }

                stats.put(cacheName, CacheStatistics.builder()
                        .cacheName(cacheName)
                        .entryCount(keys.size())
                        .totalMemoryBytes(totalMemory)
                        .build());
            }

            return stats;
        }

        /**
         * Refresh cache entry with new data.
         */
        public void refreshCacheEntry(String cacheName, String key, Object value) {
            log.info("Refreshing cache entry: {} key: {}", cacheName, key);

            Cache cache = cacheManager.getCache(cacheName);
            if (cache != null) {
                cache.evict(key);
                cache.put(key, value);
                publishInvalidationEvent(cacheName, key, "Cache refresh");
            }
        }

        private Set<String> scanKeys(String pattern) {
            Set<String> keys = new HashSet<>();

            try {
                ScanOptions options = ScanOptions.scanOptions()
                        .match(pattern)
                        .count(100)
                        .build();

                Cursor<byte[]> cursor = redisTemplate.getConnectionFactory()
                        .getConnection()
                        .scan(options);

                while (cursor.hasNext()) {
                    keys.add(new String(cursor.next()));
                }

                cursor.close();
            } catch (Exception e) {
                log.error("Error scanning Redis keys: {}", e.getMessage());
            }

            return keys;
        }

        private void publishInvalidationEvent(String cacheName, String key, String reason) {
            eventPublisher.publishEvent(new CacheInvalidationEvent(
                    cacheName, key, reason, LocalDateTime.now()));
        }
    }

    /**
     * Cache invalidation monitoring service.
     */
    @Component
    @Slf4j
    public static class CacheInvalidationMonitor {
        private final List<CacheInvalidationEvent> recentEvents = new ArrayList<>();
        private static final int MAX_EVENTS = 1000;

        /**
         * Track cache invalidation events.
         */
        @EventListener
        public void trackInvalidation(CacheInvalidationEvent event) {
            synchronized (recentEvents) {
                recentEvents.add(event);

                // Keep only recent events
                if (recentEvents.size() > MAX_EVENTS) {
                    recentEvents.remove(0);
                }
            }

            log.info("Cache invalidation: cache={}, key={}, reason={}",
                    event.getCacheName(), event.getCacheKey(), event.getReason());
        }

        /**
         * Get recent invalidation events.
         */
        public List<CacheInvalidationEvent> getRecentEvents(int limit) {
            synchronized (recentEvents) {
                return recentEvents.stream()
                        .sorted((e1, e2) -> e2.getTimestamp().compareTo(e1.getTimestamp()))
                        .limit(limit)
                        .collect(Collectors.toList());
            }
        }

        /**
         * Get invalidation statistics.
         */
        public Map<String, Long> getInvalidationStats() {
            synchronized (recentEvents) {
                return recentEvents.stream()
                        .collect(Collectors.groupingBy(
                                CacheInvalidationEvent::getCacheName,
                                Collectors.counting()));
            }
        }
    }

    /**
     * REST controller for cache invalidation operations.
     */
    @org.springframework.web.bind.annotation.RestController
    @org.springframework.web.bind.annotation.RequestMapping("/api/v1/admin/cache")
    @Slf4j
    @RequiredArgsConstructor
    public static class CacheInvalidationController {
        private final CacheInvalidationService cacheInvalidationService;
        private final CacheInvalidationMonitor monitor;

        @org.springframework.web.bind.annotation.DeleteMapping("/{cacheName}/{key}")
        public org.springframework.http.ResponseEntity<String> evictCache(
                @org.springframework.web.bind.annotation.PathVariable String cacheName,
                @org.springframework.web.bind.annotation.PathVariable String key) {

            cacheInvalidationService.evictCache(cacheName, key);
            return org.springframework.http.ResponseEntity.ok(
                    "Cache evicted: " + cacheName + "::" + key);
        }

        @org.springframework.web.bind.annotation.DeleteMapping("/{cacheName}")
        public org.springframework.http.ResponseEntity<String> clearCache(
                @org.springframework.web.bind.annotation.PathVariable String cacheName) {

            cacheInvalidationService.clearCache(cacheName);
            return org.springframework.http.ResponseEntity.ok("Cache cleared: " + cacheName);
        }

        @org.springframework.web.bind.annotation.DeleteMapping("/pattern")
        public org.springframework.http.ResponseEntity<String> evictByPattern(
                @org.springframework.web.bind.annotation.RequestParam String pattern) {

            int count = cacheInvalidationService.evictByPattern(pattern);
            return org.springframework.http.ResponseEntity.ok(
                    "Evicted " + count + " entries matching pattern: " + pattern);
        }

        @org.springframework.web.bind.annotation.GetMapping("/stats")
        public org.springframework.http.ResponseEntity<Map<String, CacheStatistics>> getCacheStats() {
            Map<String, CacheStatistics> stats = cacheInvalidationService.getCacheStatistics();
            return org.springframework.http.ResponseEntity.ok(stats);
        }

        @org.springframework.web.bind.annotation.GetMapping("/invalidations")
        public org.springframework.http.ResponseEntity<List<CacheInvalidationEvent>> getRecentInvalidations(
                @org.springframework.web.bind.annotation.RequestParam(defaultValue = "50") int limit) {

            List<CacheInvalidationEvent> events = monitor.getRecentEvents(limit);
            return org.springframework.http.ResponseEntity.ok(events);
        }

        @org.springframework.web.bind.annotation.GetMapping("/invalidations/stats")
        public org.springframework.http.ResponseEntity<Map<String, Long>> getInvalidationStats() {
            Map<String, Long> stats = monitor.getInvalidationStats();
            return org.springframework.http.ResponseEntity.ok(stats);
        }
    }

    /**
     * Domain models.
     */
    @jakarta.persistence.Entity
    @jakarta.persistence.Table(name = "users")
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class User {
        @jakarta.persistence.Id
        @jakarta.persistence.GeneratedValue(strategy = jakarta.persistence.GenerationType.IDENTITY)
        private Long id;
        private String username;
        private String email;
        private Boolean active;
    }

    public interface UserRepository extends org.springframework.data.jpa.repository.JpaRepository<User, Long> {
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class CacheStatistics {
        private String cacheName;
        private Integer entryCount;
        private Long totalMemoryBytes;
    }
}
```

## Configuration Examples

```yaml
# application.yml

spring:
  cache:
    type: redis
    redis:
      time-to-live: 600000  # 10 minutes default
      cache-null-values: false
      key-prefix: "myapp::"
      use-key-prefix: true

  # Task scheduling for cache invalidation
  task:
    scheduling:
      pool:
        size: 5

# Redis configuration
  data:
    redis:
      host: localhost
      port: 6379
      timeout: 2000ms

# Cache-specific TTL configurations
cache:
  ttl:
    users: 1800000        # 30 minutes
    products: 3600000     # 1 hour
    orders: 300000        # 5 minutes
    analytics: 86400000   # 24 hours
    sessions: 86400000    # 24 hours

# Scheduled invalidation settings
cache-invalidation:
  schedules:
    analytics-cleanup: "0 0 2 * * *"  # Daily at 2 AM
    inactive-users: "0 0 * * * *"     # Hourly
    stale-sessions: "0 */10 * * * *"  # Every 10 minutes
    product-refresh: "0 0 */6 * * *"  # Every 6 hours
```

## Best Practices

1. **Use Appropriate Invalidation Strategy**
   - Time-based: For data with predictable staleness
   - Event-driven: For data changed by application
   - Manual: For admin operations and debugging
   - Pattern-based: For related data structures

2. **Implement Cascading Invalidation**
   - Clear related cache entries together
   - Use domain events for coordination
   - Consider data dependencies
   - Avoid orphaned cache entries

3. **Handle Invalidation Failures**
   - Log invalidation errors
   - Don't fail primary operations
   - Implement retry mechanisms
   - Monitor cache consistency

4. **Use TransactionalEventListener**
   - Invalidate after transaction commit
   - Prevents premature cache clearing
   - Ensures data consistency
   - Handles transaction rollbacks

5. **Monitor Invalidation Patterns**
   - Track invalidation frequency
   - Identify invalidation hotspots
   - Measure cache consistency
   - Alert on excessive invalidations

6. **Use SCAN for Pattern Matching**
   - Avoid blocking KEYS command
   - Use cursor-based iteration
   - Set appropriate batch sizes
   - Handle large key sets efficiently

7. **Coordinate Cross-Service Invalidation**
   - Use message queue (Kafka, RabbitMQ)
   - Publish invalidation events
   - Handle eventual consistency
   - Version cache entries

8. **Test Invalidation Thoroughly**
   - Verify cache clearing after updates
   - Test cascading invalidation
   - Validate event-driven invalidation
   - Check pattern-based clearing

## Related Documentation

- [Spring Cache Abstraction](https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache)
- [Redis SCAN Command](https://redis.io/commands/scan/)
- [Spring Events](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#context-functionality-events)
- [Cache Invalidation Patterns](https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/Strategies.html)
