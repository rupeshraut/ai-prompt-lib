# META PROMPT: Apache Flink 2.0 FedNow Payment Processing

## Project Context

This is an Apache Flink 2.0 application implementing real-time FedNow payment message processing with ISO 20022 message standards, external service integrations, and exactly-once processing guarantees.

## Core Technologies

- **Apache Flink 2.0** - Stream processing framework
- **Java 17** (LTS) - Modern Java features (records, sealed classes, pattern matching)
- **Apache Kafka** - Message broker for input/output streams
- **IBM MQ / ActiveMQ** - JMS integration for FircoSoft sanctions screening
- **ISO 20022** - Payment message standards (pacs.008, pacs.002, camt.056)
- **Jackson** - XML/JSON parsing for ISO 20022 messages
- **Flink State Backend** - RocksDB for stateful processing
- **JUnit 5 & Flink Test Harness** - Testing framework

## Architecture Overview

### Two-Job Architecture (Production Recommended)

The pipeline is split into two Flink jobs for operational flexibility:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Job 1: Sanctions Request Flow                              │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐
│  Kafka   │───▶│ Account  │───▶│   Risk   │───▶│Send JMS  │───▶│Kafka Pending │
│  Source  │    │Validation│    │ Control  │    │ Request  │    │   Topic      │
│(pacs.008)│    │  (HTTP)  │    │  (HTTP)  │    │          │    │              │
└──────────┘    └──────────┘    └──────────┘    └──────────┘    └──────────────┘
                     │               │               │
                     ▼               ▼               ▼
                  [DLQ]           [DLQ]        FircoSoft

┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Job 2: Sanctions Response Flow                             │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐         ┌──────────────────┐
│ JMS Response │───▶     │  Kafka Responses │ (via JMS-to-Kafka Bridge)
│   Queue      │         │     Topic        │
└──────────────┘         └────────┬─────────┘
                                  │
┌──────────────┐                  │
│Kafka Pending │──────────────────┼────▶ Correlate ───▶ Account ───▶ Kafka
│    Topic     │                  │      (CoProcess)    Posting     Response
└──────────────┘──────────────────┘         │
                                            ▼
                                         [DLQ]
```

### Why Two Jobs?

| Benefit | Description |
|---------|-------------|
| **Independent Scaling** | Scale JMS consumer separately from request processing |
| **Fault Isolation** | FircoSoft issues don't block payment ingestion |
| **Operational Flexibility** | Deploy/restart jobs independently |
| **Kafka as Buffer** | Durable storage for pending payments during outages |
| **Checkpoint Efficiency** | Job 1 checkpoints complete quickly (no JMS wait) |

### Processing Steps

```
Job 1: Request Flow
1. INGEST           → Consume pacs.008 from Kafka
2. PARSE            → Parse ISO 20022 XML to domain objects
3. VALIDATE_ACCOUNT → HTTP call to Account Validation Service
4. RISK_CONTROL     → HTTP call to RCS (Risk Control System)
5. SANCTIONS_SEND   → JMS message to FircoSoft
6. WRITE_PENDING    → Write pending payment to Kafka topic

JMS Bridge (separate process):
7. JMS_TO_KAFKA     → Consume JMS responses, produce to Kafka

Job 2: Response Flow
8. CORRELATE        → Join pending payments with responses by transactionId
9. ACCOUNT_POSTING  → HTTP call to TMS (Transaction Management System)
10. COMPLETE        → Publish pacs.002 response to Kafka
```

### Package Structure

```
com.example.fednow
├── job/                    # Flink job entry points
│   ├── SanctionsRequestJob.java   # Job 1
│   └── SanctionsResponseJob.java  # Job 2
├── bridge/                 # JMS-to-Kafka bridge
│   └── JmsToKafkaBridge.java
├── function/               # Flink operators and functions
│   ├── AccountValidationFunction.java
│   ├── RiskControlFunction.java
│   ├── SanctionsJmsSenderFunction.java
│   ├── SanctionsCorrelationFunction.java
│   └── AccountPostingFunction.java
├── model/                  # Domain models
│   ├── Payment.java
│   ├── PaymentStatus.java
│   ├── ValidationResult.java
│   └── iso20022/           # ISO 20022 message types
│       ├── Pacs008.java
│       ├── Pacs002.java
│       └── Camt056.java
├── parser/                 # Message parsers
│   ├── Iso20022Parser.java
│   └── Iso20022Serializer.java
├── connector/              # External system connectors
│   ├── http/
│   │   ├── AccountValidationClient.java
│   │   ├── RiskControlClient.java
│   │   └── TmsClient.java
│   └── jms/
│       ├── FircoSoftSender.java
│       └── FircoSoftReceiver.java
├── state/                  # State management
│   ├── PaymentStateSchema.java
│   └── AsyncResponseHandler.java
├── exception/              # Custom exceptions
│   ├── ValidationException.java
│   ├── RiskControlException.java
│   ├── SanctionsException.java
│   └── PostingException.java
├── config/                 # Configuration
│   └── FlinkConfig.java
└── util/                   # Utilities
    ├── RetryUtils.java
    └── MetricsUtils.java
```

## FedNow Message Types

### pacs.008 - Credit Transfer (Inbound)

```xml
<FIToFICstmrCdtTrf>
  <GrpHdr>
    <MsgId>MSG123456</MsgId>
    <CreDtTm>2025-11-22T10:00:00Z</CreDtTm>
    <NbOfTxs>1</NbOfTxs>
    <SttlmInf>
      <SttlmMtd>CLRG</SttlmMtd>
    </SttlmInf>
  </GrpHdr>
  <CdtTrfTxInf>
    <PmtId>
      <InstrId>INSTR123</InstrId>
      <EndToEndId>E2E123</EndToEndId>
      <TxId>TXN123</TxId>
    </PmtId>
    <IntrBkSttlmAmt Ccy="USD">1000.00</IntrBkSttlmAmt>
    <Dbtr>
      <Nm>John Doe</Nm>
    </Dbtr>
    <DbtrAcct>
      <Id><Other><Id>123456789</Id></Other></Id>
    </DbtrAcct>
    <Cdtr>
      <Nm>Jane Smith</Nm>
    </Cdtr>
    <CdtrAcct>
      <Id><Other><Id>987654321</Id></Other></Id>
    </CdtrAcct>
  </CdtTrfTxInf>
</FIToFICstmrCdtTrf>
```

### pacs.002 - Status Report (Outbound)

```xml
<FIToFIPmtStsRpt>
  <GrpHdr>
    <MsgId>RESP123456</MsgId>
    <CreDtTm>2025-11-22T10:00:05Z</CreDtTm>
  </GrpHdr>
  <TxInfAndSts>
    <OrgnlInstrId>INSTR123</OrgnlInstrId>
    <OrgnlEndToEndId>E2E123</OrgnlEndToEndId>
    <TxSts>ACCP</TxSts>  <!-- ACCP, RJCT, PDNG -->
  </TxInfAndSts>
</FIToFIPmtStsRpt>
```

## External Service Contracts

### 1. Account Validation Service (HTTP)

```
POST /api/v1/accounts/validate
Content-Type: application/json

Request:
{
  "accountNumber": "123456789",
  "accountType": "CHECKING",
  "routingNumber": "021000021",
  "amount": 1000.00,
  "direction": "DEBIT" | "CREDIT"
}

Response (200 OK):
{
  "valid": true,
  "accountStatus": "ACTIVE",
  "availableBalance": 5000.00,
  "accountHolderName": "John Doe"
}

Response (400 Bad Request):
{
  "valid": false,
  "errorCode": "INSUFFICIENT_FUNDS",
  "errorMessage": "Available balance is less than requested amount"
}
```

### 2. Risk Control Service (HTTP)

```
POST /api/v1/rcs/evaluate
Content-Type: application/json

Request:
{
  "transactionId": "TXN123",
  "debitAccount": "123456789",
  "creditAccount": "987654321",
  "amount": 1000.00,
  "currency": "USD",
  "debtorName": "John Doe",
  "creditorName": "Jane Smith",
  "transactionType": "CREDIT_TRANSFER"
}

Response (200 OK):
{
  "approved": true,
  "riskScore": 15,
  "riskLevel": "LOW",
  "controlsApplied": ["VELOCITY_CHECK", "AMOUNT_LIMIT"]
}

Response (200 OK - Declined):
{
  "approved": false,
  "riskScore": 85,
  "riskLevel": "HIGH",
  "declineReason": "VELOCITY_LIMIT_EXCEEDED",
  "controlsTriggered": ["DAILY_LIMIT"]
}
```

### 3. FircoSoft Sanctions Screening (JMS)

```
Queue: SANCTIONS.REQUEST.QUEUE
Message Format: XML

Request:
<SanctionsRequest>
  <RequestId>REQ123</RequestId>
  <TransactionId>TXN123</TransactionId>
  <Parties>
    <Party type="DEBTOR">
      <Name>John Doe</Name>
      <Account>123456789</Account>
      <Country>US</Country>
    </Party>
    <Party type="CREDITOR">
      <Name>Jane Smith</Name>
      <Account>987654321</Account>
      <Country>US</Country>
    </Party>
  </Parties>
  <Amount currency="USD">1000.00</Amount>
</SanctionsRequest>

Queue: SANCTIONS.RESPONSE.QUEUE
Response:
<SanctionsResponse>
  <RequestId>REQ123</RequestId>
  <TransactionId>TXN123</TransactionId>
  <Result>CLEAR</Result>  <!-- CLEAR, HIT, POTENTIAL_HIT -->
  <Hits>
    <!-- Only if Result is HIT or POTENTIAL_HIT -->
    <Hit>
      <ListName>OFAC SDN</ListName>
      <MatchScore>95</MatchScore>
      <MatchedParty>DEBTOR</MatchedParty>
    </Hit>
  </Hits>
</SanctionsResponse>
```

### 4. Transaction Management System - TMS (HTTP)

```
POST /api/v1/tms/post
Content-Type: application/json

Request:
{
  "transactionId": "TXN123",
  "debitAccount": "123456789",
  "creditAccount": "987654321",
  "amount": 1000.00,
  "currency": "USD",
  "valueDate": "2025-11-22",
  "narrative": "FedNow Credit Transfer",
  "validationRef": "VAL123",
  "riskControlRef": "RCS123",
  "sanctionsRef": "SAN123"
}

Response (200 OK):
{
  "posted": true,
  "postingReference": "POST123456",
  "debitPostingId": "DEB123",
  "creditPostingId": "CRD123",
  "postedAt": "2025-11-22T10:00:05Z"
}

Response (400 Bad Request):
{
  "posted": false,
  "errorCode": "POSTING_FAILED",
  "errorMessage": "Debit account closed"
}
```

## Flink Processing Patterns

### 1. Async I/O Pattern (HTTP Calls)

```java
// Use AsyncDataStream for non-blocking HTTP calls
AsyncDataStream.unorderedWait(
    inputStream,
    new AccountValidationAsyncFunction(httpClient),
    30, TimeUnit.SECONDS,  // timeout
    100                     // capacity
)
```

### 2. Keyed State for Async JMS (Sanctions)

```java
// Use KeyedProcessFunction with state for async JMS correlation
public class SanctionsProcessFunction
    extends KeyedProcessFunction<String, Payment, Payment> {

    private ValueState<Payment> pendingPaymentState;

    @Override
    public void processElement(Payment payment, Context ctx, Collector<Payment> out) {
        // Store payment in state
        pendingPaymentState.update(payment);

        // Send JMS request
        sendSanctionsRequest(payment);

        // Register timer for timeout
        ctx.timerService().registerProcessingTimeTimer(
            ctx.timerService().currentProcessingTime() + TIMEOUT_MS
        );
    }

    // JMS response arrives via side input or separate stream
    public void onSanctionsResponse(SanctionsResponse response) {
        Payment payment = pendingPaymentState.value();
        // Resume processing
    }
}
```

### 3. Side Outputs for Error Handling

```java
// Define side outputs for different failure types
final OutputTag<Payment> validationFailedTag =
    new OutputTag<>("validation-failed"){};
final OutputTag<Payment> riskControlFailedTag =
    new OutputTag<>("risk-control-failed"){};
final OutputTag<Payment> sanctionsHitTag =
    new OutputTag<>("sanctions-hit"){};
final OutputTag<Payment> postingFailedTag =
    new OutputTag<>("posting-failed"){};
```

### 4. Exactly-Once with Checkpointing

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(60000); // 60 seconds
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(30000);
env.setStateBackend(new EmbeddedRocksDBStateBackend());
```

## Error Handling Strategy

### Retry Policies

| Service | Max Retries | Backoff | Timeout |
|---------|-------------|---------|---------|
| Account Validation | 3 | Exponential (1s, 2s, 4s) | 10s |
| Risk Control | 3 | Exponential (1s, 2s, 4s) | 15s |
| Sanctions (JMS) | 0 | N/A | 30s |
| Account Posting | 3 | Exponential (1s, 2s, 4s) | 20s |

### Dead Letter Queues

```
fednow.payments.dlq.validation-failed
fednow.payments.dlq.risk-control-failed
fednow.payments.dlq.sanctions-hit
fednow.payments.dlq.posting-failed
fednow.payments.dlq.timeout
```

## Monitoring & Observability

### Metrics

- `fednow.payments.received` - Counter
- `fednow.payments.completed` - Counter
- `fednow.payments.failed` - Counter (tagged by failure_reason)
- `fednow.validation.latency` - Histogram
- `fednow.riskcontrol.latency` - Histogram
- `fednow.sanctions.latency` - Histogram
- `fednow.posting.latency` - Histogram
- `fednow.e2e.latency` - Histogram (end-to-end)

### Logging

```java
// Structured logging with correlation ID
log.info("Payment processing started",
    kv("transactionId", payment.getTransactionId()),
    kv("step", "VALIDATION"),
    kv("amount", payment.getAmount()));
```

## Testing Strategy

### Unit Tests
- Test individual ProcessFunctions with Flink test harness
- Mock HTTP clients and JMS connections
- Test state management and timers

### Integration Tests
- Use MiniClusterWithClientResource
- Test complete pipeline with embedded Kafka
- Test checkpoint/restore behavior

### Contract Tests
- Validate ISO 20022 message parsing
- Validate external service contracts

## Configuration

```yaml
# application.yml
flink:
  job:
    name: fednow-payment-processor
    parallelism: 4
    checkpoint:
      interval: 60000
      timeout: 120000

kafka:
  bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
  consumer:
    group-id: fednow-processor
  topics:
    input: fednow.payments.inbound
    output: fednow.payments.outbound
    dlq-prefix: fednow.payments.dlq

services:
  account-validation:
    url: ${ACCOUNT_VALIDATION_URL:http://localhost:8081}
    timeout: 10000
    retry:
      max-attempts: 3
      backoff: 1000
  risk-control:
    url: ${RISK_CONTROL_URL:http://localhost:8082}
    timeout: 15000
  tms:
    url: ${TMS_URL:http://localhost:8083}
    timeout: 20000

jms:
  broker-url: ${JMS_BROKER_URL:tcp://localhost:61616}
  sanctions:
    request-queue: SANCTIONS.REQUEST.QUEUE
    response-queue: SANCTIONS.RESPONSE.QUEUE
    timeout: 30000

state:
  backend: rocksdb
  checkpoint-dir: ${CHECKPOINT_DIR:file:///tmp/flink-checkpoints}
```

## Best Practices

### Code Quality
- Use sealed interfaces for payment states
- Use records for immutable value objects
- Keep ProcessFunctions focused and testable
- Avoid blocking operations in main processing thread

### Performance
- Use async I/O for all external calls
- Configure appropriate parallelism per operator
- Monitor backpressure and adjust capacity
- Use incremental checkpoints with RocksDB

### Reliability
- Implement idempotent operations
- Use transaction IDs for deduplication
- Handle timeouts gracefully
- Store intermediate state for recovery

### Security
- Encrypt sensitive data in state
- Use TLS for all external connections
- Mask account numbers in logs
- Implement audit logging

---

**Remember**: This meta-prompt serves as a comprehensive guide for FedNow payment processing. Always adapt to specific regulatory requirements and infrastructure constraints.
