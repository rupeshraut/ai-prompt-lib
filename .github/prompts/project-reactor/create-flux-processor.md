# Create Flux Processor

Create a new Flux processor for multi-value stream processing using Project Reactor. Flux represents a reactive type that emits 0 to N elements.

## Requirements

Your Flux processor should include:

- **Class Declaration**: Public class with descriptive name (e.g., `OrderFluxProcessor`, `EventStreamProcessor`)
- **Flux Operations**: Use appropriate methods like `map()`, `flatMap()`, `filter()`, `distinct()`, `buffer()`, `window()`
- **Stream Composition**: Combine multiple Flux sources using `merge()`, `concat()`, `zip()`, `combineLatest()`
- **Backpressure Handling**: Implement buffer strategies with `onBackpressureBuffer()`, `onBackpressureDrop()`
- **Transformations**: Include various transformations (map, flatMap, groupBy, scan, reduce)
- **Filtering**: Add conditional filtering with `filter()` and predicate logic
- **Error Handling**: Include error recovery strategies at the stream level
- **Subscribers**: Example subscribers showing how to consume the stream
- **Java 17 Features**: Use records, sealed classes, and pattern matching where applicable
- **Documentation**: Include JavaDoc comments explaining the stream processing pipeline

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Project Reactor, Spring (if needed)
- **Build Tool**: Maven with spring-boot-starter-webflux dependency
- **Naming Conventions**:
  - Method names describe the Flux operation (e.g., `processOrders()`, `streamEvents()`)
  - Use reactive naming (suffix with `Flux` for return types)
- **Annotations**: Use `@Component` or `@Service` for Spring management
- **Performance**: Consider performance implications of buffering and windowing strategies

## Template Example

```java
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;
import reactor.util.function.Tuple2;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Duration;
import java.util.List;

/**
 * Processes streams of events or data using Flux.
 * Example: Order processing, event aggregation, data transformation pipelines.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class OrderFluxProcessor {

    /**
     * Processes a stream of orders with filtering and transformation.
     */
    public Flux<OrderResponse> processOrders(Flux<Order> orderStream) {
        return orderStream
                .doOnNext(order -> log.debug("Received order: {}", order.id()))
                .filter(order -> order.amount() > 0)
                .filter(order -> order.customerId() != null)
                .map(this::enrichOrder)
                .flatMap(this::validateAndEnrichOrder)
                .doOnError(error -> log.error("Error processing order", error))
                .onErrorResume(Exception.class, e -> {
                    log.warn("Skipping invalid order due to: {}", e.getMessage());
                    return Flux.empty();
                })
                .doFinally(signalType -> log.info("Order processing stream completed: {}", signalType));
    }

    /**
     * Groups orders by customer and aggregates statistics.
     */
    public Flux<CustomerOrderStats> aggregateOrdersByCustomer(Flux<Order> orderStream) {
        return orderStream
                .groupBy(Order::customerId)
                .flatMap(groupedFlux -> {
                    Long customerId = groupedFlux.key();
                    return groupedFlux
                            .reduce(
                                    new OrderAggregation(customerId, 0, 0.0, List.of()),
                                    (agg, order) -> agg.addOrder(order)
                            )
                            .map(agg -> new CustomerOrderStats(
                                    customerId,
                                    agg.orderCount,
                                    agg.totalAmount,
                                    agg.orders
                            ));
                })
                .doOnNext(stats -> log.info("Customer {} stats: count={}, total={}",
                        stats.customerId(), stats.orderCount(), stats.totalAmount()))
                .doOnError(error -> log.error("Error aggregating orders", error));
    }

    /**
     * Buffers orders and processes them in batches.
     * Example: Send batches to external service every 100 orders or 5 seconds.
     */
    public Flux<BatchProcessResult> processBatchedOrders(Flux<Order> orderStream) {
        return orderStream
                .buffer(100, Duration.ofSeconds(5))
                .filter(batch -> !batch.isEmpty())
                .doOnNext(batch -> log.info("Processing batch of {} orders", batch.size()))
                .flatMap(batch -> processBatch(batch)
                        .onErrorResume(e -> {
                            log.error("Batch processing failed, retrying...", e);
                            return processBatch(batch)
                                    .retryWhen(reactor.util.retry.Retry.backoff(3, Duration.ofSeconds(1)));
                        })
                )
                .doOnComplete(() -> log.info("All batches processed"));
    }

    /**
     * Windows orders into time-based chunks.
     * Example: Analyze order patterns every 10 seconds.
     */
    public Flux<OrderWindowStats> analyzeOrderWindows(Flux<Order> orderStream) {
        return orderStream
                .window(Duration.ofSeconds(10))
                .flatMap(window -> {
                    return window
                            .reduce(
                                    new WindowAggregation(0, 0.0),
                                    (agg, order) -> new WindowAggregation(
                                            agg.count + 1,
                                            agg.totalAmount + order.amount()
                                    )
                            )
                            .map(agg -> new OrderWindowStats(
                                    System.currentTimeMillis(),
                                    agg.count,
                                    agg.totalAmount
                            ));
                })
                .doOnNext(stats -> log.info("Window stats: count={}, total={}",
                        stats.orderCount(), stats.totalAmount()))
                .doOnError(error -> log.error("Error in windowed analysis", error));
    }

    /**
     * Merges multiple order streams from different sources.
     */
    public Flux<OrderResponse> mergeOrderStreams(
            Flux<Order> onlineOrders,
            Flux<Order> mobileOrders,
            Flux<Order> inStoreOrders) {

        return Flux.merge(onlineOrders, mobileOrders, inStoreOrders)
                .doOnNext(order -> log.debug("Received order from source"))
                .flatMap(this::validateAndEnrichOrder)
                .parallel(4)
                .runOn(Schedulers.parallel())
                .sequential()
                .doOnError(error -> log.error("Error in merged stream", error));
    }

    /**
     * Combines multiple streams to correlate data.
     * Example: Combine orders with inventory checks.
     */
    public Flux<OrderWithInventory> correlateWithInventory(
            Flux<Order> orders,
            Flux<InventoryCheck> inventoryChecks) {

        return Flux.zip(orders, inventoryChecks)
                .map(tuple -> new OrderWithInventory(
                        tuple.getT1(),
                        tuple.getT2()
                ))
                .doOnNext(combined -> log.debug("Combined order with inventory"))
                .doOnError(error -> log.error("Error correlating streams", error));
    }

    /**
     * Implements exponential backoff retry strategy.
     */
    public Flux<OrderResponse> processWithRetry(Flux<Order> orders) {
        return orders
                .flatMap(order ->
                        Mono.fromCallable(() -> validateOrder(order))
                                .retryWhen(reactor.util.retry.Retry.backoff(3, Duration.ofMillis(100))
                                        .filter(throwable -> throwable instanceof TransientException)
                                        .onRetryExhaustedThrow((retrySignal, throwable) ->
                                                new ProcessingException("Failed after retries", throwable)
                                        )
                                )
                                .doOnNext(validated -> log.info("Order validation succeeded: {}", order.id()))
                                .map(v -> new OrderResponse(order.id(), "SUCCESS"))
                                .onErrorResume(ProcessingException.class, e -> {
                                    log.error("Processing failed: {}", e.getMessage());
                                    return Mono.just(new OrderResponse(order.id(), "FAILED"));
                                })
                                .subscribeOn(Schedulers.boundedElastic())
                );
    }

    /**
     * Distinct orders by customer ID, avoiding duplicates.
     */
    public Flux<Order> deduplicateOrders(Flux<Order> orders) {
        return orders
                .distinct(Order::customerId)
                .doOnNext(order -> log.debug("Distinct order: {}", order.id()));
    }

    /**
     * Scans orders for running statistics.
     */
    public Flux<OrderRunningStats> computeRunningStats(Flux<Order> orders) {
        return orders
                .scan(
                        new OrderRunningStats(0, 0.0, 0.0),
                        (stats, order) -> new OrderRunningStats(
                                stats.totalOrders + 1,
                                stats.totalAmount + order.amount(),
                                (stats.totalAmount + order.amount()) / (stats.totalOrders + 1)
                        )
                )
                .skip(1)  // Skip initial value
                .doOnNext(stats -> log.debug("Running stats: orders={}, total={}, avg={}",
                        stats.totalOrders, stats.totalAmount, stats.averageAmount()));
    }

    // Helper methods
    private Order enrichOrder(Order order) {
        log.debug("Enriching order: {}", order.id());
        return order;  // Simplified; would add more data
    }

    private Mono<OrderResponse> validateAndEnrichOrder(Order order) {
        return Mono.just(new OrderResponse(order.id(), "PROCESSED"));
    }

    private Mono<BatchProcessResult> processBatch(List<Order> batch) {
        return Mono.fromCallable(() -> {
            log.info("Processing batch of {} orders", batch.size());
            return new BatchProcessResult(batch.size(), "SUCCESS");
        });
    }

    private Order validateOrder(Order order) {
        if (order.amount() <= 0) {
            throw new TransientException("Invalid amount");
        }
        return order;
    }

    // DTOs
    public record Order(Long id, Long customerId, Double amount) {}
    public record OrderResponse(Long id, String status) {}
    public record CustomerOrderStats(Long customerId, int orderCount, Double totalAmount, List<Order> orders) {}
    public record BatchProcessResult(int size, String status) {}
    public record OrderWindowStats(Long timestamp, int orderCount, Double totalAmount) {}
    public record OrderWithInventory(Order order, InventoryCheck inventory) {}
    public record OrderRunningStats(int totalOrders, Double totalAmount, Double averageAmount) {}
    public record InventoryCheck(Long orderId, Boolean available) {}

    // Helper records
    private record OrderAggregation(Long customerId, int orderCount, Double totalAmount, List<Order> orders) {
        OrderAggregation addOrder(Order order) {
            List<Order> newOrders = new java.util.ArrayList<>(orders);
            newOrders.add(order);
            return new OrderAggregation(customerId, orderCount + 1, totalAmount + order.amount(), newOrders);
        }
    }

    private record WindowAggregation(int count, Double totalAmount) {}

    // Custom exceptions
    public static class TransientException extends RuntimeException {
        public TransientException(String message) {
            super(message);
        }
    }

    public static class ProcessingException extends RuntimeException {
        public ProcessingException(String message, Throwable cause) {
            super(message, cause);
        }
    }
}
```

## Key Patterns

### 1. Filtering and Transformation
```java
flux
    .filter(item -> item.isValid())
    .map(item -> transform(item))
    .flatMap(item -> asyncOperation(item));
```

### 2. Buffering and Batching
```java
flux.buffer(100, Duration.ofSeconds(5))  // Buffer 100 items or timeout
    .flatMap(batch -> processBatch(batch));
```

### 3. Grouping and Aggregation
```java
flux.groupBy(item -> item.category())
    .flatMap(grouped -> grouped.reduce(accumulator));
```

### 4. Combining Streams
```java
Flux.merge(stream1, stream2, stream3)  // Interleave items
Flux.concat(stream1, stream2)          // Sequential concatenation
Flux.zip(stream1, stream2)             // Pair items from streams
```

### 5. Error Handling at Stream Level
```java
flux
    .onErrorResume(Exception.class, e -> Flux.empty())
    .doOnError(error -> log.error("Error", error))
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
```

## Testing Patterns

```java
// Test Flux emission
StepVerifier.create(processOrders(orderFlux))
    .expectNextMatches(response -> response.id() > 0)
    .expectComplete()
    .verify(Duration.ofSeconds(10));

// Test with virtual time
StepVerifier.withVirtualTime(() -> processWithDelay())
    .thenAwait(Duration.ofSeconds(5))
    .expectNextCount(5)
    .verifyComplete();
```

## Related Documentation

- [Project Reactor Flux Documentation](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html)
- [Flux Operators Guide](https://projectreactor.io/docs/core/release/reference/)
- [Spring WebFlux Data Access](https://spring.io/projects/spring-data-r2dbc)
