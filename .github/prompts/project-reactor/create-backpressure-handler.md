# Create Backpressure Handler

Create backpressure handling strategies for managing data flow in reactive streams. Backpressure prevents fast producers from overwhelming slow consumers.

## Requirements

Your backpressure handler should include:

- **Backpressure Strategies**: Implement buffer, drop, error, and ignore strategies
- **Buffer Capacity**: Configure buffer sizes and overflow handling
- **Request Management**: Demonstrate requesting specific numbers of items from publisher
- **Flow Control**: Implement pull-based demand signaling
- **Batch Processing**: Group items for efficient processing
- **Timeouts**: Add timing constraints to prevent indefinite waiting
- **Monitoring**: Track backpressure metrics and dropped items
- **Recovery**: Handle backpressure scenarios gracefully
- **Examples**: Real-world scenarios like data ingestion and rate limiting
- **Documentation**: Clear explanations of each strategy

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Project Reactor
- **Build Tool**: Maven
- **Annotations**: Use `@Component` or `@Service` as appropriate
- **Naming**: Descriptive method names reflecting the backpressure strategy

## Template Example

```java
import org.springframework.stereotype.Component;
import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Duration;

/**
 * Demonstrates various backpressure handling strategies in reactive streams.
 * Backpressure prevents producers from overwhelming consumers.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class BackpressureHandler {

    /**
     * Buffer strategy: Collects items in an unbounded buffer.
     * Use when you have occasional slow consumers.
     * WARNING: Can cause OutOfMemoryError if not monitored.
     */
    public Flux<DataBatch> handleWithBuffer(Flux<DataItem> dataStream) {
        log.info("Processing with buffer backpressure strategy");

        return dataStream
                .buffer(1000)  // Buffer up to 1000 items
                .doOnNext(batch -> log.debug("Processing batch of {} items", batch.size()))
                .map(this::processBatch)
                .doOnError(error -> log.error("Error in buffer strategy", error));
    }

    /**
     * Bounded buffer strategy: Collects items in a bounded buffer with overflow handling.
     * Better than unbounded buffer for memory management.
     */
    public Flux<DataBatch> handleWithBoundedBuffer(Flux<DataItem> dataStream) {
        log.info("Processing with bounded buffer backpressure strategy");

        return dataStream
                .buffer(500, Duration.ofSeconds(2))  // 500 items OR 2 seconds
                .doOnNext(batch -> {
                    if (batch.isEmpty()) {
                        log.debug("Empty batch received (timeout)");
                    } else {
                        log.debug("Processing bounded batch of {} items", batch.size());
                    }
                })
                .filter(batch -> !batch.isEmpty())
                .map(this::processBatch)
                .onBackpressureBuffer(100,  // Maximum buffer size
                        buffered -> log.warn("Backpressure buffer at capacity, {} items buffered", buffered),
                        BufferOverflowStrategy.DROP_OLDEST)
                .doOnError(error -> log.error("Error in bounded buffer strategy", error));
    }

    /**
     * Drop strategy: Drops items when the consumer can't keep up.
     * Use for non-critical data where missing some items is acceptable.
     */
    public Flux<ProcessedItem> handleWithDrop(Flux<DataItem> dataStream) {
        log.info("Processing with drop backpressure strategy");

        return dataStream
                .onBackpressureDrop(dropped -> {
                    log.warn("Dropped item due to backpressure: {}", dropped.id());
                    // Could send to dead letter queue or log for audit
                })
                .map(this::processItem)
                .doOnError(error -> log.error("Error in drop strategy", error));
    }

    /**
     * Error strategy: Signals an error when consumer can't keep up.
     * Use for critical operations where dropping data is not acceptable.
     */
    public Flux<ProcessedItem> handleWithError(Flux<DataItem> dataStream) {
        log.info("Processing with error backpressure strategy");

        return dataStream
                .onBackpressureError()
                .map(this::processItem)
                .doOnError(error -> log.error("Backpressure error occurred", error));
    }

    /**
     * Latest item strategy: Keeps only the latest item, dropping intermediate ones.
     * Useful for bursty producers where only the latest state matters.
     */
    public Flux<ProcessedItem> handleWithLatest(Flux<DataItem> dataStream) {
        log.info("Processing with latest backpressure strategy");

        return dataStream
                .onBackpressureLatest()  // Keeps latest, drops others
                .map(this::processItem)
                .doOnError(error -> log.error("Error in latest strategy", error));
    }

    /**
     * Request-based strategy: Manually request specific number of items.
     * Provides explicit control over demand signaling.
     */
    public Mono<Void> handleWithManualRequest(Flux<DataItem> dataStream) {
        log.info("Processing with manual request backpressure strategy");

        return dataStream
                .subscribe(
                        item -> {
                            log.debug("Processing item: {}", item.id());
                            // Process with delay to simulate slow consumer
                            Thread.sleep(100);
                        },
                        error -> log.error("Stream error", error),
                        () -> log.info("Stream completed"),
                        subscription -> {
                            // Request items in batches (pull-based demand)
                            subscription.request(10);  // Initial request for 10 items
                            // Would request more after processing batches
                        }
                )
                .then();
    }

    /**
     * Windowing strategy: Groups items into time-based windows.
     * Useful for aggregation and batching operations.
     */
    public Flux<WindowStats> handleWithTimeWindow(Flux<DataItem> dataStream) {
        log.info("Processing with time window backpressure strategy");

        return dataStream
                .window(Duration.ofSeconds(5))  // 5-second windows
                .flatMap(window -> window
                        .reduce(new WindowAggregation(0, 0.0),
                                (agg, item) -> agg.addItem(item))
                        .map(agg -> new WindowStats(System.currentTimeMillis(), agg.count, agg.total)))
                .doOnNext(stats -> log.debug("Window stats: count={}, total={}", stats.count, stats.total))
                .doOnError(error -> log.error("Error in window strategy", error));
    }

    /**
     * Rate-limited strategy: Controls item emission rate to prevent flooding.
     * Use for regulated data processing with specific throughput requirements.
     */
    public Flux<ProcessedItem> handleWithRateLimit(Flux<DataItem> dataStream, int itemsPerSecond) {
        log.info("Processing with rate limiting, {} items/sec", itemsPerSecond);

        return dataStream
                .delayElement(Duration.ofMillis(1000 / itemsPerSecond))
                .map(this::processItem)
                .doOnNext(item -> log.debug("Rate-limited item processed: {}", item.id()))
                .doOnError(error -> log.error("Error in rate limit strategy", error));
    }

    /**
     * Sampling strategy: Takes only periodic items, dropping others.
     * Useful for high-frequency data streams where sampling is acceptable.
     */
    public Flux<ProcessedItem> handleWithSampling(Flux<DataItem> dataStream) {
        log.info("Processing with sampling backpressure strategy");

        return dataStream
                .sample(Duration.ofMillis(500))  // Emit latest every 500ms
                .map(this::processItem)
                .doOnError(error -> log.error("Error in sampling strategy", error));
    }

    /**
     * Combination strategy: Buffer with fallback to drop if buffer is full.
     * Balances between preserving data and preventing memory exhaustion.
     */
    public Flux<ProcessedItem> handleWithCombination(Flux<DataItem> dataStream) {
        log.info("Processing with combination backpressure strategy");

        return dataStream
                .onBackpressureBuffer(
                        500,  // Buffer size
                        dropped -> {
                            log.warn("Buffer full, dropping item: {}", dropped.id());
                            // Could persist dropped items to dead letter queue
                        },
                        BufferOverflowStrategy.DROP_OLDEST)
                .map(this::processItem)
                .timeout(Duration.ofSeconds(30))
                .retry(3)
                .doOnError(error -> log.error("Error in combination strategy", error));
    }

    /**
     * Backpressure-aware consumer: Implements custom demand handling.
     */
    public Mono<Void> backpressureAwareConsumer(Flux<DataItem> dataStream) {
        return dataStream
                .subscribe(
                        item -> {
                            log.debug("Item received: {}", item.id());
                            // Simulate processing with variable time
                            processingTime(item);
                        },
                        error -> log.error("Consumer error", error),
                        () -> log.info("Consumer completed")
                )
                .then();
    }

    /**
     * Monitoring backpressure: Track metrics about backpressure events.
     */
    public Flux<ProcessedItem> monitorBackpressure(Flux<DataItem> dataStream) {
        var droppedCount = new java.util.concurrent.atomic.AtomicLong(0);
        var bufferedMax = new java.util.concurrent.atomic.AtomicLong(0);

        return dataStream
                .onBackpressureBuffer(1000,
                        dropped -> {
                            long count = droppedCount.incrementAndGet();
                            if (count % 100 == 0) {
                                log.warn("Total dropped items: {}", count);
                            }
                        },
                        BufferOverflowStrategy.DROP_OLDEST)
                .map(this::processItem)
                .doFinally(signal -> {
                    log.info("Stream processing completed. Signal: {}", signal);
                    log.info("Total dropped: {}", droppedCount.get());
                });
    }

    /**
     * Adaptive backpressure: Adjust buffer size based on processing load.
     */
    public Flux<ProcessedItem> adaptiveBackpressure(Flux<DataItem> dataStream) {
        var processingTime = new java.util.concurrent.atomic.AtomicLong(0);

        return dataStream
                .flatMap(item -> {
                    long startTime = System.nanoTime();
                    return Mono.fromCallable(() -> processItem(item))
                            .doFinally(signal -> {
                                long elapsed = System.nanoTime() - startTime;
                                processingTime.set(elapsed);
                                if (elapsed > 1_000_000_000) {  // > 1 second
                                    log.warn("Slow processing detected: {} ms", elapsed / 1_000_000);
                                }
                            });
                },
                calculateAdaptiveBufferSize(processingTime))  // Dynamic concurrency
                .doOnError(error -> log.error("Error in adaptive backpressure", error));
    }

    // Helper methods
    private DataBatch processBatch(java.util.List<DataItem> batch) {
        return new DataBatch(batch.size(), batch);
    }

    private ProcessedItem processItem(DataItem item) {
        return new ProcessedItem(item.id(), "PROCESSED");
    }

    private void processingTime(DataItem item) {
        try {
            Thread.sleep(50);  // Simulate processing
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private int calculateAdaptiveBufferSize(java.util.concurrent.atomic.AtomicLong processingTime) {
        long time = processingTime.get();
        if (time > 1_000_000_000) {
            return 10;  // Reduce concurrency for slow processing
        } else if (time > 100_000_000) {
            return 50;
        } else {
            return 100;  // Increase concurrency for fast processing
        }
    }

    // Helper classes
    private record DataItem(Long id, String data) {}
    private record ProcessedItem(Long id, String status) {}
    private record DataBatch(int size, java.util.List<DataItem> items) {}
    private record WindowStats(Long timestamp, int count, Double total) {}

    private static class WindowAggregation {
        int count;
        Double total;

        WindowAggregation(int count, Double total) {
            this.count = count;
            this.total = total;
        }

        WindowAggregation addItem(DataItem item) {
            return new WindowAggregation(count + 1, total);
        }
    }
}
```

## Backpressure Strategy Comparison

| Strategy | Use Case | Memory Impact | Data Loss |
|----------|----------|---------------|-----------|
| Buffer | Occasional slow consumers | High | No |
| Bounded Buffer | Known capacity limits | Medium | Possible |
| Drop | Non-critical data | Low | Yes |
| Error | Critical operations | Medium | No (fails fast) |
| Latest | Bursty producers | Low | Yes (but keeps latest) |
| Rate Limit | Regulated throughput | Low | No |
| Sample | High-frequency data | Low | Yes |

## Testing

```java
@Test
void testDropStrategy() {
    StepVerifier.create(handler.handleWithDrop(dataFlux))
        .expectNextCount(expectedCount)
        .expectComplete()
        .verify(Duration.ofSeconds(10));
}
```

## Related Documentation

- [Project Reactor Backpressure](https://projectreactor.io/docs/core/release/reference/#advanced.multicast)
- [Backpressure Strategies](https://www.baeldung.com/reactor-backpressure)
