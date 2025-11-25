# Create Reactive Test

Create comprehensive unit and integration tests for reactive code using StepVerifier, TestPublisher, and other Reactor testing utilities.

## Requirements

Your reactive tests should include:

- **StepVerifier**: Test Mono/Flux emissions with expectations
- **Virtual Time**: Use virtual time for testing time-dependent operations without delays
- **Expectations**: Assert on emitted values, errors, and completion
- **TestPublisher**: Simulate publishers for testing operators
- **Scenarios**: Happy path, error conditions, timeout, and cancellation
- **Mocking**: Mock reactive dependencies with mockito-reactor
- **Integration Tests**: Test with real database or containers
- **Performance Tests**: Measure reactive chain performance
- **Concurrency Tests**: Verify thread safety and scheduler behavior
- **Documentation**: Clear test descriptions and setup

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: JUnit 5, Reactor Test, Mockito, TestContainers
- **Annotations**: Use `@Test`, `@DisplayName`, `@ExtendWith`
- **Naming**: Test method names describe the scenario being tested

## Template Example

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.boot.test.context.SpringBootTest;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import reactor.test.StepVerifier;
import reactor.test.publisher.TestPublisher;
import reactor.test.scheduler.VirtualTimeScheduler;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;
import java.time.Duration;

/**
 * Comprehensive tests for reactive business logic using Reactor Test utilities.
 */
@ExtendWith(MockitoExtension.class)
@DisplayName("Order Service Reactive Tests")
class OrderServiceReactiveTest {

    @Mock
    private OrderRepository orderRepository;

    @Mock
    private PaymentService paymentService;

    private OrderService orderService;

    @BeforeEach
    void setup() {
        orderService = new OrderService(orderRepository, paymentService);
    }

    // ===================== MONO TESTS =====================

    @Test
    @DisplayName("should retrieve order successfully")
    void testGetOrderSuccess() {
        // Arrange
        Order expectedOrder = new Order(1L, "TEST-ORDER", OrderStatus.PENDING);
        when(orderRepository.findById(1L))
                .thenReturn(Mono.just(expectedOrder));

        // Act & Assert
        StepVerifier.create(orderService.getOrder(1L))
                .expectNext(expectedOrder)
                .expectComplete()
                .verify(Duration.ofSeconds(5));

        // Verify interaction
        verify(orderRepository).findById(1L);
    }

    @Test
    @DisplayName("should handle not found error")
    void testGetOrderNotFound() {
        // Arrange
        when(orderRepository.findById(999L))
                .thenReturn(Mono.empty());

        // Act & Assert
        StepVerifier.create(orderService.getOrder(999L))
                .expectError(OrderNotFoundException.class)
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should retry on transient failure")
    void testGetOrderWithRetry() {
        // Arrange - fail once, then succeed
        when(orderRepository.findById(1L))
                .thenReturn(Mono.error(new RuntimeException("Transient error")))
                .thenReturn(Mono.just(new Order(1L, "TEST", OrderStatus.PENDING)));

        // Act & Assert
        StepVerifier.create(orderService.getOrderWithRetry(1L))
                .expectNextMatches(order -> order.id().equals(1L))
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should handle timeout")
    void testGetOrderTimeout() {
        // Arrange - simulate slow operation
        when(orderRepository.findById(1L))
                .thenReturn(Mono.just(new Order(1L, "TEST", OrderStatus.PENDING))
                        .delayElement(Duration.ofSeconds(10)));

        // Act & Assert
        StepVerifier.create(orderService.getOrderWithTimeout(1L, 1))
                .expectError(java.util.concurrent.TimeoutException.class)
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should use fallback value on error")
    void testGetOrderWithFallback() {
        // Arrange
        when(orderRepository.findById(1L))
                .thenReturn(Mono.error(new DatabaseException("Connection failed")));

        // Act & Assert
        StepVerifier.create(orderService.getOrderWithFallback(1L))
                .expectNextMatches(order -> order.id().equals(-1L))  // fallback id
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should transform error to domain exception")
    void testGetOrderErrorTransformation() {
        // Arrange
        when(orderRepository.findById(1L))
                .thenReturn(Mono.error(new DatabaseException("DB error")));

        // Act & Assert
        StepVerifier.create(orderService.getOrder(1L))
                .expectErrorSatisfies(error -> {
                    assertThat(error).isInstanceOf(OrderException.class);
                    assertThat(error.getMessage()).contains("Failed to retrieve");
                })
                .verify(Duration.ofSeconds(5));
    }

    // ===================== FLUX TESTS =====================

    @Test
    @DisplayName("should retrieve all orders successfully")
    void testGetAllOrdersSuccess() {
        // Arrange
        Order order1 = new Order(1L, "ORDER-1", OrderStatus.PENDING);
        Order order2 = new Order(2L, "ORDER-2", OrderStatus.SHIPPED);
        when(orderRepository.findAll())
                .thenReturn(Flux.just(order1, order2));

        // Act & Assert
        StepVerifier.create(orderService.getAllOrders())
                .expectNext(order1)
                .expectNext(order2)
                .expectComplete()
                .verify(Duration.ofSeconds(5));

        verify(orderRepository).findAll();
    }

    @Test
    @DisplayName("should emit multiple orders and complete")
    void testGetOrdersByStatusEmission() {
        // Arrange
        Flux<Order> orders = Flux.just(
                new Order(1L, "ORDER-1", OrderStatus.PENDING),
                new Order(2L, "ORDER-2", OrderStatus.PENDING),
                new Order(3L, "ORDER-3", OrderStatus.PENDING)
        );
        when(orderRepository.findByStatus(OrderStatus.PENDING))
                .thenReturn(orders);

        // Act & Assert
        StepVerifier.create(orderService.getOrdersByStatus(OrderStatus.PENDING))
                .expectNextCount(3)
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should handle empty flux")
    void testGetOrdersByStatusEmpty() {
        // Arrange
        when(orderRepository.findByStatus(OrderStatus.CANCELLED))
                .thenReturn(Flux.empty());

        // Act & Assert
        StepVerifier.create(orderService.getOrdersByStatus(OrderStatus.CANCELLED))
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should filter orders by criteria")
    void testFilterOrdersByAmount() {
        // Arrange
        Flux<Order> orders = Flux.just(
                new Order(1L, "ORDER-1", OrderStatus.PENDING, 50.0),
                new Order(2L, "ORDER-2", OrderStatus.PENDING, 150.0),
                new Order(3L, "ORDER-3", OrderStatus.PENDING, 250.0)
        );

        // Act & Assert
        StepVerifier.create(
                orders.filter(order -> order.amount() >= 100.0))
                .expectNextMatches(order -> order.amount() >= 100.0)
                .expectNextMatches(order -> order.amount() >= 100.0)
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should buffer items correctly")
    void testBufferOrders() {
        // Arrange
        Flux<Order> orders = Flux.range(1, 10)
                .map(i -> new Order((long) i, "ORDER-" + i, OrderStatus.PENDING));

        // Act & Assert
        StepVerifier.create(
                orders.buffer(3)
                        .map(java.util.List::size))
                .expectNext(3, 3, 3, 1)  // 3 + 3 + 3 + 1 = 10 items in 4 batches
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    // ===================== INTEGRATION TESTS =====================

    @Test
    @DisplayName("should create order with payment successfully")
    void testCreateOrderIntegration() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest(1L, 100.0);
        Order savedOrder = new Order(1L, "NEW-ORDER", OrderStatus.PENDING);

        when(paymentService.process(request.amount()))
                .thenReturn(Mono.just("PAYMENT-ID"));
        when(orderRepository.save(any(Order.class)))
                .thenReturn(Mono.just(savedOrder));

        // Act & Assert
        StepVerifier.create(orderService.createOrder(request))
                .expectNext(savedOrder)
                .expectComplete()
                .verify(Duration.ofSeconds(5));

        verify(paymentService).process(100.0);
        verify(orderRepository).save(any(Order.class));
    }

    @Test
    @DisplayName("should rollback on payment failure")
    void testCreateOrderPaymentFailure() {
        // Arrange
        CreateOrderRequest request = new CreateOrderRequest(1L, 100.0);
        when(paymentService.process(request.amount()))
                .thenReturn(Mono.error(new PaymentException("Insufficient funds")));

        // Act & Assert
        StepVerifier.create(orderService.createOrder(request))
                .expectError(PaymentException.class)
                .verify(Duration.ofSeconds(5));

        verify(orderRepository, never()).save(any());
    }

    // ===================== VIRTUAL TIME TESTS =====================

    @Test
    @DisplayName("should delay emission correctly with virtual time")
    void testDelayedEmissionWithVirtualTime() {
        // Act & Assert
        StepVerifier.withVirtualTime(() ->
                Mono.just(new Order(1L, "TEST", OrderStatus.PENDING))
                        .delayElement(Duration.ofSeconds(5))
        )
        .thenAwait(Duration.ofSeconds(5))
        .expectNextMatches(order -> order.id().equals(1L))
        .expectComplete()
        .verify();
    }

    @Test
    @DisplayName("should emit items at scheduled intervals")
    void testScheduledEmissionWithVirtualTime() {
        // Act & Assert
        StepVerifier.withVirtualTime(() ->
                Flux.interval(Duration.ofSeconds(1))
                        .take(3)
        )
        .thenAwait(Duration.ofSeconds(3))
        .expectNextCount(3)
        .expectComplete()
        .verify();
    }

    // ===================== TEST PUBLISHER TESTS =====================

    @Test
    @DisplayName("should handle TestPublisher emission")
    void testPublisherEmission() {
        // Arrange
        TestPublisher<Order> testPublisher = TestPublisher.create();

        // Act
        StepVerifier.create(testPublisher.flux())
                .then(() -> testPublisher.next(new Order(1L, "TEST-1", OrderStatus.PENDING)))
                .expectNextMatches(order -> order.id().equals(1L))
                .then(() -> testPublisher.next(new Order(2L, "TEST-2", OrderStatus.SHIPPED)))
                .expectNextMatches(order -> order.id().equals(2L))
                .then(testPublisher::complete)
                .expectComplete()
                .verify(Duration.ofSeconds(5));
    }

    @Test
    @DisplayName("should handle TestPublisher error")
    void testPublisherError() {
        // Arrange
        TestPublisher<Order> testPublisher = TestPublisher.create();

        // Act & Assert
        StepVerifier.create(testPublisher.flux())
                .then(() -> testPublisher.error(new OrderException("Test error")))
                .expectError(OrderException.class)
                .verify(Duration.ofSeconds(5));
    }

    // ===================== HELPER CLASSES =====================

    private record CreateOrderRequest(Long userId, Double amount) {}
    private record Order(Long id, String orderNumber, OrderStatus status) {
        Order(Long id, String orderNumber, OrderStatus status, Double amount) {
            this(id, orderNumber, status);
            this.amount = amount;
        }
        private Double amount = 0.0;
    }
    private enum OrderStatus {
        PENDING, SHIPPED, DELIVERED, CANCELLED
    }

    private static class OrderException extends RuntimeException {
        OrderException(String message) { super(message); }
    }
    private static class OrderNotFoundException extends OrderException {
        OrderNotFoundException(String message) { super(message); }
    }
    private static class DatabaseException extends RuntimeException {}
    private static class PaymentException extends RuntimeException {
        PaymentException(String message) { super(message); }
    }

    interface OrderRepository {
        Mono<Order> findById(Long id);
        Flux<Order> findAll();
        Flux<Order> findByStatus(OrderStatus status);
        Mono<Order> save(Order order);
    }

    interface PaymentService {
        Mono<String> process(Double amount);
    }
}
```

## Common Testing Patterns

### StepVerifier Patterns

```java
// Test with expectations
StepVerifier.create(mono)
    .expectNext(expectedValue)
    .expectComplete()
    .verify();

// Test error handling
StepVerifier.create(mono)
    .expectError(SpecificException.class)
    .verify();

// Test with conditions
StepVerifier.create(flux)
    .expectNextMatches(item -> item.status() == "ACTIVE")
    .expectComplete()
    .verify();
```

### Virtual Time

```java
// Test time-based operations without delays
StepVerifier.withVirtualTime(() ->
    Mono.just(value).delayElement(Duration.ofSeconds(5))
)
.thenAwait(Duration.ofSeconds(5))
.expectNext(value)
.expectComplete()
.verify();
```

### TestPublisher

```java
// Simulate publisher behavior
TestPublisher<Item> publisher = TestPublisher.create();
StepVerifier.create(publisher.flux())
    .then(() -> publisher.next(item1))
    .expectNext(item1)
    .then(publisher::complete)
    .expectComplete()
    .verify();
```

## Related Documentation

- [Reactor Test Documentation](https://projectreactor.io/docs/core/release/reference/#testing)
- [StepVerifier API](https://projectreactor.io/docs/core/release/api/reactor/test/StepVerifier.html)
- [JUnit 5 Guide](https://junit.org/junit5/docs/current/user-guide/)
