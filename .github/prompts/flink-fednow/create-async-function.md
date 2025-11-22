# Create Async Function

Create Flink AsyncFunction for non-blocking HTTP service calls in payment processing.

## Requirements

The async function should include:
- `RichAsyncFunction` implementation
- Non-blocking HTTP client (Java HttpClient or OkHttp)
- Retry logic with exponential backoff
- Timeout handling
- Side output for failures
- Metrics collection
- Proper resource lifecycle (open/close)

## Code Style

- Use Java 17 features (HttpClient, CompletableFuture)
- Use records for request/response DTOs
- Implement proper error handling
- Add structured logging with transaction context

## Async Function Template

```java
package com.example.fednow.function;

import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.async.ResultFuture;
import org.apache.flink.streaming.api.functions.async.RichAsyncFunction;
import org.apache.flink.metrics.Counter;
import org.apache.flink.metrics.Histogram;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.Collections;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;

/**
 * Async function for Account Validation HTTP calls.
 *
 * <p>Performs non-blocking HTTP requests to the Account Validation Service
 * with retry logic and timeout handling.
 */
@Slf4j
public class AccountValidationAsyncFunction
    extends RichAsyncFunction<Payment, Payment> {

    private static final int MAX_RETRIES = 3;
    private static final long INITIAL_BACKOFF_MS = 1000;
    private static final long MAX_BACKOFF_MS = 8000;

    private final String serviceUrl;
    private final long timeoutMs;

    private transient HttpClient httpClient;
    private transient ObjectMapper objectMapper;
    private transient Counter successCounter;
    private transient Counter failureCounter;
    private transient Counter retryCounter;
    private transient Histogram latencyHistogram;

    public AccountValidationAsyncFunction(String serviceUrl, long timeoutMs) {
        this.serviceUrl = serviceUrl;
        this.timeoutMs = timeoutMs;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);

        // Initialize HTTP client
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofMillis(timeoutMs))
            .build();

        this.objectMapper = new ObjectMapper();

        // Register metrics
        this.successCounter = getRuntimeContext()
            .getMetricGroup()
            .counter("accountValidation.success");
        this.failureCounter = getRuntimeContext()
            .getMetricGroup()
            .counter("accountValidation.failure");
        this.retryCounter = getRuntimeContext()
            .getMetricGroup()
            .counter("accountValidation.retry");
        this.latencyHistogram = getRuntimeContext()
            .getMetricGroup()
            .histogram("accountValidation.latency", new DescriptiveStatisticsHistogram(1000));

        log.info("AccountValidationAsyncFunction initialized: url={}", serviceUrl);
    }

    @Override
    public void asyncInvoke(Payment payment, ResultFuture<Payment> resultFuture) {
        long startTime = System.currentTimeMillis();

        log.debug("Starting account validation: transactionId={}, debitAccount={}, creditAccount={}",
            payment.getTransactionId(),
            maskAccount(payment.getDebitAccount()),
            maskAccount(payment.getCreditAccount()));

        // Validate both debit and credit accounts
        CompletableFuture<ValidationResult> debitValidation =
            validateAccount(payment, "DEBIT", payment.getDebitAccount(), 0);

        CompletableFuture<ValidationResult> creditValidation =
            validateAccount(payment, "CREDIT", payment.getCreditAccount(), 0);

        // Combine both validations
        CompletableFuture.allOf(debitValidation, creditValidation)
            .thenAccept(v -> {
                try {
                    ValidationResult debitResult = debitValidation.get();
                    ValidationResult creditResult = creditValidation.get();

                    if (debitResult.isValid() && creditResult.isValid()) {
                        // Both accounts valid
                        payment.setValidationStatus(ValidationStatus.VALID);
                        payment.setDebitValidationRef(debitResult.getReference());
                        payment.setCreditValidationRef(creditResult.getReference());
                        payment.addProcessingStep("ACCOUNT_VALIDATION", "SUCCESS");

                        successCounter.inc();
                        latencyHistogram.update(System.currentTimeMillis() - startTime);

                        log.info("Account validation successful: transactionId={}",
                            payment.getTransactionId());

                        resultFuture.complete(Collections.singleton(payment));
                    } else {
                        // Validation failed
                        handleValidationFailure(payment, debitResult, creditResult, resultFuture);
                    }
                } catch (Exception e) {
                    handleException(payment, e, resultFuture);
                }
            })
            .exceptionally(ex -> {
                handleException(payment, ex, resultFuture);
                return null;
            });
    }

    /**
     * Validate a single account with retry logic.
     */
    private CompletableFuture<ValidationResult> validateAccount(
            Payment payment, String direction, String accountNumber, int retryCount) {

        AccountValidationRequest request = new AccountValidationRequest(
            accountNumber,
            payment.getAccountType(direction),
            payment.getRoutingNumber(direction),
            payment.getAmount(),
            direction
        );

        return sendHttpRequest(request)
            .thenApply(response -> {
                if (response.statusCode() == 200) {
                    return parseSuccessResponse(response.body());
                } else if (response.statusCode() >= 400 && response.statusCode() < 500) {
                    // Client error - don't retry
                    return parseErrorResponse(response.body());
                } else {
                    // Server error - throw for retry
                    throw new RuntimeException("Server error: " + response.statusCode());
                }
            })
            .exceptionally(ex -> {
                if (retryCount < MAX_RETRIES) {
                    retryCounter.inc();
                    long backoff = calculateBackoff(retryCount);

                    log.warn("Account validation retry: transactionId={}, direction={}, attempt={}, backoff={}ms",
                        payment.getTransactionId(), direction, retryCount + 1, backoff);

                    return validateAccountWithDelay(payment, direction, accountNumber, retryCount + 1, backoff)
                        .join();
                } else {
                    log.error("Account validation failed after {} retries: transactionId={}, direction={}",
                        MAX_RETRIES, payment.getTransactionId(), direction, ex);
                    return ValidationResult.failed("MAX_RETRIES_EXCEEDED", ex.getMessage());
                }
            });
    }

    /**
     * Retry with delay.
     */
    private CompletableFuture<ValidationResult> validateAccountWithDelay(
            Payment payment, String direction, String accountNumber, int retryCount, long delayMs) {

        return CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.MILLISECONDS.sleep(delayMs);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            return null;
        }).thenCompose(v -> validateAccount(payment, direction, accountNumber, retryCount));
    }

    /**
     * Send HTTP request.
     */
    private CompletableFuture<HttpResponse<String>> sendHttpRequest(AccountValidationRequest request) {
        try {
            String requestBody = objectMapper.writeValueAsString(request);

            HttpRequest httpRequest = HttpRequest.newBuilder()
                .uri(URI.create(serviceUrl + "/api/v1/accounts/validate"))
                .header("Content-Type", "application/json")
                .header("X-Correlation-Id", request.correlationId())
                .timeout(Duration.ofMillis(timeoutMs))
                .POST(HttpRequest.BodyPublishers.ofString(requestBody))
                .build();

            return httpClient.sendAsync(httpRequest, HttpResponse.BodyHandlers.ofString());
        } catch (Exception e) {
            return CompletableFuture.failedFuture(e);
        }
    }

    /**
     * Calculate exponential backoff.
     */
    private long calculateBackoff(int retryCount) {
        long backoff = INITIAL_BACKOFF_MS * (1L << retryCount);
        return Math.min(backoff, MAX_BACKOFF_MS);
    }

    /**
     * Handle validation failure.
     */
    private void handleValidationFailure(
            Payment payment,
            ValidationResult debitResult,
            ValidationResult creditResult,
            ResultFuture<Payment> resultFuture) {

        String failureReason = !debitResult.isValid()
            ? "DEBIT_" + debitResult.getErrorCode()
            : "CREDIT_" + creditResult.getErrorCode();

        payment.setValidationStatus(ValidationStatus.FAILED);
        payment.setFailureReason(failureReason);
        payment.addProcessingStep("ACCOUNT_VALIDATION", "FAILED: " + failureReason);

        failureCounter.inc();

        log.warn("Account validation failed: transactionId={}, reason={}",
            payment.getTransactionId(), failureReason);

        // Output to side output via context (handled by caller)
        resultFuture.complete(Collections.singleton(payment));
    }

    /**
     * Handle exceptions.
     */
    private void handleException(Payment payment, Throwable ex, ResultFuture<Payment> resultFuture) {
        payment.setValidationStatus(ValidationStatus.ERROR);
        payment.setFailureReason("SYSTEM_ERROR");
        payment.setErrorMessage(ex.getMessage());
        payment.addProcessingStep("ACCOUNT_VALIDATION", "ERROR: " + ex.getMessage());

        failureCounter.inc();

        log.error("Account validation error: transactionId={}",
            payment.getTransactionId(), ex);

        resultFuture.complete(Collections.singleton(payment));
    }

    /**
     * Mask account number for logging.
     */
    private String maskAccount(String accountNumber) {
        if (accountNumber == null || accountNumber.length() < 4) {
            return "****";
        }
        return "****" + accountNumber.substring(accountNumber.length() - 4);
    }

    @Override
    public void timeout(Payment payment, ResultFuture<Payment> resultFuture) {
        payment.setValidationStatus(ValidationStatus.TIMEOUT);
        payment.setFailureReason("TIMEOUT");
        payment.addProcessingStep("ACCOUNT_VALIDATION", "TIMEOUT");

        failureCounter.inc();

        log.warn("Account validation timeout: transactionId={}", payment.getTransactionId());

        resultFuture.complete(Collections.singleton(payment));
    }

    @Override
    public void close() throws Exception {
        // HttpClient doesn't need explicit close in Java 17+
        super.close();
    }
}

/**
 * Request DTO for account validation.
 */
public record AccountValidationRequest(
    String accountNumber,
    String accountType,
    String routingNumber,
    BigDecimal amount,
    String direction,
    String correlationId
) {
    public AccountValidationRequest(String accountNumber, String accountType,
            String routingNumber, BigDecimal amount, String direction) {
        this(accountNumber, accountType, routingNumber, amount, direction,
            UUID.randomUUID().toString());
    }
}

/**
 * Response DTO for account validation.
 */
public record AccountValidationResponse(
    boolean valid,
    String accountStatus,
    BigDecimal availableBalance,
    String accountHolderName,
    String reference,
    String errorCode,
    String errorMessage
) {}

/**
 * Validation result with success/failure information.
 */
@Data
@Builder
public class ValidationResult {
    private boolean valid;
    private String reference;
    private String accountStatus;
    private BigDecimal availableBalance;
    private String errorCode;
    private String errorMessage;

    public static ValidationResult success(String reference, String accountStatus, BigDecimal balance) {
        return ValidationResult.builder()
            .valid(true)
            .reference(reference)
            .accountStatus(accountStatus)
            .availableBalance(balance)
            .build();
    }

    public static ValidationResult failed(String errorCode, String errorMessage) {
        return ValidationResult.builder()
            .valid(false)
            .errorCode(errorCode)
            .errorMessage(errorMessage)
            .build();
    }
}
```

## Risk Control Async Function

```java
package com.example.fednow.function;

/**
 * Async function for Risk Control Service HTTP calls.
 */
@Slf4j
public class RiskControlAsyncFunction
    extends RichAsyncFunction<Payment, Payment> {

    // Similar structure to AccountValidationAsyncFunction
    // with RCS-specific request/response handling

    @Override
    public void asyncInvoke(Payment payment, ResultFuture<Payment> resultFuture) {
        // Skip if validation already failed
        if (payment.getValidationStatus() != ValidationStatus.VALID) {
            resultFuture.complete(Collections.singleton(payment));
            return;
        }

        RiskControlRequest request = buildRiskControlRequest(payment);

        sendHttpRequest(request)
            .thenAccept(response -> {
                RiskControlResponse rcsResponse = parseResponse(response);

                if (rcsResponse.isApproved()) {
                    payment.setRiskControlStatus(RiskControlStatus.APPROVED);
                    payment.setRiskScore(rcsResponse.getRiskScore());
                    payment.setRiskControlRef(rcsResponse.getReference());
                    payment.addProcessingStep("RISK_CONTROL", "APPROVED");

                    successCounter.inc();
                } else {
                    payment.setRiskControlStatus(RiskControlStatus.DECLINED);
                    payment.setRiskDeclineReason(rcsResponse.getDeclineReason());
                    payment.addProcessingStep("RISK_CONTROL", "DECLINED: " + rcsResponse.getDeclineReason());

                    failureCounter.inc();
                }

                resultFuture.complete(Collections.singleton(payment));
            })
            .exceptionally(ex -> {
                handleException(payment, ex, resultFuture);
                return null;
            });
    }
}
```

## Account Posting Async Function

```java
package com.example.fednow.function;

/**
 * Async function for TMS (Transaction Management System) HTTP calls.
 */
@Slf4j
public class AccountPostingAsyncFunction
    extends RichAsyncFunction<Payment, Payment> {

    @Override
    public void asyncInvoke(Payment payment, ResultFuture<Payment> resultFuture) {
        // Skip if any previous step failed
        if (!payment.isReadyForPosting()) {
            resultFuture.complete(Collections.singleton(payment));
            return;
        }

        PostingRequest request = PostingRequest.builder()
            .transactionId(payment.getTransactionId())
            .debitAccount(payment.getDebitAccount())
            .creditAccount(payment.getCreditAccount())
            .amount(payment.getAmount())
            .currency(payment.getCurrency())
            .valueDate(payment.getValueDate())
            .narrative("FedNow Credit Transfer")
            .validationRef(payment.getDebitValidationRef())
            .riskControlRef(payment.getRiskControlRef())
            .sanctionsRef(payment.getSanctionsRef())
            .build();

        sendHttpRequest(request)
            .thenAccept(response -> {
                if (response.isPosted()) {
                    payment.setPostingStatus(PostingStatus.POSTED);
                    payment.setPostingReference(response.getPostingReference());
                    payment.addProcessingStep("ACCOUNT_POSTING", "SUCCESS");

                    successCounter.inc();

                    log.info("Payment posted successfully: transactionId={}, postingRef={}",
                        payment.getTransactionId(), response.getPostingReference());
                } else {
                    payment.setPostingStatus(PostingStatus.FAILED);
                    payment.setPostingErrorCode(response.getErrorCode());
                    payment.addProcessingStep("ACCOUNT_POSTING", "FAILED: " + response.getErrorCode());

                    failureCounter.inc();
                }

                resultFuture.complete(Collections.singleton(payment));
            })
            .exceptionally(ex -> {
                handleException(payment, ex, resultFuture);
                return null;
            });
    }
}
```

## Best Practices

### Async I/O
- Use `unorderedWait` for better throughput when order doesn't matter
- Set appropriate timeout and capacity
- Handle timeout callback properly

### Retry Logic
- Use exponential backoff
- Set maximum retry count
- Don't retry on client errors (4xx)

### Resource Management
- Initialize HTTP clients in `open()`
- Reuse connections with HttpClient
- Clean up in `close()` if needed

### Observability
- Register Flink metrics for monitoring
- Use structured logging with correlation IDs
- Mask sensitive data in logs

### Error Handling
- Use side outputs for different failure types
- Preserve error context for debugging
- Handle both expected failures and exceptions
