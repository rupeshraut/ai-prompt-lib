# Data Quality & Validation Library

Comprehensive prompts for implementing data validation, schema validation, and idempotency controls to ensure data integrity, prevent duplicate processing, and maintain high data quality in production microservices.

## Available Prompts

### 1. Create Schema Validation
Implement JSON Schema validation for API requests, events, and data transformations.

| Feature | Description |
|---------|-------------|
| JSON Schema Validation | Validate against JSON Schema specifications |
| Bean Validation (JSR-380) | Declarative validation with annotations |
| Custom Validators | Business rule validation |
| Error Messages | Detailed validation error responses |
| Nested Object Validation | Recursive validation of complex structures |
| Collection Validation | Validate list and map elements |
| Conditional Validation | Context-dependent validation rules |
| Schema Registry | Centralized schema management |
| Schema Evolution | Backward/forward compatibility checks |
| Performance | Compiled schema validation |

### 2. Create Idempotency and Deduplication
Implement idempotent operations and deduplication to ensure exactly-once processing semantics.

| Feature | Description |
|---------|-------------|
| Idempotency Keys | Client and server-generated unique keys |
| Duplicate Detection | Identify duplicate requests and events |
| State Management | Track processed operations with TTL |
| Response Caching | Return cached results for duplicates |
| Distributed Locks | Redis-based coordination |
| Database Constraints | Unique indexes for deduplication |
| Event Deduplication | Stream processing deduplication |
| Retry Safety | Handle retries without side effects |
| Audit Trail | Log duplicate detection events |
| Flink Integration | Stateful deduplication in streaming |

## Quick Start

### 1. Setup Schema Validation

```
@workspace Use .github/prompts/data-quality-validation/create-schema-validation.md

Add JSON Schema validation for order creation API with custom business rules
```

### 2. Implement Idempotency

```
@workspace Use .github/prompts/data-quality-validation/create-idempotency-deduplication.md

Configure idempotency for payment processing with 24-hour key retention
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Request                            │
│                    POST /api/orders                          │
│                    Idempotency-Key: uuid                     │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Schema Validation │
            │  (JSON Schema)     │
            │                    │
            │  ✓ Required fields │
            │  ✓ Data types      │
            │  ✓ Constraints     │
            │  ✓ Business rules  │
            └────────┬───────────┘
                     │ Valid
                     ▼
            ┌────────────────────┐
            │  Idempotency Check │
            │  (Redis + DB)      │
            └────────┬───────────┘
                     │
         ┌───────────┴───────────┐
         │                       │
         ▼                       ▼
    Already Processed      New Request
         │                       │
         │                       ▼
         │              ┌─────────────────┐
         │              │ Create Record   │
         │              │ Status:         │
         │              │ IN_PROGRESS     │
         │              └────────┬────────┘
         │                       │
         │                       ▼
         │              ┌─────────────────┐
         │              │ Process Order   │
         │              │ (Business Logic)│
         │              └────────┬────────┘
         │                       │
         │                       ▼
         │              ┌─────────────────┐
         │              │ Update Record   │
         │              │ Status:         │
         │              │ COMPLETED       │
         │              │ Cache Response  │
         │              └────────┬────────┘
         │                       │
         └───────────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Return Response   │
            │  (Original or      │
            │   Cached)          │
            └────────────────────┘

Schema Validation Flow:

    JSON Request
         │
         ▼
    ┌─────────────────────┐
    │  JSON Schema        │
    │  Validation         │
    │  - Structure        │
    │  - Types            │
    │  - Required fields  │
    └──────┬──────────────┘
           │ Valid
           ▼
    ┌─────────────────────┐
    │  Bean Validation    │
    │  (@Valid)           │
    │  - @NotNull         │
    │  - @Size            │
    │  - @Email           │
    │  - @Pattern         │
    └──────┬──────────────┘
           │ Valid
           ▼
    ┌─────────────────────┐
    │  Custom Validators  │
    │  - Business rules   │
    │  - Cross-field      │
    │  - Database checks  │
    └──────┬──────────────┘
           │ Valid
           ▼
    Process Request

Deduplication in Flink:

    Kafka Source
         │
         ▼
    ┌─────────────────────┐
    │  keyBy(eventId)     │
    └──────┬──────────────┘
           │
           ▼
    ┌─────────────────────┐
    │  Deduplication      │
    │  Function           │
    │  (Keyed State)      │
    │  - Check if seen    │
    │  - Update state     │
    │  - TTL: 24h         │
    └──────┬──────────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
  Unique     Duplicates
  Events     (Side Output)
     │           │
     ▼           ▼
  Process    Log/Monitor
```

## Integration Patterns

### Spring Boot Controller with Validation

```java
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
@Validated
public class OrderController {

    private final OrderService orderService;
    private final IdempotencyService idempotencyService;
    private final SchemaValidator schemaValidator;

    /**
     * Create order with schema validation and idempotency.
     */
    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody @Valid CreateOrderRequest request) {

        // Validate against JSON Schema
        schemaValidator.validate(request, "create-order-schema.json");

        // Execute with idempotency guarantee
        OrderResponse response = idempotencyService.executeIdempotent(
            idempotencyKey,
            OrderResponse.class,
            () -> orderService.createOrder(request)
        );

        return ResponseEntity.ok(response);
    }
}

/**
 * Order request with Bean Validation.
 */
@Data
@Schema(description = "Create order request")
public class CreateOrderRequest {

    @NotNull(message = "Customer ID is required")
    @Pattern(regexp = "^CUST-[0-9]{6}$", message = "Invalid customer ID format")
    private String customerId;

    @NotNull(message = "Items list is required")
    @NotEmpty(message = "Order must contain at least one item")
    @Valid
    private List<OrderItem> items;

    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.01", message = "Amount must be positive")
    @DecimalMax(value = "1000000.00", message = "Amount exceeds maximum")
    private BigDecimal amount;

    @NotNull(message = "Currency is required")
    @Pattern(regexp = "^(USD|EUR|GBP)$", message = "Unsupported currency")
    private String currency;

    @Email(message = "Invalid email format")
    private String notificationEmail;

    @ValidOrderPriority  // Custom validator
    private String priority;
}

/**
 * Custom validator for business rules.
 */
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = OrderPriorityValidator.class)
public @interface ValidOrderPriority {
    String message() default "Invalid order priority";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class OrderPriorityValidator implements ConstraintValidator<ValidOrderPriority, String> {

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true;  // Use @NotNull for null checks
        }

        return value.matches("^(LOW|MEDIUM|HIGH|URGENT)$");
    }
}
```

### JSON Schema Validation

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class JsonSchemaValidator {

    private final Map<String, JsonSchema> schemaCache = new ConcurrentHashMap<>();
    private final ObjectMapper objectMapper;

    /**
     * Validate object against JSON Schema.
     */
    public void validate(Object obj, String schemaName) {
        try {
            JsonSchema schema = getSchema(schemaName);
            JsonNode jsonNode = objectMapper.valueToTree(obj);

            Set<ValidationMessage> errors = schema.validate(jsonNode);

            if (!errors.isEmpty()) {
                String errorMessages = errors.stream()
                    .map(ValidationMessage::getMessage)
                    .collect(Collectors.joining(", "));

                throw new SchemaValidationException(
                    "Schema validation failed: " + errorMessages
                );
            }

            log.debug("Schema validation passed: {}", schemaName);

        } catch (Exception e) {
            log.error("Schema validation error: {}", schemaName, e);
            throw new SchemaValidationException("Validation failed", e);
        }
    }

    private JsonSchema getSchema(String schemaName) throws Exception {
        return schemaCache.computeIfAbsent(schemaName, name -> {
            try {
                InputStream schemaStream = getClass()
                    .getResourceAsStream("/schemas/" + name);

                JsonNode schemaNode = objectMapper.readTree(schemaStream);

                JsonSchemaFactory factory = JsonSchemaFactory.getInstance(
                    SpecVersion.VersionFlag.V7
                );

                return factory.getSchema(schemaNode);

            } catch (Exception e) {
                throw new RuntimeException("Failed to load schema: " + name, e);
            }
        });
    }
}
```

### Idempotency with Redis and Database

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class IdempotencyService {

    private final RedisTemplate<String, String> redisTemplate;
    private final IdempotencyRecordRepository repository;
    private final ObjectMapper objectMapper;

    private static final String IDEMPOTENCY_PREFIX = "idempotency:";
    private static final Duration DEFAULT_TTL = Duration.ofHours(24);

    /**
     * Execute operation with idempotency guarantee.
     */
    public <T> T executeIdempotent(
            String idempotencyKey,
            Class<T> responseType,
            Supplier<T> operation) {

        String key = IDEMPOTENCY_PREFIX + idempotencyKey;

        // Check Redis cache first
        String cachedResult = redisTemplate.opsForValue().get(key);
        if (cachedResult != null) {
            log.info("Idempotency cache hit: {}", idempotencyKey);
            return deserialize(cachedResult, responseType);
        }

        // Check database
        Optional<IdempotencyRecord> record = repository.findById(idempotencyKey);
        if (record.isPresent() && record.get().isCompleted()) {
            log.info("Idempotency database hit: {}", idempotencyKey);
            T result = deserialize(record.get().getResponse(), responseType);
            cacheResult(key, record.get().getResponse());
            return result;
        }

        // Execute new operation
        return executeNewOperation(idempotencyKey, key, operation);
    }

    @Transactional
    private <T> T executeNewOperation(
            String idempotencyKey,
            String redisKey,
            Supplier<T> operation) {

        // Create in-progress record
        IdempotencyRecord record = new IdempotencyRecord();
        record.setIdempotencyKey(idempotencyKey);
        record.setStatus(OperationStatus.IN_PROGRESS);

        try {
            repository.save(record);

            // Execute operation
            T result = operation.get();

            // Update as completed
            String serialized = serialize(result);
            record.setResponse(serialized);
            record.setStatus(OperationStatus.COMPLETED);
            repository.save(record);

            // Cache result
            cacheResult(redisKey, serialized);

            return result;

        } catch (Exception e) {
            record.setStatus(OperationStatus.FAILED);
            record.setErrorMessage(e.getMessage());
            repository.save(record);
            throw e;
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
```

### Flink Deduplication

```java
/**
 * Stateful deduplication function for Flink.
 */
public class DeduplicationFunction extends KeyedProcessFunction<String, Event, Event> {

    private transient ValueState<Boolean> seenState;

    @Override
    public void open(Configuration parameters) throws Exception {
        StateTtlConfig ttlConfig = StateTtlConfig
            .newBuilder(Time.hours(24))
            .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
            .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
            .build();

        ValueStateDescriptor<Boolean> descriptor =
            new ValueStateDescriptor<>("seen", Boolean.class);
        descriptor.enableTimeToLive(ttlConfig);

        seenState = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(
            Event event,
            Context ctx,
            Collector<Event> out) throws Exception {

        Boolean seen = seenState.value();

        if (seen != null && seen) {
            // Duplicate - output to side output
            ctx.output(duplicateOutputTag, event);
            return;
        }

        // First occurrence
        seenState.update(true);
        out.collect(event);
    }

    private static final OutputTag<Event> duplicateOutputTag =
        new OutputTag<Event>("duplicates") {};
}
```

## Key Concepts

### Schema Validation
Validating data structure and content against predefined schemas (JSON Schema) and business rules (Bean Validation).

### Idempotency
Ensuring operations can be safely retried without side effects. Same request produces same result.

### Deduplication
Identifying and filtering duplicate events/requests based on unique identifiers.

### Idempotency Key
Unique identifier for an operation, either client-provided or server-generated from request parameters.

### State TTL
Time-to-live for deduplication state in streaming systems. Balances memory usage with duplicate detection window.

### Bean Validation (JSR-380)
Declarative validation using annotations (@NotNull, @Size, @Valid) on domain objects.

## Best Practices

1. **Layer Validation Approaches**
   - JSON Schema for structure validation
   - Bean Validation for field constraints
   - Custom validators for business rules
   - Database constraints for uniqueness

2. **Client-Provided Idempotency Keys**
   - Accept Idempotency-Key header for API calls
   - Validate key format (UUID recommended)
   - Return 400 for invalid keys
   - Document idempotency behavior

3. **Set Appropriate TTL**
   - API idempotency: 24 hours minimum
   - Financial transactions: 30-90 days
   - Stream deduplication: 1-24 hours
   - Consider storage costs vs duplicate risk

4. **Return Cached Responses**
   - Return identical response for duplicate requests
   - Include same HTTP status codes
   - Add X-Idempotent-Replay header

5. **Handle Edge Cases**
   - In-progress operations (return 409 Conflict)
   - Failed operations (allow retry with same key)
   - Expired keys (treat as new request)

6. **Monitor Data Quality**
   - Track validation failure rates
   - Monitor duplicate detection rate
   - Alert on validation error spikes
   - Log schema validation failures

7. **Schema Evolution**
   - Use backward-compatible changes
   - Version schemas explicitly
   - Test compatibility before deployment
   - Maintain schema registry

8. **Performance Optimization**
   - Compile and cache JSON schemas
   - Use Redis for fast duplicate checks
   - Async validation where possible
   - Batch database constraint checks

## Configuration

```yaml
# application.yml
validation:
  json-schema:
    enabled: true
    schema-location: classpath:/schemas
    cache-schemas: true
    fail-fast: true

  bean-validation:
    fail-fast: false
    validate-return-value: true

idempotency:
  enabled: true
  cache-ttl: 24h
  cleanup-interval: 6h
  retention-days: 30

spring:
  redis:
    host: ${REDIS_HOST:localhost}
    port: 6379

  datasource:
    url: jdbc:postgresql://localhost:5432/idempotency
```

## Troubleshooting

### Validation Always Failing
1. Check schema file location
2. Verify schema is valid JSON Schema
3. Review validation error messages
4. Test with schema validation tools

### Duplicate Detection Not Working
1. Verify idempotency key format
2. Check Redis connectivity
3. Review database unique constraints
4. Check TTL configuration

### High Idempotency Storage
1. Reduce retention period
2. Implement cleanup jobs
3. Use shorter TTL in Redis
4. Partition by date

### Schema Evolution Breaking Clients
1. Test backward compatibility
2. Version schemas explicitly
3. Use optional fields for new data
4. Deprecate gradually

## Related Documentation

- [JSON Schema Specification](https://json-schema.org/)
- [Bean Validation (JSR-380)](https://beanvalidation.org/)
- [Stripe Idempotency](https://stripe.com/docs/api/idempotent_requests)
- [Apache Flink State](https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/datastream/fault-tolerance/state/)

## Library Navigation

This library focuses on data quality and validation. For related capabilities:
- See **observability** library for validation metrics and alerting
- See **resilience-operations** library for handling validation failures
- See **security-compliance** library for data encryption and masking
