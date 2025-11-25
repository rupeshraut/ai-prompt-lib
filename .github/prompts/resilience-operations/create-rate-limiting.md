# Create Rate Limiting

Implement rate limiting and traffic shaping for Spring Boot applications using Resilience4j. Protect services from traffic spikes, enforce API quotas, and prevent resource exhaustion through intelligent request throttling.

## Requirements

Your rate limiting implementation should include:

- **Request Throttling**: Limit requests per time window (seconds, minutes, hours)
- **Per-User Rate Limits**: Different quotas for user tiers (free, premium, enterprise)
- **IP-Based Limiting**: Rate limit by client IP address
- **API Key Quotas**: Track and enforce API key usage limits
- **Distributed Rate Limiting**: Redis-backed counters for multi-instance deployments
- **Graceful Degradation**: Queue or reject excess requests appropriately
- **Rate Limit Headers**: Return remaining quota in HTTP headers
- **Custom Responses**: Return 429 Too Many Requests with retry information
- **Metrics and Monitoring**: Track rate limit violations and usage patterns
- **Dynamic Configuration**: Adjust rate limits without redeployment

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x, Resilience4j, Redis
- **Rate Limiting**: Resilience4j RateLimiter, Bucket4j for advanced scenarios
- **Configuration**: YAML-based with external configuration support

## Template Example

```java
import io.github.resilience4j.ratelimiter.RateLimiter;
import io.github.resilience4j.ratelimiter.RateLimiterConfig;
import io.github.resilience4j.ratelimiter.RateLimiterRegistry;
import io.github.resilience4j.ratelimiter.RequestNotPermitted;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import io.github.resilience4j.reactor.ratelimiter.operator.RateLimiterOperator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.StringRedisTemplate;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import reactor.core.publisher.Mono;
import lombok.extern.slf4j.Slf4j;
import lombok.RequiredArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;
import java.time.Duration;
import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

/**
 * Rate limiting implementation using Resilience4j and Redis.
 * Protects APIs from abuse and enforces usage quotas.
 */
@Slf4j
public class RateLimitingPatterns {

    /**
     * Rate limiter configuration for different API tiers.
     */
    @Configuration
    @Slf4j
    public static class RateLimiterConfiguration {

        /**
         * Default rate limiter configuration.
         * 100 requests per 60 seconds.
         */
        @Bean
        public RateLimiterConfig defaultRateLimiterConfig() {
            return RateLimiterConfig.custom()
                    // Maximum number of permits per refresh period
                    .limitForPeriod(100)
                    // Time period for limit refresh
                    .limitRefreshPeriod(Duration.ofSeconds(60))
                    // Maximum time to wait for permission
                    .timeoutDuration(Duration.ofMillis(500))
                    .build();
        }

        /**
         * Rate limiter registry with tier-specific configurations.
         */
        @Bean
        public RateLimiterRegistry rateLimiterRegistry(RateLimiterConfig defaultRateLimiterConfig) {
            RateLimiterRegistry registry = RateLimiterRegistry.of(defaultRateLimiterConfig);

            // Free tier - 10 requests per minute
            RateLimiterConfig freeTierConfig = RateLimiterConfig.custom()
                    .limitForPeriod(10)
                    .limitRefreshPeriod(Duration.ofMinutes(1))
                    .timeoutDuration(Duration.ofMillis(100))
                    .build();
            registry.rateLimiter("free-tier", freeTierConfig);

            // Premium tier - 100 requests per minute
            RateLimiterConfig premiumTierConfig = RateLimiterConfig.custom()
                    .limitForPeriod(100)
                    .limitRefreshPeriod(Duration.ofMinutes(1))
                    .timeoutDuration(Duration.ofMillis(500))
                    .build();
            registry.rateLimiter("premium-tier", premiumTierConfig);

            // Enterprise tier - 1000 requests per minute
            RateLimiterConfig enterpriseTierConfig = RateLimiterConfig.custom()
                    .limitForPeriod(1000)
                    .limitRefreshPeriod(Duration.ofMinutes(1))
                    .timeoutDuration(Duration.ofMillis(1000))
                    .build();
            registry.rateLimiter("enterprise-tier", enterpriseTierConfig);

            // IP-based rate limiter - 50 requests per minute per IP
            RateLimiterConfig ipBasedConfig = RateLimiterConfig.custom()
                    .limitForPeriod(50)
                    .limitRefreshPeriod(Duration.ofMinutes(1))
                    .timeoutDuration(Duration.ofMillis(200))
                    .build();
            registry.rateLimiter("ip-based", ipBasedConfig);

            // Register event listeners
            registry.rateLimiter("free-tier")
                    .getEventPublisher()
                    .onSuccess(event -> log.debug("Free tier request permitted"))
                    .onFailure(event -> log.warn("Free tier rate limit exceeded"));

            return registry;
        }
    }

    /**
     * API service with rate limiting using annotations.
     */
    @Service
    @Slf4j
    public static class UserApiService {

        /**
         * Get user profile with rate limiting.
         * Falls back to cached profile when rate limit exceeded.
         */
        @io.github.resilience4j.ratelimiter.annotation.RateLimiter(
                name = "premium-tier",
                fallbackMethod = "getUserProfileFallback")
        public UserProfile getUserProfile(String userId) {
            log.info("Fetching user profile: {}", userId);

            // Simulate database query
            return UserProfile.builder()
                    .userId(userId)
                    .username("user" + userId)
                    .email(userId + "@example.com")
                    .tier("premium")
                    .build();
        }

        /**
         * Fallback method when rate limit is exceeded.
         */
        private UserProfile getUserProfileFallback(String userId, RequestNotPermitted ex) {
            log.warn("Rate limit exceeded for user profile request: {}", userId);

            // Return cached or minimal profile
            return UserProfile.builder()
                    .userId(userId)
                    .username("Cached User")
                    .email("Rate limit exceeded")
                    .tier("unknown")
                    .build();
        }

        /**
         * Search users with rate limiting.
         */
        @io.github.resilience4j.ratelimiter.annotation.RateLimiter(name = "free-tier")
        public java.util.List<UserProfile> searchUsers(String query) {
            log.info("Searching users with query: {}", query);

            // Simulate search operation
            return java.util.List.of(
                    UserProfile.builder().userId("1").username("user1").build(),
                    UserProfile.builder().userId("2").username("user2").build()
            );
        }
    }

    /**
     * Programmatic rate limiting with custom logic.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class OrderApiService {
        private final RateLimiterRegistry rateLimiterRegistry;

        /**
         * Create order with dynamic rate limiter based on user tier.
         */
        public OrderResponse createOrder(String userId, OrderRequest request) {
            // Get user's tier
            String userTier = getUserTier(userId);
            String rateLimiterName = userTier + "-tier";

            RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter(rateLimiterName);

            // Attempt to acquire permission
            boolean permitted = rateLimiter.acquirePermission();

            if (!permitted) {
                log.warn("Rate limit exceeded for user: {} (tier: {})", userId, userTier);
                throw new RateLimitExceededException(
                        "Rate limit exceeded. Please try again later.",
                        rateLimiter.getRateLimiterConfig().getLimitRefreshPeriod());
            }

            log.info("Order created for user: {} (tier: {})", userId, userTier);

            // Process order
            return OrderResponse.builder()
                    .orderId("ORDER-" + System.currentTimeMillis())
                    .status("CREATED")
                    .userId(userId)
                    .amount(request.getAmount())
                    .build();
        }

        /**
         * Try-with-timeout pattern for rate limiting.
         */
        public OrderResponse createOrderWithTimeout(String userId, OrderRequest request) {
            String userTier = getUserTier(userId);
            RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter(userTier + "-tier");

            try {
                // Wait up to 1 second for permission
                boolean permitted = rateLimiter.acquirePermission(1);

                if (!permitted) {
                    throw new RateLimitExceededException(
                            "Rate limit exceeded after waiting",
                            rateLimiter.getRateLimiterConfig().getLimitRefreshPeriod());
                }

                return processOrder(userId, request);

            } catch (RequestNotPermitted ex) {
                log.warn("Rate limit timeout for user: {}", userId);
                throw new RateLimitExceededException(
                        "Service temporarily unavailable. Please try again.",
                        Duration.ofSeconds(30));
            }
        }

        private String getUserTier(String userId) {
            // Retrieve user tier from database or cache
            return "premium";  // Default for demo
        }

        private OrderResponse processOrder(String userId, OrderRequest request) {
            return OrderResponse.builder()
                    .orderId("ORDER-" + System.currentTimeMillis())
                    .status("CREATED")
                    .userId(userId)
                    .amount(request.getAmount())
                    .build();
        }
    }

    /**
     * Distributed rate limiting using Redis.
     * Supports multi-instance deployments.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class DistributedRateLimiter {
        private final StringRedisTemplate redisTemplate;

        /**
         * Check and increment rate limit counter in Redis.
         * Returns true if request is permitted.
         */
        public boolean allowRequest(String key, int maxRequests, Duration window) {
            String redisKey = "rate_limit:" + key;
            Long currentCount = redisTemplate.opsForValue().increment(redisKey);

            if (currentCount == null) {
                log.error("Redis increment failed for key: {}", redisKey);
                return true;  // Fail open
            }

            // Set expiration on first request
            if (currentCount == 1) {
                redisTemplate.expire(redisKey, window.getSeconds(), TimeUnit.SECONDS);
            }

            boolean permitted = currentCount <= maxRequests;

            if (!permitted) {
                log.warn("Distributed rate limit exceeded for key: {} (count: {}, limit: {})",
                        key, currentCount, maxRequests);
            }

            return permitted;
        }

        /**
         * Get remaining quota for key.
         */
        public RateLimitQuota getQuota(String key, int maxRequests) {
            String redisKey = "rate_limit:" + key;
            String countStr = redisTemplate.opsForValue().get(redisKey);
            int current = countStr != null ? Integer.parseInt(countStr) : 0;
            int remaining = Math.max(0, maxRequests - current);

            Long ttl = redisTemplate.getExpire(redisKey, TimeUnit.SECONDS);
            Duration resetIn = ttl != null && ttl > 0 ? Duration.ofSeconds(ttl) : Duration.ZERO;

            return RateLimitQuota.builder()
                    .limit(maxRequests)
                    .remaining(remaining)
                    .resetIn(resetIn)
                    .build();
        }

        /**
         * Reset rate limit for key (admin operation).
         */
        public void resetRateLimit(String key) {
            String redisKey = "rate_limit:" + key;
            redisTemplate.delete(redisKey);
            log.info("Rate limit reset for key: {}", key);
        }
    }

    /**
     * IP-based rate limiting filter.
     */
    @Component
    @Slf4j
    @RequiredArgsConstructor
    public static class IpRateLimitFilter extends org.springframework.web.filter.OncePerRequestFilter {
        private final DistributedRateLimiter distributedRateLimiter;

        // 50 requests per minute per IP
        private static final int MAX_REQUESTS_PER_IP = 50;
        private static final Duration WINDOW = Duration.ofMinutes(1);

        @Override
        protected void doFilterInternal(
                jakarta.servlet.http.HttpServletRequest request,
                jakarta.servlet.http.HttpServletResponse response,
                jakarta.servlet.FilterChain filterChain)
                throws jakarta.servlet.ServletException, java.io.IOException {

            String clientIp = getClientIpAddress(request);
            String rateLimitKey = "ip:" + clientIp;

            // Check rate limit
            boolean permitted = distributedRateLimiter.allowRequest(
                    rateLimitKey, MAX_REQUESTS_PER_IP, WINDOW);

            if (!permitted) {
                log.warn("IP rate limit exceeded for: {}", clientIp);

                // Get quota for response headers
                RateLimitQuota quota = distributedRateLimiter.getQuota(rateLimitKey, MAX_REQUESTS_PER_IP);

                // Set rate limit headers
                response.setHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS_PER_IP));
                response.setHeader("X-RateLimit-Remaining", "0");
                response.setHeader("X-RateLimit-Reset", String.valueOf(quota.getResetIn().getSeconds()));
                response.setHeader("Retry-After", String.valueOf(quota.getResetIn().getSeconds()));

                response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
                response.getWriter().write("{\"error\":\"Rate limit exceeded\",\"retryAfter\":" +
                        quota.getResetIn().getSeconds() + "}");
                return;
            }

            // Add rate limit headers to successful responses
            RateLimitQuota quota = distributedRateLimiter.getQuota(rateLimitKey, MAX_REQUESTS_PER_IP);
            response.setHeader("X-RateLimit-Limit", String.valueOf(MAX_REQUESTS_PER_IP));
            response.setHeader("X-RateLimit-Remaining", String.valueOf(quota.getRemaining()));
            response.setHeader("X-RateLimit-Reset", String.valueOf(quota.getResetIn().getSeconds()));

            filterChain.doFilter(request, response);
        }

        private String getClientIpAddress(HttpServletRequest request) {
            String ip = request.getHeader("X-Forwarded-For");
            if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
                ip = request.getHeader("X-Real-IP");
            }
            if (ip == null || ip.isEmpty() || "unknown".equalsIgnoreCase(ip)) {
                ip = request.getRemoteAddr();
            }
            // Handle multiple IPs (take first)
            if (ip != null && ip.contains(",")) {
                ip = ip.split(",")[0].trim();
            }
            return ip;
        }
    }

    /**
     * Reactive rate limiting for WebFlux applications.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class ReactiveApiService {
        private final RateLimiterRegistry rateLimiterRegistry;

        /**
         * Process request with reactive rate limiting.
         */
        public Mono<ApiResponse> processRequestReactive(String userId, ApiRequest request) {
            RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter("premium-tier");

            return Mono.just(request)
                    .transformDeferred(RateLimiterOperator.of(rateLimiter))
                    .flatMap(req -> {
                        log.info("Processing request for user: {}", userId);
                        return Mono.just(ApiResponse.builder()
                                .message("Request processed successfully")
                                .userId(userId)
                                .timestamp(LocalDateTime.now())
                                .build());
                    })
                    .onErrorResume(RequestNotPermitted.class, error -> {
                        log.warn("Rate limit exceeded for user: {}", userId);
                        return Mono.just(ApiResponse.builder()
                                .message("Rate limit exceeded")
                                .userId(userId)
                                .timestamp(LocalDateTime.now())
                                .build());
                    });
        }
    }

    /**
     * REST controller with rate limiting.
     */
    @RestController
    @RequestMapping("/api/v1")
    @Slf4j
    @RequiredArgsConstructor
    public static class RateLimitedApiController {
        private final UserApiService userApiService;
        private final OrderApiService orderApiService;
        private final DistributedRateLimiter distributedRateLimiter;

        /**
         * Get user profile (rate limited by annotation).
         */
        @GetMapping("/users/{userId}")
        public ResponseEntity<UserProfile> getUserProfile(@PathVariable String userId) {
            UserProfile profile = userApiService.getUserProfile(userId);
            return ResponseEntity.ok(profile);
        }

        /**
         * Create order (programmatic rate limiting).
         */
        @PostMapping("/orders")
        public ResponseEntity<OrderResponse> createOrder(
                @RequestHeader("X-User-Id") String userId,
                @RequestBody OrderRequest request) {

            try {
                OrderResponse response = orderApiService.createOrder(userId, request);
                return ResponseEntity.ok(response);

            } catch (RateLimitExceededException ex) {
                return ResponseEntity
                        .status(HttpStatus.TOO_MANY_REQUESTS)
                        .header("Retry-After", String.valueOf(ex.getRetryAfter().getSeconds()))
                        .body(OrderResponse.builder()
                                .status("RATE_LIMITED")
                                .message(ex.getMessage())
                                .build());
            }
        }

        /**
         * Admin endpoint to get rate limit quota.
         */
        @GetMapping("/admin/rate-limits/{key}")
        public ResponseEntity<RateLimitQuota> getQuota(@PathVariable String key) {
            RateLimitQuota quota = distributedRateLimiter.getQuota(key, 100);
            return ResponseEntity.ok(quota);
        }

        /**
         * Admin endpoint to reset rate limit.
         */
        @PostMapping("/admin/rate-limits/{key}/reset")
        public ResponseEntity<String> resetRateLimit(@PathVariable String key) {
            distributedRateLimiter.resetRateLimit(key);
            return ResponseEntity.ok("Rate limit reset for: " + key);
        }
    }

    /**
     * Custom exception for rate limit violations.
     */
    public static class RateLimitExceededException extends RuntimeException {
        @Getter
        private final Duration retryAfter;

        public RateLimitExceededException(String message, Duration retryAfter) {
            super(message);
            this.retryAfter = retryAfter;
        }
    }

    /**
     * Global exception handler for rate limiting.
     */
    @org.springframework.web.bind.annotation.RestControllerAdvice
    @Slf4j
    public static class RateLimitExceptionHandler {

        @org.springframework.web.bind.annotation.ExceptionHandler(RequestNotPermitted.class)
        public ResponseEntity<ErrorResponse> handleRateLimitExceeded(RequestNotPermitted ex) {
            log.warn("Rate limit exceeded: {}", ex.getMessage());

            return ResponseEntity
                    .status(HttpStatus.TOO_MANY_REQUESTS)
                    .header("Retry-After", "60")
                    .body(ErrorResponse.builder()
                            .error("Rate limit exceeded")
                            .message("Too many requests. Please try again later.")
                            .retryAfter(60)
                            .build());
        }

        @org.springframework.web.bind.annotation.ExceptionHandler(RateLimitExceededException.class)
        public ResponseEntity<ErrorResponse> handleRateLimitExceeded(RateLimitExceededException ex) {
            return ResponseEntity
                    .status(HttpStatus.TOO_MANY_REQUESTS)
                    .header("Retry-After", String.valueOf(ex.getRetryAfter().getSeconds()))
                    .body(ErrorResponse.builder()
                            .error("Rate limit exceeded")
                            .message(ex.getMessage())
                            .retryAfter((int) ex.getRetryAfter().getSeconds())
                            .build());
        }
    }

    /**
     * Domain models.
     */
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class UserProfile {
        private String userId;
        private String username;
        private String email;
        private String tier;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class OrderRequest {
        private String productId;
        private Integer quantity;
        private Double amount;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class OrderResponse {
        private String orderId;
        private String status;
        private String userId;
        private String message;
        private Double amount;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class ApiRequest {
        private String action;
        private java.util.Map<String, Object> data;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class ApiResponse {
        private String message;
        private String userId;
        private LocalDateTime timestamp;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class RateLimitQuota {
        private Integer limit;
        private Integer remaining;
        private Duration resetIn;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class ErrorResponse {
        private String error;
        private String message;
        private Integer retryAfter;
    }
}
```

## Configuration Examples

```yaml
# application.yml

resilience4j:
  ratelimiter:
    configs:
      default:
        limit-for-period: 100
        limit-refresh-period: 60s
        timeout-duration: 500ms
        register-health-indicator: true

    instances:
      # Free tier - 10 requests per minute
      free-tier:
        limit-for-period: 10
        limit-refresh-period: 1m
        timeout-duration: 100ms

      # Premium tier - 100 requests per minute
      premium-tier:
        limit-for-period: 100
        limit-refresh-period: 1m
        timeout-duration: 500ms

      # Enterprise tier - 1000 requests per minute
      enterprise-tier:
        limit-for-period: 1000
        limit-refresh-period: 1m
        timeout-duration: 1s

      # IP-based - 50 requests per minute
      ip-based:
        limit-for-period: 50
        limit-refresh-period: 1m
        timeout-duration: 200ms

# Redis configuration for distributed rate limiting
spring:
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

# Actuator for monitoring
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,ratelimiters
  endpoint:
    health:
      show-details: always
  health:
    ratelimiters:
      enabled: true
```

## Best Practices

1. **Choose Appropriate Limits**
   - Base limits on API capacity and SLA requirements
   - Free tier: 10-50 requests/minute
   - Premium tier: 100-500 requests/minute
   - Enterprise tier: 1000+ requests/minute

2. **Return Proper HTTP Headers**
   - `X-RateLimit-Limit`: Maximum requests allowed
   - `X-RateLimit-Remaining`: Requests remaining in window
   - `X-RateLimit-Reset`: Seconds until limit resets
   - `Retry-After`: When to retry (for 429 responses)

3. **Use Distributed Rate Limiting for Scaling**
   - Redis for shared counters across instances
   - Consistent hashing for better distribution
   - Handle Redis failures gracefully (fail open)

4. **Implement Multiple Rate Limit Layers**
   - IP-based: Prevent abuse from single source
   - User-based: Enforce tier quotas
   - API key-based: Track partner usage
   - Global: Protect overall system capacity

5. **Provide Clear Error Messages**
   - Explain why request was rejected
   - Include retry information
   - Suggest upgrade path for premium features

6. **Monitor Rate Limit Metrics**
   - Track violation rates per tier
   - Alert on sustained limit breaches
   - Analyze patterns for capacity planning

7. **Test Rate Limiting Behavior**
   - Verify limits are enforced correctly
   - Test across multiple instances
   - Ensure proper header inclusion
   - Validate fallback behavior

8. **Consider Burst Allowances**
   - Allow temporary spikes within reason
   - Use token bucket algorithm for smooth traffic
   - Balance protection with user experience

## Related Documentation

- [Resilience4j RateLimiter](https://resilience4j.readme.io/docs/ratelimiter)
- [Redis Rate Limiting Patterns](https://redis.io/docs/manual/patterns/rate-limiter/)
- [HTTP 429 Too Many Requests](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/429)
- [API Rate Limiting Best Practices](https://cloud.google.com/architecture/rate-limiting-strategies-techniques)
