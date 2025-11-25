# Create Health Checks & Readiness Probes

Implement comprehensive health checks and readiness probes for Spring Boot applications. Monitor application and dependency health for load balancing and orchestration decisions.

## Requirements

Your health check implementation should include:

- **Liveness Probes**: Is the application running and responsive?
- **Readiness Probes**: Is the application ready to handle traffic?
- **Custom Indicators**: Database, cache, message queue health
- **HTTP Endpoints**: `/health/live`, `/health/ready` endpoints
- **Health Details**: Detailed status information for diagnostics
- **Dependency Checks**: Verify external service availability
- **Graceful Degradation**: Continue with reduced functionality
- **Metrics Integration**: Track health check results
- **Timeout Configuration**: Prevent hanging checks
- **Kubernetes Integration**: Compatible with K8s probes

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x, Spring Actuator
- **Health**: Spring Boot Health Indicators
- **Monitoring**: Micrometer metrics

## Template Example

```java
import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.actuate.health.Status;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
import org.springframework.jdbc.core.JdbcTemplate;
import com.zaxxer.hikari.HikariDataSource;
import lombok.extern.slf4j.Slf4j;
import lombok.AllArgsConstructor;
import java.util.HashMap;
import java.util.Map;

/**
 * Health checks and readiness probes for Spring Boot applications.
 * Monitors application and dependency health.
 */
@Slf4j
public class HealthChecks {

    /**
     * Database health indicator.
     */
    @Component("database")
    @Slf4j
    @AllArgsConstructor
    public static class DatabaseHealthIndicator implements HealthIndicator {
        private final JdbcTemplate jdbcTemplate;

        @Value("${health.database.query:SELECT 1}")
        private String healthCheckQuery;

        @Override
        public Health health() {
            try {
                long startTime = System.currentTimeMillis();
                Integer result = jdbcTemplate.queryForObject(healthCheckQuery, Integer.class);
                long duration = System.currentTimeMillis() - startTime;

                if (result != null) {
                    return Health.up()
                            .withDetail("duration_ms", duration)
                            .withDetail("status", "Database is responding")
                            .build();
                } else {
                    return Health.down()
                            .withDetail("reason", "Database query returned null")
                            .build();
                }
            } catch (Exception e) {
                log.error("Database health check failed", e);
                return Health.down()
                        .withException(e)
                        .withDetail("error", e.getMessage())
                        .build();
            }
        }
    }

    /**
     * Cache health indicator.
     */
    @Component("cache")
    @Slf4j
    @AllArgsConstructor
    public static class CacheHealthIndicator implements HealthIndicator {
        private final org.springframework.cache.CacheManager cacheManager;

        @Override
        public Health health() {
            try {
                // Check if cache is accessible
                org.springframework.cache.Cache cache = cacheManager.getCache("default");
                if (cache == null) {
                    return Health.down()
                            .withDetail("reason", "Default cache not found")
                            .build();
                }

                // Try a simple cache operation
                String testKey = "__health_check__";
                cache.put(testKey, "test_value");
                Object value = cache.get(testKey, Object.class);
                cache.evict(testKey);

                if ("test_value".equals(value)) {
                    return Health.up()
                            .withDetail("status", "Cache is operational")
                            .build();
                } else {
                    return Health.down()
                            .withDetail("reason", "Cache operation failed")
                            .build();
                }
            } catch (Exception e) {
                log.error("Cache health check failed", e);
                return Health.down()
                        .withException(e)
                        .build();
            }
        }
    }

    /**
     * External service health indicator.
     */
    @Component("external-service")
    @Slf4j
    @AllArgsConstructor
    public static class ExternalServiceHealthIndicator implements HealthIndicator {
        private final RestTemplate restTemplate;

        @Value("${health.external-service.url:http://external-api.example.com/health}")
        private String externalServiceUrl;

        @Value("${health.external-service.timeout-ms:5000}")
        private long timeoutMs;

        @Override
        public Health health() {
            try {
                long startTime = System.currentTimeMillis();
                var response = restTemplate.getForObject(externalServiceUrl, Map.class);
                long duration = System.currentTimeMillis() - startTime;

                if (duration > timeoutMs) {
                    return Health.degraded()
                            .withDetail("duration_ms", duration)
                            .withDetail("warning", "Service responding but slow")
                            .build();
                }

                return Health.up()
                        .withDetail("duration_ms", duration)
                        .withDetail("status", "External service is healthy")
                        .build();

            } catch (org.springframework.web.client.ConnectTimeoutException e) {
                log.warn("External service health check timeout");
                return Health.down()
                        .withDetail("reason", "Connection timeout")
                        .build();
            } catch (Exception e) {
                log.error("External service health check failed", e);
                return Health.down()
                        .withException(e)
                        .build();
            }
        }
    }

    /**
     * Message queue health indicator.
     */
    @Component("message-queue")
    @Slf4j
    @AllArgsConstructor
    public static class MessageQueueHealthIndicator implements HealthIndicator {
        @Value("${health.message-queue.enabled:false}")
        private boolean enabled;

        @Override
        public Health health() {
            if (!enabled) {
                return Health.unknown()
                        .withDetail("status", "Message queue health check disabled")
                        .build();
            }

            try {
                // Check message queue connectivity
                boolean isConnected = checkMessageQueueConnectivity();

                if (isConnected) {
                    return Health.up()
                            .withDetail("status", "Message queue is operational")
                            .build();
                } else {
                    return Health.down()
                            .withDetail("reason", "Cannot connect to message queue")
                            .build();
                }

            } catch (Exception e) {
                log.error("Message queue health check failed", e);
                return Health.down()
                        .withException(e)
                        .build();
            }
        }

        private boolean checkMessageQueueConnectivity() {
            // Implement message queue connectivity check
            return true;
        }
    }

    /**
     * Disk space health indicator.
     */
    @Component("disk-space")
    @Slf4j
    public static class DiskSpaceHealthIndicator implements HealthIndicator {
        @Value("${health.disk-space.threshold-mb:100}")
        private long thresholdMb;

        @Override
        public Health health() {
            java.io.File root = new java.io.File("/");
            long freeSpace = root.getFreeSpace();
            long freeMb = freeSpace / (1024 * 1024);

            if (freeMb < thresholdMb) {
                return Health.down()
                        .withDetail("free_space_mb", freeMb)
                        .withDetail("threshold_mb", thresholdMb)
                        .withDetail("reason", "Disk space below threshold")
                        .build();
            }

            return Health.up()
                    .withDetail("free_space_mb", freeMb)
                    .withDetail("status", "Sufficient disk space available")
                    .build();
        }
    }

    /**
     * Memory health indicator.
     */
    @Component("memory")
    @Slf4j
    public static class MemoryHealthIndicator implements HealthIndicator {
        @Value("${health.memory.threshold-percent:90}")
        private int thresholdPercent;

        @Override
        public Health health() {
            Runtime runtime = Runtime.getRuntime();
            long totalMemory = runtime.totalMemory();
            long freeMemory = runtime.freeMemory();
            long usedMemory = totalMemory - freeMemory;
            int usagePercent = (int) ((usedMemory * 100) / totalMemory);

            if (usagePercent > thresholdPercent) {
                return Health.degraded()
                        .withDetail("used_percent", usagePercent)
                        .withDetail("threshold_percent", thresholdPercent)
                        .withDetail("used_mb", usedMemory / (1024 * 1024))
                        .withDetail("total_mb", totalMemory / (1024 * 1024))
                        .build();
            }

            return Health.up()
                    .withDetail("used_percent", usagePercent)
                    .withDetail("used_mb", usedMemory / (1024 * 1024))
                    .withDetail("total_mb", totalMemory / (1024 * 1024))
                    .build();
        }
    }

    /**
     * Readiness probe implementation.
     */
    @Service
    @Slf4j
    @AllArgsConstructor
    public static class ReadinessProbe {
        private final org.springframework.boot.actuate.health.HealthEndpoint healthEndpoint;

        /**
         * Check if application is ready to receive traffic.
         */
        public boolean isReady() {
            try {
                org.springframework.boot.actuate.health.HealthComponent health =
                        healthEndpoint.health();

                if (health.getStatus() == Status.UP) {
                    log.debug("Application is ready");
                    return true;
                } else if (health.getStatus() == Status.DEGRADED) {
                    log.warn("Application is degraded but ready");
                    return true;
                } else {
                    log.warn("Application is not ready: {}", health.getStatus());
                    return false;
                }

            } catch (Exception e) {
                log.error("Error checking readiness", e);
                return false;
            }
        }

        /**
         * Check specific component readiness.
         */
        public boolean isComponentReady(String componentName) {
            try {
                org.springframework.boot.actuate.health.HealthComponent health =
                        healthEndpoint.health();

                // Implementation would check specific component status
                return health.getStatus() != Status.DOWN;

            } catch (Exception e) {
                log.error("Error checking component readiness: {}", componentName, e);
                return false;
            }
        }
    }

    /**
     * Liveness probe implementation.
     */
    @Service
    @Slf4j
    public static class LivenessProbe {

        /**
         * Check if application is alive (not hanging/deadlocked).
         */
        public boolean isAlive() {
            try {
                // Simple check: can we allocate memory?
                byte[] test = new byte[1024 * 1024];
                return true;
            } catch (OutOfMemoryError e) {
                log.error("Application is not alive - OutOfMemoryError", e);
                return false;
            }
        }

        /**
         * Check if main thread is responsive.
         */
        public boolean isMainThreadResponsive(long timeoutMs) {
            Thread mainThread = Thread.currentThread();
            java.util.concurrent.CountDownLatch latch = new java.util.concurrent.CountDownLatch(1);

            new Thread(() -> {
                latch.countDown();
            }).start();

            try {
                return latch.await(timeoutMs, java.util.concurrent.TimeUnit.MILLISECONDS);
            } catch (InterruptedException e) {
                return false;
            }
        }
    }

    /**
     * Best practices for health checks.
     */
    public static class BestPractices {
        /**
         * 1. Separate liveness and readiness probes
         *    - Liveness: Is the application running?
         *    - Readiness: Should it receive traffic?
         */

        /**
         * 2. Check all critical dependencies
         *    - Database connectivity
         *    - Cache availability
         *    - Message queue connection
         *    - External service availability
         */

        /**
         * 3. Set appropriate timeouts
         *    - Keep check duration short (< 5 seconds)
         *    - Prevent cascading failures
         *    - Fail fast on unavailable dependencies
         */

        /**
         * 4. Use health check thresholds
         *    - Memory usage > 90%: DEGRADED
         *    - Disk space < 100MB: DOWN
         *    - Database response > 5s: DEGRADED
         */

        /**
         * 5. Implement graceful degradation
         *    - Continue with reduced functionality
         *    - Cache stale data if DB unavailable
         *    - Return 503 when critical service down
         */

        /**
         * 6. Monitor health check results
         *    - Track check success rate
         *    - Alert on repeated failures
         *    - Correlate with performance metrics
         */

        /**
         * 7. Document probe configuration
         *    - Initial delay times
         *    - Failure thresholds
         *    - Period between checks
         *    - Timeout durations
         */

        /**
         * 8. Test health checks regularly
         *    - Test with dependencies offline
         *    - Verify correct status codes
         *    - Test timeout behavior
         */
    }
}
```

## Configuration Examples

```properties
# application.properties

# Health check configuration
management.health.defaults.enabled=true
management.health.diskspace.enabled=true
management.health.memory.enabled=true

# Endpoint exposure
management.endpoints.web.exposure.include=health,info
management.endpoint.health.show-details=when-authorized

# Custom health check timeouts
health.database.query=SELECT 1
health.external-service.url=http://external-api.example.com/health
health.external-service.timeout-ms=5000
health.message-queue.enabled=true
health.disk-space.threshold-mb=100
health.memory.threshold-percent=90
```

## Kubernetes Probe Configuration

```yaml
livenessProbe:
  httpGet:
    path: /health/live
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  timeoutSeconds: 5
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 2
```

## Related Documentation

- [Spring Boot Actuator Health](https://spring.io/blog/2020/03/25/liveness-and-readiness-probes-with-spring-boot)
- [Kubernetes Liveness/Readiness Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Micrometer Metrics](https://micrometer.io/)
```
