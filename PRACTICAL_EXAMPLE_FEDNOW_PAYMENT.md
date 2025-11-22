# Practical Example: FedNow Payment Processing Pipeline

This guide demonstrates how to use the Flink/FedNow prompt templates to build a complete real-time payment processing pipeline using prompt chaining methodology.

## Two-Job Architecture Overview

The pipeline is split into **two Flink jobs** plus a **JMS-to-Kafka bridge** for production resilience:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Job 1: Sanctions Request Flow                              │
└─────────────────────────────────────────────────────────────────────────────────┘

Step 1-2        Step 3          Step 4          Step 5          Step 6
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Kafka   │───▶│ Account  │───▶│   Risk   │───▶│Send JMS  │───▶│  Kafka   │
│  Source  │    │Validation│    │ Control  │    │ Request  │    │ Pending  │
│(pacs.008)│    │  (HTTP)  │    │  (HTTP)  │    │          │    │  Topic   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────┘
                     │               │               │
                     ▼               ▼               ▼
                  [DLQ]           [DLQ]        FircoSoft

┌─────────────────────────────────────────────────────────────────────────────────┐
│                      JMS-to-Kafka Bridge (Step 7)                               │
└─────────────────────────────────────────────────────────────────────────────────┘
│   JMS Response Queue  │───▶│  Kafka Responses Topic  │

┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Job 2: Sanctions Response Flow                             │
└─────────────────────────────────────────────────────────────────────────────────┘

Step 8          Step 9          Step 10         Step 11
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│Correlate │───▶│ Account  │───▶│ Generate │───▶│  Kafka   │
│ Pending  │    │ Posting  │    │ pacs.002 │    │ Response │
│+Response │    │  (HTTP)  │    │          │    │   Sink   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     │               │
     ▼               ▼
  [DLQ]           [DLQ]
```

### Why Two Jobs?

| Benefit | Description |
|---------|-------------|
| **Independent Scaling** | Scale JMS consumer separately from request processing |
| **Fault Isolation** | FircoSoft issues don't block payment ingestion |
| **Operational Flexibility** | Deploy/restart jobs independently |
| **Kafka as Buffer** | Durable storage for pending payments during outages |

## Complete 11-Step Prompt Chain

### Step 1: Create Payment Domain Model

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-payment-model.md

Create the Payment domain model with:
- All FedNow message fields (transactionId, amount, debtor/creditor info)
- Processing status enums (PaymentStatus, ValidationStatus, etc.)
- Helper methods for state transitions
- Flink serialization support
```

**Expected Output:**
```java
// Payment.java - ~200 lines
@Data
@Builder
public class Payment implements Serializable {
    private String transactionId;
    private BigDecimal amount;
    private String currency;
    private String debtorName;
    private String debitAccount;
    // ... all fields

    public boolean isReadyForSanctions() { ... }
    public boolean isReadyForPosting() { ... }
}

// PaymentStatus.java, ValidationStatus.java, etc.
```

### Step 2: Create ISO 20022 Parser

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-iso20022-parser.md

Create parsers for:
- pacs.008 (Credit Transfer) - inbound parsing
- pacs.002 (Status Report) - outbound serialization
Include:
- Jackson XML annotations
- Validation against FedNow rules
- Flink MapFunction wrappers
```

**Expected Output:**
```java
// Pacs008Parser.java - parses XML to Payment
public class Pacs008Parser {
    public Payment parse(String xml) throws Pacs008ParseException { ... }
}

// Pacs002Serializer.java - generates response XML
public class Pacs002Serializer {
    public String serialize(Payment payment) { ... }
}

// Iso20022ParserFunction.java - Flink MapFunction
public class Iso20022ParserFunction extends RichMapFunction<String, Payment> { ... }
```

### Step 3: Create Account Validation Async Function

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-async-function.md

Create AccountValidationAsyncFunction that:
- Validates both debit and credit accounts in parallel
- Calls HTTP endpoint: POST /api/v1/accounts/validate
- Implements retry with exponential backoff (1s, 2s, 4s)
- Timeout: 10 seconds
- Tracks metrics: success, failure, latency
```

**Expected Output:**
```java
@Slf4j
public class AccountValidationAsyncFunction
    extends RichAsyncFunction<Payment, Payment> {

    private static final int MAX_RETRIES = 3;

    @Override
    public void asyncInvoke(Payment payment, ResultFuture<Payment> resultFuture) {
        CompletableFuture<ValidationResult> debitValidation = validateAccount(...);
        CompletableFuture<ValidationResult> creditValidation = validateAccount(...);

        CompletableFuture.allOf(debitValidation, creditValidation)
            .thenAccept(v -> {
                // Combine results and complete
            });
    }
}
```

### Step 4: Create Risk Control Async Function

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-async-function.md

Create RiskControlAsyncFunction that:
- Evaluates transaction risk for both debit and credit
- Calls HTTP endpoint: POST /api/v1/rcs/evaluate
- Skips if validation already failed
- Timeout: 15 seconds
- Captures risk score and controls applied
```

**Expected Output:**
```java
@Slf4j
public class RiskControlAsyncFunction
    extends RichAsyncFunction<Payment, Payment> {

    @Override
    public void asyncInvoke(Payment payment, ResultFuture<Payment> resultFuture) {
        // Skip if validation failed
        if (payment.getValidationStatus() != ValidationStatus.VALID) {
            resultFuture.complete(Collections.singleton(payment));
            return;
        }

        sendHttpRequest(buildRiskControlRequest(payment))
            .thenAccept(response -> {
                // Process RCS response
            });
    }
}
```

### Step 5: Create JMS Sender Function (Job 1)

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-sanctions-request-job.md

Create SanctionsJmsSenderFunction that:
- Sends JMS message to FircoSoft (SANCTIONS.REQUEST.QUEUE)
- Uses transacted session for reliability
- Emits payment for downstream Kafka pending sink
- Routes failures to side output
```

**Expected Output:**
```java
public class SanctionsJmsSenderFunction
    extends ProcessFunction<Payment, Payment> {

    @Override
    public void processElement(Payment payment, Context ctx, Collector<Payment> out) {
        try {
            SanctionsRequest request = buildSanctionsRequest(payment);
            sendJmsRequest(payment, request);

            payment.setSanctionsStatus(SanctionsStatus.PENDING);
            payment.setSanctionsRequestId(request.getRequestId());

            out.collect(payment);  // To Kafka pending topic
        } catch (JMSException e) {
            ctx.output(JMS_SEND_FAILED, payment);
        }
    }
}
```

### Step 6: Create Sanctions Request Job (Job 1)

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-sanctions-request-job.md

Create SanctionsRequestJob that:
- Chains steps 1-5 (parse, validate, risk control, JMS send)
- Writes pending payments to Kafka topic
- Handles side outputs for failures
- Uses separate checkpoint directory
```

**Expected Output:**
```java
@Slf4j
public class SanctionsRequestJob {

    public static void main(String[] args) throws Exception {
        // Build pipeline: Kafka -> Parse -> Validate -> RCS -> JMS -> Kafka Pending
        env.execute("FedNow Sanctions Request Job");
    }
}
```

### Step 7: Create JMS-to-Kafka Bridge

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-sanctions-response-job.md

Create JmsToKafkaBridge that:
- Consumes from SANCTIONS.RESPONSE.QUEUE via JMS
- Produces to Kafka responses topic
- Uses transactionId as Kafka key for co-partitioning
- Runs as separate process (not Flink)
```

**Expected Output:**
```java
public class JmsToKafkaBridge {

    public void start() throws Exception {
        while (running) {
            Message message = messageConsumer.receive(1000);
            if (message instanceof TextMessage textMessage) {
                SanctionsResponse response = parseResponse(textMessage.getText());
                kafkaProducer.send(new ProducerRecord<>(
                    kafkaTopic,
                    response.getTransactionId(),
                    toJson(response)
                ));
                textMessage.acknowledge();
            }
        }
    }
}
```

### Step 8: Create Sanctions Correlation Function (Job 2)

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-sanctions-response-job.md

Create SanctionsCorrelationFunction that:
- Joins pending payments (Kafka) with responses (Kafka)
- Uses KeyedCoProcessFunction for correlation by transactionId
- Registers timeout timer for missing responses
- Routes to side output on HIT, POTENTIAL_HIT, or TIMEOUT
```

**Expected Output:**
```java
public class SanctionsCorrelationFunction
    extends KeyedCoProcessFunction<String, Payment, SanctionsResponse, Payment> {

    @Override
    public void processElement1(Payment payment, Context ctx, Collector<Payment> out) {
        SanctionsResponse response = pendingResponseState.value();
        if (response != null) {
            processCorrelation(payment, response, ctx, out);
            pendingResponseState.clear();
        } else {
            pendingPaymentState.update(payment);
            registerTimeout(ctx);
        }
    }

    @Override
    public void processElement2(SanctionsResponse response, Context ctx, Collector<Payment> out) {
        Payment payment = pendingPaymentState.value();
        if (payment != null) {
            cancelTimer(ctx);
            processCorrelation(payment, response, ctx, out);
            pendingPaymentState.clear();
        } else {
            pendingResponseState.update(response);
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Payment> out) {
        Payment payment = pendingPaymentState.value();
        if (payment != null) {
            payment.setSanctionsStatus(SanctionsStatus.TIMEOUT);
            ctx.output(SANCTIONS_TIMEOUT, payment);
        }
    }
}
```

### Step 9: Create Account Posting Async Function

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-async-function.md

Create AccountPostingAsyncFunction that:
- Posts debit and credit entries to TMS
- Calls HTTP endpoint: POST /api/v1/tms/post
- Includes all validation/RCS/sanctions references
- Timeout: 20 seconds
- Only processes if sanctions cleared
```

**Expected Output:**
```java
@Slf4j
public class AccountPostingAsyncFunction
    extends RichAsyncFunction<Payment, Payment> {

    @Override
    public void asyncInvoke(Payment payment, ResultFuture<Payment> resultFuture) {
        if (!payment.isReadyForPosting()) {
            resultFuture.complete(Collections.singleton(payment));
            return;
        }

        PostingRequest request = PostingRequest.builder()
            .transactionId(payment.getTransactionId())
            .validationRef(payment.getDebitValidationRef())
            .riskControlRef(payment.getRiskControlRef())
            .sanctionsRef(payment.getSanctionsRef())
            .build();

        sendHttpRequest(request)
            .thenAccept(response -> {
                // Mark as posted or failed
            });
    }
}
```

### Step 10: Create Sanctions Response Job (Job 2)

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-sanctions-response-job.md

Create SanctionsResponseJob that:
- Consumes pending payments and responses from Kafka
- Correlates by transactionId using CoProcessFunction
- Posts to TMS on CLEAR status
- Generates pacs.002 responses
- Uses separate checkpoint directory from Job 1
```

**Expected Output:**
```java
@Slf4j
public class SanctionsResponseJob {

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = createExecutionEnvironment(config);
        buildPipeline(env, config);
        env.execute("FedNow Sanctions Response Job");
    }

    private static void buildPipeline(StreamExecutionEnvironment env, FlinkConfig config) {
        // Pending payments from Job 1
        DataStream<Payment> pendingPayments = env.fromSource(pendingSource, ...);

        // Sanctions responses from JMS bridge
        DataStream<SanctionsResponse> responses = env.fromSource(responsesSource, ...);

        // Correlate by transactionId
        SingleOutputStreamOperator<Payment> correlated = pendingPayments
            .keyBy(Payment::getTransactionId)
            .connect(responses.keyBy(SanctionsResponse::getTransactionId))
            .process(new SanctionsCorrelationFunction(30_000));

        // Account Posting (Async) - only CLEAR status
        SingleOutputStreamOperator<Payment> posted = AsyncDataStream.unorderedWait(
            correlated.filter(p -> p.getSanctionsStatus() == SanctionsStatus.CLEAR),
            new AccountPostingAsyncFunction(...), 20, SECONDS, 100);

        // Generate pacs.002 and sink
        posted.map(new Pacs002GeneratorFunction()).sinkTo(kafkaSink);

        // Handle side outputs (SANCTIONS_HIT, SANCTIONS_TIMEOUT, POSTING_FAILED)
        handleSideOutputs(correlated, posted, config);
    }
}
```

## Testing the Pipeline

### Step 11: Create Unit Tests

**Prompt:**
```
@workspace Use .github/prompts/flink-fednow/create-flink-test.md

Create unit tests for:
- Pacs008Parser with valid/invalid XML
- AccountValidationAsyncFunction with mock HTTP
- SanctionsCorrelationFunction with test harness
- Complete pipeline integration test
```

**Expected Output:**
```java
// Pacs008ParserTest.java
class Pacs008ParserTest {
    @Test
    void shouldParseValidMessage() { ... }

    @Test
    void shouldThrowOnInvalidXml() { ... }
}

// AccountValidationAsyncFunctionTest.java
class AccountValidationAsyncFunctionTest {
    @Test
    void shouldValidateAccountsSuccessfully() { ... }

    @Test
    void shouldRetryOnServerError() { ... }
}

// SanctionsCorrelationFunctionTest.java
class SanctionsCorrelationFunctionTest {
    @Test
    void shouldCorrelatePaymentWithResponse() { ... }

    @Test
    void shouldTimeoutOnMissingResponse() { ... }
}

// PaymentProcessingJobIntegrationTest.java
class PaymentProcessingJobIntegrationTest {
    @Test
    void shouldProcessPaymentEndToEnd() { ... }
}
```

## Configuration

### application.yml (Two-Job Architecture)

```yaml
# Job 1: Sanctions Request Flow
flink:
  job1:
    name: fednow-sanctions-request-job
    parallelism: 4
    checkpoint:
      interval: 60000
      timeout: 60000           # Shorter - no JMS wait
      dir: s3://bucket/flink-checkpoints/job1

# Job 2: Sanctions Response Flow
  job2:
    name: fednow-sanctions-response-job
    parallelism: 4
    checkpoint:
      interval: 60000
      timeout: 120000          # Longer - handles state
      dir: s3://bucket/flink-checkpoints/job2

kafka:
  bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}
  consumer:
    group-id: fednow-processor
  topics:
    input: fednow.payments.inbound
    output: fednow.payments.outbound
    pending: fednow.payments.pending-sanctions    # Job 1 -> Job 2
    sanctions-responses: fednow.sanctions.responses  # Bridge -> Job 2
    dlq-prefix: fednow.payments.dlq

services:
  account-validation:
    url: ${ACCOUNT_VALIDATION_URL}
    timeout: 10000
  risk-control:
    url: ${RISK_CONTROL_URL}
    timeout: 15000
  tms:
    url: ${TMS_URL}
    timeout: 20000

jms:
  broker-url: ${JMS_BROKER_URL}
  sanctions:
    request-queue: SANCTIONS.REQUEST.QUEUE
    response-queue: SANCTIONS.RESPONSE.QUEUE
    timeout: 30000
```

### Deployment Order

```bash
# 1. Deploy JMS-to-Kafka bridge first
kubectl apply -f jms-kafka-bridge.yaml

# 2. Deploy Job 2 (consumer of pending + responses)
flink run -d sanctions-response-job.jar

# 3. Deploy Job 1 (producer of pending)
flink run -d sanctions-request-job.jar
```

## Error Handling Summary

| Step | Failure Type | Side Output Tag | DLQ Topic |
|------|--------------|-----------------|-----------|
| Parse | Parse error | - | fednow.payments.dlq.parse-error |
| Validation | Invalid account | VALIDATION_FAILED | fednow.payments.dlq.validation-failed |
| Risk Control | Declined | RISK_CONTROL_FAILED | fednow.payments.dlq.risk-control-failed |
| Sanctions | Hit/Timeout | SANCTIONS_HIT | fednow.payments.dlq.sanctions-hit |
| Posting | Failed | POSTING_FAILED | fednow.payments.dlq.posting-failed |

## Metrics to Monitor

```
# Throughput
fednow.payments.received - total payments ingested
fednow.payments.completed - successfully processed
fednow.payments.failed - total failures

# Step Latencies (histograms)
fednow.validation.latency
fednow.riskcontrol.latency
fednow.sanctions.latency
fednow.posting.latency
fednow.e2e.latency - end-to-end processing time

# Error Rates
fednow.validation.failure.rate
fednow.sanctions.hit.rate
fednow.sanctions.timeout.rate
```

## Summary

This 11-step prompt chain creates a production-ready FedNow payment processing pipeline with:

- **Two-job architecture** for operational flexibility and fault isolation
- **Exactly-once processing** via Flink checkpointing
- **Non-blocking HTTP** calls with async I/O
- **JMS integration** for sanctions screening with FircoSoft
- **Kafka-based correlation** for async responses (via JMS-to-Kafka bridge)
- **Comprehensive error handling** with DLQ routing
- **Full observability** with metrics and structured logging
- **Independent scaling** of request and response processing

### Component Summary

| Component | Type | Purpose |
|-----------|------|---------|
| SanctionsRequestJob | Flink Job | Parse, validate, RCS check, send JMS, write pending |
| JmsToKafkaBridge | Standalone | Consume JMS responses, produce to Kafka |
| SanctionsResponseJob | Flink Job | Correlate, post accounts, generate response |

Each prompt template is designed to work independently or as part of the chain, following the same patterns established in the Spring Boot templates.
