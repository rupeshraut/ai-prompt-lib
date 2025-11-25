# Create Reactive Controller

Create a new Spring WebFlux REST controller using reactive endpoints. Reactive controllers handle HTTP requests asynchronously and return Mono/Flux publishers instead of blocking responses.

## Requirements

Your reactive controller should include:

- **Class Declaration**: Public controller class with `@RestController` and `@RequestMapping` annotations
- **Reactive Endpoints**: HTTP methods that return `Mono<ResponseEntity<T>>` or `Mono<T>` or `Flux<T>`
- **HTTP Methods**: Implement GET, POST, PUT, DELETE, PATCH with proper status codes
- **Request Validation**: Use `@Valid` and validation annotations on request DTOs
- **Exception Handling**: Handle validation errors and business exceptions gracefully
- **Response Types**: Return appropriate HTTP status codes (200, 201, 204, 400, 404, 500)
- **Pagination**: Support pagination for list endpoints with Pageable or custom parameters
- **Path Variables & Query Params**: Properly annotate with `@PathVariable`, `@RequestParam`, `@RequestBody`
- **Content Negotiation**: Support JSON responses with proper media types
- **Documentation**: Include Swagger/OpenAPI comments or JavaDoc for API endpoints
- **Java 17 Features**: Use records for DTOs, sealed classes where applicable

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Spring WebFlux, Project Reactor
- **Annotations**: `@RestController`, `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`
- **Naming Conventions**:
  - Controller class suffix with `Controller` (e.g., `OrderController`, `UserController`)
  - Method names describe the operation (e.g., `getOrder()`, `createOrder()`, `updateOrder()`)
- **Response Wrapping**: Use `ResponseEntity` for fine-grained control over HTTP responses
- **Error Handling**: Centralized error handling via `@ExceptionHandler` methods
- **Logging**: Structured logging for request/response tracking

## Template Example

```java
import org.springframework.web.bind.annotation.*;
import org.springframework.http.ResponseEntity;
import org.springframework.http.HttpStatus;
import org.springframework.data.domain.Pageable;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import jakarta.validation.Valid;
import java.net.URI;
import java.time.LocalDateTime;

/**
 * REST API controller for order management using reactive endpoints.
 * All endpoints use Project Reactor Mono/Flux for non-blocking operations.
 */
@Slf4j
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;
    private final OrderMapper mapper;

    /**
     * GET /api/v1/orders/{id}
     * Retrieves a single order by ID.
     */
    @GetMapping("/{id}")
    public Mono<ResponseEntity<OrderResponse>> getOrder(@PathVariable Long id) {
        log.info("GET /api/v1/orders/{}", id);

        return orderService.getOrderById(id)
                .map(order -> ResponseEntity.ok(mapper.toResponse(order)))
                .onErrorResume(OrderNotFoundException.class, e ->
                        Mono.just(ResponseEntity.notFound().build()))
                .doOnError(error -> log.error("Error fetching order {}: {}", id, error.getMessage()));
    }

    /**
     * GET /api/v1/orders?page=0&size=20&status=PENDING
     * Retrieves paginated list of orders with filtering.
     */
    @GetMapping
    public Flux<OrderResponse> listOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(required = false) String status) {

        log.info("GET /api/v1/orders?page={}&size={}&status={}", page, size, status);

        OrderSearchCriteria criteria = new OrderSearchCriteria(
                status != null ? OrderStatus.valueOf(status) : null,
                null,
                null
        );

        return orderService.searchOrders(criteria)
                .skip((long) page * size)
                .take(size)
                .map(mapper::toResponse)
                .doOnError(error -> log.error("Error listing orders", error));
    }

    /**
     * GET /api/v1/orders/user/{userId}
     * Retrieves all orders for a specific user.
     */
    @GetMapping("/user/{userId}")
    public Flux<OrderResponse> getUserOrders(@PathVariable Long userId) {
        log.info("GET /api/v1/orders/user/{}", userId);

        return orderService.userOrders(userId)
                .map(mapper::toResponse)
                .doOnError(error -> log.error("Error fetching user orders", error));
    }

    /**
     * POST /api/v1/orders
     * Creates a new order.
     * Status: 201 Created with Location header
     */
    @PostMapping
    public Mono<ResponseEntity<OrderResponse>> createOrder(@Valid @RequestBody CreateOrderRequest request) {
        log.info("POST /api/v1/orders - Creating order for user: {}", request.userId());

        return orderService.createOrder(request)
                .map(order -> ResponseEntity
                        .created(URI.create("/api/v1/orders/" + order.id()))
                        .body(order))
                .onErrorResume(ValidationException.class, e ->
                        Mono.just(ResponseEntity.badRequest().build()))
                .onErrorResume(PaymentFailedException.class, e ->
                        Mono.just(ResponseEntity.status(HttpStatus.PAYMENT_REQUIRED).build()))
                .onErrorResume(UserNotFoundException.class, e ->
                        Mono.just(ResponseEntity.notFound().build()))
                .onErrorResume(Exception.class, e -> {
                    log.error("Unexpected error creating order", e);
                    return Mono.just(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build());
                })
                .doOnSuccess(response -> log.info("Order created with status: {}", response.getStatusCode()));
    }

    /**
     * PUT /api/v1/orders/{id}
     * Updates an existing order.
     */
    @PutMapping("/{id}")
    public Mono<ResponseEntity<OrderResponse>> updateOrder(
            @PathVariable Long id,
            @Valid @RequestBody UpdateOrderRequest request) {

        log.info("PUT /api/v1/orders/{}", id);

        return orderService.updateOrder(id, request)
                .map(order -> ResponseEntity.ok(mapper.toResponse(order)))
                .onErrorResume(OrderNotFoundException.class, e ->
                        Mono.just(ResponseEntity.notFound().build()))
                .onErrorResume(BusinessException.class, e ->
                        Mono.just(ResponseEntity.status(HttpStatus.CONFLICT).build()))
                .doOnError(error -> log.error("Error updating order {}", id, error));
    }

    /**
     * DELETE /api/v1/orders/{id}
     * Cancels an order with optional reason.
     */
    @DeleteMapping("/{id}")
    public Mono<ResponseEntity<Void>> cancelOrder(
            @PathVariable Long id,
            @RequestParam(defaultValue = "Customer request") String reason) {

        log.info("DELETE /api/v1/orders/{}", id);

        return orderService.cancelOrder(id, reason)
                .then(Mono.just(ResponseEntity.noContent().<Void>build()))
                .onErrorResume(OrderNotFoundException.class, e ->
                        Mono.just(ResponseEntity.notFound().build()))
                .onErrorResume(BusinessException.class, e ->
                        Mono.just(ResponseEntity.status(HttpStatus.CONFLICT).build()))
                .doOnSuccess(response -> log.info("Order {} cancelled", id))
                .doOnError(error -> log.error("Error cancelling order {}", id, error));
    }

    /**
     * GET /api/v1/orders/{id}/status
     * Gets the current status of an order.
     */
    @GetMapping("/{id}/status")
    public Mono<ResponseEntity<OrderStatusResponse>> getOrderStatus(@PathVariable Long id) {
        log.info("GET /api/v1/orders/{}/status", id);

        return orderService.getOrderById(id)
                .map(order -> new OrderStatusResponse(order.id(), order.status().name()))
                .map(ResponseEntity::ok)
                .onErrorResume(OrderNotFoundException.class, e ->
                        Mono.just(ResponseEntity.notFound().build()))
                .doOnError(error -> log.error("Error getting order status", error));
    }

    /**
     * PATCH /api/v1/orders/{id}/status
     * Updates the status of an order (admin only).
     */
    @PatchMapping("/{id}/status")
    public Mono<ResponseEntity<OrderResponse>> updateOrderStatus(
            @PathVariable Long id,
            @RequestBody UpdateOrderStatusRequest request) {

        log.info("PATCH /api/v1/orders/{}/status - New status: {}", id, request.status());

        return orderService.updateOrderStatus(id, request.status())
                .map(order -> ResponseEntity.ok(mapper.toResponse(order)))
                .onErrorResume(OrderNotFoundException.class, e ->
                        Mono.just(ResponseEntity.notFound().build()))
                .onErrorResume(IllegalArgumentException.class, e ->
                        Mono.just(ResponseEntity.badRequest().build()))
                .doOnError(error -> log.error("Error updating order status", error));
    }

    /**
     * GET /api/v1/orders/statistics?userId={userId}
     * Retrieves order statistics for a user.
     */
    @GetMapping("/statistics")
    public Mono<ResponseEntity<OrderStatistics>> getStatistics(@RequestParam Long userId) {
        log.info("GET /api/v1/orders/statistics?userId={}", userId);

        return orderService.getOrderStatistics(userId)
                .map(ResponseEntity::ok)
                .onErrorResume(Exception.class, e ->
                        Mono.just(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build()))
                .doOnError(error -> log.error("Error fetching statistics", error));
    }

    /**
     * GET /api/v1/orders/search?minAmount=100&maxAmount=500&status=DELIVERED
     * Advanced search with multiple criteria.
     */
    @GetMapping("/search")
    public Flux<OrderResponse> searchOrders(
            @RequestParam(required = false) Double minAmount,
            @RequestParam(required = false) Double maxAmount,
            @RequestParam(required = false) String status) {

        log.info("GET /api/v1/orders/search");

        OrderSearchCriteria criteria = new OrderSearchCriteria(
                status != null ? OrderStatus.valueOf(status) : null,
                minAmount,
                maxAmount
        );

        return orderService.searchOrders(criteria)
                .map(mapper::toResponse)
                .doOnError(error -> log.error("Error searching orders", error));
    }

    // Exception handlers
    @ExceptionHandler(ValidationException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleValidationException(
            ValidationException ex) {
        log.warn("Validation error: {}", ex.getMessage());
        return Mono.just(ResponseEntity
                .badRequest()
                .body(new ErrorResponse("VALIDATION_ERROR", ex.getMessage(), LocalDateTime.now())));
    }

    @ExceptionHandler(OrderNotFoundException.class)
    public Mono<ResponseEntity<ErrorResponse>> handleOrderNotFound(
            OrderNotFoundException ex) {
        log.warn("Order not found: {}", ex.getMessage());
        return Mono.just(ResponseEntity
                .notFound()
                .build());
    }

    @ExceptionHandler(Exception.class)
    public Mono<ResponseEntity<ErrorResponse>> handleGenericException(
            Exception ex) {
        log.error("Unexpected error", ex);
        return Mono.just(ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred", LocalDateTime.now())));
    }

    // DTOs
    public record OrderResponse(
            Long id,
            Long userId,
            java.util.List<OrderItem> items,
            Double total,
            String status,
            LocalDateTime createdAt) {}

    public record CreateOrderRequest(
            @jakarta.validation.constraints.NotNull
            Long userId,
            java.util.List<OrderItem> items,
            Double total) {}

    public record UpdateOrderRequest(
            java.util.List<OrderItem> items) {}

    public record UpdateOrderStatusRequest(String status) {}

    public record OrderStatusResponse(Long id, String status) {}

    public record OrderSearchCriteria(
            OrderStatus status,
            Double minAmount,
            Double maxAmount) {}

    public record OrderStatistics(
            Long userId,
            int totalOrders,
            long completedOrders,
            Double totalAmount) {}

    public record OrderItem(
            Long productId,
            int quantity,
            Double price) {}

    public record ErrorResponse(
            String code,
            String message,
            LocalDateTime timestamp) {}

    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    }
}
```

## HTTP Status Codes Reference

| Method | Status | Scenario |
|--------|--------|----------|
| GET | 200 | Success |
| GET | 404 | Not found |
| POST | 201 | Created successfully |
| POST | 400 | Validation error |
| PUT | 200 | Updated successfully |
| PUT | 404 | Resource not found |
| DELETE | 204 | Deleted successfully |
| DELETE | 404 | Resource not found |

## Key Patterns

### 1. Reactive Response Mapping
```java
@GetMapping("/{id}")
public Mono<ResponseEntity<OrderResponse>> getOrder(@PathVariable Long id) {
    return service.findById(id)
            .map(ResponseEntity::ok)
            .onErrorResume(NotFoundException.class, e ->
                Mono.just(ResponseEntity.notFound().build()));
}
```

### 2. List with Pagination
```java
@GetMapping
public Flux<OrderResponse> listOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size) {
    return service.all()
            .skip((long) page * size)
            .take(size)
            .map(mapper::toResponse);
}
```

### 3. Create with Location Header
```java
@PostMapping
public Mono<ResponseEntity<OrderResponse>> create(@Valid @RequestBody CreateOrderRequest req) {
    return service.create(req)
            .map(order -> ResponseEntity
                .created(URI.create("/orders/" + order.id()))
                .body(toResponse(order)));
}
```

### 4. Centralized Error Handling
```java
@ExceptionHandler(ValidationException.class)
public Mono<ResponseEntity<ErrorResponse>> handleValidation(ValidationException ex) {
    return Mono.just(ResponseEntity.badRequest()
        .body(new ErrorResponse(ex.getMessage())));
}
```

## Testing Patterns

```java
@Test
void testGetOrder() {
    webTestClient
        .get()
        .uri("/api/v1/orders/1")
        .exchange()
        .expectStatus().isOk()
        .expectBody(OrderResponse.class);
}
```

## Related Documentation

- [Spring WebFlux Guide](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Spring Data Validation](https://spring.io/guides/gs/validating-form-input/)
- [REST API Best Practices](https://restfulapi.net/)
