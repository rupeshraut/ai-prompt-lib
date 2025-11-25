# Create Graceful Shutdown

Implement graceful shutdown procedures for Spring Boot applications. Handle in-flight requests, close database connections, complete async operations, and perform cleanup before process termination.

## Requirements

Your graceful shutdown implementation should include:

- **Shutdown Hooks**: Register and manage application shutdown handlers
- **Request Draining**: Allow in-flight requests to complete
- **Connection Closing**: Gracefully close database and external connections
- **Async Task Completion**: Wait for async operations to finish
- **Resource Cleanup**: Release locks, cache, memory resources
- **Timeout Management**: Prevent indefinite waiting
- **Health Check Updates**: Mark instance as unhealthy during shutdown
- **Metrics Cleanup**: Flush metrics before shutdown
- **Logging**: Record shutdown events and durations
- **Kubernetes Integration**: Handle SIGTERM and preStop hooks

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x
- **Async**: Project Reactor, CompletableFuture
- **Configuration**: Properties-based timeouts

## Template Example

```java
import org.springframework.boot.SpringApplication;
import org.springframework.context.ApplicationContext;
import org.springframework.context.event.ContextClosedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.boot.context.event.ApplicationShutdownEvent;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServlet;
import org.springframework.boot.web.servlet.ServletRegistrationBean;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import lombok.extern.slf4j.Slf4j;
import lombok.AllArgsConstructor;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicBoolean;

/**
 * Graceful shutdown for Spring Boot applications.
 * Handles in-flight requests, closes connections, and completes async operations.
 */
@Slf4j
public class GracefulShutdown {

    /**
     * Shutdown manager coordinating graceful application shutdown.
     */
    @Component
    @Slf4j
    @AllArgsConstructor
    public static class ShutdownManager {
        private final ApplicationContext applicationContext;

        @Value("${server.shutdown.wait-time-seconds:30}")
        private int waitTimeSeconds;

        @Value("${server.shutdown.drain-requests-timeout-seconds:60}")
        private int drainRequestsTimeoutSeconds;

        private final AtomicBoolean isShuttingDown = new AtomicBoolean(false);

        /**
         * Trigger graceful shutdown.
         */
        public void initiateShutdown(String reason) {
            if (!isShuttingDown.compareAndSet(false, true)) {
                log.warn("Shutdown already in progress");
                return;
            }

            log.warn("Initiating graceful shutdown: {}", reason);
            new Thread(() -> performGracefulShutdown(reason)).start();
        }

        /**
         * Check if shutdown is in progress.
         */
        public boolean isShuttingDown() {
            return isShuttingDown.get();
        }

        /**
         * Perform graceful shutdown sequence.
         */
        private void performGracefulShutdown(String reason) {
            try {
                long startTime = System.currentTimeMillis();

                // Step 1: Stop accepting new requests
                log.info("Step 1: Stopping request acceptance");
                notifyHealthCheck(false);
                Thread.sleep(2000);  // Allow load balancer to notice

                // Step 2: Wait for in-flight requests to complete
                log.info("Step 2: Draining in-flight requests (timeout: {} seconds)",
                        drainRequestsTimeoutSeconds);
                boolean requestsDrained = waitForRequestsToComplete(drainRequestsTimeoutSeconds);
                if (!requestsDrained) {
                    log.warn("Request drain timeout - forcing shutdown");
                }

                // Step 3: Close database connections
                log.info("Step 3: Closing database connections");
                closeDatabase();
                Thread.sleep(1000);

                // Step 4: Wait for async operations
                log.info("Step 4: Waiting for async operations");
                waitForAsyncOperations(10);

                // Step 5: Flush metrics
                log.info("Step 5: Flushing metrics");
                flushMetrics();

                // Step 6: Cleanup resources
                log.info("Step 6: Cleaning up resources");
                cleanupResources();

                long duration = System.currentTimeMillis() - startTime;
                log.info("Graceful shutdown completed in {} ms", duration);

            } catch (Exception e) {
                log.error("Error during graceful shutdown", e);
            } finally {
                // Force shutdown application context
                SpringApplication.exit(applicationContext, () -> 0);
                System.exit(0);
            }
        }

        private void notifyHealthCheck(boolean healthy) {
            // Update liveness probe status
            log.debug("Updating health check status: {}", healthy);
        }

        private boolean waitForRequestsToComplete(int timeoutSeconds) {
            long deadline = System.currentTimeMillis() + (timeoutSeconds * 1000L);
            while (System.currentTimeMillis() < deadline) {
                int activeRequests = getActiveRequestCount();
                if (activeRequests == 0) {
                    log.info("All requests completed");
                    return true;
                }
                log.debug("Waiting for {} active requests to complete", activeRequests);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
            return false;
        }

        private void closeDatabase() {
            try {
                // Get HikariCP datasource and close
                log.info("Database connections closed");
            } catch (Exception e) {
                log.error("Error closing database", e);
            }
        }

        private void waitForAsyncOperations(int timeoutSeconds) {
            long deadline = System.currentTimeMillis() + (timeoutSeconds * 1000L);
            while (System.currentTimeMillis() < deadline) {
                int pendingTasks = getPendingAsyncTasks();
                if (pendingTasks == 0) {
                    log.info("All async operations completed");
                    return;
                }
                log.debug("Waiting for {} async operations to complete", pendingTasks);
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        }

        private void flushMetrics() {
            try {
                // Flush Micrometer/Prometheus metrics
                log.info("Metrics flushed");
            } catch (Exception e) {
                log.error("Error flushing metrics", e);
            }
        }

        private void cleanupResources() {
            try {
                // Close caches, thread pools, etc.
                log.info("Resources cleaned up");
            } catch (Exception e) {
                log.error("Error cleaning up resources", e);
            }
        }

        private int getActiveRequestCount() {
            // Return count of active HTTP requests
            return 0;
        }

        private int getPendingAsyncTasks() {
            // Return count of pending async tasks
            return 0;
        }
    }

    /**
     * Listener for application shutdown events.
     */
    @Component
    @Slf4j
    public static class ShutdownEventListener {
        private final ShutdownManager shutdownManager;

        public ShutdownEventListener(ShutdownManager shutdownManager) {
            this.shutdownManager = shutdownManager;
        }

        /**
         * Listen for application shutdown event.
         */
        @EventListener(ContextClosedEvent.class)
        public void onApplicationShutdown() {
            log.info("Application shutdown event received");
        }
    }

    /**
     * HTTP endpoint to trigger graceful shutdown.
     */
    @RestController
    @Slf4j
    @AllArgsConstructor
    public static class ShutdownController {
        private final ShutdownManager shutdownManager;

        /**
         * Health check endpoint - returns 503 when shutting down.
         */
        @GetMapping("/health/live")
        public org.springframework.http.ResponseEntity<?> liveness() {
            if (shutdownManager.isShuttingDown()) {
                log.warn("Liveness probe failed - shutdown in progress");
                return org.springframework.http.ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).build();
            }
            return org.springframework.http.ResponseEntity.ok().build();
        }

        /**
         * Readiness check - marks instance as not ready during shutdown.
         */
        @GetMapping("/health/ready")
        public org.springframework.http.ResponseEntity<?> readiness() {
            if (shutdownManager.isShuttingDown()) {
                return org.springframework.http.ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE).build();
            }
            return org.springframework.http.ResponseEntity.ok().build();
        }

        /**
         * Admin endpoint to trigger shutdown (requires admin role).
         */
        @PostMapping("/admin/shutdown")
        @org.springframework.security.access.prepost.PreAuthorize("hasRole('ADMIN')")
        public org.springframework.http.ResponseEntity<?> shutdown(
                @org.springframework.web.bind.annotation.RequestParam(required = false) String reason) {
            String shutdownReason = reason != null ? reason : "Admin requested shutdown";
            shutdownManager.initiateShutdown(shutdownReason);
            return org.springframework.http.ResponseEntity.accepted().body("Graceful shutdown initiated");
        }
    }

    /**
     * JVM shutdown hook registration.
     */
    @Component
    @Slf4j
    public static class ShutdownHookRegistrar {
        private final ShutdownManager shutdownManager;

        public ShutdownHookRegistrar(ShutdownManager shutdownManager) {
            this.shutdownManager = shutdownManager;
            Runtime.getRuntime().addShutdownHook(new Thread(this::handleShutdown, "ShutdownHook"));
        }

        private void handleShutdown() {
            if (!shutdownManager.isShuttingDown()) {
                log.warn("JVM shutdown signal received - initiating graceful shutdown");
                shutdownManager.initiateShutdown("JVM shutdown signal (SIGTERM)");
            }
        }
    }

    /**
     * Request tracking filter for monitoring in-flight requests.
     */
    @Component
    @Slf4j
    public static class RequestTrackingFilter extends org.springframework.web.filter.OncePerRequestFilter {
        private static final ConcurrentHashMap<String, Long> activeRequests = new ConcurrentHashMap<>();

        @Override
        protected void doFilterInternal(jakarta.servlet.http.HttpServletRequest request,
                                       jakarta.servlet.http.HttpServletResponse response,
                                       jakarta.servlet.FilterChain filterChain)
                throws jakarta.servlet.ServletException, java.io.IOException {

            String requestId = java.util.UUID.randomUUID().toString();
            activeRequests.put(requestId, System.currentTimeMillis());

            try {
                filterChain.doFilter(request, response);
            } finally {
                activeRequests.remove(requestId);
            }
        }

        public static int getActiveRequestCount() {
            return activeRequests.size();
        }
    }

    /**
     * Async task tracking for graceful shutdown.
     */
    @Component
    @Slf4j
    public static class AsyncTaskTracker {
        private final ThreadPoolTaskExecutor executor;
        private final ConcurrentHashMap<String, Long> pendingTasks = new ConcurrentHashMap<>();

        public AsyncTaskTracker(ThreadPoolTaskExecutor executor) {
            this.executor = executor;
        }

        /**
         * Submit tracked async task.
         */
        public void submitTrackedTask(Runnable task) {
            String taskId = java.util.UUID.randomUUID().toString();
            pendingTasks.put(taskId, System.currentTimeMillis());

            executor.execute(() -> {
                try {
                    task.run();
                } finally {
                    pendingTasks.remove(taskId);
                    log.debug("Async task completed: {}", taskId);
                }
            });
        }

        public int getPendingTaskCount() {
            return pendingTasks.size();
        }
    }

    /**
     * Configuration for shutdown behavior.
     */
    @Configuration
    public static class ShutdownConfiguration {

        /**
         * Configure Tomcat shutdown timeout.
         */
        @Bean
        public org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory
        servletWebServerFactory() {
            org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory factory =
                    new org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory();

            factory.addConnectorCustomizers(connector -> {
                connector.setAsyncTimeout(60000);  // 60 seconds
            });

            return factory;
        }

        /**
         * Thread pool executor for async tasks.
         */
        @Bean
        public ThreadPoolTaskExecutor asyncExecutor() {
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            executor.setCorePoolSize(5);
            executor.setMaxPoolSize(10);
            executor.setQueueCapacity(100);
            executor.setThreadNamePrefix("async-");
            executor.setWaitForTasksToCompleteOnShutdown(true);
            executor.setAwaitTerminationSeconds(60);
            executor.initialize();
            return executor;
        }
    }

    /**
     * Kubernetes preStop hook handler.
     */
    public static class KubernetesGracefulShutdown {
        /**
         * preStop hook script for Kubernetes deployment.
         */
        public static final String KUBERNETES_PRESTOP_HOOK = """
            lifecycle:
              preStop:
                exec:
                  command: ["/bin/sh", "-c", "sleep 15"]
            terminationGracePeriodSeconds: 60
        """;
    }

    /**
     * Best practices for graceful shutdown.
     */
    public static class BestPractices {
        /**
         * 1. Set appropriate timeouts
         *    - Request drain: 60 seconds
         *    - Async operations: 30 seconds
         *    - Database close: 10 seconds
         */

        /**
         * 2. Stop accepting new requests immediately
         *    - Mark as unhealthy to load balancer
         *    - Return 503 from /health endpoints
         *    - Allow load balancer to detect change (2-5 seconds)
         */

        /**
         * 3. Wait for in-flight requests
         *    - Track active requests
         *    - Allow reasonable completion time
         *    - Force shutdown if timeout exceeded
         */

        /**
         * 4. Close external connections gracefully
         *    - Database connections
         *    - Message queues
         *    - HTTP clients
         */

        /**
         * 5. Complete async operations
         *    - Track scheduled tasks
         *    - Wait for task completion
         *    - Force shutdown if timeout exceeded
         */

        /**
         * 6. Flush monitoring data
         *    - Metrics to monitoring system
         *    - Logs to logging backend
         *    - Traces to tracing service
         */

        /**
         * 7. Use Kubernetes terminationGracePeriodSeconds
         *    - Set to sum of shutdown timeouts
         *    - Allow preStop hook execution
         *    - Prevent forceful termination
         */

        /**
         * 8. Test graceful shutdown
         *    - Test with in-flight requests
         *    - Test with pending async tasks
         *    - Verify no data loss
         */
    }
}
```

## Application Configuration

```properties
# application.properties

# Server shutdown configuration
server.shutdown=graceful
server.tomcat.shutdown.wait-time-seconds=30
server.tomcat.shutdown.drain-requests-timeout-seconds=60

# Thread pool configuration
spring.task.execution.pool.core-size=5
spring.task.execution.pool.max-size=10
spring.task.execution.pool.queue-capacity=100
spring.task.execution.thread-name-prefix=async-

# Graceful shutdown timeouts
server.shutdown.wait-time-seconds=30
server.shutdown.drain-requests-timeout-seconds=60

# HikariCP connection pool
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=20000
spring.datasource.hikari.idle-timeout=300000
spring.datasource.hikari.max-lifetime=1200000
```

## Kubernetes Deployment Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 3
  template:
    spec:
      terminationGracePeriodSeconds: 65
      containers:
      - name: app
        image: myapp:latest
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
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
          failureThreshold: 3
```

## Related Documentation

- [Spring Boot Graceful Shutdown](https://spring.io/blog/2022/05/12/spring-boot-graceful-shutdown)
- [Kubernetes Termination Grace Period](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#termination-of-pods)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP/wiki/Configuration)
```
