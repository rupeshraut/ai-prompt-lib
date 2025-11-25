# Create Event-Time Processing

Create event-time semantics in Apache Flink using watermarks for handling late and out-of-order data. Event-time processing is essential for accurate windowed aggregations.

## Requirements

Your event-time implementation should include:

- **Watermark Strategies**: Implement watermark assigner strategies for event-time progression
- **Out-of-Order Handling**: Process late-arriving events correctly
- **Allowed Lateness**: Configure windows to accept events arriving after window close
- **Watermark Idleness**: Handle idle sources that don't emit watermarks
- **Custom Watermarks**: Create custom watermark logic for specific patterns
- **Side Output Late Data**: Route late events to separate streams
- **Timestamp Assignment**: Extract and assign event timestamps from data
- **Lag Monitoring**: Track and report watermark lag
- **Timestamp Skew**: Handle clock skew between distributed systems
- **Documentation**: Clear explanation of event-time semantics

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0 with modern APIs
- **Watermark Strategies**: Use WatermarkStrategy interface
- **Handling**: Proper timestamp and watermark lifecycle management

## Template Example

```java
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.functions.timestamps.BoundedOutOfOrdernessTimestampExtractor;
import org.apache.flink.streaming.api.watermark.Watermark;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.streaming.api.watermark.WatermarkStrategy;
import org.apache.flink.streaming.api.watermark.WatermarkGenerator;
import org.apache.flink.streaming.api.watermark.WatermarkOutput;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;
import lombok.extern.slf4j.Slf4j;
import java.time.Instant;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Demonstrates event-time processing with watermarks in Apache Flink 2.0.
 * Event-time semantics allow correct handling of late and out-of-order data.
 */
@Slf4j
public class EventTimeProcessing {

    /**
     * Simple watermark strategy: Perfect watermarks (no late data expected).
     * Use for: Ordered data sources where all events arrive on-time.
     */
    public static DataStream<Event> perfectWatermarks(DataStream<String> source) {
        return source
                .map(EventTimeProcessing::parseEvent)
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.<Event>forMonotonousTimestamps()
                                .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                )
                .doOnNext(event -> log.debug("Event: {} at {}", event.id(), event.timestamp()));
    }

    /**
     * Bounded out-of-orderness watermark: Allow bounded delay of late events.
     * Use for: Networks with variable latency but bounded delay.
     * Example: Allow events up to 5 seconds late.
     */
    public static DataStream<Event> boundedOutOfOrderWatermarks(DataStream<String> source) {
        return source
                .map(EventTimeProcessing::parseEvent)
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.<Event>forBoundedOutOfOrderness(
                                java.time.Duration.ofSeconds(5))  // Max 5 seconds late
                                .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                )
                .doOnNext(event -> log.info("Event with bounded OoO: {}", event.id()));
    }

    /**
     * Custom watermark strategy: Fine-grained control over watermark generation.
     * Use for: Complex scenarios requiring custom watermark logic.
     */
    public static DataStream<Event> customWatermarkStrategy(DataStream<String> source) {
        return source
                .map(EventTimeProcessing::parseEvent)
                .assignTimestampsAndWatermarks(
                        new WatermarkStrategy<Event>() {
                            @Override
                            public WatermarkGenerator<Event> createWatermarkGenerator(
                                    WatermarkGeneratorSupplier.Context context) {
                                return new CustomWatermarkGenerator();
                            }

                            @Override
                            public TimestampAssigner<Event> createTimestampAssigner(
                                    TimestampAssignerSupplier.Context context) {
                                return (event, recordTimestamp) -> event.timestamp();
                            }
                        }.withIdleness(java.time.Duration.ofSeconds(30))  // Idle timeout
                )
                .doOnNext(event -> log.debug("Custom watermark event: {}", event.id()));
    }

    /**
     * Session gap watermark: Watermarks progress based on event activity.
     * Use for: User session analysis, event clustering.
     */
    public static DataStream<SessionEvent> sessionGapWatermarks(DataStream<String> source) {
        return source
                .map(EventTimeProcessing::parseEvent)
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.<Event>forBoundedOutOfOrderness(
                                java.time.Duration.ofSeconds(2))
                                .withIdleness(java.time.Duration.ofSeconds(10))  // 10s idle = session end
                                .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                )
                .map(event -> new SessionEvent(event.sessionId(), event.timestamp()))
                .doOnNext(event -> log.info("Session event: {}", event.sessionId()));
    }

    /**
     * Handling late arrivals: Accept and process events after window close.
     * Use allowedLateness() to extend window computation period.
     */
    public static DataStream<WindowResult> handlingLateArrivals(DataStream<Event> events) {
        return events
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.<Event>forBoundedOutOfOrderness(
                                java.time.Duration.ofSeconds(2))
                                .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                )
                .keyBy(Event::sessionId)
                .window(TumblingEventTimeWindows.of(Time.minutes(1)))
                .allowedLateness(Time.seconds(30))  // Accept events 30s after window closes
                .process(new TimeWindowFunction())
                .doOnNext(result -> log.info("Window result: {} events", result.count()));
    }

    /**
     * Late data side output: Route late events to separate stream.
     * Use for: Separate analysis of late-arriving data.
     */
    public static class LateDataSideOutput {
        static final OutputTag<LateEvent> lateEventsTag = new OutputTag<LateEvent>("late-events") {};

        public static SingleOutputStreamOperator<WindowResult> processWithLateSideOutput(
                DataStream<Event> events) {

            return events
                    .assignTimestampsAndWatermarks(
                            WatermarkStrategy.<Event>forBoundedOutOfOrderness(
                                    java.time.Duration.ofSeconds(5))
                                    .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                    )
                    .keyBy(Event::sessionId)
                    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
                    .allowedLateness(Time.seconds(60))
                    .sideOutputLateData(lateEventsTag)
                    .process(new TimeWindowFunction())
                    .doOnNext(result -> log.info("On-time window: {}", result.count()));
        }

        public static DataStream<LateEvent> extractLateEvents(
                SingleOutputStreamOperator<WindowResult> mainStream) {
            return mainStream
                    .getSideOutput(lateEventsTag)
                    .doOnNext(late -> log.warn("Late event: {} arrived at {}", late.id(), late.arrivalTime()));
        }
    }

    /**
     * Monitoring watermark lag: Track how far behind events are from wall-clock time.
     * Use for: Alerting when processing falls behind.
     */
    public static DataStream<WatermarkLagMetric> monitorWatermarkLag(DataStream<Event> events) {
        return events
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.<Event>forBoundedOutOfOrderness(
                                java.time.Duration.ofSeconds(10))
                                .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                )
                .map(event -> {
                    long wallClockTime = System.currentTimeMillis();
                    long eventTime = event.timestamp();
                    long lag = wallClockTime - eventTime;

                    if (lag > 60000) {  // 1 minute
                        log.warn("High watermark lag: {} ms", lag);
                    }

                    return new WatermarkLagMetric(event.id(), lag, wallClockTime);
                })
                .doOnNext(metric -> log.debug("Watermark lag for {}: {} ms",
                        metric.eventId(), metric.lagMs()));
    }

    /**
     * Idleness handling: Gracefully handle sources that stop emitting.
     * Watermarks won't progress without events in source.
     */
    public static DataStream<Event> handlingIdleSources(DataStream<String> source) {
        return source
                .map(EventTimeProcessing::parseEvent)
                .assignTimestampsAndWatermarks(
                        WatermarkStrategy.<Event>forBoundedOutOfOrderness(
                                java.time.Duration.ofSeconds(5))
                                .withIdleness(java.time.Duration.ofSeconds(30))  // Force WM every 30s
                                .withTimestampAssigner((event, recordTimestamp) -> event.timestamp())
                )
                .doOnNext(event -> log.debug("Event from active source: {}", event.id()));
    }

    /**
     * Timestamp skew handling: Deal with clock differences across systems.
     * Use when different machines have unsynchronized clocks.
     */
    public static DataStream<Event> handlingTimestampSkew(DataStream<Event> events) {
        var maxSkew = java.time.Duration.ofSeconds(5);

        return events
                .assignTimestampsAndWatermarks(
                        new WatermarkStrategy<Event>() {
                            @Override
                            public WatermarkGenerator<Event> createWatermarkGenerator(
                                    WatermarkGeneratorSupplier.Context context) {
                                return new SkewTolerantWatermarkGenerator(maxSkew);
                            }

                            @Override
                            public TimestampAssigner<Event> createTimestampAssigner(
                                    TimestampAssignerSupplier.Context context) {
                                return (event, recordTimestamp) -> {
                                    // Cap timestamp at current time
                                    return Math.min(event.timestamp(), System.currentTimeMillis());
                                };
                            }
                        }
                )
                .doOnNext(event -> log.debug("Skew-tolerant event: {}", event.id()));
    }

    /**
     * Complex watermark strategy: Multiple watermarks for different partitions.
     * Use for: Multi-partition sources with varying progress.
     */
    public static DataStream<Event> multiPartitionWatermarks(DataStream<Event> events) {
        return events
                .assignTimestampsAndWatermarks(
                        new WatermarkStrategy<Event>() {
                            @Override
                            public WatermarkGenerator<Event> createWatermarkGenerator(
                                    WatermarkGeneratorSupplier.Context context) {
                                return new PerPartitionWatermarkGenerator();
                            }

                            @Override
                            public TimestampAssigner<Event> createTimestampAssigner(
                                    TimestampAssignerSupplier.Context context) {
                                return (event, recordTimestamp) -> event.timestamp();
                            }
                        }
                )
                .doOnNext(event -> log.debug("Multi-partition event: {}", event.id()));
    }

    /**
     * Processing time watermarks (not recommended but sometimes needed):
     * Falls back to processing time if event time is unavailable.
     */
    public static DataStream<Event> fallbackToProcessingTime(DataStream<Event> events) {
        return events
                .assignTimestampsAndWatermarks(
                        new WatermarkStrategy<Event>() {
                            @Override
                            public WatermarkGenerator<Event> createWatermarkGenerator(
                                    WatermarkGeneratorSupplier.Context context) {
                                return new FallbackWatermarkGenerator();
                            }

                            @Override
                            public TimestampAssigner<Event> createTimestampAssigner(
                                    TimestampAssignerSupplier.Context context) {
                                return (event, recordTimestamp) -> {
                                    if (event.hasEventTime()) {
                                        return event.timestamp();
                                    } else {
                                        return System.currentTimeMillis();
                                    }
                                };
                            }
                        }
                )
                .doOnNext(event -> log.debug("Fallback watermark event: {}", event.id()));
    }

    // Custom Watermark Generators
    private static class CustomWatermarkGenerator implements WatermarkGenerator<Event> {
        private long maxTimestamp = Long.MIN_VALUE;

        @Override
        public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
            maxTimestamp = Math.max(maxTimestamp, event.timestamp());
        }

        @Override
        public void onPeriodicEmit(WatermarkOutput output) {
            output.emitWatermark(new Watermark(maxTimestamp - 5000));
        }
    }

    private static class SkewTolerantWatermarkGenerator implements WatermarkGenerator<Event> {
        private final java.time.Duration maxSkew;
        private long maxTimestamp = Long.MIN_VALUE;

        SkewTolerantWatermarkGenerator(java.time.Duration maxSkew) {
            this.maxSkew = maxSkew;
        }

        @Override
        public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
            maxTimestamp = Math.max(maxTimestamp, event.timestamp());
        }

        @Override
        public void onPeriodicEmit(WatermarkOutput output) {
            long skewBuffer = maxSkew.toMillis();
            output.emitWatermark(new Watermark(Math.max(0, maxTimestamp - skewBuffer)));
        }
    }

    private static class PerPartitionWatermarkGenerator implements WatermarkGenerator<Event> {
        private final java.util.Map<String, Long> partitionMaxTimestamp = new java.util.HashMap<>();

        @Override
        public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
            String partition = event.partition();
            partitionMaxTimestamp.put(partition,
                    Math.max(partitionMaxTimestamp.getOrDefault(partition, 0L), event.timestamp()));
        }

        @Override
        public void onPeriodicEmit(WatermarkOutput output) {
            if (!partitionMaxTimestamp.isEmpty()) {
                long minTimestamp = partitionMaxTimestamp.values().stream()
                        .mapToLong(Long::longValue)
                        .min()
                        .orElse(0);
                output.emitWatermark(new Watermark(minTimestamp));
            }
        }
    }

    private static class FallbackWatermarkGenerator implements WatermarkGenerator<Event> {
        private long maxTimestamp = Long.MIN_VALUE;

        @Override
        public void onEvent(Event event, long eventTimestamp, WatermarkOutput output) {
            if (event.hasEventTime()) {
                maxTimestamp = Math.max(maxTimestamp, event.timestamp());
            } else {
                maxTimestamp = Math.max(maxTimestamp, System.currentTimeMillis());
            }
        }

        @Override
        public void onPeriodicEmit(WatermarkOutput output) {
            output.emitWatermark(new Watermark(maxTimestamp - 1000));
        }
    }

    // Helper classes and records
    public record Event(String id, String sessionId, long timestamp, String partition, String data) {
        public boolean hasEventTime() { return timestamp > 0; }
    }
    public record SessionEvent(String sessionId, long timestamp) {}
    public record WindowResult(int count, double sum, long windowEnd) {}
    public record LateEvent(String id, long arrivalTime) {}
    public record WatermarkLagMetric(String eventId, long lagMs, long wallClockTime) {}

    // Window processing function
    private static class TimeWindowFunction extends ProcessWindowFunction<Event, WindowResult, String, TimeWindow> {
        @Override
        public void process(String key, ProcessWindowFunction<Event, WindowResult, String, TimeWindow>.Context context,
                           Iterable<Event> elements, Collector<WindowResult> out) {
            int count = 0;
            double sum = 0;

            for (Event e : elements) {
                count++;
                // Process event...
            }

            out.collect(new WindowResult(count, sum, context.window().getEnd()));
        }
    }

    private static Event parseEvent(String line) {
        // Parse event from string
        String[] parts = line.split(",");
        return new Event(parts[0], parts[1], Long.parseLong(parts[2]), parts[3], parts[4]);
    }
}
```

## Event-Time Processing Strategies

| Strategy | Use Case | Benefits | Trade-offs |
|----------|----------|----------|-----------|
| **Monotonous** | Perfect ordering | Simple, no configuration | Fails with any late data |
| **Bounded OoO** | Common networks | Handles realistic delays | Must choose bound carefully |
| **Custom** | Complex patterns | Maximum control | Complex to implement |

## Related Documentation

- [Event Time & Watermarks](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/concepts/time/)
- [Watermark Strategies](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/event-time/generating_watermarks/)
- [Allowed Lateness](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/operators/windows/#allowed-lateness)
