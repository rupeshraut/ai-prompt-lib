# Create Flink Job

Create an Apache Flink 2.0 streaming job for payment processing.

## Requirements

The Flink job should include:
- `StreamExecutionEnvironment` configuration
- Checkpointing with exactly-once semantics
- RocksDB state backend configuration
- Kafka source and sink connectors
- Pipeline topology with operators
- Side outputs for error handling
- Metrics registration

## Code Style

- Use Java 17 features
- Follow Flink naming conventions
- Use builder patterns for configuration
- Add comprehensive logging
- Include JavaDoc for the job class

## Job Template

```java
package com.example.fednow.job;

import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.restartstrategy.RestartStrategies;
import org.apache.flink.api.common.time.Time;
import org.apache.flink.connector.kafka.sink.KafkaSink;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.contrib.streaming.state.EmbeddedRocksDBStateBackend;
import org.apache.flink.streaming.api.CheckpointingMode;
import org.apache.flink.streaming.api.datastream.AsyncDataStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.OutputTag;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.TimeUnit;

/**
 * Main Flink job for FedNow payment processing.
 *
 * <p>Processing Pipeline:
 * <ol>
 *   <li>Consume pacs.008 messages from Kafka</li>
 *   <li>Validate accounts via HTTP</li>
 *   <li>Perform risk control checks via HTTP</li>
 *   <li>Screen for sanctions via JMS</li>
 *   <li>Post transactions via HTTP</li>
 *   <li>Produce pacs.002 responses to Kafka</li>
 * </ol>
 */
@Slf4j
public class PaymentProcessingJob {

    // Side output tags for error handling
    public static final OutputTag<Payment> VALIDATION_FAILED =
        new OutputTag<>("validation-failed") {};
    public static final OutputTag<Payment> RISK_CONTROL_FAILED =
        new OutputTag<>("risk-control-failed") {};
    public static final OutputTag<Payment> SANCTIONS_HIT =
        new OutputTag<>("sanctions-hit") {};
    public static final OutputTag<Payment> POSTING_FAILED =
        new OutputTag<>("posting-failed") {};

    public static void main(String[] args) throws Exception {
        // Load configuration
        FlinkConfig config = FlinkConfig.load(args);

        // Create execution environment
        StreamExecutionEnvironment env = createExecutionEnvironment(config);

        // Build and execute pipeline
        buildPipeline(env, config);

        // Execute job
        env.execute(config.getJobName());
    }

    /**
     * Configure the Flink execution environment.
     */
    private static StreamExecutionEnvironment createExecutionEnvironment(FlinkConfig config) {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // Set parallelism
        env.setParallelism(config.getParallelism());

        // Configure checkpointing
        env.enableCheckpointing(config.getCheckpointInterval());
        env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
        env.getCheckpointConfig().setMinPauseBetweenCheckpoints(config.getMinPauseBetweenCheckpoints());
        env.getCheckpointConfig().setCheckpointTimeout(config.getCheckpointTimeout());
        env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);
        env.getCheckpointConfig().setTolerableCheckpointFailureNumber(3);

        // Configure state backend
        env.setStateBackend(new EmbeddedRocksDBStateBackend(true));
        env.getCheckpointConfig().setCheckpointStorage(config.getCheckpointDir());

        // Configure restart strategy
        env.setRestartStrategy(RestartStrategies.fixedDelayRestart(
            3,                          // max attempts
            Time.seconds(10)            // delay between attempts
        ));

        log.info("Flink environment configured: parallelism={}, checkpoint={}ms",
            config.getParallelism(), config.getCheckpointInterval());

        return env;
    }

    /**
     * Build the payment processing pipeline.
     */
    private static void buildPipeline(StreamExecutionEnvironment env, FlinkConfig config) {
        // Step 1: Create Kafka source
        KafkaSource<String> kafkaSource = createKafkaSource(config);

        DataStream<String> rawMessages = env
            .fromSource(kafkaSource, WatermarkStrategy.noWatermarks(), "kafka-source")
            .name("Kafka Source")
            .uid("kafka-source");

        // Step 2: Parse ISO 20022 messages
        DataStream<Payment> payments = rawMessages
            .map(new Iso20022ParserFunction())
            .name("Parse ISO 20022")
            .uid("parse-iso20022");

        // Step 3: Account Validation (Async HTTP)
        SingleOutputStreamOperator<Payment> validated = AsyncDataStream.unorderedWait(
            payments,
            new AccountValidationAsyncFunction(config.getAccountValidationUrl()),
            config.getAccountValidationTimeout(),
            TimeUnit.MILLISECONDS,
            100  // capacity
        )
        .name("Account Validation")
        .uid("account-validation");

        // Step 4: Risk Control (Async HTTP)
        SingleOutputStreamOperator<Payment> riskChecked = AsyncDataStream.unorderedWait(
            validated,
            new RiskControlAsyncFunction(config.getRiskControlUrl()),
            config.getRiskControlTimeout(),
            TimeUnit.MILLISECONDS,
            100
        )
        .name("Risk Control")
        .uid("risk-control");

        // Step 5: Sanctions Screening (Keyed Process with JMS)
        SingleOutputStreamOperator<Payment> sanctionsChecked = riskChecked
            .keyBy(Payment::getTransactionId)
            .process(new SanctionsProcessFunction(config.getJmsConfig()))
            .name("Sanctions Screening")
            .uid("sanctions-screening");

        // Step 6: Account Posting (Async HTTP)
        SingleOutputStreamOperator<Payment> posted = AsyncDataStream.unorderedWait(
            sanctionsChecked,
            new AccountPostingAsyncFunction(config.getTmsUrl()),
            config.getTmsTimeout(),
            TimeUnit.MILLISECONDS,
            100
        )
        .name("Account Posting")
        .uid("account-posting");

        // Step 7: Generate pacs.002 response
        DataStream<String> responses = posted
            .map(new Pacs002GeneratorFunction())
            .name("Generate Response")
            .uid("generate-response");

        // Step 8: Sink to Kafka
        KafkaSink<String> kafkaSink = createKafkaSink(config);
        responses.sinkTo(kafkaSink)
            .name("Kafka Sink")
            .uid("kafka-sink");

        // Handle side outputs (failures)
        handleSideOutputs(validated, riskChecked, sanctionsChecked, posted, config);
    }

    /**
     * Handle side outputs for different failure types.
     */
    private static void handleSideOutputs(
            SingleOutputStreamOperator<Payment> validated,
            SingleOutputStreamOperator<Payment> riskChecked,
            SingleOutputStreamOperator<Payment> sanctionsChecked,
            SingleOutputStreamOperator<Payment> posted,
            FlinkConfig config) {

        // Validation failures -> DLQ
        validated.getSideOutput(VALIDATION_FAILED)
            .map(new FailedPaymentSerializer("VALIDATION_FAILED"))
            .sinkTo(createDlqSink(config, "validation-failed"))
            .name("DLQ: Validation Failed")
            .uid("dlq-validation-failed");

        // Risk control failures -> DLQ
        riskChecked.getSideOutput(RISK_CONTROL_FAILED)
            .map(new FailedPaymentSerializer("RISK_CONTROL_FAILED"))
            .sinkTo(createDlqSink(config, "risk-control-failed"))
            .name("DLQ: Risk Control Failed")
            .uid("dlq-risk-control-failed");

        // Sanctions hits -> DLQ
        sanctionsChecked.getSideOutput(SANCTIONS_HIT)
            .map(new FailedPaymentSerializer("SANCTIONS_HIT"))
            .sinkTo(createDlqSink(config, "sanctions-hit"))
            .name("DLQ: Sanctions Hit")
            .uid("dlq-sanctions-hit");

        // Posting failures -> DLQ
        posted.getSideOutput(POSTING_FAILED)
            .map(new FailedPaymentSerializer("POSTING_FAILED"))
            .sinkTo(createDlqSink(config, "posting-failed"))
            .name("DLQ: Posting Failed")
            .uid("dlq-posting-failed");
    }

    /**
     * Create Kafka source for inbound messages.
     */
    private static KafkaSource<String> createKafkaSource(FlinkConfig config) {
        return KafkaSource.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setTopics(config.getInputTopic())
            .setGroupId(config.getConsumerGroupId())
            .setStartingOffsets(OffsetsInitializer.committedOffsets(OffsetResetStrategy.EARLIEST))
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();
    }

    /**
     * Create Kafka sink for outbound responses.
     */
    private static KafkaSink<String> createKafkaSink(FlinkConfig config) {
        return KafkaSink.<String>builder()
            .setBootstrapServers(config.getKafkaBootstrapServers())
            .setRecordSerializer(KafkaRecordSerializationSchema.builder()
                .setTopic(config.getOutputTopic())
                .setValueSerializationSchema(new SimpleStringSchema())
                .build())
            .setDeliveryGuarantee(DeliveryGuarantee.EXACTLY_ONCE)
            .setTransactionalIdPrefix(config.getTransactionalIdPrefix())
            .build();
    }

    /**
     * Create Kafka sink for dead letter queue.
     */
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

## Configuration Class

```java
package com.example.fednow.config;

import lombok.Builder;
import lombok.Data;
import org.apache.flink.api.java.utils.ParameterTool;

@Data
@Builder
public class FlinkConfig {
    private String jobName;
    private int parallelism;
    private long checkpointInterval;
    private long minPauseBetweenCheckpoints;
    private long checkpointTimeout;
    private String checkpointDir;

    private String kafkaBootstrapServers;
    private String consumerGroupId;
    private String inputTopic;
    private String outputTopic;
    private String dlqTopicPrefix;
    private String transactionalIdPrefix;

    private String accountValidationUrl;
    private long accountValidationTimeout;
    private String riskControlUrl;
    private long riskControlTimeout;
    private String tmsUrl;
    private long tmsTimeout;

    private JmsConfig jmsConfig;

    public static FlinkConfig load(String[] args) {
        ParameterTool params = ParameterTool.fromArgs(args);

        return FlinkConfig.builder()
            .jobName(params.get("job.name", "fednow-payment-processor"))
            .parallelism(params.getInt("parallelism", 4))
            .checkpointInterval(params.getLong("checkpoint.interval", 60000))
            .minPauseBetweenCheckpoints(params.getLong("checkpoint.min-pause", 30000))
            .checkpointTimeout(params.getLong("checkpoint.timeout", 120000))
            .checkpointDir(params.get("checkpoint.dir", "file:///tmp/flink-checkpoints"))
            .kafkaBootstrapServers(params.get("kafka.bootstrap-servers", "localhost:9092"))
            .consumerGroupId(params.get("kafka.consumer.group-id", "fednow-processor"))
            .inputTopic(params.get("kafka.topics.input", "fednow.payments.inbound"))
            .outputTopic(params.get("kafka.topics.output", "fednow.payments.outbound"))
            .dlqTopicPrefix(params.get("kafka.topics.dlq-prefix", "fednow.payments.dlq"))
            .transactionalIdPrefix(params.get("kafka.transactional-id-prefix", "fednow-tx"))
            .accountValidationUrl(params.get("services.account-validation.url"))
            .accountValidationTimeout(params.getLong("services.account-validation.timeout", 10000))
            .riskControlUrl(params.get("services.risk-control.url"))
            .riskControlTimeout(params.getLong("services.risk-control.timeout", 15000))
            .tmsUrl(params.get("services.tms.url"))
            .tmsTimeout(params.getLong("services.tms.timeout", 20000))
            .jmsConfig(JmsConfig.fromParams(params))
            .build();
    }
}
```

## Best Practices

### Operator UID
- Always set `.uid()` for every operator
- UIDs are required for savepoint compatibility
- Use descriptive, stable names

### Checkpointing
- Enable exactly-once for financial transactions
- Configure appropriate intervals based on throughput
- Use incremental checkpoints with RocksDB

### Parallelism
- Set per-operator parallelism for bottleneck operators
- Match parallelism to Kafka partition count for sources
- Consider downstream system capacity

### Error Handling
- Use side outputs for different failure types
- Send failures to dedicated DLQ topics
- Include failure metadata for debugging

### Metrics
- Register custom metrics for business monitoring
- Track latency at each processing step
- Monitor backpressure indicators
