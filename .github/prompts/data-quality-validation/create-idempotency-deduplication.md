# Create Idempotency and Deduplication

Implement idempotent operations and deduplication strategies to ensure exactly-once processing semantics in distributed systems. Prevent duplicate processing of requests, events, and messages across microservices and stream processing pipelines.

## Requirements

Your idempotency and deduplication implementation should include:

- **Idempotency Keys**: Generate and validate unique operation identifiers
- **Duplicate Detection**: Identify and filter duplicate requests/events
- **State Management**: Track processed operations with TTL
- **Response Caching**: Cache idempotent operation results
- **Distributed Coordination**: Redis-based distributed locks and state
- **Database Constraints**: Unique indexes for duplicate prevention
- **Event Deduplication**: Kafka and stream processing deduplication
- **Retry Safety**: Handle retries without side effects
- **Client-Provided Keys**: Accept and validate client idempotency tokens
- **Audit Trail**: Log duplicate detection events

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Project Reactor, Apache Flink
- **Storage**: Redis, PostgreSQL
- **Libraries**: Spring Data Redis, Resilience4j

## Template: Spring Boot Idempotency Service

```java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.time.Duration;
import java.util.UUID;
import java.util.Optional;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import jakarta.persistence.Id;
import jakarta.persistence.Column;
import org.springframework.data.jpa.repository.JpaRepository;

/**
 * Idempotency service for ensuring exactly-once operation semantics.
 * Uses Redis for fast duplicate detection and PostgreSQL for durable storage.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class IdempotencyService {
    private final RedisTemplate<String, String> redisTemplate;
    private final IdempotencyRecordRepository repository;
    private final ObjectMapper objectMapper;

    private static final String IDEMPOTENCY_PREFIX = "idempotency:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);

    /**
     * Execute operation with idempotency guarantee.
     * Returns cached result if operation already completed.
     */
    public <T> T executeIdempotent(
            String idempotencyKey,
            Class<T> responseType,
            IdempotentOperation<T> operation) {

        String key = IDEMPOTENCY_PREFIX + idempotencyKey;

        // Check Redis cache first (fast path)
        String cachedResult = redisTemplate.opsForValue().get(key);
        if (cachedResult != null) {
            log.info("Idempotency hit in Redis: {}", idempotencyKey);
            return deserialize(cachedResult, responseType);
        }

        // Check database for durable record
        Optional<IdempotencyRecord> record = repository.findById(idempotencyKey);
        if (record.isPresent() && record.get().getStatus() == OperationStatus.COMPLETED) {
            log.info("Idempotency hit in database: {}", idempotencyKey);
            T result = deserialize(record.get().getResponse(), responseType);

            // Refresh Redis cache
            cacheResult(key, record.get().getResponse());
            return result;
        }

        // Execute operation (new request)
        return executeNewOperation(idempotencyKey, key, operation);
    }

    /**
     * Execute new operation with idempotency tracking.
     */
    @Transactional
    private <T> T executeNewOperation(
            String idempotencyKey,
            String redisKey,
            IdempotentOperation<T> operation) {

        // Create in-progress record to prevent concurrent execution
        IdempotencyRecord record = new IdempotencyRecord();
        record.setIdempotencyKey(idempotencyKey);
        record.setStatus(OperationStatus.IN_PROGRESS);
        record.setStartedAt(System.currentTimeMillis());

        try {
            repository.save(record);
        } catch (Exception e) {
            // Race condition: another thread already started
            log.warn("Concurrent execution detected: {}", idempotencyKey);
            throw new IdempotencyConflictException("Operation already in progress");
        }

        try {
            // Execute actual operation
            log.info("Executing new operation: {}", idempotencyKey);
            T result = operation.execute();

            // Update record as completed
            String serializedResult = serialize(result);
            record.setResponse(serializedResult);
            record.setStatus(OperationStatus.COMPLETED);
            record.setCompletedAt(System.currentTimeMillis());
            repository.save(record);

            // Cache result in Redis
            cacheResult(redisKey, serializedResult);

            log.info("Operation completed: {}", idempotencyKey);
            return result;

        } catch (Exception e) {
            // Mark as failed
            record.setStatus(OperationStatus.FAILED);
            record.setErrorMessage(e.getMessage());
            record.setCompletedAt(System.currentTimeMillis());
            repository.save(record);

            log.error("Operation failed: {}", idempotencyKey, e);
            throw new IdempotentOperationException("Operation failed", e);
        }
    }

    /**
     * Generate idempotency key from request parameters.
     */
    public String generateIdempotencyKey(Object... params) {
        String combined = String.join(":",
            java.util.Arrays.stream(params)
                .map(String::valueOf)
                .toArray(String[]::new));

        return UUID.nameUUIDFromBytes(combined.getBytes()).toString();
    }

    /**
     * Validate client-provided idempotency key.
     */
    public boolean isValidIdempotencyKey(String key) {
        if (key == null || key.isBlank()) {
            return false;
        }

        // UUID format validation
        try {
            UUID.fromString(key);
            return true;
        } catch (IllegalArgumentException e) {
            return false;
        }
    }

    private void cacheResult(String key, String value) {
        redisTemplate.opsForValue().set(key, value, DEFAULT_TTL);
    }

    private <T> T deserialize(String json, Class<T> type) {
        try {
            return objectMapper.readValue(json, type);
        } catch (Exception e) {
            throw new SerializationException("Failed to deserialize", e);
        }
    }

    private String serialize(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (Exception e) {
            throw new SerializationException("Failed to serialize", e);
        }
    }
}

/**
 * Functional interface for idempotent operations.
 */
@FunctionalInterface
interface IdempotentOperation<T> {
    T execute();
}

/**
 * Idempotency record entity for durable storage.
 */
@Entity
@Table(name = "idempotency_records",
       indexes = {@Index(name = "idx_idempotency_key", columnList = "idempotency_key", unique = true)})
@Data
@NoArgsConstructor
class IdempotencyRecord {
    @Id
    @Column(name = "idempotency_key", nullable = false, unique = true, length = 64)
    private String idempotencyKey;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OperationStatus status;

    @Column(columnDefinition = "TEXT")
    private String response;

    @Column
    private String errorMessage;

    @Column(nullable = false)
    private Long startedAt;

    @Column
    private Long completedAt;
}

enum OperationStatus {
    IN_PROGRESS,
    COMPLETED,
    FAILED
}

interface IdempotencyRecordRepository extends JpaRepository<IdempotencyRecord, String> {
}
```

## Usage Example: Payment Controller with Idempotency

```java
@Slf4j
@RestController
@RequestMapping("/api/payments")
@RequiredArgsConstructor
public class PaymentController {
    private final IdempotencyService idempotencyService;
    private final PaymentService paymentService;

    /**
     * Create payment with client-provided idempotency key.
     */
    @PostMapping
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid PaymentRequest request) {

        // Validate idempotency key
        if (!idempotencyService.isValidIdempotencyKey(idempotencyKey)) {
            return ResponseEntity.badRequest()
                .body(new ErrorResponse("Invalid idempotency key format"));
        }

        // Execute with idempotency guarantee
        PaymentResponse response = idempotencyService.executeIdempotent(
            idempotencyKey,
            PaymentResponse.class,
            () -> paymentService.processPayment(request)
        );

        return ResponseEntity.ok(response);
    }

    /**
     * Create refund with server-generated idempotency key.
     */
    @PostMapping("/{paymentId}/refund")
    public ResponseEntity<RefundResponse> createRefund(
            @PathVariable String paymentId,
            @RequestBody @Valid RefundRequest request) {

        // Generate idempotency key from request parameters
        String idempotencyKey = idempotencyService.generateIdempotencyKey(
            "refund", paymentId, request.amount(), request.reason()
        );

        RefundResponse response = idempotencyService.executeIdempotent(
            idempotencyKey,
            RefundResponse.class,
            () -> paymentService.processRefund(paymentId, request)
        );

        return ResponseEntity.ok(response);
    }
}
```

## Reactive Idempotency with Project Reactor

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class ReactiveIdempotencyService {
    private final ReactiveRedisTemplate<String, String> redisTemplate;
    private final ReactiveIdempotencyRepository repository;
    private final ObjectMapper objectMapper;

    private static final String IDEMPOTENCY_PREFIX = "idempotency:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);

    /**
     * Execute operation with reactive idempotency guarantee.
     */
    public <T> Mono<T> executeIdempotent(
            String idempotencyKey,
            Class<T> responseType,
            Mono<T> operation) {

        String key = IDEMPOTENCY_PREFIX + idempotencyKey;

        return checkCache(key, responseType)
            .switchIfEmpty(checkDatabase(idempotencyKey, responseType, key))
            .switchIfEmpty(executeNewOperation(idempotencyKey, key, operation, responseType))
            .doOnSuccess(result -> log.info("Idempotent operation completed: {}", idempotencyKey));
    }

    private <T> Mono<T> checkCache(String key, Class<T> responseType) {
        return redisTemplate.opsForValue().get(key)
            .doOnNext(cached -> log.info("Idempotency hit in Redis"))
            .map(json -> deserialize(json, responseType));
    }

    private <T> Mono<T> checkDatabase(
            String idempotencyKey,
            Class<T> responseType,
            String redisKey) {

        return repository.findById(idempotencyKey)
            .filter(record -> record.getStatus() == OperationStatus.COMPLETED)
            .doOnNext(record -> log.info("Idempotency hit in database"))
            .flatMap(record -> {
                T result = deserialize(record.getResponse(), responseType);

                // Refresh Redis cache
                return redisTemplate.opsForValue()
                    .set(redisKey, record.getResponse(), DEFAULT_TTL)
                    .thenReturn(result);
            });
    }

    private <T> Mono<T> executeNewOperation(
            String idempotencyKey,
            String redisKey,
            Mono<T> operation,
            Class<T> responseType) {

        // Create in-progress record
        IdempotencyRecord record = new IdempotencyRecord();
        record.setIdempotencyKey(idempotencyKey);
        record.setStatus(OperationStatus.IN_PROGRESS);
        record.setStartedAt(System.currentTimeMillis());

        return repository.save(record)
            .flatMap(saved -> operation
                .flatMap(result -> {
                    // Update as completed
                    String serialized = serialize(result);
                    saved.setResponse(serialized);
                    saved.setStatus(OperationStatus.COMPLETED);
                    saved.setCompletedAt(System.currentTimeMillis());

                    return repository.save(saved)
                        .then(redisTemplate.opsForValue().set(redisKey, serialized, DEFAULT_TTL))
                        .thenReturn(result);
                })
                .onErrorResume(error -> {
                    // Update as failed
                    saved.setStatus(OperationStatus.FAILED);
                    saved.setErrorMessage(error.getMessage());
                    saved.setCompletedAt(System.currentTimeMillis());

                    return repository.save(saved)
                        .then(Mono.error(new IdempotentOperationException("Operation failed", error)));
                })
            )
            .onErrorResume(error -> {
                if (error instanceof DuplicateKeyException) {
                    log.warn("Concurrent execution detected: {}", idempotencyKey);
                    return Mono.error(new IdempotencyConflictException("Operation already in progress"));
                }
                return Mono.error(error);
            });
    }

    private <T> T deserialize(String json, Class<T> type) {
        try {
            return objectMapper.readValue(json, type);
        } catch (Exception e) {
            throw new SerializationException("Failed to deserialize", e);
        }
    }

    private String serialize(Object obj) {
        try {
            return objectMapper.writeValueAsString(obj);
        } catch (Exception e) {
            throw new SerializationException("Failed to serialize", e);
        }
    }
}
```

## Apache Flink Deduplication

```java
/**
 * Stateful deduplication function for Flink streaming.
 * Uses keyed state to track processed event IDs.
 */
@Slf4j
public class DeduplicationFunction extends KeyedProcessFunction<String, Event, Event> {

    // State to track seen event IDs
    private transient ValueState<Boolean> seenState;

    // State TTL config: cleanup after 24 hours
    private static final StateTtlConfig TTL_CONFIG = StateTtlConfig
        .newBuilder(Time.hours(24))
        .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
        .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
        .build();

    @Override
    public void open(Configuration parameters) throws Exception {
        ValueStateDescriptor<Boolean> descriptor =
            new ValueStateDescriptor<>("seen", Boolean.class);
        descriptor.enableTimeToLive(TTL_CONFIG);
        seenState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(
            Event event,
            Context ctx,
            Collector<Event> out) throws Exception {

        // Check if already seen
        Boolean seen = seenState.value();

        if (seen != null && seen) {
            // Duplicate - skip
            log.debug("Duplicate event detected: {}", event.id());
            ctx.output(duplicateOutputTag, event);
            return;
        }

        // First occurrence - mark as seen and emit
        seenState.update(true);
        log.debug("New event: {}", event.id());
        out.collect(event);
    }

    private static final OutputTag<Event> duplicateOutputTag =
        new OutputTag<Event>("duplicates") {};
}

/**
 * Flink job with deduplication.
 */
public class DeduplicatedStreamJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Event> events = env
            .addSource(new KafkaSource<>(...))
            .keyBy(Event::customerId);

        // Apply deduplication
        SingleOutputStreamOperator<Event> deduplicated = events
            .keyBy(Event::eventId)
            .process(new DeduplicationFunction());

        // Main output: unique events
        deduplicated.addSink(new ProcessedEventSink());

        // Side output: duplicates for monitoring
        deduplicated
            .getSideOutput(duplicateOutputTag)
            .addSink(new DuplicateEventSink());

        env.execute("Deduplicated Stream Processing");
    }
}
```

## Database Unique Constraints for Deduplication

```sql
-- Idempotency records table
CREATE TABLE idempotency_records (
    idempotency_key VARCHAR(64) PRIMARY KEY,
    status VARCHAR(20) NOT NULL,
    response TEXT,
    error_message TEXT,
    started_at BIGINT NOT NULL,
    completed_at BIGINT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index for cleanup queries
CREATE INDEX idx_idempotency_status_created
ON idempotency_records(status, created_at);

-- Payment deduplication via unique constraint
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    idempotency_key VARCHAR(64) NOT NULL UNIQUE,
    customer_id UUID NOT NULL,
    amount DECIMAL(19, 2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Ensure payment uniqueness per customer and reference
CREATE UNIQUE INDEX idx_payment_dedup
ON payments(customer_id, external_reference_id);

-- Event processing deduplication
CREATE TABLE processed_events (
    event_id VARCHAR(64) PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    processed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processing_time_ms INTEGER
);

-- Partition by date for efficient cleanup
CREATE TABLE processed_events_partitioned (
    event_id VARCHAR(64) NOT NULL,
    event_type VARCHAR(50) NOT NULL,
    processed_at TIMESTAMP NOT NULL,
    processing_date DATE NOT NULL,
    PRIMARY KEY (event_id, processing_date)
) PARTITION BY RANGE (processing_date);
```

## Configuration

```yaml
# application.yml
spring:
  redis:
    host: ${REDIS_HOST:localhost}
    port: ${REDIS_PORT:6379}
    timeout: 2000ms
    lettuce:
      pool:
        max-active: 20
        max-idle: 10
        min-idle: 5

  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/idempotency
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:secret}

  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true

idempotency:
  cache-ttl: 24h
  cleanup-interval: 6h
  retention-days: 30

deduplication:
  enabled: true
  window-duration: 24h
  bloom-filter-size: 10000000
  bloom-filter-fpp: 0.01
```

## Best Practices

1. **Use Client-Provided Keys for External APIs**: Accept Idempotency-Key header for payment and order APIs to support client retries
2. **Generate Keys for Internal Operations**: Create deterministic keys from request parameters for internal idempotency
3. **Set Appropriate TTL**: Keep idempotency records for 24 hours minimum, longer for financial transactions (30-90 days)
4. **Cache Results in Redis**: Use Redis for fast duplicate detection, PostgreSQL for durability
5. **Handle Concurrent Requests**: Use database unique constraints to prevent race conditions during concurrent duplicate requests
6. **Return Cached Responses**: Return identical response for duplicate requests, including HTTP status codes
7. **Implement Cleanup Jobs**: Regularly purge expired idempotency records to prevent unbounded growth
8. **Monitor Duplicate Rates**: Track duplicate detection metrics to identify retry storms or client issues
9. **Use State TTL in Flink**: Configure state TTL to automatically cleanup deduplication state in streaming jobs
10. **Validate Key Format**: Enforce UUID or hash format for idempotency keys to prevent collisions

## Related Documentation

- [Stripe Idempotency Guide](https://stripe.com/docs/api/idempotent_requests)
- [Apache Flink State TTL](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/state/#state-time-to-live-ttl)
- [Spring Data Redis](https://spring.io/projects/spring-data-redis)
- [PostgreSQL Unique Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
