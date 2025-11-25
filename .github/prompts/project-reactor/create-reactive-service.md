# Create Reactive Service

Create a new business logic service using reactive patterns with Project Reactor and Spring. Reactive services implement business logic asynchronously using Mono and Flux.

## Requirements

Your reactive service should include:

- **Class Declaration**: Public service class with descriptive name using `@Service` annotation
- **Dependency Injection**: Inject repositories, other services, and external clients via constructor
- **Mono/Flux Returns**: All public methods return `Mono<T>` or `Flux<T>` instead of blocking types
- **Business Logic**: Implement domain logic using reactive operators (map, flatMap, filter, etc.)
- **Transaction Management**: Use `@Transactional` where needed for reactive operations
- **Validation**: Include input validation with proper error propagation
- **Error Handling**: Implement comprehensive error recovery and fallback strategies
- **Logging**: Add structured logging at key operation points
- **Composition**: Demonstrate composition of multiple reactive operations
- **Testing**: Include testable design with clear separation of concerns
- **Documentation**: JavaDoc comments explaining reactive flow and side effects

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Spring Data R2DBC or reactive MongoDB, Project Reactor
- **Annotations**: `@Service`, `@Transactional`, `@RequiredArgsConstructor` from Lombok
- **Naming Conventions**:
  - Method names should describe the business operation (e.g., `createOrder()`, `findUserWithOrders()`)
  - Avoid verb prefixes when the method returns Mono/Flux (e.g., `orders()` instead of `getOrders()`)
- **Error Handling**: Use custom exceptions and error strategies appropriate to each operation
- **Performance**: Consider filtering, mapping, and flatMapping order to minimize computational overhead

## Template Example

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import reactor.core.scheduler.Schedulers;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Instant;
import java.time.LocalDateTime;
import java.util.UUID;

/**
 * Reactive business service for order management.
 * Implements non-blocking operations for order creation, retrieval, and updates.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final UserRepository userRepository;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    private final InventoryService inventoryService;

    /**
     * Creates a new order with validation, payment processing, and notifications.
     * Returns a Mono that emits the created order or an error.
     */
    @Transactional
    public Mono<OrderResponse> createOrder(CreateOrderRequest request) {
        log.info("Creating order for user: {}", request.userId());

        return Mono.defer(() -> {
            // Validate request
            if (request.items() == null || request.items().isEmpty()) {
                return Mono.error(new ValidationException("Order must contain at least one item"));
            }

            return userRepository.findById(request.userId())
                    .switchIfEmpty(Mono.error(new UserNotFoundException("User not found")))
                    .flatMap(user -> validateOrderItems(request))
                    .flatMap(validRequest -> processPayment(validRequest))
                    .flatMap(this::persistOrder)
                    .flatMap(order -> notifyOrderCreation(order)
                            .then(Mono.just(order)))
                    .map(this::toOrderResponse)
                    .doOnNext(response -> log.info("Order created: {}", response.id()))
                    .doOnError(error -> log.error("Failed to create order", error));
        })
        .onErrorResume(ValidationException.class, e -> {
            log.warn("Validation failed: {}", e.getMessage());
            return Mono.error(new BadRequestException(e.getMessage()));
        })
        .onErrorResume(PaymentException.class, e -> {
            log.error("Payment processing failed", e);
            return Mono.error(new PaymentFailedException(e.getMessage()));
        })
        .subscribeOn(Schedulers.boundedElastic());
    }

    /**
     * Retrieves orders for a specific user with full details.
     * Returns a Flux that emits each order.
     */
    public Flux<OrderResponse> userOrders(Long userId) {
        log.debug("Fetching orders for user: {}", userId);

        return userRepository.existsById(userId)
                .flatMapMany(exists -> {
                    if (!exists) {
                        return Flux.error(new UserNotFoundException("User not found"));
                    }
                    return orderRepository.findByUserId(userId)
                            .doOnNext(order -> log.debug("Found order: {}", order.id()));
                })
                .map(this::toOrderResponse)
                .doOnError(error -> log.error("Error fetching user orders", error))
                .doOnComplete(() -> log.debug("Completed fetching user orders"))
                .subscribeOn(Schedulers.boundedElastic());
    }

    /**
     * Updates an existing order (only if not yet shipped).
     */
    @Transactional
    public Mono<OrderResponse> updateOrder(Long orderId, UpdateOrderRequest request) {
        log.info("Updating order: {}", orderId);

        return orderRepository.findById(orderId)
                .switchIfEmpty(Mono.error(new OrderNotFoundException("Order not found")))
                .filter(order -> !order.status().equals(OrderStatus.SHIPPED))
                .switchIfEmpty(Mono.error(new BusinessException("Cannot update shipped orders")))
                .flatMap(order -> validateOrderItems(request.items())
                        .map(validItems -> updateOrderItems(order, validItems)))
                .flatMap(orderRepository::save)
                .flatMap(order -> notifyOrderUpdate(order)
                        .then(Mono.just(order)))
                .map(this::toOrderResponse)
                .doOnNext(response -> log.info("Order updated: {}", response.id()))
                .doOnError(error -> log.error("Failed to update order", error));
    }

    /**
     * Cancels an order with refund processing.
     */
    @Transactional
    public Mono<Void> cancelOrder(Long orderId, String reason) {
        log.info("Cancelling order: {} with reason: {}", orderId, reason);

        return orderRepository.findById(orderId)
                .switchIfEmpty(Mono.error(new OrderNotFoundException("Order not found")))
                .filter(order -> canCancel(order))
                .switchIfEmpty(Mono.error(new BusinessException("Order cannot be cancelled")))
                .flatMap(order -> paymentService.refund(order.paymentId())
                        .flatMap(refund -> {
                            order.setCancelledAt(LocalDateTime.now());
                            order.setStatus(OrderStatus.CANCELLED);
                            return orderRepository.save(order);
                        }))
                .flatMap(order -> notifyOrderCancellation(order))
                .then()
                .doOnSuccess(v -> log.info("Order cancelled: {}", orderId))
                .doOnError(error -> log.error("Failed to cancel order", error));
    }

    /**
     * Searches orders by criteria with filtering.
     */
    public Flux<OrderResponse> searchOrders(OrderSearchCriteria criteria) {
        log.info("Searching orders with criteria: {}", criteria);

        return orderRepository.findByStatus(criteria.status())
                .filter(order -> matchesCriteria(order, criteria))
                .doOnNext(order -> log.debug("Matched order: {}", order.id()))
                .map(this::toOrderResponse)
                .doOnError(error -> log.error("Error searching orders", error))
                .subscribeOn(Schedulers.boundedElastic());
    }

    /**
     * Processes multiple orders in batch with error recovery.
     */
    public Flux<OrderResponse> processBatchOrders(Flux<CreateOrderRequest> orders) {
        log.info("Processing batch of orders");

        return orders
                .flatMap(request -> createOrder(request)
                        .onErrorResume(Exception.class, e -> {
                            log.warn("Skipping order due to: {}", e.getMessage());
                            return Flux.empty();
                        }), 4)  // Concurrency of 4
                .doOnComplete(() -> log.info("Batch processing completed"))
                .doOnError(error -> log.error("Batch processing failed", error));
    }

    /**
     * Aggregates order statistics by status.
     */
    public Mono<OrderStatistics> getOrderStatistics(Long userId) {
        log.info("Computing order statistics for user: {}", userId);

        return userOrders(userId)
                .collectList()
                .map(orders -> {
                    int total = orders.size();
                    long completed = orders.stream()
                            .filter(o -> o.id() != null)
                            .count();
                    double totalAmount = orders.stream()
                            .mapToDouble(o -> o.total())
                            .sum();

                    return new OrderStatistics(userId, total, completed, totalAmount);
                })
                .doOnSuccess(stats -> log.info("Statistics computed: {}", stats))
                .doOnError(error -> log.error("Error computing statistics", error));
    }

    /**
     * Monitors orders with custom business logic.
     */
    public Mono<Void> monitorOrderFulfillment(Long orderId) {
        log.info("Starting fulfillment monitoring for order: {}", orderId);

        return orderRepository.findById(orderId)
                .switchIfEmpty(Mono.error(new OrderNotFoundException("Order not found")))
                .flatMap(order -> notifyOrderConfirmation(order)
                        .then(inventoryService.processInventory(orderId))
                        .then(paymentService.capturePayment(order.paymentId()))
                        .doOnSuccess(v -> log.info("Order fulfillment process completed: {}", orderId)))
                .then()
                .doOnError(error -> log.error("Order fulfillment failed", error));
    }

    // Helper methods
    private Mono<CreateOrderRequest> validateOrderItems(CreateOrderRequest request) {
        return Mono.fromCallable(() -> {
            if (request.items() == null || request.items().isEmpty()) {
                throw new ValidationException("Order must contain items");
            }
            request.items().forEach(item -> {
                if (item.quantity() <= 0) {
                    throw new ValidationException("Item quantity must be positive");
                }
            });
            return request;
        });
    }

    private Mono<Void> validateOrderItems(java.util.List<OrderItem> items) {
        return Mono.fromRunnable(() -> {
            if (items == null || items.isEmpty()) {
                throw new ValidationException("Order must contain items");
            }
        });
    }

    private Mono<CreateOrderRequest> processPayment(CreateOrderRequest request) {
        return paymentService.processPayment(request.userId(), request.total())
                .map(payment -> request)
                .onErrorMap(e -> new PaymentException("Payment failed", e));
    }

    private Mono<Order> persistOrder(CreateOrderRequest request) {
        Order order = new Order(
                null,
                request.userId(),
                request.items(),
                request.total(),
                OrderStatus.PENDING,
                LocalDateTime.now(),
                null
        );
        return orderRepository.save(order);
    }

    private Mono<Void> notifyOrderCreation(Order order) {
        return notificationService.sendOrderConfirmation(order.userId(), order.id())
                .doOnError(error -> log.warn("Failed to send creation notification", error))
                .onErrorResume(e -> Mono.empty());
    }

    private Mono<Void> notifyOrderUpdate(Order order) {
        return notificationService.sendOrderUpdated(order.userId(), order.id())
                .onErrorResume(e -> Mono.empty());
    }

    private Mono<Void> notifyOrderCancellation(Order order) {
        return notificationService.sendOrderCancelled(order.userId(), order.id())
                .onErrorResume(e -> Mono.empty());
    }

    private Mono<Void> notifyOrderConfirmation(Order order) {
        return notificationService.sendOrderConfirmed(order.userId(), order.id())
                .onErrorResume(e -> Mono.empty());
    }

    private boolean canCancel(Order order) {
        return !order.status().equals(OrderStatus.SHIPPED) &&
               !order.status().equals(OrderStatus.DELIVERED) &&
               !order.status().equals(OrderStatus.CANCELLED);
    }

    private Order updateOrderItems(Order order, java.util.List<OrderItem> items) {
        order.setItems(items);
        order.setUpdatedAt(LocalDateTime.now());
        return order;
    }

    private boolean matchesCriteria(Order order, OrderSearchCriteria criteria) {
        if (criteria.minAmount() != null && order.total() < criteria.minAmount()) {
            return false;
        }
        if (criteria.maxAmount() != null && order.total() > criteria.maxAmount()) {
            return false;
        }
        return true;
    }

    private OrderResponse toOrderResponse(Order order) {
        return new OrderResponse(
                order.id(),
                order.userId(),
                order.items(),
                order.total(),
                order.status().name(),
                order.createdAt()
        );
    }

    // DTOs and Domain Models
    public record CreateOrderRequest(Long userId, java.util.List<OrderItem> items, Double total) {}
    public record UpdateOrderRequest(java.util.List<OrderItem> items) {}
    public record OrderResponse(Long id, Long userId, java.util.List<OrderItem> items, Double total, String status, LocalDateTime createdAt) {}
    public record OrderSearchCriteria(OrderStatus status, Double minAmount, Double maxAmount) {}
    public record OrderStatistics(Long userId, int totalOrders, long completedOrders, Double totalAmount) {}
    public record OrderItem(Long productId, int quantity, Double price) {}

    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    }

    // Domain Model
    public static class Order {
        private Long id;
        private Long userId;
        private java.util.List<OrderItem> items;
        private Double total;
        private OrderStatus status;
        private LocalDateTime createdAt;
        private LocalDateTime cancelledAt;
        private LocalDateTime updatedAt;
        private String paymentId;

        // Constructor, getters, setters...
        public Order(Long id, Long userId, java.util.List<OrderItem> items, Double total,
                    OrderStatus status, LocalDateTime createdAt, String paymentId) {
            this.id = id;
            this.userId = userId;
            this.items = items;
            this.total = total;
            this.status = status;
            this.createdAt = createdAt;
            this.paymentId = paymentId;
        }

        // Getters and setters omitted for brevity
        public Long id() { return id; }
        public Long userId() { return userId; }
        public java.util.List<OrderItem> items() { return items; }
        public void setItems(java.util.List<OrderItem> items) { this.items = items; }
        public Double total() { return total; }
        public OrderStatus status() { return status; }
        public void setStatus(OrderStatus status) { this.status = status; }
        public LocalDateTime createdAt() { return createdAt; }
        public LocalDateTime cancelledAt() { return cancelledAt; }
        public void setCancelledAt(LocalDateTime cancelledAt) { this.cancelledAt = cancelledAt; }
        public LocalDateTime updatedAt() { return updatedAt; }
        public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
        public String paymentId() { return paymentId; }
    }

    // Custom exceptions
    public static class ValidationException extends RuntimeException {
        public ValidationException(String message) { super(message); }
    }

    public static class UserNotFoundException extends RuntimeException {
        public UserNotFoundException(String message) { super(message); }
    }

    public static class OrderNotFoundException extends RuntimeException {
        public OrderNotFoundException(String message) { super(message); }
    }

    public static class PaymentException extends RuntimeException {
        public PaymentException(String message, Throwable cause) { super(message, cause); }
    }

    public static class PaymentFailedException extends RuntimeException {
        public PaymentFailedException(String message) { super(message); }
    }

    public static class BusinessException extends RuntimeException {
        public BusinessException(String message) { super(message); }
    }

    public static class BadRequestException extends RuntimeException {
        public BadRequestException(String message) { super(message); }
    }
}
```

## Key Patterns

### 1. Reactive Composition
```java
service.getUserWithOrders(userId)
    .flatMap(user -> enrichWithOrders(user))
    .flatMap(enrichedUser -> saveAudit(enrichedUser))
    .map(this::toResponse);
```

### 2. Error Recovery with Fallback
```java
repository.findById(id)
    .onErrorResume(NotFoundException.class, e ->
        cache.getOrDefault(id, Mono.empty()))
    .onErrorReturn(defaultValue);
```

### 3. Concurrent Operations
```java
Mono.zip(
    userService.getUser(userId),
    orderService.getOrders(userId),
    paymentService.getPaymentMethods(userId)
).map(tuple -> combineResults(tuple.getT1(), tuple.getT2(), tuple.getT3()));
```

### 4. Batch Processing
```java
flux
    .flatMap(item -> processItem(item), 4)  // Concurrency limit of 4
    .collectList()
    .flatMap(results -> saveBatch(results));
```

## Testing Patterns

```java
@Test
void testCreateOrder() {
    StepVerifier.create(orderService.createOrder(request))
        .expectNextMatches(response -> response.id() > 0)
        .expectComplete()
        .verify(Duration.ofSeconds(5));
}
```

## Related Documentation

- [Spring Data R2DBC](https://spring.io/projects/spring-data-r2dbc)
- [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Reactor Core](https://projectreactor.io/docs/core/release/reference/)
