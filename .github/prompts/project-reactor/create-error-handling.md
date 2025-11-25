# Create Error Handling

Create comprehensive error handling strategies for reactive streams using Project Reactor. Error handling is critical for building resilient reactive applications.

## Requirements

Your error handling implementation should include:

- **Error Recovery**: Implement `onErrorResume()`, `onErrorReturn()`, and `onErrorMap()`
- **Retry Logic**: Use `retry()` and `retryWhen()` with backoff strategies
- **Error Transformation**: Convert errors to appropriate types with context
- **Fallback Handling**: Provide fallback values or streams
- **Catch and Handle**: Catch specific exception types with different strategies
- **Error Logging**: Comprehensive logging at each error point
- **Circuit Breaker**: Implement circuit breaker pattern for external calls
- **Timeout Handling**: Handle timeout exceptions specifically
- **Error Context**: Propagate error context through the reactive chain
- **Testing**: Examples of testing error scenarios

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Project Reactor, Spring Boot
- **Annotations**: Use `@Component` or `@Service` as appropriate
- **Naming**: Method names should indicate the error handling strategy

## Template Example

```java
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import reactor.util.retry.Retry;
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Duration;
import java.time.Instant;

/**
 * Demonstrates comprehensive error handling strategies for reactive streams.
 * Includes retry, fallback, circuit breaker, and timeout handling.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class ReactiveErrorHandler {

    private final CircuitBreakerRegistry circuitBreakerRegistry;

    /**
     * Basic error recovery: Return alternative value on error.
     */
    public Mono<User> getUserWithDefault(String userId) {
        return userRepository.findById(userId)
                .doOnNext(user -> log.info("User found: {}", user.id()))
                .onErrorReturn(new User("unknown", "Unknown User"))
                .doOnError(error -> log.error("Error retrieving user", error));
    }

    /**
     * Resume with alternative source: Fall back to cached/default source.
     */
    public Mono<User> getUserWithFallback(String userId) {
        return userRepository.findById(userId)
                .doOnNext(user -> log.info("User found from database: {}", user.id()))
                .onErrorResume(exception -> {
                    log.warn("Database error, falling back to cache: {}", exception.getMessage());
                    return cacheService.getUser(userId)
                            .doOnNext(user -> log.info("User found from cache: {}", user.id()));
                })
                .onErrorResume(exception -> {
                    log.error("Cache failed, using default user");
                    return Mono.just(new User("default", "Default User"));
                });
    }

    /**
     * Error transformation: Convert low-level errors to domain-specific exceptions.
     */
    public Mono<Order> createOrderWithErrorTransform(CreateOrderRequest request) {
        return orderService.create(request)
                .onErrorMap(DatabaseException.class, e ->
                        new OrderCreationException("Failed to create order in database", e))
                .onErrorMap(ValidationException.class, e ->
                        new BadRequestException("Invalid order data: " + e.getMessage()))
                .onErrorMap(IllegalStateException.class, e ->
                        new ConflictException("Order creation conflict: " + e.getMessage()))
                .doOnError(error -> log.error("Order creation error: {}", error.getMessage()));
    }

    /**
     * Specific error handling: Different strategies for different exception types.
     */
    public Mono<PaymentResult> processPaymentWithSpecificHandling(PaymentRequest request) {
        return paymentService.process(request)
                .doOnNext(result -> log.info("Payment processed: {}", result.transactionId()))
                .onErrorResume(TimeoutException.class, e -> {
                    log.warn("Payment timeout, returning pending status");
                    return Mono.just(new PaymentResult("PENDING", "timeout"));
                })
                .onErrorResume(NetworkException.class, e -> {
                    log.warn("Network error, retrying...");
                    return retryWithBackoff(3, Duration.ofMillis(100));
                })
                .onErrorResume(InvalidPaymentException.class, e -> {
                    log.error("Invalid payment details: {}", e.getMessage());
                    return Mono.error(new BadRequestException(e.getMessage()));
                })
                .onErrorResume(e -> {
                    log.error("Unexpected payment error", e);
                    return Mono.error(new InternalServerException("Payment processing failed"));
                });
    }

    /**
     * Retry with exponential backoff: Automatically retry on transient failures.
     */
    public Mono<Data> fetchWithRetry(String dataId) {
        return dataService.fetch(dataId)
                .doOnNext(data -> log.info("Data fetched successfully: {}", data.id()))
                .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
                        .filter(throwable -> isTransientError(throwable))
                        .doBeforeRetry(signal -> log.warn("Retrying after {} attempt: {}",
                                signal.totalRetries(),
                                signal.failure().getMessage()))
                        .onRetryExhaustedThrow((retrySignal, throwable) -> {
                            log.error("Failed after {} retries", retrySignal.totalRetries());
                            return new DataFetchException("Max retries exceeded", throwable);
                        }))
                .doOnError(error -> log.error("Data fetch failed", error));
    }

    /**
     * Retry with jitter: Add randomness to prevent thundering herd.
     */
    public Mono<Data> fetchWithJitterRetry(String dataId) {
        return dataService.fetch(dataId)
                .retryWhen(Retry.backoff(5, Duration.ofMillis(100))
                        .jitter(0.5)  // 50% jitter
                        .filter(throwable -> isTransientError(throwable)))
                .doOnError(error -> log.error("Data fetch failed after retries", error));
    }

    /**
     * Retry with custom condition: Only retry on specific errors.
     */
    public Mono<Data> fetchWithConditionalRetry(String dataId) {
        return dataService.fetch(dataId)
                .retryWhen(Retry.max(3)
                        .filter(throwable -> {
                            if (throwable instanceof TimeoutException) {
                                log.warn("Timeout error, will retry");
                                return true;
                            } else if (throwable instanceof ConnectionException) {
                                log.warn("Connection error, will retry");
                                return true;
                            } else {
                                log.error("Non-retryable error: {}", throwable.getMessage());
                                return false;
                            }
                        }))
                .doOnError(error -> log.error("Failed to fetch after conditional retries", error));
    }

    /**
     * Timeout handling: Handle requests that exceed time limits.
     */
    public Mono<Data> fetchWithTimeout(String dataId, long timeoutSeconds) {
        return dataService.fetch(dataId)
                .timeout(Duration.ofSeconds(timeoutSeconds),
                        Mono.error(new TimeoutException("Request exceeded " + timeoutSeconds + " seconds")))
                .onErrorResume(java.util.concurrent.TimeoutException.class, e -> {
                    log.warn("Operation timeout for {}", dataId);
                    return cacheService.getOrDefault(dataId);
                })
                .doOnError(error -> log.error("Fetch error with timeout", error));
    }

    /**
     * Circuit breaker pattern: Prevent cascading failures by stopping requests.
     */
    public Mono<Data> fetchWithCircuitBreaker(String dataId) {
        CircuitBreaker circuitBreaker = circuitBreakerRegistry.circuitBreaker("dataServiceCB");

        return dataService.fetch(dataId)
                .transformDeferred(circuitBreaker.toReactorOperator())
                .doOnNext(data -> log.info("Fetched via circuit breaker: {}", data.id()))
                .onErrorResume(e -> {
                    if (circuitBreaker.getState().toString().equals("OPEN")) {
                        log.error("Circuit breaker is OPEN, using fallback");
                        return getFallbackData(dataId);
                    }
                    return Mono.error(e);
                });
    }

    /**
     * Bulkhead pattern: Limit concurrency to prevent resource exhaustion.
     */
    public Flux<Data> fetchMultipleWithBulkhead(Flux<String> dataIds) {
        return dataIds
                .flatMap(id -> dataService.fetch(id),
                        3,  // Limit concurrency to 3
                        2)  // Prefetch queue size
                .doOnError(error -> log.error("Bulkhead error", error));
    }

    /**
     * Recover with retry and fallback: Multi-layer error handling.
     */
    public Mono<Data> robustFetch(String dataId) {
        return dataService.fetch(dataId)
                .doOnNext(data -> log.info("Data fetched from primary source"))
                // Retry on transient errors
                .retryWhen(Retry.backoff(2, Duration.ofMillis(50))
                        .filter(throwable -> isTransientError(throwable))
                        .doBeforeRetry(signal -> log.warn("Retrying primary fetch")))
                // Fallback to cache
                .onErrorResume(exception -> {
                    log.warn("Primary source failed, trying cache");
                    return cacheService.get(dataId)
                            .doOnNext(data -> log.info("Data fetched from cache"));
                })
                // Fallback to default
                .onErrorResume(exception -> {
                    log.error("All sources failed, using default");
                    return Mono.just(new Data(dataId, "default"));
                });
    }

    /**
     * Error context: Pass error details through reactive chain.
     */
    public Mono<Data> fetchWithErrorContext(String dataId) {
        return dataService.fetch(dataId)
                .doOnError(error -> {
                    var context = new ErrorContext(
                            Instant.now(),
                            error.getClass().getSimpleName(),
                            error.getMessage(),
                            dataId
                    );
                    log.error("Error occurred: {}", context);
                })
                .onErrorMap(error -> new DataFetchException(
                        "Failed to fetch: " + error.getMessage(),
                        error,
                        dataId));
    }

    /**
     * Graceful degradation: Continue processing even with some failures.
     */
    public Flux<Data> fetchMultipleWithGracefulDegradation(Flux<String> dataIds) {
        return dataIds
                .flatMap(id -> dataService.fetch(id)
                        .onErrorResume(e -> {
                            log.warn("Failed to fetch {}, using partial result", id);
                            return Mono.just(new Data(id, "PARTIAL"));
                        }))
                .doOnError(error -> log.error("Unexpected error in batch", error));
    }

    /**
     * Error aggregation: Collect errors and report them together.
     */
    public Mono<BatchResult> processBatchWithErrorAggregation(Flux<Item> items) {
        var successCount = new java.util.concurrent.atomic.AtomicInteger(0);
        var errors = new java.util.concurrent.CopyOnWriteArrayList<ProcessingError>();

        return items
                .flatMap(item ->
                        processItem(item)
                                .doOnSuccess(result -> successCount.incrementAndGet())
                                .onErrorResume(error -> {
                                    errors.add(new ProcessingError(item.id(), error.getMessage()));
                                    return Mono.empty();
                                }))
                .collectList()
                .map(results -> new BatchResult(
                        successCount.get(),
                        errors.size(),
                        errors))
                .doOnError(error -> log.error("Batch processing error", error));
    }

    /**
     * Exponential backoff with max delay.
     */
    private Mono<Data> retryWithBackoff(int maxRetries, Duration initialDelay) {
        return Mono.error(new Exception("Initial attempt"))
                .retryWhen(Retry.backoff(maxRetries, initialDelay)
                        .maxBackoff(Duration.ofSeconds(10))
                        .doBeforeRetry(signal -> log.warn("Backing off before retry")));
    }

    /**
     * Determine if error is transient (retryable).
     */
    private boolean isTransientError(Throwable throwable) {
        return throwable instanceof TimeoutException ||
               throwable instanceof ConnectionException ||
               (throwable instanceof java.io.IOException);
    }

    /**
     * Get fallback data when primary source fails.
     */
    private Mono<Data> getFallbackData(String dataId) {
        return cacheService.get(dataId)
                .switchIfEmpty(Mono.just(new Data(dataId, "FALLBACK")));
    }

    /**
     * Process a single item.
     */
    private Mono<Data> processItem(Item item) {
        return Mono.fromCallable(() -> new Data(item.id(), "processed"));
    }

    // Helper classes and records
    private record User(String id, String name) {}
    private record Order(String id) {}
    private record CreateOrderRequest(String items) {}
    private record PaymentRequest(Double amount) {}
    private record PaymentResult(String status, String reason) {}
    private record Data(String id, String value) {}
    private record Item(String id) {}
    private record ErrorContext(Instant timestamp, String exceptionType, String message, String context) {}
    private record BatchResult(int successCount, int errorCount, java.util.List<ProcessingError> errors) {}
    private record ProcessingError(String itemId, String errorMessage) {}

    // Exception classes
    public static class OrderCreationException extends RuntimeException {
        public OrderCreationException(String message, Throwable cause) {
            super(message, cause);
        }
    }

    public static class BadRequestException extends RuntimeException {
        public BadRequestException(String message) {
            super(message);
        }
    }

    public static class ConflictException extends RuntimeException {
        public ConflictException(String message) {
            super(message);
        }
    }

    public static class DataFetchException extends RuntimeException {
        private final String dataId;

        public DataFetchException(String message, Throwable cause, String dataId) {
            super(message, cause);
            this.dataId = dataId;
        }

        public String getDataId() {
            return dataId;
        }
    }

    public static class DatabaseException extends RuntimeException {}
    public static class ValidationException extends RuntimeException {}
    public static class InvalidPaymentException extends RuntimeException {}
    public static class InternalServerException extends RuntimeException {
        public InternalServerException(String message) {
            super(message);
        }
    }
    public static class NetworkException extends RuntimeException {}
    public static class ConnectionException extends RuntimeException {}

    // Placeholder services
    interface UserRepository {
        Mono<User> findById(String id);
    }
    interface OrderService {
        Mono<Order> create(CreateOrderRequest request);
    }
    interface PaymentService {
        Mono<PaymentResult> process(PaymentRequest request);
    }
    interface DataService {
        Mono<Data> fetch(String id);
    }
    interface CacheService {
        Mono<User> getUser(String id);
        Mono<Data> get(String id);
        Mono<Data> getOrDefault(String id);
    }
}
```

## Error Handling Strategy Comparison

| Strategy | Use Case | Characteristics |
|----------|----------|-----------------|
| onErrorReturn | Non-critical data | Fast, provides default value |
| onErrorResume | Has alternative source | Tries fallback source |
| retry | Transient failures | Automatic retry with backoff |
| timeout | Long operations | Prevents indefinite waiting |
| Circuit Breaker | External calls | Prevents cascading failures |
| Custom mapping | Domain errors | Provides context-specific errors |

## Testing

```java
@Test
void testErrorRecovery() {
    StepVerifier.create(handler.getUserWithFallback("123"))
        .expectNextMatches(user -> user.name().equals("Unknown"))
        .expectComplete()
        .verify();
}
```

## Related Documentation

- [Project Reactor Error Handling](https://projectreactor.io/docs/core/release/reference/#error-handling)
- [Resilience4j Guide](https://resilience4j.readme.io/)
- [Spring Cloud Circuit Breaker](https://spring.io/projects/spring-cloud-circuitbreaker)
