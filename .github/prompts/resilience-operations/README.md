# Resilience & Operations Library

Comprehensive prompts for implementing resilience patterns and operational best practices in production microservices. Covers graceful shutdown, health checks, circuit breakers, and rate limiting for building fault-tolerant distributed systems.

## Available Prompts

### 1. Create Graceful Shutdown
Implement graceful shutdown to drain active requests, close connections, and ensure zero data loss during deployments.

| Feature | Description |
|---------|-------------|
| Request Draining | Complete in-flight requests before shutdown |
| Connection Closing | Gracefully close database and cache connections |
| Kubernetes Integration | PreStop hooks and readiness probes |
| Timeout Configuration | Configurable shutdown grace period |
| Event Listeners | Spring Boot shutdown hooks |
| Thread Pool Shutdown | Ordered termination of executors |
| Message Queue Drain | Complete message processing before exit |
| Zero Downtime Deploy | Rolling deployments with health checks |

### 2. Create Health Checks
Implement comprehensive health checks for Kubernetes readiness/liveness probes and service mesh integration.

| Feature | Description |
|---------|-------------|
| Liveness Probes | Detect and restart unhealthy containers |
| Readiness Probes | Control traffic routing to healthy instances |
| Custom Health Indicators | Database, cache, message queue checks |
| Downstream Dependencies | Verify external service availability |
| Startup Probes | Handle slow-starting applications |
| Health Check Aggregation | Combined status from multiple checks |
| Circuit Breaker Integration | Report circuit breaker state |
| Performance | Non-blocking health checks |

### 3. Create Circuit Breaker
Implement circuit breaker pattern to prevent cascading failures and enable graceful degradation.

| Feature | Description |
|---------|-------------|
| Resilience4j Integration | Industry-standard circuit breaker |
| State Management | Closed, Open, Half-Open states |
| Failure Threshold | Configurable failure rate triggers |
| Timeout Protection | Prevent slow calls from blocking threads |
| Fallback Strategies | Return cached data or default responses |
| Metrics Integration | Track circuit breaker state changes |
| Reactive Support | Project Reactor integration |
| Spring Boot Auto-Config | Declarative circuit breaker setup |

### 4. Create Rate Limiting
Implement rate limiting to protect services from traffic spikes and prevent resource exhaustion.

| Feature | Description |
|---------|-------------|
| Request Rate Limiting | Limit requests per second/minute/hour |
| User-Based Limits | Per-user or per-tenant rate limits |
| Token Bucket Algorithm | Smooth rate limiting with bursts |
| Distributed Rate Limiting | Redis-based shared rate limiters |
| API Key Throttling | Different limits per API tier |
| HTTP 429 Responses | Proper rate limit exceeded handling |
| Retry-After Headers | Inform clients when to retry |
| Metrics Tracking | Monitor rate limit hits and rejections |

## Quick Start

### 1. Implement Graceful Shutdown

```
@workspace Use .github/prompts/resilience-operations/create-graceful-shutdown.md

Configure graceful shutdown with 30s grace period for Kubernetes deployments
```

### 2. Setup Health Checks

```
@workspace Use .github/prompts/resilience-operations/create-health-checks.md

Add liveness and readiness probes with database and cache health indicators
```

### 3. Add Circuit Breaker

```
@workspace Use .github/prompts/resilience-operations/create-circuit-breaker.md

Implement circuit breaker for external payment service with fallback to cached data
```

### 4. Configure Rate Limiting

```
@workspace Use .github/prompts/resilience-operations/create-rate-limiting.md

Add rate limiting: 1000 requests/minute per user, 10000/minute per service
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Load Balancer                          │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │  API Gateway    │ ← Rate Limiter (Global)
        │  (Ingress)      │   1M requests/hour
        └────────┬────────┘
                 │
         ┌───────┴────────┐
         │                │
         ▼                ▼
    ┌─────────┐      ┌─────────┐
    │Service A│      │Service B│
    │         │      │         │
    │ Health  │      │ Health  │
    │ Checks: │      │ Checks: │
    │ /health │      │ /health │
    │ /ready  │      │ /ready  │
    │         │      │         │
    │ Rate    │      │ Rate    │
    │ Limiter │      │ Limiter │
    │ (User)  │      │ (User)  │
    │ 100/min │      │ 100/min │
    └────┬────┘      └────┬────┘
         │                │
         │ Circuit        │ Circuit
         │ Breaker        │ Breaker
         │                │
         ▼                ▼
    ┌─────────┐      ┌─────────┐
    │External │      │External │
    │ Payment │      │  Email  │
    │ Service │      │ Service │
    └─────────┘      └─────────┘

Graceful Shutdown Flow:
1. SIGTERM received
2. Stop accepting new requests (readiness=false)
3. Wait for in-flight requests (30s grace period)
4. Close database connections
5. Shutdown thread pools
6. Process exits

Circuit Breaker States:
CLOSED → (50% failures) → OPEN
OPEN → (60s wait) → HALF_OPEN
HALF_OPEN → (5 successful calls) → CLOSED
```

## Integration Patterns

### Spring Boot Service with Full Resilience

```java
@Configuration
public class ResilienceConfig {

    /**
     * Circuit breaker for external payment service.
     */
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> defaultCustomizer() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofSeconds(30))
                .slidingWindowSize(10)
                .build())
            .timeLimiterConfig(TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(3))
                .build())
            .build());
    }

    /**
     * Rate limiter with 100 requests per minute.
     */
    @Bean
    public RateLimiter rateLimiter() {
        return RateLimiter.of("payment-service", RateLimiterConfig.custom()
            .limitRefreshPeriod(Duration.ofMinutes(1))
            .limitForPeriod(100)
            .timeoutDuration(Duration.ofSeconds(5))
            .build());
    }
}

@Service
@RequiredArgsConstructor
public class PaymentService {
    private final CircuitBreakerFactory circuitBreakerFactory;
    private final RateLimiter rateLimiter;
    private final PaymentClient paymentClient;

    public Payment processPayment(PaymentRequest request) {
        CircuitBreaker circuitBreaker = circuitBreakerFactory.create("payment");

        // Apply rate limiter and circuit breaker
        return rateLimiter.executeSupplier(
            () -> circuitBreaker.run(
                () -> paymentClient.process(request),
                throwable -> getFallbackPayment(request)
            )
        );
    }

    private Payment getFallbackPayment(PaymentRequest request) {
        // Return cached payment or queue for retry
        return Payment.queued(request);
    }
}
```

### Kubernetes Deployment with Health Checks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    spec:
      containers:
      - name: order-service
        image: order-service:1.0.0
        ports:
        - containerPort: 8080

        # Startup probe: allow 5 minutes for application start
        startupProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          failureThreshold: 30

        # Liveness probe: restart if unhealthy
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3

        # Readiness probe: remove from load balancer if not ready
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 2

        # Graceful shutdown
        lifecycle:
          preStop:
            exec:
              command:
              - /bin/sh
              - -c
              - sleep 15

        env:
        - name: SPRING_LIFECYCLE_TIMEOUT_PER_SHUTDOWN_PHASE
          value: "30s"

      terminationGracePeriodSeconds: 60
```

### Rate Limiting with Redis

```java
@Configuration
public class RateLimitingConfig {

    @Bean
    public RedisRateLimiter redisRateLimiter(ReactiveRedisTemplate<String, String> redisTemplate) {
        return new RedisRateLimiter(100, 100); // 100 requests per second
    }
}

@RestController
@RequiredArgsConstructor
public class OrderController {

    private final RedisRateLimiter rateLimiter;

    @PostMapping("/api/orders")
    public Mono<ResponseEntity<Order>> createOrder(
            @RequestHeader("X-User-ID") String userId,
            @RequestBody CreateOrderRequest request) {

        // Apply rate limiting per user
        return rateLimiter.isAllowed(userId, userId)
            .flatMap(response -> {
                if (!response.isAllowed()) {
                    return Mono.just(
                        ResponseEntity.status(429)
                            .header("X-RateLimit-Remaining", "0")
                            .header("X-RateLimit-Retry-After", "60")
                            .build()
                    );
                }

                return orderService.createOrder(request)
                    .map(order -> ResponseEntity.ok()
                        .header("X-RateLimit-Remaining",
                            String.valueOf(response.getTokensRemaining()))
                        .body(order)
                    );
            });
    }
}
```

## Key Concepts

### Graceful Shutdown
Ensures all in-flight requests complete before application terminates. Prevents connection errors during rolling deployments.

### Health Checks
Monitoring endpoints that report application health status. Used by orchestrators (Kubernetes) to route traffic and restart containers.

### Circuit Breaker
Prevents cascading failures by stopping requests to failing services. Implements fast-fail with fallback strategies.

### Rate Limiting
Controls request rate to prevent resource exhaustion. Protects backend services from traffic spikes and DDoS attacks.

### Bulkhead Pattern
Isolates thread pools and connection pools to prevent resource contention across different operations.

### Retry with Backoff
Automatically retries failed requests with exponential backoff to handle transient failures.

## Best Practices

1. **Implement All Four Patterns Together**
   - Graceful shutdown for zero-downtime deployments
   - Health checks for traffic management
   - Circuit breakers for fault isolation
   - Rate limiting for resource protection

2. **Configure Appropriate Timeouts**
   - Graceful shutdown: 30-60 seconds
   - Health check timeout: 3-5 seconds
   - Circuit breaker wait: 30-60 seconds
   - Rate limit window: 1 minute to 1 hour

3. **Use Separate Liveness and Readiness**
   - Liveness: restart if application is deadlocked
   - Readiness: remove from load balancer if dependencies are down

4. **Set Circuit Breaker Thresholds**
   - Failure rate: 50% over 10 requests
   - Timeout: 3-5 seconds per request
   - Wait in open state: 30-60 seconds

5. **Implement Fallback Strategies**
   - Return cached data
   - Return default values
   - Queue requests for retry
   - Degrade functionality gracefully

6. **Monitor Resilience Metrics**
   - Circuit breaker state changes
   - Rate limit rejections
   - Health check failures
   - Graceful shutdown duration

7. **Test Failure Scenarios**
   - Kill pods during traffic
   - Inject latency in dependencies
   - Simulate database connection pool exhaustion
   - Test with traffic spikes

8. **Use Distributed Rate Limiting**
   - Redis-based rate limiters for multi-instance services
   - Token bucket algorithm for burst handling
   - Per-user and per-service limits

## Operational Commands

### Health Check Verification

```bash
# Liveness check
curl http://localhost:8080/actuator/health/liveness

# Readiness check
curl http://localhost:8080/actuator/health/readiness

# Full health details
curl http://localhost:8080/actuator/health

# Kubernetes health check logs
kubectl logs -f pod/order-service-abc123 | grep health
```

### Circuit Breaker Management

```bash
# Force circuit breaker to open (testing)
curl -X POST http://localhost:8080/actuator/circuitbreakers/payment-service/open

# Force circuit breaker to close
curl -X POST http://localhost:8080/actuator/circuitbreakers/payment-service/close

# Get circuit breaker state
curl http://localhost:8080/actuator/circuitbreakers
```

### Graceful Shutdown Testing

```bash
# Send SIGTERM to process
kill -TERM <pid>

# Kubernetes rolling update (triggers graceful shutdown)
kubectl rollout restart deployment/order-service

# Monitor shutdown logs
kubectl logs -f pod/order-service-abc123 --tail=100
```

## Troubleshooting

### Graceful Shutdown Not Working
1. Check `terminationGracePeriodSeconds` is sufficient (60s+)
2. Verify PreStop hook is configured
3. Check application logs for shutdown timeout errors
4. Ensure connection pools have shutdown hooks

### Health Checks Failing
1. Verify health check endpoints are accessible
2. Check health check timeout is sufficient
3. Review health indicator implementation
4. Check database/cache connectivity

### Circuit Breaker Stuck Open
1. Review failure rate threshold (too low?)
2. Check if downstream service recovered
3. Verify circuit breaker timeout configuration
4. Force transition to half-open for testing

### Rate Limiting Too Aggressive
1. Increase rate limit thresholds
2. Implement tiered rate limits (free/paid)
3. Use burst capacity for traffic spikes
4. Monitor rate limit metrics

## Related Documentation

- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Kubernetes Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)

## Library Navigation

This library focuses on resilience and operations. For related capabilities:
- See **caching-optimization** library for cache-aside patterns and cache warming
- See **observability** library for metrics and alerting on resilience patterns
- See **advanced-testing** library for chaos engineering and resilience testing
