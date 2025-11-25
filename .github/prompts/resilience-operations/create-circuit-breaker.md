# Create Circuit Breaker

Implement Resilience4j circuit breaker patterns for Spring Boot applications. Protect microservices from cascading failures by monitoring service health and preventing calls to failing dependencies.

## Requirements

Your circuit breaker implementation should include:

- **Resilience4j Integration**: Configure circuit breaker with Spring Boot
- **State Management**: Handle CLOSED, OPEN, and HALF_OPEN states
- **Failure Threshold**: Configure failure rate and slow call thresholds
- **Fallback Mechanisms**: Provide alternative responses when circuit opens
- **Metrics and Monitoring**: Expose circuit breaker metrics via Actuator
- **Reactive Support**: Circuit breakers for Project Reactor applications
- **Custom Configuration**: Per-service circuit breaker settings
- **Event Listeners**: Track state transitions and failures
- **Time-Based Recovery**: Automatic state transition after wait duration
- **Testing Strategies**: Verify circuit breaker behavior under failure conditions

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x, Resilience4j
- **Reactive**: Project Reactor integration
- **Configuration**: YAML-based declarative configuration

## Template Example

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.event.CircuitBreakerEvent;
import io.github.resilience4j.circuitbreaker.event.CircuitBreakerOnErrorEvent;
import io.github.resilience4j.circuitbreaker.event.CircuitBreakerOnStateTransitionEvent;
import io.github.resilience4j.reactor.circuitbreaker.operator.CircuitBreakerOperator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;
import org.springframework.beans.factory.annotation.Qualifier;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import lombok.extern.slf4j.Slf4j;
import lombok.RequiredArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;
import java.time.Duration;
import java.io.IOException;

/**
 * Circuit breaker implementation using Resilience4j.
 * Protects against cascading failures by failing fast when dependencies are unavailable.
 */
@Slf4j
public class CircuitBreakerPatterns {

    /**
     * Circuit breaker configuration for multiple services.
     */
    @Configuration
    @Slf4j
    public static class CircuitBreakerConfiguration {

        /**
         * Default circuit breaker configuration.
         * 50% failure rate threshold, minimum 10 calls.
         */
        @Bean
        public CircuitBreakerConfig defaultCircuitBreakerConfig() {
            return CircuitBreakerConfig.custom()
                    // Failure rate threshold - open circuit if 50% of calls fail
                    .failureRateThreshold(50)
                    // Slow call threshold - consider calls slower than 2s as failures
                    .slowCallRateThreshold(50)
                    .slowCallDurationThreshold(Duration.ofSeconds(2))
                    // Minimum number of calls before calculating failure rate
                    .minimumNumberOfCalls(10)
                    // Number of calls in half-open state to determine recovery
                    .permittedNumberOfCallsInHalfOpenState(5)
                    // Wait duration before transitioning from OPEN to HALF_OPEN
                    .waitDurationInOpenState(Duration.ofSeconds(30))
                    // Sliding window size for calculating failure rate
                    .slidingWindowSize(100)
                    .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
                    // Automatic transition from OPEN to HALF_OPEN
                    .automaticTransitionFromOpenToHalfOpenEnabled(true)
                    // Record specific exceptions as failures
                    .recordExceptions(IOException.class, RuntimeException.class)
                    .ignoreExceptions(IllegalArgumentException.class)
                    .build();
        }

        /**
         * Circuit breaker registry with custom configurations.
         */
        @Bean
        public CircuitBreakerRegistry circuitBreakerRegistry(
                CircuitBreakerConfig defaultCircuitBreakerConfig) {

            CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(defaultCircuitBreakerConfig);

            // Payment service - more aggressive circuit breaker
            CircuitBreakerConfig paymentConfig = CircuitBreakerConfig.custom()
                    .failureRateThreshold(30)  // Open at 30% failure rate
                    .minimumNumberOfCalls(5)
                    .waitDurationInOpenState(Duration.ofSeconds(60))
                    .slidingWindowSize(50)
                    .build();
            registry.circuitBreaker("payment-service", paymentConfig);

            // Notification service - more lenient (non-critical)
            CircuitBreakerConfig notificationConfig = CircuitBreakerConfig.custom()
                    .failureRateThreshold(70)  // Open at 70% failure rate
                    .minimumNumberOfCalls(20)
                    .waitDurationInOpenState(Duration.ofSeconds(15))
                    .slidingWindowSize(100)
                    .build();
            registry.circuitBreaker("notification-service", notificationConfig);

            // Register event listeners
            registry.circuitBreaker("payment-service")
                    .getEventPublisher()
                    .onStateTransition(event ->
                        log.warn("Payment service circuit breaker state transition: {} -> {}",
                                event.getStateTransition().getFromState(),
                                event.getStateTransition().getToState()))
                    .onError(event ->
                        log.error("Payment service circuit breaker error: {}",
                                event.getThrowable().getMessage()));

            return registry;
        }

        /**
         * RestTemplate for external service calls.
         */
        @Bean
        public RestTemplate restTemplate() {
            return new RestTemplate();
        }
    }

    /**
     * Payment service client with circuit breaker protection.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class PaymentServiceClient {
        private final RestTemplate restTemplate;
        private final CircuitBreakerRegistry circuitBreakerRegistry;

        private static final String PAYMENT_SERVICE_URL = "http://payment-service:8080";

        /**
         * Process payment with circuit breaker protection.
         * Falls back to queuing payment when service unavailable.
         */
        @io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker(
                name = "payment-service",
                fallbackMethod = "processPaymentFallback")
        public PaymentResponse processPayment(PaymentRequest request) {
            log.info("Processing payment for order: {}", request.getOrderId());

            String url = PAYMENT_SERVICE_URL + "/api/v1/payments";
            PaymentResponse response = restTemplate.postForObject(url, request, PaymentResponse.class);

            log.info("Payment processed successfully: {}", response.getTransactionId());
            return response;
        }

        /**
         * Fallback method when circuit is open or call fails.
         */
        private PaymentResponse processPaymentFallback(PaymentRequest request, Exception ex) {
            log.warn("Payment service unavailable for order: {}. Queueing payment. Error: {}",
                    request.getOrderId(), ex.getMessage());

            // Queue payment for later processing
            queuePaymentForLaterProcessing(request);

            return PaymentResponse.builder()
                    .transactionId("QUEUED-" + request.getOrderId())
                    .status("QUEUED")
                    .message("Payment queued for processing when service is available")
                    .build();
        }

        /**
         * Retrieve payment status with circuit breaker.
         */
        @io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker(
                name = "payment-service",
                fallbackMethod = "getPaymentStatusFallback")
        public PaymentStatus getPaymentStatus(String transactionId) {
            log.debug("Retrieving payment status for transaction: {}", transactionId);

            String url = PAYMENT_SERVICE_URL + "/api/v1/payments/" + transactionId;
            return restTemplate.getForObject(url, PaymentStatus.class);
        }

        /**
         * Fallback for payment status retrieval.
         */
        private PaymentStatus getPaymentStatusFallback(String transactionId, Exception ex) {
            log.warn("Unable to retrieve payment status for transaction: {}. Error: {}",
                    transactionId, ex.getMessage());

            return PaymentStatus.builder()
                    .transactionId(transactionId)
                    .status("UNKNOWN")
                    .message("Payment service temporarily unavailable")
                    .build();
        }

        /**
         * Programmatic circuit breaker usage (alternative to annotation).
         */
        public PaymentResponse processPaymentProgrammatic(PaymentRequest request) {
            CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("payment-service");

            return circuitBreaker.executeSupplier(() -> {
                log.info("Processing payment programmatically for order: {}", request.getOrderId());
                String url = PAYMENT_SERVICE_URL + "/api/v1/payments";
                return restTemplate.postForObject(url, request, PaymentResponse.class);
            });
        }

        private void queuePaymentForLaterProcessing(PaymentRequest request) {
            // Queue to message broker (Kafka, RabbitMQ) for async processing
            log.info("Payment queued for order: {}", request.getOrderId());
        }
    }

    /**
     * Reactive circuit breaker for Project Reactor applications.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class ReactivePaymentServiceClient {
        private final org.springframework.web.reactive.function.client.WebClient webClient;
        private final CircuitBreakerRegistry circuitBreakerRegistry;

        /**
         * Process payment reactively with circuit breaker.
         */
        public Mono<PaymentResponse> processPaymentReactive(PaymentRequest request) {
            CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("payment-service");

            return webClient.post()
                    .uri("http://payment-service:8080/api/v1/payments")
                    .bodyValue(request)
                    .retrieve()
                    .bodyToMono(PaymentResponse.class)
                    .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
                    .doOnError(error -> log.error("Payment processing failed: {}", error.getMessage()))
                    .onErrorResume(error -> {
                        log.warn("Circuit breaker fallback activated for order: {}", request.getOrderId());
                        return Mono.just(PaymentResponse.builder()
                                .transactionId("QUEUED-" + request.getOrderId())
                                .status("QUEUED")
                                .message("Payment queued due to service unavailability")
                                .build());
                    });
        }

        /**
         * Batch payment processing with circuit breaker.
         */
        public Flux<PaymentResponse> processPaymentsBatch(Flux<PaymentRequest> requests) {
            CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("payment-service");

            return requests
                    .flatMap(request -> webClient.post()
                            .uri("http://payment-service:8080/api/v1/payments")
                            .bodyValue(request)
                            .retrieve()
                            .bodyToMono(PaymentResponse.class)
                            .transformDeferred(CircuitBreakerOperator.of(circuitBreaker))
                            .onErrorResume(error -> {
                                log.warn("Payment failed for order: {}. Error: {}",
                                        request.getOrderId(), error.getMessage());
                                return Mono.just(PaymentResponse.builder()
                                        .transactionId("FAILED-" + request.getOrderId())
                                        .status("FAILED")
                                        .message(error.getMessage())
                                        .build());
                            }));
        }
    }

    /**
     * Circuit breaker state monitoring service.
     */
    @Service
    @Slf4j
    @RequiredArgsConstructor
    public static class CircuitBreakerMonitoringService {
        private final CircuitBreakerRegistry circuitBreakerRegistry;

        /**
         * Get current state of circuit breaker.
         */
        public CircuitBreakerStatus getCircuitBreakerStatus(String name) {
            CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(name);
            CircuitBreaker.Metrics metrics = circuitBreaker.getMetrics();

            return CircuitBreakerStatus.builder()
                    .name(name)
                    .state(circuitBreaker.getState().name())
                    .failureRate(metrics.getFailureRate())
                    .slowCallRate(metrics.getSlowCallRate())
                    .numberOfSuccessfulCalls(metrics.getNumberOfSuccessfulCalls())
                    .numberOfFailedCalls(metrics.getNumberOfFailedCalls())
                    .numberOfSlowCalls(metrics.getNumberOfSlowCalls())
                    .numberOfNotPermittedCalls(metrics.getNumberOfNotPermittedCalls())
                    .build();
        }

        /**
         * Force transition to specific state (for testing/maintenance).
         */
        public void transitionToState(String name, String targetState) {
            CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(name);

            switch (targetState.toUpperCase()) {
                case "OPEN":
                    circuitBreaker.transitionToOpenState();
                    log.warn("Circuit breaker '{}' manually transitioned to OPEN", name);
                    break;
                case "CLOSED":
                    circuitBreaker.transitionToClosedState();
                    log.info("Circuit breaker '{}' manually transitioned to CLOSED", name);
                    break;
                case "HALF_OPEN":
                    circuitBreaker.transitionToHalfOpenState();
                    log.info("Circuit breaker '{}' manually transitioned to HALF_OPEN", name);
                    break;
                case "FORCED_OPEN":
                    circuitBreaker.transitionToForcedOpenState();
                    log.warn("Circuit breaker '{}' manually transitioned to FORCED_OPEN", name);
                    break;
                default:
                    throw new IllegalArgumentException("Invalid state: " + targetState);
            }
        }

        /**
         * Reset circuit breaker metrics.
         */
        public void resetCircuitBreaker(String name) {
            CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker(name);
            circuitBreaker.reset();
            log.info("Circuit breaker '{}' has been reset", name);
        }
    }

    /**
     * REST controller exposing circuit breaker operations.
     */
    @RestController
    @Slf4j
    @RequiredArgsConstructor
    public static class CircuitBreakerController {
        private final PaymentServiceClient paymentServiceClient;
        private final CircuitBreakerMonitoringService monitoringService;

        /**
         * Process payment through circuit breaker.
         */
        @org.springframework.web.bind.annotation.PostMapping("/api/v1/orders/{orderId}/payment")
        public org.springframework.http.ResponseEntity<PaymentResponse> processPayment(
                @PathVariable String orderId,
                @org.springframework.web.bind.annotation.RequestBody PaymentRequest request) {

            log.info("Received payment request for order: {}", orderId);
            request.setOrderId(orderId);

            PaymentResponse response = paymentServiceClient.processPayment(request);
            return org.springframework.http.ResponseEntity.ok(response);
        }

        /**
         * Get circuit breaker status.
         */
        @GetMapping("/api/v1/admin/circuit-breakers/{name}")
        public org.springframework.http.ResponseEntity<CircuitBreakerStatus> getCircuitBreakerStatus(
                @PathVariable String name) {

            CircuitBreakerStatus status = monitoringService.getCircuitBreakerStatus(name);
            return org.springframework.http.ResponseEntity.ok(status);
        }

        /**
         * Reset circuit breaker.
         */
        @org.springframework.web.bind.annotation.PostMapping("/api/v1/admin/circuit-breakers/{name}/reset")
        public org.springframework.http.ResponseEntity<String> resetCircuitBreaker(
                @PathVariable String name) {

            monitoringService.resetCircuitBreaker(name);
            return org.springframework.http.ResponseEntity.ok("Circuit breaker reset: " + name);
        }

        /**
         * Transition circuit breaker state.
         */
        @org.springframework.web.bind.annotation.PostMapping("/api/v1/admin/circuit-breakers/{name}/state/{targetState}")
        public org.springframework.http.ResponseEntity<String> transitionState(
                @PathVariable String name,
                @PathVariable String targetState) {

            monitoringService.transitionToState(name, targetState);
            return org.springframework.http.ResponseEntity.ok(
                    "Circuit breaker transitioned to " + targetState);
        }
    }

    /**
     * Domain models for payment operations.
     */
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class PaymentRequest {
        private String orderId;
        private String customerId;
        private Double amount;
        private String currency;
        private String paymentMethod;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class PaymentResponse {
        private String transactionId;
        private String status;
        private String message;
        private Double amount;
        private java.time.LocalDateTime processedAt;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class PaymentStatus {
        private String transactionId;
        private String status;
        private String message;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @lombok.Builder
    public static class CircuitBreakerStatus {
        private String name;
        private String state;
        private Float failureRate;
        private Float slowCallRate;
        private Integer numberOfSuccessfulCalls;
        private Integer numberOfFailedCalls;
        private Integer numberOfSlowCalls;
        private Long numberOfNotPermittedCalls;
    }
}
```

## Configuration Examples

```yaml
# application.yml

resilience4j:
  circuitbreaker:
    configs:
      default:
        # Circuit breaker opens when 50% of calls fail
        failure-rate-threshold: 50
        # Consider calls slower than 2 seconds as failures
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 2s
        # Minimum number of calls before calculating failure rate
        minimum-number-of-calls: 10
        # Number of calls in half-open state
        permitted-number-of-calls-in-half-open-state: 5
        # Wait 30 seconds before attempting recovery
        wait-duration-in-open-state: 30s
        # Sliding window size for tracking calls
        sliding-window-size: 100
        sliding-window-type: COUNT_BASED
        # Automatic transition from OPEN to HALF_OPEN
        automatic-transition-from-open-to-half-open-enabled: true
        # Record these exceptions as failures
        record-exceptions:
          - java.io.IOException
          - java.util.concurrent.TimeoutException
          - org.springframework.web.client.ResourceAccessException
        # Ignore these exceptions (don't count as failures)
        ignore-exceptions:
          - java.lang.IllegalArgumentException

    instances:
      # Payment service - critical, fail fast
      payment-service:
        base-config: default
        failure-rate-threshold: 30
        minimum-number-of-calls: 5
        wait-duration-in-open-state: 60s
        sliding-window-size: 50

      # Notification service - non-critical, more lenient
      notification-service:
        base-config: default
        failure-rate-threshold: 70
        minimum-number-of-calls: 20
        wait-duration-in-open-state: 15s
        sliding-window-size: 100

      # External API - aggressive circuit breaker
      external-api:
        failure-rate-threshold: 25
        minimum-number-of-calls: 10
        wait-duration-in-open-state: 120s
        slow-call-duration-threshold: 5s

# Actuator endpoints for monitoring
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,circuitbreakers
  endpoint:
    health:
      show-details: always
  health:
    circuitbreakers:
      enabled: true
  metrics:
    distribution:
      percentiles-histogram:
        resilience4j.circuitbreaker.calls: true
```

## Maven Dependencies

```xml
<dependencies>
    <!-- Resilience4j Spring Boot 3 Starter -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>2.1.0</version>
    </dependency>

    <!-- Resilience4j Circuit Breaker -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-circuitbreaker</artifactId>
        <version>2.1.0</version>
    </dependency>

    <!-- Resilience4j Reactor (for reactive support) -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-reactor</artifactId>
        <version>2.1.0</version>
    </dependency>

    <!-- Micrometer for metrics -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-micrometer</artifactId>
        <version>2.1.0</version>
    </dependency>

    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

## Best Practices

1. **Choose Appropriate Thresholds**
   - Critical services: 25-30% failure rate, fail fast
   - Non-critical services: 50-70% failure rate, more lenient
   - Adjust based on SLA requirements

2. **Configure Minimum Calls**
   - Set minimum 5-10 calls before calculating failure rate
   - Prevents premature circuit opening on initial failures
   - Balance between responsiveness and stability

3. **Set Realistic Timeouts**
   - Wait duration in open state: 30-60 seconds for most services
   - Longer waits for external APIs: 120+ seconds
   - Consider downstream recovery time

4. **Implement Fallback Strategies**
   - Return cached data when available
   - Queue requests for later processing
   - Provide degraded functionality
   - Never throw exceptions from fallback methods

5. **Monitor Circuit Breaker States**
   - Expose metrics via Actuator endpoints
   - Alert on state transitions to OPEN
   - Track failure rates and slow call rates
   - Monitor number of rejected calls

6. **Use Slow Call Detection**
   - Mark calls slower than threshold as failures
   - Prevent resource exhaustion from slow dependencies
   - Set slow call duration based on SLA

7. **Test Circuit Breaker Behavior**
   - Simulate downstream failures in tests
   - Verify fallback methods are invoked
   - Test state transitions (CLOSED -> OPEN -> HALF_OPEN)
   - Ensure proper recovery after failures

8. **Combine with Other Patterns**
   - Retry: Attempt before circuit breaker opens
   - Timeout: Prevent indefinite waits
   - Bulkhead: Isolate thread pools
   - Rate Limiter: Prevent overwhelming downstream services

## Related Documentation

- [Resilience4j Documentation](https://resilience4j.readme.io/docs/circuitbreaker)
- [Spring Cloud Circuit Breaker](https://spring.io/projects/spring-cloud-circuitbreaker)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Resilience4j Spring Boot Integration](https://resilience4j.readme.io/docs/getting-started-3)
