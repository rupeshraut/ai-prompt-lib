# Create Scheduler Config

Configure thread schedulers and execution contexts for optimal reactive performance. Schedulers control which thread executes reactive operations.

## Requirements

Your scheduler configuration should include:

- **Scheduler Types**: Understand and configure different scheduler types (parallel, boundedElastic, single, etc.)
- **Thread Pool Configuration**: Configure thread pool sizes and queue behavior
- **Custom Schedulers**: Create custom schedulers for specific use cases
- **Scheduler Selection**: Choose appropriate schedulers for different operations
- **Context Propagation**: Maintain context across scheduler boundaries
- **Performance Tuning**: Configure optimal settings for application requirements
- **Monitoring**: Log and monitor scheduler behavior
- **Spring Integration**: Configure schedulers as Spring beans
- **Documentation**: Clear guidance on scheduler selection
- **Testing**: Test scheduler behavior with virtual time

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot, Project Reactor
- **Annotations**: Use `@Configuration`, `@Bean`
- **Naming**: Descriptive scheduler names for logging

## Template Example

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Bean;
import org.springframework.beans.factory.annotation.Value;
import reactor.core.scheduler.Scheduler;
import reactor.core.scheduler.Schedulers;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import lombok.extern.slf4j.Slf4j;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Scheduler configuration for reactive operations.
 * Configures thread pools and execution contexts for optimal performance.
 */
@Slf4j
@Configuration
public class ReactorSchedulerConfig {

    @Value("${reactor.scheduler.parallel.threads:4}")
    private int parallelThreads;

    @Value("${reactor.scheduler.bounded.threads:50}")
    private int boundedElasticThreads;

    @Value("${reactor.scheduler.bounded.queue:100000}")
    private int boundedElasticQueueSize;

    @Value("${reactor.scheduler.bounded.timeout:60}")
    private int boundedElasticTimeout;

    /**
     * Default scheduler for I/O operations with bounded thread pool.
     * Use for: Database operations, HTTP calls, file I/O.
     * Default bounded elastic scheduler is usually sufficient.
     */
    public Scheduler ioScheduler() {
        return Schedulers.boundedElastic();
    }

    /**
     * Scheduler for CPU-intensive operations.
     * Use for: Heavy computations, data transformations.
     * Uses parallel scheduler with number of threads = CPU cores.
     */
    public Scheduler cpuScheduler() {
        return Schedulers.parallel();
    }

    /**
     * Single-threaded scheduler for sequential operations.
     * Use for: Operations requiring single thread, maintaining order.
     * WARNING: Blocking this thread blocks all dependent operations.
     */
    public Scheduler singleScheduler() {
        return Schedulers.single();
    }

    /**
     * Immediate scheduler - no thread switch (runs on current thread).
     * Use for: Simple transformations, minimal overhead.
     */
    public Scheduler immediateScheduler() {
        return Schedulers.immediate();
    }

    /**
     * Custom bounded elastic scheduler with specific configuration.
     * For: Database operations with custom thread pool sizing.
     */
    @Bean("databaseScheduler")
    public Scheduler customDatabaseScheduler() {
        return Schedulers.newBoundedElastic(
                boundedElasticThreads,          // Max concurrent threads
                boundedElasticQueueSize,        // Queue capacity
                "db-scheduler",                 // Thread name prefix
                boundedElasticTimeout,          // Thread idle timeout (seconds)
                true                            // Daemon threads
        );
    }

    /**
     * Custom scheduler for external API calls.
     * Use for: Throttled external service calls.
     */
    @Bean("apiScheduler")
    public Scheduler customApiScheduler() {
        return Schedulers.newBoundedElastic(
                20,                    // Limited threads for API rate limiting
                500,                   // Smaller queue
                "api-scheduler",
                30,
                true
        );
    }

    /**
     * Custom scheduler for compute-heavy operations.
     * Optimized for: Data processing, calculations.
     */
    @Bean("computeScheduler")
    public Scheduler customComputeScheduler() {
        return Schedulers.newParallel(
                "compute-scheduler",
                parallelThreads,               // Number of threads
                true                           // Daemon threads
        );
    }

    /**
     * Custom scheduler with custom thread factory for monitoring.
     */
    @Bean("monitoredScheduler")
    public Scheduler monitoredScheduler() {
        ThreadFactory threadFactory = r -> {
            Thread thread = new Thread(r, "monitored-scheduler");
            thread.setDaemon(true);
            return thread;
        };

        return Schedulers.newParallel(
                "monitored",
                Runtime.getRuntime().availableProcessors(),
                threadFactory
        );
    }

    /**
     * Scheduler with thread naming for debugging.
     */
    @Bean("debugScheduler")
    public Scheduler debugScheduler() {
        AtomicInteger threadCounter = new AtomicInteger(0);
        ThreadFactory threadFactory = r -> {
            int threadId = threadCounter.incrementAndGet();
            Thread thread = new Thread(r, "debug-" + threadId);
            thread.setUncaughtExceptionHandler((t, e) ->
                    log.error("Uncaught exception in thread {}: {}", t.getName(), e.getMessage(), e)
            );
            thread.setDaemon(true);
            return thread;
        };

        return Schedulers.newParallel("debug", 4, threadFactory);
    }

    /**
     * Virtual thread scheduler (Java 19+).
     * Use for: Maximum concurrency with minimal overhead.
     */
    @Bean("virtualThreadScheduler")
    public Scheduler virtualThreadScheduler() {
        return Schedulers.newVirtualThreadPerTaskExecutor("virtual-scheduler");
    }

    /**
     * Executor-based scheduler with custom executor.
     */
    @Bean("customExecutorScheduler")
    public Scheduler customExecutorScheduler() {
        var executor = Executors.newFixedThreadPool(
                10,
                r -> {
                    Thread t = new Thread(r, "custom-executor");
                    t.setDaemon(true);
                    return t;
                }
        );

        return Schedulers.fromExecutor(executor);
    }

    /**
     * Example: Using different schedulers in reactive chains.
     */
    public Mono<ProcessedData> processDataWithSchedulers(RawData data) {
        return Mono.just(data)
                // Parse on current thread (lightweight)
                .map(this::parseData)
                // Heavy computation on compute scheduler
                .subscribeOn(customComputeScheduler())
                .flatMap(parsed -> {
                    // Database operation on database scheduler
                    return databaseOperation(parsed)
                            .subscribeOn(customDatabaseScheduler());
                })
                // API call on api scheduler
                .flatMap(dbResult ->
                        apiCall(dbResult)
                                .subscribeOn(customApiScheduler())
                )
                .publishOn(Schedulers.single())  // Final step on single thread
                .doOnNext(result -> log.info("Processed: {}", result))
                .doOnError(error -> log.error("Processing error", error));
    }

    /**
     * Thread-safe data processing with scheduler isolation.
     */
    public Flux<ProcessedData> processStreamWithIsolation(Flux<RawData> dataStream) {
        return dataStream
                // Incoming data on I/O scheduler
                .publishOn(ioScheduler())
                .doOnNext(data -> log.debug("Received on I/O scheduler: {}", data.id()))
                // Process on compute scheduler
                .publishOn(customComputeScheduler())
                .map(this::parseData)
                .map(this::transformData)
                .doOnNext(data -> log.debug("Processed on compute scheduler: {}", data.id()))
                // Persist on database scheduler
                .publishOn(customDatabaseScheduler())
                .flatMap(this::persistData)
                .doOnNext(data -> log.debug("Persisted on database scheduler: {}", data.id()));
    }

    /**
     * Scheduler selection guidelines.
     * Demonstrates when to use each scheduler type.
     */
    public interface SchedulerGuidelines {
        /**
         * Parallel Scheduler
         * - Use for: CPU-intensive operations, data transformations
         * - Threads: Number of CPU cores
         * - Characteristics: Fixed thread pool, no queuing
         * - Example: Math computations, aggregations
         */
        default Mono<Integer> cpuIntensiveOperation() {
            return Mono.fromCallable(() -> {
                // Heavy computation
                return computeExpensiveValue();
            }).subscribeOn(Schedulers.parallel());
        }

        /**
         * Bounded Elastic Scheduler
         * - Use for: I/O operations, external service calls
         * - Threads: Dynamic, up to configurable limit
         * - Characteristics: Queue-based, grows as needed
         * - Example: Database queries, HTTP calls
         */
        default Mono<String> ioOperation() {
            return Mono.fromCallable(() -> {
                // Database or HTTP operation
                return fetchFromDatabase();
            }).subscribeOn(Schedulers.boundedElastic());
        }

        /**
         * Single Scheduler
         * - Use for: Sequential operations, shared state
         * - Threads: 1
         * - Characteristics: All work serialized
         * - Example: Event sourcing, ordered processing
         */
        default Mono<Void> sequentialOperation() {
            return Mono.just("data")
                    .subscribeOn(Schedulers.single())
                    .then();
        }

        /**
         * Immediate Scheduler
         * - Use for: Lightweight transformations
         * - Threads: Current thread (no switch)
         * - Characteristics: Minimal overhead
         * - Example: Simple map operations
         */
        default Mono<String> lightweightOperation() {
            return Mono.just("data")
                    .map(s -> s.toUpperCase())
                    .subscribeOn(Schedulers.immediate());
        }

        int computeExpensiveValue();
        String fetchFromDatabase();
    }

    /**
     * Scheduler monitoring and metrics.
     */
    public void logSchedulerMetrics() {
        log.info("=== Scheduler Metrics ===");
        log.info("CPU cores: {}", Runtime.getRuntime().availableProcessors());
        log.info("Total threads: {}", Thread.activeCount());
        log.info("Free memory: {} MB", Runtime.getRuntime().freeMemory() / (1024 * 1024));
    }

    /**
     * Best practices for scheduler usage.
     */
    public static class BestPractices {
        /**
         * Rule 1: Use appropriate scheduler for operation type.
         * - I/O operations → boundedElastic
         * - CPU operations → parallel
         * - Sequential logic → single
         */

        /**
         * Rule 2: Avoid blocking the I/O scheduler.
         * Wrap blocking calls in boundedElastic.
         */

        /**
         * Rule 3: Use publishOn() for intermediate scheduler changes.
         * Use subscribeOn() only at subscription time.
         */

        /**
         * Rule 4: Be careful with single scheduler.
         * Never block it - can deadlock entire chain.
         */

        /**
         * Rule 5: Monitor scheduler queue sizes.
         * Growing queues indicate bottlenecks.
         */

        /**
         * Rule 6: Use custom schedulers for rate limiting.
         * Control thread pool size to throttle requests.
         */

        /**
         * Rule 7: Consider virtual threads (Java 19+).
         * Enables millions of concurrent operations.
         */
    }

    // Helper classes
    private record RawData(Long id, String value) {}
    private record ProcessedData(Long id, String processed) {}

    private ProcessedData parseData(RawData data) {
        return new ProcessedData(data.id(), data.value().toUpperCase());
    }

    private ProcessedData transformData(ProcessedData data) {
        return new ProcessedData(data.id(), data.processed + "_TRANSFORMED");
    }

    private Mono<ProcessedData> databaseOperation(ProcessedData data) {
        return Mono.just(data)
                .doOnNext(d -> log.debug("Database op: {}", d.id()));
    }

    private Mono<ProcessedData> apiCall(ProcessedData data) {
        return Mono.just(data)
                .doOnNext(d -> log.debug("API call: {}", d.id()));
    }

    private Mono<ProcessedData> persistData(ProcessedData data) {
        return Mono.just(data)
                .doOnNext(d -> log.debug("Persisting: {}", d.id()));
    }
}
```

## Scheduler Selection Decision Tree

```
Is it I/O?
├─ YES: Use Schedulers.boundedElastic()
│       └─ Blocking calls, HTTP, Database
└─ NO: Is it CPU-intensive?
       ├─ YES: Use Schedulers.parallel()
       │       └─ Computations, Transformations
       └─ NO: Is ordering critical?
               ├─ YES: Use Schedulers.single()
               │       └─ Sequential processing
               └─ NO: Use Schedulers.immediate()
                      └─ Lightweight transforms
```

## Performance Tuning

| Setting | Impact | Recommendation |
|---------|--------|-----------------|
| Parallel threads | CPU utilization | = CPU cores |
| Bounded threads | I/O concurrency | 2-4x CPU cores |
| Queue size | Memory usage | Monitor backpressure |
| Thread timeout | Resource cleanup | 60-120 seconds |

## Related Documentation

- [Reactor Scheduler Guide](https://projectreactor.io/docs/core/release/reference/#scheduler)
- [Java Virtual Threads (Project Loom)](https://openjdk.org/projects/loom/)
- [Thread Pool Configuration](https://www.baeldung.com/spring-scheduler-configuration)
