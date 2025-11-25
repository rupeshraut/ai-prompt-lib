# Create Flink Testing

Create comprehensive testing strategies for Apache Flink 2.0 jobs including unit tests, integration tests, and end-to-end testing. Proper testing ensures job correctness, performance, and fault tolerance.

## Requirements

Your Flink testing implementation should include:

- **Unit Tests**: Test individual operators and functions in isolation
- **Test Harness**: Use Flink test harnesses for operator testing
- **Mini Cluster**: Integration tests with MiniClusterWithClientResource
- **Test Utilities**: MockSourceFunction, CollectSink for controlled testing
- **State Testing**: Validate state consistency and recovery
- **Checkpoint Testing**: Verify checkpoint creation and recovery
- **Time Testing**: Test event time, processing time, watermarks
- **Error Injection**: Test failure scenarios and recovery
- **Test Containers**: Integration tests with Kafka, databases using Testcontainers
- **Performance Testing**: Benchmark throughput and latency
- **Property-Based Testing**: Use QuickCheck-style testing for edge cases
- **Documentation**: Testing best practices and patterns

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0, JUnit 5, AssertJ
- **Approach**: Use test harness for unit tests, mini cluster for integration
- **Test Isolation**: Each test should be independent
- **Assertions**: Use AssertJ for fluent assertions
- **Mocking**: Use Mockito for external dependencies

## Template Example

```java
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.streaming.api.functions.sink.SinkFunction;
import org.apache.flink.test.util.MiniClusterWithClientResource;
import org.apache.flink.runtime.testutils.MiniClusterResourceConfiguration;
import org.apache.flink.streaming.api.operators.*;
import org.apache.flink.streaming.util.*;
import org.apache.flink.util.Collector;
import org.junit.jupiter.api.*;
import org.assertj.core.api.Assertions;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * Demonstrates comprehensive testing strategies for Apache Flink 2.0 jobs.
 * Covers unit tests, integration tests, state testing, and fault tolerance validation.
 */
@Slf4j
public class FlinkTestingExamples {

    // ==================== Unit Tests with Test Harness ====================

    /**
     * Test stateless map function.
     */
    @Test
    public void testStatelessMapFunction() throws Exception {
        // Create test harness for map operator
        OneInputStreamOperatorTestHarness<Event, EnrichedEvent> testHarness =
                new OneInputStreamOperatorTestHarness<>(
                        new StreamMap<>(new EventEnrichmentFunction())
                );

        testHarness.open();

        // Process test data
        testHarness.processElement(new Event("1", "click", 1000L), 1000L);
        testHarness.processElement(new Event("2", "view", 2000L), 2000L);

        // Verify output
        List<EnrichedEvent> output = new ArrayList<>();
        testHarness.getOutput().forEach(record -> {
            output.add(record.getValue());
        });

        Assertions.assertThat(output)
                .hasSize(2)
                .extracting(EnrichedEvent::enrichedField)
                .containsExactly("CLICK", "VIEW");

        testHarness.close();
    }

    /**
     * Test stateful process function with ValueState.
     */
    @Test
    public void testStatefulProcessFunction() throws Exception {
        // Create test harness
        KeyedOneInputStreamOperatorTestHarness<String, Transaction, Alert> testHarness =
                new KeyedOneInputStreamOperatorTestHarness<>(
                        new KeyedProcessOperator<>(new FraudDetectionFunction()),
                        Transaction::userId,
                        TypeInformation.of(String.class)
                );

        testHarness.open();

        // Process transactions for same user
        testHarness.processElement(new Transaction("user1", 100.0, 1000L), 1000L);
        testHarness.processElement(new Transaction("user1", 200.0, 2000L), 2000L);
        testHarness.processElement(new Transaction("user1", 5000.0, 3000L), 3000L);

        // Verify fraud alert was triggered
        List<Alert> output = extractOutput(testHarness);

        Assertions.assertThat(output)
                .hasSize(1)
                .first()
                .extracting(Alert::reason)
                .isEqualTo("Large transaction");

        testHarness.close();
    }

    /**
     * Test window operator with event time.
     */
    @Test
    public void testWindowOperator() throws Exception {
        KeyedOneInputStreamOperatorTestHarness<String, Event, WindowResult> testHarness =
                createWindowTestHarness();

        testHarness.open();

        // Add events in first window (0-5000ms)
        testHarness.processElement(new Event("user1", "click", 1000L), 1000L);
        testHarness.processElement(new Event("user1", "view", 2000L), 2000L);

        // Add watermark to trigger window
        testHarness.processWatermark(new Watermark(5000L));

        // Verify window output
        List<WindowResult> output = extractOutput(testHarness);

        Assertions.assertThat(output)
                .hasSize(1)
                .first()
                .satisfies(result -> {
                    Assertions.assertThat(result.count()).isEqualTo(2);
                    Assertions.assertThat(result.userId()).isEqualTo("user1");
                });

        testHarness.close();
    }

    /**
     * Test broadcast state function.
     */
    @Test
    public void testBroadcastStateFunction() throws Exception {
        MapStateDescriptor<String, Rule> ruleDescriptor = new MapStateDescriptor<>(
                "rules",
                TypeInformation.of(String.class),
                TypeInformation.of(Rule.class)
        );

        TwoInputStreamOperatorTestHarness<Event, Rule, ProcessedEvent> testHarness =
                new TwoInputStreamOperatorTestHarness<>(
                        new CoBroadcastWithKeyedOperator<>(
                                new RuleBasedProcessor(),
                                List.of(ruleDescriptor)
                        )
                );

        testHarness.open();

        // Process broadcast rule
        testHarness.processBroadcastElement(new Rule("rule1", "FILTER"), 1000L);

        // Process regular events
        testHarness.processElement1(new Event("1", "click", 2000L), 2000L);
        testHarness.processElement1(new Event("2", "view", 3000L), 3000L);

        List<ProcessedEvent> output = extractOutput(testHarness);

        Assertions.assertThat(output)
                .hasSize(2)
                .allMatch(e -> e.ruleApplied().equals("rule1"));

        testHarness.close();
    }

    // ==================== Integration Tests with Mini Cluster ====================

    private static final MiniClusterWithClientResource flinkCluster =
            new MiniClusterWithClientResource(
                    new MiniClusterResourceConfiguration.Builder()
                            .setNumberSlotsPerTaskManager(2)
                            .setNumberTaskManagers(1)
                            .build()
            );

    @BeforeAll
    public static void setupCluster() {
        flinkCluster.before();
    }

    @AfterAll
    public static void teardownCluster() {
        flinkCluster.after();
    }

    /**
     * Integration test with mini cluster.
     */
    @Test
    public void testJobWithMiniCluster() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(2);

        // Create test data source
        List<Event> testData = Arrays.asList(
                new Event("1", "click", 1000L),
                new Event("2", "view", 2000L),
                new Event("3", "click", 3000L)
        );

        // Collect sink for verification
        List<EnrichedEvent> results = new CopyOnWriteArrayList<>();

        DataStream<Event> events = env.fromCollection(testData);
        events.map(new EventEnrichmentFunction())
                .addSink(new CollectSink<>(results));

        env.execute("Test Job");

        // Verify results
        Assertions.assertThat(results)
                .hasSize(3)
                .extracting(EnrichedEvent::enrichedField)
                .containsExactlyInAnyOrder("CLICK", "VIEW", "CLICK");
    }

    /**
     * Test checkpoint creation and recovery.
     */
    @Test
    public void testCheckpointRecovery() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.enableCheckpointing(1000);
        env.setParallelism(1);

        // Source with checkpoint support
        CountingSource source = new CountingSource(100);
        List<Long> results = new CopyOnWriteArrayList<>();

        env.addSource(source)
                .map(x -> x * 2)
                .addSink(new CollectSink<>(results));

        env.execute("Checkpoint Test");

        // Verify state was checkpointed
        Assertions.assertThat(source.getCheckpointCount()).isGreaterThan(0);
        Assertions.assertThat(results).hasSize(100);
    }

    /**
     * Test watermark propagation.
     */
    @Test
    public void testWatermarkPropagation() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        List<Long> watermarks = new CopyOnWriteArrayList<>();

        env.fromElements(
                new Event("1", "click", 1000L),
                new Event("2", "view", 2000L),
                new Event("3", "click", 3000L)
        )
        .assignTimestampsAndWatermarks(
                WatermarkStrategy.<Event>forMonotonousTimestamps()
                        .withTimestampAssigner((event, timestamp) -> event.timestamp())
        )
        .process(new WatermarkCapturingFunction(watermarks))
        .addSink(new DiscardingSink<>());

        env.execute("Watermark Test");

        Assertions.assertThat(watermarks)
                .isNotEmpty()
                .isSorted();
    }

    // ==================== State Testing ====================

    /**
     * Test state snapshot and restore.
     */
    @Test
    public void testStateSnapshotAndRestore() throws Exception {
        KeyedOneInputStreamOperatorTestHarness<String, Event, Result> testHarness =
                new KeyedOneInputStreamOperatorTestHarness<>(
                        new KeyedProcessOperator<>(new StatefulFunction()),
                        Event::userId,
                        TypeInformation.of(String.class)
                );

        testHarness.open();

        // Process some data
        testHarness.processElement(new Event("user1", "click", 1000L), 1000L);
        testHarness.processElement(new Event("user1", "view", 2000L), 2000L);

        // Take snapshot
        OperatorSubtaskState snapshot = testHarness.snapshot(1L, 2000L);

        testHarness.close();

        // Create new harness and restore state
        KeyedOneInputStreamOperatorTestHarness<String, Event, Result> restoredHarness =
                new KeyedOneInputStreamOperatorTestHarness<>(
                        new KeyedProcessOperator<>(new StatefulFunction()),
                        Event::userId,
                        TypeInformation.of(String.class)
                );

        restoredHarness.setup();
        restoredHarness.initializeState(snapshot);
        restoredHarness.open();

        // Process more data - should continue from restored state
        restoredHarness.processElement(new Event("user1", "click", 3000L), 3000L);

        List<Result> output = extractOutput(restoredHarness);

        // Verify state was restored correctly (count should be 3, not 1)
        Assertions.assertThat(output)
                .hasSize(1)
                .first()
                .extracting(Result::count)
                .isEqualTo(3);

        restoredHarness.close();
    }

    // ==================== Test Containers Integration ====================

    /**
     * Integration test with Kafka using Testcontainers.
     */
    @Test
    public void testWithKafkaContainer() throws Exception {
        try (KafkaContainer kafka = new KafkaContainer(
                DockerImageName.parse("confluentinc/cp-kafka:7.4.0"))) {

            kafka.start();

            StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
            env.setParallelism(1);

            // Configure Kafka source
            KafkaSource<Event> source = KafkaSource.<Event>builder()
                    .setBootstrapServers(kafka.getBootstrapServers())
                    .setTopics("test-topic")
                    .setValueOnlyDeserializer(new EventDeserializer())
                    .build();

            // Configure Kafka sink
            KafkaSink<Event> sink = KafkaSink.<Event>builder()
                    .setBootstrapServers(kafka.getBootstrapServers())
                    .setRecordSerializer(new EventSerializer("output-topic"))
                    .build();

            // Run job
            env.fromSource(source, WatermarkStrategy.noWatermarks(), "Kafka Source")
                    .map(e -> new Event(e.id(), e.type().toUpperCase(), e.timestamp()))
                    .sinkTo(sink);

            // Produce test messages to Kafka
            produceTestMessages(kafka.getBootstrapServers(), "test-topic");

            // Execute job
            env.execute("Kafka Integration Test");

            // Verify output
            List<Event> consumed = consumeMessages(kafka.getBootstrapServers(), "output-topic");
            Assertions.assertThat(consumed).isNotEmpty();
        }
    }

    // ==================== Performance Testing ====================

    /**
     * Benchmark throughput test.
     */
    @Test
    public void testThroughputPerformance() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(4);

        long startTime = System.currentTimeMillis();
        int recordCount = 1_000_000;

        ThroughputMeasuringSink sink = new ThroughputMeasuringSink();

        env.addSource(new NumberSequenceSource(0, recordCount))
                .map(x -> x * 2)
                .addSink(sink);

        env.execute("Throughput Test");

        long duration = System.currentTimeMillis() - startTime;
        double throughput = recordCount / (duration / 1000.0);

        log.info("Processed {} records in {}ms ({} records/sec)",
                recordCount, duration, throughput);

        Assertions.assertThat(throughput).isGreaterThan(10000);
    }

    // ==================== Helper Classes ====================

    private static class EventEnrichmentFunction implements MapFunction<Event, EnrichedEvent> {
        @Override
        public EnrichedEvent map(Event event) {
            return new EnrichedEvent(
                    event.id(),
                    event.type(),
                    event.type().toUpperCase(),
                    event.timestamp()
            );
        }
    }

    private static class FraudDetectionFunction extends KeyedProcessFunction<String, Transaction, Alert> {
        private transient ValueState<Double> totalAmountState;

        @Override
        public void open(Configuration parameters) {
            totalAmountState = getRuntimeContext().getState(
                    new ValueStateDescriptor<>("total", Double.class)
            );
        }

        @Override
        public void processElement(Transaction txn, Context ctx, Collector<Alert> out) throws Exception {
            Double total = totalAmountState.value();
            if (total == null) total = 0.0;

            total += txn.amount();
            totalAmountState.update(total);

            // Alert on large transaction
            if (txn.amount() > 1000) {
                out.collect(new Alert(txn.userId(), "Large transaction", txn.amount()));
            }
        }
    }

    private static class CollectSink<T> implements SinkFunction<T> {
        private final List<T> results;

        CollectSink(List<T> results) {
            this.results = results;
        }

        @Override
        public void invoke(T value, Context context) {
            results.add(value);
        }
    }

    private static class CountingSource implements SourceFunction<Long>, CheckpointedFunction {
        private final long count;
        private volatile boolean running = true;
        private long current = 0;
        private int checkpointCount = 0;
        private transient ListState<Long> state;

        CountingSource(long count) {
            this.count = count;
        }

        @Override
        public void run(SourceContext<Long> ctx) throws Exception {
            while (running && current < count) {
                synchronized (ctx.getCheckpointLock()) {
                    ctx.collect(current++);
                }
                Thread.sleep(1);
            }
        }

        @Override
        public void cancel() {
            running = false;
        }

        @Override
        public void snapshotState(FunctionSnapshotContext context) throws Exception {
            state.clear();
            state.add(current);
            checkpointCount++;
        }

        @Override
        public void initializeState(FunctionInitializationContext context) throws Exception {
            state = context.getOperatorStateStore().getListState(
                    new ListStateDescriptor<>("current", Long.class)
            );

            if (context.isRestored()) {
                for (Long l : state.get()) {
                    current = l;
                }
            }
        }

        public int getCheckpointCount() {
            return checkpointCount;
        }
    }

    private static class WatermarkCapturingFunction extends ProcessFunction<Event, Event> {
        private final List<Long> watermarks;

        WatermarkCapturingFunction(List<Long> watermarks) {
            this.watermarks = watermarks;
        }

        @Override
        public void processElement(Event value, Context ctx, Collector<Event> out) {
            watermarks.add(ctx.timestamp());
            out.collect(value);
        }
    }

    private static <T> List<T> extractOutput(AbstractStreamOperatorTestHarness<?, ?> harness) {
        List<T> output = new ArrayList<>();
        harness.getOutput().forEach(record -> {
            if (record.isRecord()) {
                output.add((T) record.getValue());
            }
        });
        return output;
    }

    // ==================== Data Models ====================

    public record Event(String id, String type, long timestamp) {
        public String userId() { return id; }
    }

    public record EnrichedEvent(String id, String type, String enrichedField, long timestamp) {}

    public record Transaction(String userId, double amount, long timestamp) {}

    public record Alert(String userId, String reason, double amount) {}

    public record WindowResult(String userId, int count, long windowEnd) {}

    public record Rule(String id, String type) {}

    public record ProcessedEvent(String id, String type, String ruleApplied) {}

    public record Result(String userId, int count) {}
}
```

## Test Configuration

```xml
<!-- pom.xml test dependencies -->
<dependencies>
    <!-- Flink Test Utilities -->
    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-test-utils</artifactId>
        <version>2.0.0</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-runtime</artifactId>
        <version>2.0.0</version>
        <type>test-jar</type>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.apache.flink</groupId>
        <artifactId>flink-streaming-java</artifactId>
        <version>2.0.0</version>
        <type>test-jar</type>
        <scope>test</scope>
    </dependency>

    <!-- JUnit 5 -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.0</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.24.2</version>
        <scope>test</scope>
    </dependency>

    <!-- Testcontainers -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>testcontainers</artifactId>
        <version>1.19.0</version>
        <scope>test</scope>
    </dependency>

    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>kafka</artifactId>
        <version>1.19.0</version>
        <scope>test</scope>
    </dependency>

    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.5.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## JUnit Configuration

```properties
# junit-platform.properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4
```

## Test Utilities

```java
/**
 * Common test utilities for Flink testing.
 */
public class FlinkTestUtils {

    /**
     * Create test environment with standard configuration.
     */
    public static StreamExecutionEnvironment createTestEnv() {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);
        env.setRestartStrategy(RestartStrategies.noRestart());
        return env;
    }

    /**
     * Assert collection contains elements in any order.
     */
    public static <T> void assertContainsInAnyOrder(Collection<T> actual, T... expected) {
        Assertions.assertThat(actual)
                .containsExactlyInAnyOrder(expected);
    }

    /**
     * Wait for condition with timeout.
     */
    public static void waitForCondition(Supplier<Boolean> condition, Duration timeout) {
        long deadline = System.currentTimeMillis() + timeout.toMillis();
        while (System.currentTimeMillis() < deadline) {
            if (condition.get()) {
                return;
            }
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException(e);
            }
        }
        throw new AssertionError("Condition not met within timeout");
    }
}
```

## Best Practices

1. **Test Isolation**: Each test should be completely independent. Use fresh environments and clean state.

2. **Test Harness First**: Use test harness for unit testing operators. It's faster and more focused than mini cluster tests.

3. **Deterministic Tests**: Avoid time-dependent tests. Use controlled time (ManualClock) or explicit watermarks.

4. **State Testing**: Always test state snapshot and restore. Verify state is correctly checkpointed and recovered.

5. **Watermark Testing**: Test watermark generation and propagation explicitly. Verify late data handling.

6. **Error Scenarios**: Test failure cases: missing data, malformed input, checkpoint failures, restarts.

7. **Integration Tests**: Use Testcontainers for integration tests with Kafka, databases. Avoid hardcoded ports/hosts.

8. **Performance Tests**: Include performance benchmarks in CI. Track throughput and latency over time.

9. **Test Coverage**: Aim for 80%+ code coverage. Cover edge cases: empty windows, late arrivals, state cleanup.

10. **Parallel Testing**: Run tests in parallel for faster feedback. Use JUnit 5 parallel execution.

## Related Documentation

- [Flink Testing](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/testing/)
- [Test Harness](https://nightlies.apache.org/flink/flink-docs-release-2.0/api/java/org/apache/flink/streaming/util/AbstractStreamOperatorTestHarness.html)
- [MiniCluster](https://nightlies.apache.org/flink/flink-docs-release-2.0/api/java/org/apache/flink/test/util/MiniClusterWithClientResource.html)
- [Testcontainers](https://www.testcontainers.org/)
- [AssertJ](https://assertj.github.io/doc/)
