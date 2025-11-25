# Create WebFlux Client

Create a non-blocking HTTP client using Spring WebFlux's `WebClient`. This client enables asynchronous, non-blocking communication with external REST APIs.

## Requirements

Your WebFlux client should include:

- **WebClient Bean Configuration**: Static or instance method to create/configure WebClient
- **Base URL Management**: Configurable base URL for API endpoints
- **HTTP Methods**: GET, POST, PUT, DELETE operations returning Mono/Flux
- **Request Building**: Proper headers, body, query parameters, path variables
- **Response Handling**: Extract response body, headers, and status codes
- **Error Handling**: Handle HTTP errors (4xx, 5xx) and network errors gracefully
- **Timeout Configuration**: Set request timeouts to prevent hanging connections
- **Retry Logic**: Implement exponential backoff retry for transient failures
- **Logging**: Request/response logging for debugging
- **Testing**: Examples using MockWebServer or WebTestClient
- **Java 17 Features**: Use records for request/response DTOs

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot, Spring WebFlux, Project Reactor
- **Build Tool**: Maven with spring-boot-starter-webflux dependency
- **Configuration**: Use `@Configuration` for WebClient bean definition
- **Dependency Injection**: Inject WebClient via constructor or field injection
- **Naming**: Descriptive method names (e.g., `getUser()`, `createOrder()`)

## Template Example

```java
import org.springframework.stereotype.Component;
import org.springframework.web.reactive.function.client.WebClient;
import org.springframework.web.reactive.function.client.WebClientResponseException;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import reactor.util.retry.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Duration;
import java.util.List;

/**
 * Non-blocking HTTP client for external payment service integration.
 * Uses Spring WebFlux WebClient for reactive, non-blocking operations.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentServiceClient {

    private final WebClient webClient;
    private static final String BASE_URL = "/api/v1/payments";
    private static final int TIMEOUT_SECONDS = 10;
    private static final int MAX_RETRIES = 3;

    /**
     * Initiates a payment request asynchronously.
     */
    public Mono<PaymentResponse> initiatePayment(PaymentRequest request) {
        log.info("Initiating payment for order: {}", request.orderId());

        return webClient.post()
                .uri(BASE_URL + "/initiate")
                .header("X-Request-ID", generateRequestId())
                .bodyValue(request)
                .retrieve()
                .onStatus(
                        status -> status.is4xxClientError(),
                        response -> handleClientError(response))
                .onStatus(
                        status -> status.is5xxServerError(),
                        response -> handleServerError(response))
                .bodyToMono(PaymentResponse.class)
                .timeout(Duration.ofSeconds(TIMEOUT_SECONDS))
                .retryWhen(Retry.backoff(MAX_RETRIES, Duration.ofMillis(100))
                        .filter(throwable -> isTransient(throwable))
                        .onRetryExhaustedThrow((retrySignal, throwable) ->
                                new PaymentException("Payment initiation failed after retries", throwable)))
                .doOnSuccess(response -> log.info("Payment initiated: {}", response.transactionId()))
                .doOnError(error -> log.error("Failed to initiate payment", error));
    }

    /**
     * Retrieves payment status by transaction ID.
     */
    public Mono<PaymentStatusResponse> getPaymentStatus(String transactionId) {
        log.debug("Fetching payment status for transaction: {}", transactionId);

        return webClient.get()
                .uri(BASE_URL + "/status/{id}", transactionId)
                .retrieve()
                .bodyToMono(PaymentStatusResponse.class)
                .timeout(Duration.ofSeconds(TIMEOUT_SECONDS))
                .retryWhen(Retry.backoff(2, Duration.ofMillis(50)))
                .doOnSuccess(response -> log.debug("Payment status: {}", response.status()))
                .doOnError(error -> log.error("Error fetching payment status", error));
    }

    /**
     * Confirms a payment that was previously authorized.
     */
    public Mono<PaymentResponse> confirmPayment(String transactionId, ConfirmPaymentRequest request) {
        log.info("Confirming payment: {}", transactionId);

        return webClient.put()
                .uri(BASE_URL + "/confirm/{id}", transactionId)
                .bodyValue(request)
                .retrieve()
                .onStatus(
                        status -> status.is4xxClientError(),
                        response -> handleClientError(response))
                .onStatus(
                        status -> status.is5xxServerError(),
                        response -> handleServerError(response))
                .bodyToMono(PaymentResponse.class)
                .timeout(Duration.ofSeconds(TIMEOUT_SECONDS))
                .retryWhen(Retry.backoff(MAX_RETRIES, Duration.ofMillis(100)))
                .doOnSuccess(response -> log.info("Payment confirmed: {}", response.transactionId()))
                .doOnError(error -> log.error("Failed to confirm payment", error));
    }

    /**
     * Cancels a pending payment.
     */
    public Mono<Void> cancelPayment(String transactionId, CancelPaymentRequest request) {
        log.info("Cancelling payment: {}", transactionId);

        return webClient.delete()
                .uri(BASE_URL + "/cancel/{id}", transactionId)
                .bodyValue(request)
                .retrieve()
                .onStatus(
                        status -> status.is4xxClientError(),
                        response -> handleClientError(response))
                .toBodilessEntity()
                .then()
                .timeout(Duration.ofSeconds(TIMEOUT_SECONDS))
                .doOnSuccess(v -> log.info("Payment cancelled: {}", transactionId))
                .doOnError(error -> log.error("Failed to cancel payment", error));
    }

    /**
     * Requests a refund for a completed transaction.
     */
    public Mono<RefundResponse> requestRefund(String transactionId, RefundRequest request) {
        log.info("Requesting refund for transaction: {}", transactionId);

        return webClient.post()
                .uri(BASE_URL + "/{id}/refund", transactionId)
                .bodyValue(request)
                .retrieve()
                .bodyToMono(RefundResponse.class)
                .timeout(Duration.ofSeconds(TIMEOUT_SECONDS))
                .retryWhen(Retry.backoff(2, Duration.ofMillis(100)))
                .doOnSuccess(response -> log.info("Refund initiated: {}", response.refundId()))
                .doOnError(error -> log.error("Failed to request refund", error));
    }

    /**
     * Streams payment events from webhook.
     */
    public Flux<PaymentEvent> streamPaymentEvents(String orderId) {
        log.info("Streaming payment events for order: {}", orderId);

        return webClient.get()
                .uri(BASE_URL + "/events/stream?orderId={orderId}", orderId)
                .retrieve()
                .bodyToFlux(PaymentEvent.class)
                .timeout(Duration.ofMinutes(5))
                .doOnNext(event -> log.debug("Received payment event: {}", event.type()))
                .doOnError(error -> log.error("Error streaming payment events", error))
                .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
    }

    /**
     * Batch processes multiple payments.
     */
    public Flux<PaymentResponse> batchPayments(Flux<PaymentRequest> requests) {
        log.info("Processing batch of payments");

        return requests
                .flatMap(request ->
                        initiatePayment(request)
                                .onErrorResume(PaymentException.class, e -> {
                                    log.warn("Skipping failed payment for order: {}", request.orderId());
                                    return Mono.empty();
                                }),
                        5)  // Concurrency of 5
                .doOnComplete(() -> log.info("Batch processing completed"))
                .doOnError(error -> log.error("Batch processing failed", error));
    }

    /**
     * Queries payment history with pagination.
     */
    public Flux<PaymentRecord> getPaymentHistory(String orderId, int page, int size) {
        log.info("Fetching payment history for order: {} page: {} size: {}", orderId, page, size);

        return webClient.get()
                .uri(uriBuilder -> uriBuilder
                        .path(BASE_URL + "/history/{orderId}")
                        .queryParam("page", page)
                        .queryParam("size", size)
                        .build(orderId))
                .retrieve()
                .bodyToFlux(PaymentRecord.class)
                .timeout(Duration.ofSeconds(TIMEOUT_SECONDS))
                .retryWhen(Retry.backoff(2, Duration.ofMillis(100)))
                .doOnError(error -> log.error("Error fetching payment history", error));
    }

    // Error handling methods
    private Mono<? extends Throwable> handleClientError(
            org.springframework.web.reactive.function.client.ClientResponse response) {
        return response.bodyToMono(ErrorResponse.class)
                .flatMap(errorBody -> {
                    log.warn("Client error [{}]: {}", response.statusCode(), errorBody.message());
                    return Mono.error(new PaymentClientException(errorBody.message(), response.statusCode()));
                });
    }

    private Mono<? extends Throwable> handleServerError(
            org.springframework.web.reactive.function.client.ClientResponse response) {
        log.error("Server error [{}]", response.statusCode());
        return Mono.error(new PaymentServerException("External service error", response.statusCode()));
    }

    private boolean isTransient(Throwable throwable) {
        return throwable instanceof java.net.ConnectException ||
               throwable instanceof java.util.concurrent.TimeoutException ||
               throwable instanceof PaymentServerException;
    }

    private String generateRequestId() {
        return java.util.UUID.randomUUID().toString();
    }

    // DTOs
    public record PaymentRequest(
            Long orderId,
            Double amount,
            String currency,
            String paymentMethod) {}

    public record PaymentResponse(
            String transactionId,
            String status,
            Double amount,
            Long timestamp) {}

    public record PaymentStatusResponse(
            String transactionId,
            String status,
            String reason) {}

    public record ConfirmPaymentRequest(
            String authorizationCode) {}

    public record CancelPaymentRequest(
            String reason) {}

    public record RefundRequest(
            Double amount,
            String reason) {}

    public record RefundResponse(
            String refundId,
            String transactionId,
            Double amount,
            String status) {}

    public record PaymentEvent(
            String type,
            String transactionId,
            Long timestamp) {}

    public record PaymentRecord(
            String transactionId,
            Double amount,
            String status,
            Long date) {}

    public record ErrorResponse(
            String code,
            String message) {}

    // Custom exceptions
    public static class PaymentException extends RuntimeException {
        public PaymentException(String message, Throwable cause) {
            super(message, cause);
        }
    }

    public static class PaymentClientException extends RuntimeException {
        private final org.springframework.http.HttpStatusCode statusCode;

        public PaymentClientException(String message, org.springframework.http.HttpStatusCode statusCode) {
            super(message);
            this.statusCode = statusCode;
        }

        public org.springframework.http.HttpStatusCode getStatusCode() {
            return statusCode;
        }
    }

    public static class PaymentServerException extends RuntimeException {
        private final org.springframework.http.HttpStatusCode statusCode;

        public PaymentServerException(String message, org.springframework.http.HttpStatusCode statusCode) {
            super(message);
            this.statusCode = statusCode;
        }

        public org.springframework.http.HttpStatusCode getStatusCode() {
            return statusCode;
        }
    }
}
```

## Configuration Example

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient paymentWebClient(WebClient.Builder builder) {
        return builder
                .baseUrl("https://payment-service.example.com/api/v1")
                .defaultHeader("Content-Type", "application/json")
                .defaultHeader("Accept", "application/json")
                .defaultHeader("User-Agent", "MyApp/1.0")
                .filter(ExchangeFilterFunction.ofRequestProcessor(request -> {
                    log.debug("Request: {} {}", request.getMethod(), request.getURL());
                    return Mono.just(request);
                }))
                .filter(ExchangeFilterFunction.ofResponseProcessor(response -> {
                    log.debug("Response: {}", response.getStatusCode());
                    return Mono.just(response);
                }))
                .build();
    }
}
```

## Key Patterns

### 1. Basic Request-Response
```java
webClient.get()
    .uri("/endpoint/{id}", id)
    .retrieve()
    .bodyToMono(Response.class);
```

### 2. Error Handling
```java
.retrieve()
.onStatus(status -> status.is4xxClientError(), response -> handleError(response))
.bodyToMono(Response.class);
```

### 3. Retry with Backoff
```java
.retryWhen(Retry.backoff(3, Duration.ofMillis(100))
    .filter(throwable -> isRetryable(throwable)));
```

### 4. Streaming
```java
webClient.get()
    .uri("/stream")
    .retrieve()
    .bodyToFlux(Event.class);
```

## Testing

```java
@Test
void testInitiatePayment() {
    PaymentResponse response = new PaymentResponse("123", "SUCCESS", 100.0, System.currentTimeMillis());

    webClient.post()
        .uri("/payments/initiate")
        .bodyValue(request)
        .exchange()
        .expectStatus().isOk()
        .expectBody(PaymentResponse.class)
        .isEqualTo(response);
}
```

## Related Documentation

- [Spring WebClient Guide](https://docs.spring.io/spring-framework/reference/web/webflux-client.html)
- [Reactor Retry Patterns](https://projectreactor.io/docs/core/release/reference/#faq.exponentialBackoff)
