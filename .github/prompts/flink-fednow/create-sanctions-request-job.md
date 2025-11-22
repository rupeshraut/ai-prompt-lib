# Create Sanctions Request Job

Create Flink Job 1 for the two-job sanctions architecture: processes payments through validation and risk control, sends JMS request to FircoSoft, and writes pending payments to Kafka.

## Architecture Overview

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
```

## Why Two Jobs?

1. **Independent Scaling** - Scale JMS consumer separately from request processing
2. **Fault Isolation** - FircoSoft issues don't block payment ingestion
3. **Operational Flexibility** - Deploy/restart jobs independently
4. **Kafka as Buffer** - Durable storage for pending payments during outages

## Job 1: Sanctions Request Job

```java
package com.example.fednow.job;

import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.connector.kafka.sink.KafkaSink;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.AsyncDataStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.OutputTag;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

/**
 * Job 1: Sanctions Request Flow
 *
 * <p>Processes incoming payments through validation and risk control,
 * sends sanctions screening request to FircoSoft via JMS, and writes
 * pending payment to Kafka for Job 2 to pick up.
 *
 * <p>Processing Pipeline:
 * <ol>
 *   <li>Consume pacs.008 messages from Kafka</li>
 *   <li>Parse ISO 20022 XML</li>
 *   <li>Validate accounts via HTTP</li>
 *   <li>Perform risk control checks via HTTP</li>
 *   <li>Send sanctions request to FircoSoft via JMS</li>
 *   <li>Write pending payment to Kafka topic</li>
 * </ol>
 */
@Slf4j
public class SanctionsRequestJob {

    // Side output tags for failures
    public static final OutputTag<Payment> VALIDATION_FAILED =
        new OutputTag<>("validation-failed") {};
    public static final OutputTag<Payment> RISK_CONTROL_FAILED =
        new OutputTag<>("risk-control-failed") {};
    public static final OutputTag<Payment> JMS_SEND_FAILED =
        new OutputTag<>("jms-send-failed") {};

    public static void main(String[] args) throws Exception {
        FlinkConfig config = FlinkConfig.load(args);

        StreamExecutionEnvironment env = createExecutionEnvironment(config);
        buildPipeline(env, config);

        env.execute("FedNow Sanctions Request Job");
    }

    private static StreamExecutionEnvironment createExecutionEnvironment(FlinkConfig config) {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        env.setParallelism(config.getParallelism());

        // Checkpointing - can be more aggressive since no JMS wait
        env.enableCheckpointing(config.getCheckpointInterval());
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        env.getCheckpointConfig().setMinPauseBetweenCheckpoints(config.getMinPauseBetweenCheckpoints());
        env.getCheckpointConfig().setCheckpointTimeout(60000); // 60s - faster than Job 2
        env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

        // RocksDB state backend
        env.setStateBackend(new EmbeddedRocksDBStateBackend(true));
        env.getCheckpointConfig().setCheckpointStorage(config.getCheckpointDir() + "/job1");

        log.info("Job 1 environment configured: parallelism={}", config.getParallelism());

        return env;
    }

    private static void buildPipeline(StreamExecutionEnvironment env, FlinkConfig config) {
        // Step 1: Kafka source for inbound payments
        KafkaSource<String> kafkaSource = KafkaSource.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setTopics(config.getInputTopic())
            .setGroupId(config.getConsumerGroupId() + "-job1")
            .setStartingOffsets(OffsetsInitializer.committedOffsets(OffsetResetStrategy.EARLIEST))
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        DataStream<String> rawMessages = env
            .fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "kafka-source")
            .name("Kafka Source")
            .uid("kafka-source-job1");

        // Step 2: Parse ISO 20022
        DataStream<Payment> payments = rawMessages
            .map(new Iso20022ParserFunction())
            .name("Parse ISO 20022")
            .uid("parse-iso20022");

        // Step 3: Account Validation (Async HTTP)
        SingleOutputStreamOperator<Payment> validated = AsyncDataStream.unorderedWait(
            payments,
            new AccountValidationAsyncFunction(
                config.getAccountValidationUrl(),
                config.getAccountValidationTimeout()
            ),
            config.getAccountValidationTimeout(),
            TimeUnit.MILLISECONDS,
            100
        )
        .name("Account Validation")
        .uid("account-validation");

        // Step 4: Risk Control (Async HTTP)
        SingleOutputStreamOperator<Payment> riskChecked = AsyncDataStream.unorderedWait(
            validated.filter(p -> p.getValidationStatus() == ValidationStatus.VALID),
            new RiskControlAsyncFunction(
                config.getRiskControlUrl(),
                config.getRiskControlTimeout()
            ),
            config.getRiskControlTimeout(),
            TimeUnit.MILLISECONDS,
            100
        )
        .name("Risk Control")
        .uid("risk-control");

        // Step 5: Send JMS Request & Write to Pending Topic
        SingleOutputStreamOperator<Payment> jmsSent = riskChecked
            .filter(p -> p.getRiskControlStatus() == RiskControlStatus.APPROVED)
            .process(new SanctionsJmsSenderFunction(config.getJmsConfig()))
            .name("Send JMS Request")
            .uid("send-jms-request");

        // Step 6: Write pending payments to Kafka (for Job 2 to correlate)
        KafkaSink<String> pendingSink = createPendingPaymentsSink(config);

        jmsSent
            .map(new PaymentToJsonFunction())
            .sinkTo(pendingSink)
            .name("Kafka Pending Sink")
            .uid("kafka-pending-sink");

        // Handle side outputs
        handleSideOutputs(validated, riskChecked, jmsSent, config);

        log.info("Job 1 pipeline built successfully");
    }

    private static KafkaSink<String> createPendingPaymentsSink(FlinkConfig config) {
        return KafkaSink.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setRecordSerializer(KafkaRecordSerializationSchema.<String>builder()
                .setTopic(config.getPendingPaymentsTopic())
                .setKeySerializationSchema(new PaymentKeySerializer())
                .setValueSerializationSchema(new SimpleStringSchema())
                .build())
            .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
            .setTransactionalIdPrefix(config.getTransactionalIdPrefix() + "-job1")
            .build();
    }

    private static void handleSideOutputs(
            SingleOutputStreamOperator<Payment> validated,
            SingleOutputStreamOperator<Payment> riskChecked,
            SingleOutputStreamOperator<Payment> jmsSent,
            FlinkConfig config) {

        // Validation failures
        validated.getSideOutput(VALIDATION_FAILED)
            .map(new FailedPaymentSerializer("VALIDATION_FAILED"))
            .sinkTo(createDlqSink(config, "validation-failed"))
            .name("DLQ: Validation Failed")
            .uid("dlq-validation-failed");

        // Risk control failures
        riskChecked.getSideOutput(RISK_CONTROL_FAILED)
            .map(new FailedPaymentSerializer("RISK_CONTROL_FAILED"))
            .sinkTo(createDlqSink(config, "risk-control-failed"))
            .name("DLQ: Risk Control Failed")
            .uid("dlq-risk-control-failed");

        // JMS send failures
        jmsSent.getSideOutput(JMS_SEND_FAILED)
            .map(new FailedPaymentSerializer("JMS_SEND_FAILED"))
            .sinkTo(createDlqSink(config, "jms-send-failed"))
            .name("DLQ: JMS Send Failed")
            .uid("dlq-jms-send-failed");
    }

    private static KafkaSink<String> createDlqSink(FlinkConfig config, String dlqSuffix) {
        return KafkaSink.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setRecordSerializer(KafkaRecordSerializationSchema.builder()
                .setTopic(config.getDlqTopicPrefix() + "." + dlqSuffix)
                .setValueSerializationSchema(new SimpleStringSchema())
                .build())
            .setDeliveryGuarantee(DeliveryGuarantee.AT_LEAST_ONCE)
            .build();
    }
}
```

## JMS Sender Function

```java
package com.example.fednow.function;

import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;
import org.apache.flink.metrics.Counter;
import lombok.extern.slf4j.Slf4j;

import javax.jms.*;
import javax.naming.InitialContext;

/**
 * Process function that sends sanctions request to FircoSoft via JMS.
 *
 * <p>This function:
 * <ol>
 *   <li>Builds XML request from Payment</li>
 *   <li>Sends to FircoSoft JMS queue</li>
 *   <li>Marks payment as SANCTIONS_PENDING</li>
 *   <li>Emits payment for downstream Kafka sink</li>
 * </ol>
 *
 * <p>Note: This does NOT wait for response. Job 2 handles correlation.
 */
@Slf4j
public class SanctionsJmsSenderFunction
    extends ProcessFunction<Payment, Payment> {

    private final JmsConfig jmsConfig;

    // JMS resources (transient - not serialized)
    private transient Connection jmsConnection;
    private transient Session jmsSession;
    private transient MessageProducer messageProducer;

    // Metrics
    private transient Counter requestsSent;
    private transient Counter sendFailures;

    public static final OutputTag<Payment> JMS_SEND_FAILED =
        new OutputTag<>("jms-send-failed") {};

    public SanctionsJmsSenderFunction(JmsConfig jmsConfig) {
        this.jmsConfig = jmsConfig;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);

        initializeJms();

        requestsSent = getRuntimeContext()
            .getMetricGroup()
            .counter("sanctions.jms.requestsSent");
        sendFailures = getRuntimeContext()
            .getMetricGroup()
            .counter("sanctions.jms.sendFailures");

        log.info("SanctionsJmsSenderFunction initialized");
    }

    private void initializeJms() throws Exception {
        ConnectionFactory connectionFactory = (ConnectionFactory)
            new InitialContext().lookup(jmsConfig.getConnectionFactoryName());

        jmsConnection = connectionFactory.createConnection(
            jmsConfig.getUsername(),
            jmsConfig.getPassword()
        );
        jmsConnection.start();

        // Use transacted session for reliability
        jmsSession = jmsConnection.createSession(true, Session.SESSION_TRANSACTED);

        Destination requestQueue = jmsSession.createQueue(jmsConfig.getRequestQueue());
        messageProducer = jmsSession.createProducer(requestQueue);
        messageProducer.setDeliveryMode(DeliveryMode.PERSISTENT);

        log.info("JMS connection established: queue={}", jmsConfig.getRequestQueue());
    }

    @Override
    public void processElement(Payment payment, Context ctx, Collector<Payment> out)
            throws Exception {

        String transactionId = payment.getTransactionId();

        try {
            // Build and send JMS request
            SanctionsRequest request = buildSanctionsRequest(payment);
            sendJmsRequest(payment, request);

            // Update payment status
            payment.setSanctionsStatus(SanctionsStatus.PENDING);
            payment.setSanctionsRequestId(request.getRequestId());
            payment.setSanctionsRequestTime(LocalDateTime.now());
            payment.addProcessingStep("SANCTIONS_REQUEST", "JMS_SENT");

            requestsSent.inc();

            log.info("Sanctions request sent: transactionId={}, requestId={}",
                transactionId, request.getRequestId());

            // Emit for Kafka pending sink
            out.collect(payment);

        } catch (JMSException e) {
            log.error("Failed to send JMS request: transactionId={}", transactionId, e);

            sendFailures.inc();

            payment.setSanctionsStatus(SanctionsStatus.ERROR);
            payment.setFailureReason("JMS_SEND_FAILED");
            payment.setErrorMessage(e.getMessage());
            payment.addProcessingStep("SANCTIONS_REQUEST", "JMS_FAILED: " + e.getMessage());

            // Route to side output
            ctx.output(JMS_SEND_FAILED, payment);
        }
    }

    private SanctionsRequest buildSanctionsRequest(Payment payment) {
        return SanctionsRequest.builder()
            .requestId(generateRequestId())
            .transactionId(payment.getTransactionId())
            .debtorName(payment.getDebtorName())
            .debtorAccount(payment.getDebitAccount())
            .debtorCountry(payment.getDebtorCountry())
            .creditorName(payment.getCreditorName())
            .creditorAccount(payment.getCreditAccount())
            .creditorCountry(payment.getCreditorCountry())
            .amount(payment.getAmount())
            .currency(payment.getCurrency())
            .build();
    }

    private void sendJmsRequest(Payment payment, SanctionsRequest request)
            throws JMSException {

        String xmlMessage = new SanctionsRequestSerializer().serialize(request);

        TextMessage jmsMessage = jmsSession.createTextMessage(xmlMessage);
        jmsMessage.setStringProperty("TransactionId", payment.getTransactionId());
        jmsMessage.setStringProperty("RequestId", request.getRequestId());
        jmsMessage.setJMSCorrelationID(payment.getTransactionId());

        messageProducer.send(jmsMessage);

        // Commit the transacted session
        jmsSession.commit();

        log.debug("JMS message committed: transactionId={}", payment.getTransactionId());
    }

    private String generateRequestId() {
        return "SAN-" + System.currentTimeMillis() + "-" +
            getRuntimeContext().getIndexOfThisSubtask();
    }

    @Override
    public void close() throws Exception {
        if (messageProducer != null) {
            messageProducer.close();
        }
        if (jmsSession != null) {
            jmsSession.close();
        }
        if (jmsConnection != null) {
            jmsConnection.close();
        }
        super.close();

        log.info("JMS resources closed");
    }
}
```

## Payment Key Serializer

```java
package com.example.fednow.serialization;

import org.apache.flink.connector.kafka.sink.KafkaRecordSerializationSchema;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.nio.charset.StandardCharsets;

/**
 * Serializes Payment transactionId as Kafka key for partitioning.
 *
 * <p>This ensures that pending payments and responses for the same
 * transaction go to the same partition in Job 2.
 */
public class PaymentKeySerializer
    implements KafkaRecordSerializationSchema<String> {

    @Override
    public ProducerRecord<byte[], byte[]> serialize(
            String json,
            KafkaSinkContext context,
            Long timestamp) {

        // Extract transactionId from JSON for key
        String transactionId = extractTransactionId(json);

        return new ProducerRecord<>(
            null, // topic set by builder
            null, // partition (let Kafka decide based on key)
            transactionId.getBytes(StandardCharsets.UTF_8),
            json.getBytes(StandardCharsets.UTF_8)
        );
    }

    private String extractTransactionId(String json) {
        // Simple extraction - use Jackson in production
        int start = json.indexOf("\"transactionId\":\"") + 17;
        int end = json.indexOf("\"", start);
        return json.substring(start, end);
    }
}
```

## Configuration Extensions

```java
// Add to FlinkConfig.java

/** Kafka topic for pending payments (between Job 1 and Job 2) */
private String pendingPaymentsTopic;

/** Kafka topic for sanctions responses (from JMS bridge) */
private String sanctionsResponsesTopic;

public static FlinkConfig load(String[] args) {
    ParameterTool params = ParameterTool.fromArgs(args);

    return FlinkConfig.builder()
        // ... existing config ...
        .pendingPaymentsTopic(params.get(
            "kafka.topics.pending",
            "fednow.payments.pending-sanctions"))
        .sanctionsResponsesTopic(params.get(
            "kafka.topics.sanctions-responses",
            "fednow.sanctions.responses"))
        .build();
}
```

## Kafka Topics for Two-Job Architecture

```yaml
# New topics required
kafka:
  topics:
    # Job 1 writes, Job 2 reads
    pending-sanctions: fednow.payments.pending-sanctions

    # JMS Bridge writes, Job 2 reads
    sanctions-responses: fednow.sanctions.responses

    # Partitioning: by transactionId (same key = same partition)
    # Retention: 7 days (longer than sanctions timeout)
```

## Best Practices

### JMS Sender
- Use transacted sessions for reliability
- Commit after each send (or batch)
- Handle connection failures with reconnection logic
- Set persistent delivery mode

### Kafka Pending Topic
- Use transactionId as key for co-partitioning
- Set retention longer than max sanctions timeout
- Consider compaction for space efficiency

### Monitoring
- Track JMS send latency
- Alert on send failures
- Monitor pending topic lag

### Deployment
- Deploy Job 1 before Job 2
- Scale independently based on throughput
- Use separate consumer groups
