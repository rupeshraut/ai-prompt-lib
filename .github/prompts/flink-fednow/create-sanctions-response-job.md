# Create Sanctions Response Job

Create Flink Job 2 for the two-job sanctions architecture: consumes JMS responses, correlates with pending payments from Kafka, and completes the payment processing flow.

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      Job 2: Sanctions Response Flow                             │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────────┐
│ JMS Response │───▶ JMS-to-Kafka Bridge (separate process)
│   Queue      │              │
└──────────────┘              ▼
                    ┌──────────────────┐
                    │  Kafka Responses │
                    │     Topic        │
                    └────────┬─────────┘
                             │
┌──────────────┐             │
│Kafka Pending │─────────────┼─────────▶ Correlate ───▶ Account ───▶ Kafka
│    Topic     │             │           (CoProcess)    Posting     Response
└──────────────┘─────────────┘                            │
                                                          ▼
                                                       [DLQ]
```

## Job 2: Sanctions Response Job

```java
package com.example.fednow.job;

import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.connector.kafka.sink.KafkaSink;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.AsyncDataStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.co.KeyedCoProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

/**
 * Job 2: Sanctions Response Flow
 *
 * <p>Correlates sanctions responses with pending payments and completes
 * the payment processing pipeline (account posting, response generation).
 *
 * <p>Processing Pipeline:
 * <ol>
 *   <li>Consume pending payments from Kafka (from Job 1)</li>
 *   <li>Consume sanctions responses from Kafka (from JMS bridge)</li>
 *   <li>Correlate by transactionId using CoProcessFunction</li>
 *   <li>Post accounts via HTTP on CLEAR status</li>
 *   <li>Generate pacs.002 response</li>
 *   <li>Produce response to Kafka</li>
 * </ol>
 */
@Slf4j
public class SanctionsResponseJob {

    // Side output tags
    public static final OutputTag<Payment> SANCTIONS_HIT =
        new OutputTag<>("sanctions-hit") {};
    public static final OutputTag<Payment> SANCTIONS_TIMEOUT =
        new OutputTag<>("sanctions-timeout") {};
    public static final OutputTag<Payment> POSTING_FAILED =
        new OutputTag<>("posting-failed") {};

    private static final long SANCTIONS_TIMEOUT_MS = 30_000; // 30 seconds

    public static void main(String[] args) throws Exception {
        FlinkConfig config = FlinkConfig.load(args);

        StreamExecutionEnvironment env = createExecutionEnvironment(config);
        buildPipeline(env, config);

        env.execute("FedNow Sanctions Response Job");
    }

    private static StreamExecutionEnvironment createExecutionEnvironment(FlinkConfig config) {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        env.setParallelism(config.getParallelism());

        // Checkpointing - needs to handle pending state
        env.enableCheckpointing(config.getCheckpointInterval());
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        env.getCheckpointConfig().setMinPauseBetweenCheckpoints(config.getMinPauseBetweenCheckpoints());
        env.getCheckpointConfig().setCheckpointTimeout(120000); // 2 min - longer for state
        env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
        env.getCheckpointConfig().setTolerableCheckpointFailureNumber(3);

        // RocksDB for correlation state
        env.setStateBackend(new EmbeddedRocksDBStateBackend(true));
        env.getCheckpointConfig().setCheckpointStorage(config.getCheckpointDir() + "/job2");

        // Restart strategy
        env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
            3, Time.seconds(10)
        ));

        log.info("Job 2 environment configured: parallelism={}", config.getParallelism());

        return env;
    }

    private static void buildPipeline(StreamExecutionEnvironment env, FlinkConfig config) {
        // Stream 1: Pending payments from Job 1
        KafkaSource<String> pendingSource = KafkaSource.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setTopics(config.getPendingPaymentsTopic())
            .setGroupId(config.getConsumerGroupId() + "-job2-pending")
            .setStartingOffsets(OffsetsInitializer.committedOffsets(OffsetResetStrategy.EARLIEST))
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        DataStream<Payment> pendingPayments = env
            .fromSource(pendingSource, WatermarkStrategy.noWatermarks(), "pending-source")
            .map(new JsonToPaymentFunction())
            .name("Pending Payments Source")
            .uid("pending-payments-source");

        // Stream 2: Sanctions responses from JMS bridge
        KafkaSource<String> responsesSource = KafkaSource.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setTopics(config.getSanctionsResponsesTopic())
            .setGroupId(config.getConsumerGroupId() + "-job2-responses")
            .setStartingOffsets(OffsetsInitializer.committedOffsets(OffsetResetStrategy.EARLIEST))
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        DataStream<SanctionsResponse> sanctionsResponses = env
            .fromSource(responsesSource, WatermarkStrategy.noWatermarks(), "responses-source")
            .map(new JsonToSanctionsResponseFunction())
            .name("Sanctions Responses Source")
            .uid("sanctions-responses-source");

        // Correlate payments with responses by transactionId
        SingleOutputStreamOperator<Payment> correlated = pendingPayments
            .keyBy(Payment::getTransactionId)
            .connect(sanctionsResponses.keyBy(SanctionsResponse::getTransactionId))
            .process(new SanctionsCorrelationFunction(SANCTIONS_TIMEOUT_MS))
            .name("Sanctions Correlation")
            .uid("sanctions-correlation");

        // Filter cleared payments for posting
        DataStream<Payment> clearedPayments = correlated
            .filter(p -> p.getSanctionsStatus() == SanctionsStatus.CLEAR);

        // Account Posting (Async HTTP)
        SingleOutputStreamOperator<Payment> posted = AsyncDataStream.unorderedWait(
            clearedPayments,
            new AccountPostingAsyncFunction(
                config.getTmsUrl(),
                config.getTmsTimeout()
            ),
            config.getTmsTimeout(),
            TimeUnit.MILLISECONDS,
            100
        )
        .name("Account Posting")
        .uid("account-posting");

        // Generate pacs.002 response
        DataStream<String> responses = posted
            .map(new Pacs002GeneratorFunction())
            .name("Generate Response")
            .uid("generate-response");

        // Sink to Kafka
        KafkaSink<String> kafkaSink = createResponseSink(config);
        responses.sinkTo(kafkaSink)
            .name("Kafka Response Sink")
            .uid("kafka-response-sink");

        // Handle side outputs
        handleSideOutputs(correlated, posted, config);

        log.info("Job 2 pipeline built successfully");
    }

    private static KafkaSink<String> createResponseSink(FlinkConfig config) {
        return KafkaSink.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setRecordSerializer(KafkaRecordSerializationSchema.builder()
                .setTopic(config.getOutputTopic())
                .setValueSerializationSchema(new SimpleStringSchema())
                .build())
            .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
            .setTransactionalIdPrefix(config.getTransactionalIdPrefix() + "-job2")
            .build();
    }

    private static void handleSideOutputs(
            SingleOutputStreamOperator<Payment> correlated,
            SingleOutputStreamOperator<Payment> posted,
            FlinkConfig config) {

        // Sanctions hits
        correlated.getSideOutput(SANCTIONS_HIT)
            .map(new FailedPaymentSerializer("SANCTIONS_HIT"))
            .sinkTo(createDlqSink(config, "sanctions-hit"))
            .name("DLQ: Sanctions Hit")
            .uid("dlq-sanctions-hit");

        // Sanctions timeouts
        correlated.getSideOutput(SANCTIONS_TIMEOUT)
            .map(new FailedPaymentSerializer("SANCTIONS_TIMEOUT"))
            .sinkTo(createDlqSink(config, "sanctions-timeout"))
            .name("DLQ: Sanctions Timeout")
            .uid("dlq-sanctions-timeout");

        // Posting failures
        posted.getSideOutput(POSTING_FAILED)
            .map(new FailedPaymentSerializer("POSTING_FAILED"))
            .sinkTo(createDlqSink(config, "posting-failed"))
            .name("DLQ: Posting Failed")
            .uid("dlq-posting-failed");
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

## Sanctions Correlation Function

```java
package com.example.fednow.function;

import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.co.KeyedCoProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;
import org.apache.flink.metrics.Counter;
import lombok.extern.slf4j.Slf4j;

/**
 * Correlates pending payments with sanctions responses.
 *
 * <p>Handles three scenarios:
 * <ol>
 *   <li>Payment arrives first: Store in state, register timeout timer</li>
 *   <li>Response arrives first: Store in state (rare but possible)</li>
 *   <li>Timeout: No response received within threshold</li>
 * </ol>
 */
@Slf4j
public class SanctionsCorrelationFunction
    extends KeyedCoProcessFunction<String, Payment, SanctionsResponse, Payment> {

    private final long timeoutMs;

    // State
    private transient ValueState<Payment> pendingPaymentState;
    private transient ValueState<SanctionsResponse> pendingResponseState;
    private transient ValueState<Long> timerState;

    // Metrics
    private transient Counter correlatedCount;
    private transient Counter timeoutCount;
    private transient Counter hitCount;
    private transient Counter clearCount;

    // Side outputs
    public static final OutputTag<Payment> SANCTIONS_HIT =
        new OutputTag<>("sanctions-hit") {};
    public static final OutputTag<Payment> SANCTIONS_TIMEOUT =
        new OutputTag<>("sanctions-timeout") {};

    public SanctionsCorrelationFunction(long timeoutMs) {
        this.timeoutMs = timeoutMs;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);

        pendingPaymentState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("pending-payment", TypeInformation.of(Payment.class))
        );

        pendingResponseState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("pending-response", TypeInformation.of(SanctionsResponse.class))
        );

        timerState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("timer", TypeInformation.of(Long.class))
        );

        correlatedCount = getRuntimeContext().getMetricGroup().counter("sanctions.correlated");
        timeoutCount = getRuntimeContext().getMetricGroup().counter("sanctions.timeout");
        hitCount = getRuntimeContext().getMetricGroup().counter("sanctions.hit");
        clearCount = getRuntimeContext().getMetricGroup().counter("sanctions.clear");

        log.info("SanctionsCorrelationFunction initialized: timeoutMs={}", timeoutMs);
    }

    @Override
    public void processElement1(Payment payment, Context ctx, Collector<Payment> out)
            throws Exception {

        String transactionId = ctx.getCurrentKey();

        log.debug("Received pending payment: transactionId={}", transactionId);

        // Check if response already arrived (rare but handle it)
        SanctionsResponse response = pendingResponseState.value();

        if (response != null) {
            // Response arrived before payment - process immediately
            log.debug("Response already present, correlating: transactionId={}", transactionId);

            processCorrelation(payment, response, ctx, out);

            // Clear state
            pendingResponseState.clear();

        } else {
            // Normal case: store payment and wait for response
            pendingPaymentState.update(payment);

            // Register timeout timer
            long timeoutTime = ctx.timerService().currentProcessingTime() + timeoutMs;
            ctx.timerService().registerProcessingTimeTimer(timeoutTime);
            timerState.update(timeoutTime);

            log.debug("Payment stored, timer registered: transactionId={}, timeout={}",
                transactionId, timeoutTime);
        }
    }

    @Override
    public void processElement2(SanctionsResponse response, Context ctx, Collector<Payment> out)
            throws Exception {

        String transactionId = ctx.getCurrentKey();

        log.debug("Received sanctions response: transactionId={}, result={}",
            transactionId, response.getResult());

        // Check if payment is waiting
        Payment payment = pendingPaymentState.value();

        if (payment != null) {
            // Normal case: payment waiting for response
            cancelTimer(ctx);
            processCorrelation(payment, response, ctx, out);

            // Clear state
            pendingPaymentState.clear();
            timerState.clear();

        } else {
            // Response arrived before payment (unlikely but handle it)
            log.warn("Response arrived before payment: transactionId={}", transactionId);
            pendingResponseState.update(response);

            // Set a shorter timer for orphan cleanup
            long cleanupTime = ctx.timerService().currentProcessingTime() + timeoutMs;
            ctx.timerService().registerProcessingTimeTimer(cleanupTime);
            timerState.update(cleanupTime);
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Payment> out)
            throws Exception {

        String transactionId = ctx.getCurrentKey();

        Payment payment = pendingPaymentState.value();
        SanctionsResponse response = pendingResponseState.value();

        if (payment != null && response == null) {
            // Timeout: no response received
            log.warn("Sanctions timeout: transactionId={}", transactionId);

            payment.setSanctionsStatus(SanctionsStatus.TIMEOUT);
            payment.setFailureReason("SANCTIONS_TIMEOUT");
            payment.addProcessingStep("SANCTIONS_CORRELATION", "TIMEOUT");

            timeoutCount.inc();

            ctx.output(SANCTIONS_TIMEOUT, payment);

        } else if (response != null && payment == null) {
            // Orphan response cleanup
            log.warn("Orphan response cleanup: transactionId={}", transactionId);
        }

        // Clear all state
        pendingPaymentState.clear();
        pendingResponseState.clear();
        timerState.clear();
    }

    private void processCorrelation(
            Payment payment,
            SanctionsResponse response,
            Context ctx,
            Collector<Payment> out) {

        correlatedCount.inc();

        switch (response.getResult()) {
            case CLEAR -> {
                payment.setSanctionsStatus(SanctionsStatus.CLEAR);
                payment.setSanctionsRef(response.getRequestId());
                payment.addProcessingStep("SANCTIONS_CORRELATION", "CLEAR");

                clearCount.inc();

                log.info("Sanctions cleared: transactionId={}", payment.getTransactionId());

                out.collect(payment);
            }

            case HIT -> {
                payment.setSanctionsStatus(SanctionsStatus.HIT);
                payment.setSanctionsHits(response.getHits());
                payment.setFailureReason("SANCTIONS_HIT");
                payment.addProcessingStep("SANCTIONS_CORRELATION",
                    "HIT: " + response.getHits().size() + " matches");

                hitCount.inc();

                log.warn("Sanctions hit: transactionId={}, hits={}",
                    payment.getTransactionId(), response.getHits().size());

                ctx.output(SANCTIONS_HIT, payment);
            }

            case POTENTIAL_HIT -> {
                payment.setSanctionsStatus(SanctionsStatus.POTENTIAL_HIT);
                payment.setSanctionsHits(response.getHits());
                payment.addProcessingStep("SANCTIONS_CORRELATION", "POTENTIAL_HIT");

                hitCount.inc();

                log.warn("Sanctions potential hit: transactionId={}",
                    payment.getTransactionId());

                ctx.output(SANCTIONS_HIT, payment);
            }

            case ERROR -> {
                payment.setSanctionsStatus(SanctionsStatus.ERROR);
                payment.setFailureReason("SANCTIONS_ERROR");
                payment.setErrorMessage(response.getResponseMessage());
                payment.addProcessingStep("SANCTIONS_CORRELATION",
                    "ERROR: " + response.getResponseMessage());

                log.error("Sanctions error: transactionId={}, message={}",
                    payment.getTransactionId(), response.getResponseMessage());

                ctx.output(SANCTIONS_TIMEOUT, payment);
            }
        }
    }

    private void cancelTimer(Context ctx) throws Exception {
        Long timerTimestamp = timerState.value();
        if (timerTimestamp != null) {
            ctx.timerService().deleteProcessingTimeTimer(timerTimestamp);
        }
    }
}
```

## JMS-to-Kafka Bridge

```java
package com.example.fednow.bridge;

import lombok.extern.slf4j.Slf4j;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;

import javax.jms.*;
import javax.naming.InitialContext;
import java.util.Properties;

/**
 * Standalone bridge that consumes JMS responses and produces to Kafka.
 *
 * <p>This runs as a separate process (not Flink) for:
 * <ul>
 *   <li>Independent scaling of JMS consumption</li>
 *   <li>Simpler JMS transaction handling</li>
 *   <li>Decoupling from Flink checkpointing</li>
 * </ul>
 *
 * <p>Alternative: Use Kafka Connect JMS Source Connector
 */
@Slf4j
public class JmsToKafkaBridge {

    private final JmsConfig jmsConfig;
    private final String kafkaBootstrapServers;
    private final String kafkaTopic;

    private volatile boolean running = true;
    private Connection jmsConnection;
    private Session jmsSession;
    private MessageConsumer messageConsumer;
    private KafkaProducer<String, String> kafkaProducer;

    public JmsToKafkaBridge(JmsConfig jmsConfig, String kafkaBootstrapServers, String kafkaTopic) {
        this.jmsConfig = jmsConfig;
        this.kafkaBootstrapServers = kafkaBootstrapServers;
        this.kafkaTopic = kafkaTopic;
    }

    public void start() throws Exception {
        initializeJms();
        initializeKafka();

        log.info("JMS-to-Kafka bridge started: jmsQueue={}, kafkaTopic={}",
            jmsConfig.getResponseQueue(), kafkaTopic);

        while (running) {
            try {
                Message message = messageConsumer.receive(1000);

                if (message instanceof TextMessage textMessage) {
                    processMessage(textMessage);
                }
            } catch (JMSException e) {
                log.error("JMS receive error", e);
                reconnectJms();
            }
        }
    }

    private void processMessage(TextMessage jmsMessage) throws Exception {
        String transactionId = jmsMessage.getStringProperty("TransactionId");
        String xmlContent = jmsMessage.getText();

        // Convert XML to JSON for Kafka
        SanctionsResponse response = new SanctionsResponseParser().parse(xmlContent);
        String json = new ObjectMapper().writeValueAsString(response);

        // Send to Kafka with transactionId as key
        ProducerRecord<String, String> record = new ProducerRecord<>(
            kafkaTopic,
            transactionId,
            json
        );

        kafkaProducer.send(record, (metadata, exception) -> {
            if (exception != null) {
                log.error("Failed to send to Kafka: transactionId={}", transactionId, exception);
            } else {
                log.debug("Sent to Kafka: transactionId={}, offset={}",
                    transactionId, metadata.offset());
            }
        });

        // Acknowledge JMS message after Kafka send
        jmsMessage.acknowledge();
    }

    private void initializeJms() throws Exception {
        ConnectionFactory cf = (ConnectionFactory)
            new InitialContext().lookup(jmsConfig.getConnectionFactoryName());

        jmsConnection = cf.createConnection(
            jmsConfig.getUsername(),
            jmsConfig.getPassword()
        );

        jmsSession = jmsConnection.createSession(false, Session.CLIENT_ACKNOWLEDGE);

        Destination queue = jmsSession.createQueue(jmsConfig.getResponseQueue());
        messageConsumer = jmsSession.createConsumer(queue);

        jmsConnection.start();
    }

    private void initializeKafka() {
        Properties props = new Properties();
        props.put("bootstrap.servers", kafkaBootstrapServers);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("acks", "all");
        props.put("enable.idempotence", "true");

        kafkaProducer = new KafkaProducer<>(props);
    }

    private void reconnectJms() {
        try {
            Thread.sleep(5000);
            close();
            initializeJms();
        } catch (Exception e) {
            log.error("Failed to reconnect JMS", e);
        }
    }

    public void stop() {
        running = false;
    }

    public void close() {
        try {
            if (messageConsumer != null) messageConsumer.close();
            if (jmsSession != null) jmsSession.close();
            if (jmsConnection != null) jmsConnection.close();
            if (kafkaProducer != null) kafkaProducer.close();
        } catch (Exception e) {
            log.error("Error closing resources", e);
        }
    }

    public static void main(String[] args) {
        JmsConfig jmsConfig = JmsConfig.fromArgs(args);
        String kafkaServers = System.getenv("KAFKA_BOOTSTRAP_SERVERS");
        String kafkaTopic = System.getenv("KAFKA_SANCTIONS_RESPONSES_TOPIC");

        JmsToKafkaBridge bridge = new JmsToKafkaBridge(jmsConfig, kafkaServers, kafkaTopic);

        Runtime.getRuntime().addShutdownHook(new Thread(bridge::stop));

        try {
            bridge.start();
        } catch (Exception e) {
            log.error("Bridge failed", e);
            System.exit(1);
        }
    }
}
```

## Best Practices

### Two-Job Coordination
- Use same Kafka key (transactionId) for co-partitioning
- Set appropriate retention on pending topic
- Monitor lag between Job 1 output and Job 2 processing

### State Management
- Keep correlation state minimal
- Set TTL on state to prevent memory growth
- Use RocksDB for large state volumes

### Timeout Handling
- Start timer from Job 1 JMS send time (passed in Payment)
- Account for Kafka latency in timeout calculation
- Consider business SLA requirements

### JMS Bridge
- Run multiple instances for high availability
- Use Kafka Connect as alternative
- Monitor JMS queue depth

### Deployment Order
1. Deploy JMS-to-Kafka bridge first
2. Deploy Job 2 (consumer of pending + responses)
3. Deploy Job 1 (producer of pending)
