# Project Reactor Reactive Programming Prompts

This directory contains custom prompts for generating reactive applications using Project Reactor and Spring WebFlux.

## Available Prompts

### Core Reactive Types

| Template | Description |
|----------|-------------|
| `create-mono-handler.md` | Single-value async operations with Mono |
| `create-flux-processor.md` | Multi-value stream processing with Flux |

### Business Logic Layer

| Template | Description |
|----------|-------------|
| `create-reactive-service.md` | Business logic services using reactive patterns |

### Web & API Layer

| Template | Description |
|----------|-------------|
| `create-reactive-controller.md` | Spring WebFlux REST endpoints with reactive returns |
| `create-webflux-client.md` | Non-blocking HTTP clients using WebClient |

### Data Access Layer

| Template | Description |
|----------|-------------|
| `create-reactive-repository.md` | Spring Data R2DBC repositories for reactive database access |

### Advanced Patterns

| Template | Description |
|----------|-------------|
| `create-backpressure-handler.md` | Handling backpressure, buffering, and flow control |
| `create-error-handling.md` | Error recovery, retry logic, and exception handling |
| `create-scheduler-config.md` | Thread scheduling, Scheduler configuration, and execution contexts |

### Testing & Quality

| Template | Description |
|----------|-------------|
| `create-reactive-test.md` | Unit and integration tests using StepVerifier and Reactor test utilities |

## Quick Start

### 1. Create Mono Handler

```
@workspace Use .github/prompts/project-reactor/create-mono-handler.md

Create a UserMonoHandler for single-value operations like user lookup
```

### 2. Create Flux Processor

```
@workspace Use .github/prompts/project-reactor/create-flux-processor.md

Create an OrderFluxProcessor for streaming order data with filtering and transformations
```

### 3. Create Reactive Service

```
@workspace Use .github/prompts/project-reactor/create-reactive-service.md

Create an OrderService with reactive methods for create, update, retrieve operations
```

### 4. Create Reactive Controller

```
@workspace Use .github/prompts/project-reactor/create-reactive-controller.md

Create an OrderController with reactive REST endpoints returning Mono/Flux responses
```

### 5. Create WebFlux Client

```
@workspace Use .github/prompts/project-reactor/create-webflux-client.md

Create a PaymentServiceClient for non-blocking external API integration
```

### 6. Create Reactive Repository

```
@workspace Use .github/prompts/project-reactor/create-reactive-repository.md

Create an OrderRepository with R2DBC for async database operations
```

### 7. Create Error Handling

```
@workspace Use .github/prompts/project-reactor/create-error-handling.md

Create error recovery strategies with retry, fallback, and circuit breaker patterns
```

### 8. Create Backpressure Handler

```
@workspace Use .github/prompts/project-reactor/create-backpressure-handler.md

Create backpressure strategies for managing data flow between producers and consumers
```

### 9. Create Scheduler Config

```
@workspace Use .github/prompts/project-reactor/create-scheduler-config.md

Create scheduler configuration for optimal thread pool management and execution
```

### 10. Create Tests

```
@workspace Use .github/prompts/project-reactor/create-reactive-test.md

Create comprehensive tests using StepVerifier and virtual time scheduling
```

## Reactive Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        HTTP Request                             │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ WebFlux REST    │
                    │ Controller      │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────────────────────┐
                    │ Reactive Business Service       │
                    │ (Mono/Flux operations)          │
                    └────────┬────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        ┌───────────┐ ┌───────────┐ ┌──────────────┐
        │ WebClient │ │  R2DBC    │ │   Cache      │
        │ (External)│ │(Database) │ │  (Fallback)  │
        └──────┬────┘ └─────┬─────┘ └──────┬───────┘
               │            │              │
        ┌──────┴────────────┴──────────────┘
        │
        ▼
Error Handling & Retry
        │
        ▼
Backpressure Management
        │
        ▼
Scheduler Configuration
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        HTTP Response                             │
└─────────────────────────────────────────────────────────────────┘
```

## Key Concepts

### Mono
- Represents 0 or 1 element
- Use for: Single values, operations that complete with one result
- Example: `getUserById()`, `createOrder()`

### Flux
- Represents 0 to N elements
- Use for: Streams, collections, time-based events
- Example: `getAllOrders()`, `streamPaymentEvents()`

### Schedulers
- **parallel()** - CPU-intensive operations (threads = CPU cores)
- **boundedElastic()** - I/O operations (blocking calls, database, HTTP)
- **single()** - Sequential operations (maintains order)
- **immediate()** - Current thread (lightweight transforms)

### Backpressure
Prevents producers from overwhelming consumers:
- **buffer()** - Collect items with capacity limit
- **drop()** - Drop items if consumer can't keep up
- **latest()** - Keep only the latest item
- **error()** - Signal error on backpressure

### Error Handling
```
onErrorReturn()    → Return fallback value
onErrorResume()    → Fall back to alternative source
retry()            → Automatic retry with backoff
timeout()          → Handle slow operations
circuit breaker    → Prevent cascading failures
```

## Technology Stack

- **Framework**: Spring Boot 3.x with Spring WebFlux
- **Reactive Library**: Project Reactor (Mono, Flux)
- **Database**: Spring Data R2DBC (reactive database access)
- **HTTP Client**: WebClient (reactive HTTP client)
- **Testing**: Reactor Test, JUnit 5, StepVerifier
- **Error Handling**: Resilience4j (circuit breaker, retry)
- **Thread Management**: Schedulers (parallel, boundedElastic, single)

## Best Practices

1. **Choose the right scheduler** for each operation
2. **Handle backpressure** explicitly in high-throughput scenarios
3. **Always provide error handling** - use retry, fallback, or circuit breaker
4. **Test reactive code** with StepVerifier and virtual time
5. **Avoid blocking** operations in the reactive chain
6. **Use `subscribeOn()` at subscription time**, `publishOn()` for intermediate changes
7. **Monitor queue sizes** and scheduler metrics
8. **Use context propagation** for correlation IDs and user information

## Common Patterns

### Request-Response Flow
```java
service.findById(id)
    .map(mapper::toResponse)
    .switchIfEmpty(Mono.error(NotFoundException))
    .onErrorResume(e -> handleError(e))
    .subscribeOn(Schedulers.boundedElastic());
```

### Error Recovery with Fallback
```java
primarySource()
    .retryWhen(Retry.backoff(3, Duration.ofMillis(100)))
    .onErrorResume(e -> fallbackSource())
    .onErrorReturn(defaultValue);
```

### Concurrent Operations
```java
Mono.zip(
    service1.getUser(id),
    service2.getOrders(id),
    service3.getPayments(id)
).map(tuple -> combine(tuple.getT1(), tuple.getT2(), tuple.getT3()));
```

### Stream Processing
```java
flux
    .filter(item -> item.isValid())
    .map(item -> transform(item))
    .buffer(100, Duration.ofSeconds(5))
    .flatMap(batch -> processBatch(batch));
```

## Troubleshooting

### Issue: Blocking occurs in reactive chain
**Solution**: Use `boundedElastic()` scheduler for blocking I/O

### Issue: Backpressure errors
**Solution**: Implement backpressure handling strategy (buffer, drop, etc.)

### Issue: Memory leaks or resource issues
**Solution**: Ensure proper cleanup with `doFinally()` or `using()`

### Issue: Tests hang or timeout
**Solution**: Use StepVerifier with timeout, or virtual time for scheduled operations

## Related Documentation

- [Project Reactor Reference Guide](https://projectreactor.io/docs/core/release/reference/)
- [Spring WebFlux Documentation](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Spring Data R2DBC Guide](https://spring.io/projects/spring-data-r2dbc)
- [Reactive Programming Tutorial](https://www.baeldung.com/reactor-core)
- [Resilience4j Documentation](https://resilience4j.readme.io/)
