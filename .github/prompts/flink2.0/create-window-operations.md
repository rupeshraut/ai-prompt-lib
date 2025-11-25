# Create Window Operations

Create windowed stream operations for aggregating data over time periods or element counts. Windows divide infinite streams into finite buckets for processing.

## Requirements

Your window operations should include:

- **Window Types**: Implement tumbling, sliding, and session windows
- **Window Functions**: Use WindowFunction, AggregateFunction, ReduceFunction with windows
- **Time Windows**: Time-based windows with various semantics (event time, processing time)
- **Count Windows**: Count-based windows for batch aggregations
- **Custom Windows**: Implement custom window assigner if needed
- **Triggers**: Custom triggers for early firing or late emissions
- **Evictors**: Evict elements before/after window application
- **Side Outputs**: Handle late arrivals separately
- **Aggregation**: Combine/reduce elements within windows
- **Output Tags**: Multiple output streams for different window results
- **Documentation**: JavaDoc explaining window semantics and timing

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **Approach**: Modern DataStream API (no legacy DataSet)
- **Annotations**: Use modern annotations and sealed classes
- **Error Handling**: Proper exception handling in window functions

## Template Example

```java
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.functions.windowing.WindowFunction;
import org.apache.flink.streaming.api.functions.windowing.AggregateFunction;
import org.apache.flink.streaming.api.windowing.assigners.*;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.windows.Window;
import org.apache.flink.streaming.api.windowing.windows.TimeWindow;
import org.apache.flink.api.common.functions.AggregateFunction;
import org.apache.flink.util.Collector;
import lombok.extern.slf4j.Slf4j;
import java.time.Instant;
import java.util.*;

/**
 * Demonstrates various window operations in Apache Flink 2.0.
 * Windows divide infinite streams into finite buckets for processing.
 */
@Slf4j
public class WindowOperations {

    /**
     * Tumbling Window: Non-overlapping, fixed-size windows.
     * Example: Aggregate metrics every 60 seconds.
     */
    public static DataStream<MetricWindow> tumblingWindowAggregation(
            DataStream<Metric> metrics) {

        return metrics
                .keyBy(Metric::sourceId)
                .window(TumblingEventTimeWindows.of(Time.seconds(60)))
                .aggregate(new MetricAggregator())
                .doOnNext(window -> log.info("Tumbling window completed: {} - {}",
                        window.startTime(), window.endTime()));
    }

    /**
     * Sliding Window: Overlapping windows with slide and size parameters.
     * Example: 5-minute window sliding every 1 minute.
     */
    public static DataStream<WindowResult> slidingWindowAnalysis(
            DataStream<SensorReading> readings) {

        return readings
                .keyBy(SensorReading::sensorId)
                .window(SlidingEventTimeWindows.of(
                        Time.minutes(5),     // Window size
                        Time.minutes(1)))    // Slide interval
                .aggregate(new SensorAggregator())
                .doOnNext(result -> log.debug("Sliding window: {} readings", result.count()));
    }

    /**
     * Session Window: Groups elements with gaps of inactivity.
     * Example: Group user events with 10-minute inactivity gap.
     */
    public static DataStream<SessionData> sessionWindowGrouping(
            DataStream<UserEvent> events) {

        return events
                .keyBy(UserEvent::userId)
                .window(EventTimeSessionWindows.withGap(Time.minutes(10)))
                .process(new SessionWindowProcessFunction())
                .doOnNext(session -> log.info("Session window: {} events for user {}",
                        session.eventCount(), session.userId()));
    }

    /**
     * Count Window: Groups fixed number of elements.
     * Example: Process every 100 events as a batch.
     */
    public static DataStream<BatchResult> countWindowBatching(
            DataStream<Record> records) {

        return records
                .keyBy(Record::partitionKey)
                .countWindow(100)  // Window of 100 elements
                .aggregate(new CountAggregator())
                .doOnNext(result -> log.debug("Count window: {} elements processed",
                        result.elementCount()));
    }

    /**
     * Processing Time Window: Uses local machine time instead of event time.
     * Useful for monitoring and alerting on live systems.
     */
    public static DataStream<LiveMetrics> processingTimeWindow(
            DataStream<SystemMetric> metrics) {

        return metrics
                .keyBy(SystemMetric::hostname)
                .window(TumblingProcessingTimeWindows.of(Time.seconds(30)))
                .aggregate(new ProcessingTimeMetricsAggregator())
                .doOnNext(m -> log.info("Live metrics for {}: cpu={}%, mem={}%",
                        m.hostname(), m.cpuUsage(), m.memoryUsage()));
    }

    /**
     * Complex Aggregate Function: Computes multiple statistics in single pass.
     */
    public static DataStream<StatisticsWindow> complexAggregation(
            DataStream<DataPoint> data) {

        return data
                .keyBy(DataPoint::category)
                .window(TumblingEventTimeWindows.of(Time.seconds(60)))
                .aggregate(new StatisticsAggregator())
                .doOnNext(stats -> log.info("Stats - min: {}, max: {}, avg: {}",
                        stats.min(), stats.max(), stats.average()));
    }

    /**
     * Window with Early Trigger: Fire window before watermark arrives.
     * Useful for low-latency results at cost of accuracy.
     */
    public static DataStream<PartialResult> earlyFiringWindow(
            DataStream<Event> events) {

        return events
                .keyBy(Event::type)
                .window(TumblingEventTimeWindows.of(Time.minutes(1)))
                .trigger(org.apache.flink.streaming.api.windowing.triggers.
                        ContinuousEventTimeTrigger.of(Time.seconds(10)))  // Fire every 10 seconds
                .aggregate(new EventAggregator())
                .doOnNext(result -> log.debug("Early-fired window result: {}",
                        result.count()));
    }

    /**
     * Side Outputs for Late Data: Route late arrivals separately.
     */
    public static class LateDataHandlingWindow {
        static final OutputTag<LateEvent> lateDataTag = new OutputTag<LateEvent>("late-data") {};

        public static SingleOutputStreamOperator<WindowedEvent> processWithLateSideOutput(
                DataStream<Event> events) {

            return events
                    .keyBy(Event::type)
                    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
                    .allowedLateness(Time.minutes(1))  // Allow 1 minute late
                    .sideOutputLateData(lateDataTag)
                    .aggregate(new SimpleAggregator())
                    .doOnNext(event -> log.info("Windowed event: {}", event.id()));
        }

        public static DataStream<LateEvent> extractLateData(
                SingleOutputStreamOperator<WindowedEvent> mainStream) {

            return mainStream
                    .getSideOutput(lateDataTag)
                    .doOnNext(late -> log.warn("Late event received: {} (arrived {} ms late)",
                            late.id(), late.delayMs()));
        }
    }

    /**
     * Custom Window Assigner: Assign windows based on custom logic.
     */
    public static DataStream<CustomWindowResult> customWindowAssigner(
            DataStream<TimestampedData> data) {

        return data
                .keyBy(TimestampedData::key)
                .window(new HourlyAlignedWindowAssigner())
                .aggregate(new CustomAggregator())
                .doOnNext(result -> log.debug("Custom window result: {}", result.id()));
    }

    /**
     * Evictor: Remove elements before window processing.
     * Example: Keep only top 10 elements by score.
     */
    public static DataStream<TopElementsWindow> tumblingWindowWithEvictor(
            DataStream<ScoredItem> items) {

        return items
                .keyBy(ScoredItem::category)
                .window(TumblingEventTimeWindows.of(Time.seconds(60)))
                .evictor(new TopKEvictor(10))  // Keep top 10 by score
                .aggregate(new ItemAggregator())
                .doOnNext(window -> log.info("Window: {} top items", window.itemCount()));
    }

    /**
     * Reduce Function: Stateless window aggregation.
     */
    public static DataStream<Sale> reduceWindowFunction(
            DataStream<Sale> sales) {

        return sales
                .keyBy(Sale::storeId)
                .window(TumblingEventTimeWindows.of(Time.hours(1)))
                .reduce((sale1, sale2) -> new Sale(
                        sale1.storeId(),
                        sale1.amount() + sale2.amount(),
                        Math.max(sale1.timestamp(), sale2.timestamp())))
                .doOnNext(total -> log.info("Hourly store total: ${}", total.amount()));
    }

    /**
     * Process Window Function: Full access to window contents and context.
     */
    public static DataStream<DetailedWindowResult> processWindowFunction(
            DataStream<Order> orders) {

        return orders
                .keyBy(Order::customerId)
                .window(TumblingEventTimeWindows.of(Time.days(1)))
                .process(new DetailedOrderProcessor())
                .doOnNext(result -> log.info("Daily orders: {}, total: ${}, avg: ${}",
                        result.orderCount(), result.totalAmount(), result.averageAmount()));
    }

    /**
     * Global Window: Single window for entire stream (use with care).
     * Only suitable with custom trigger.
     */
    public static DataStream<StreamStatistics> globalWindowWithTrigger(
            DataStream<Value> values) {

        return values
                .windowAll(GlobalWindows.create())
                .trigger(org.apache.flink.streaming.api.windowing.triggers.
                        CountTrigger.of(1000))  // Fire every 1000 elements
                .aggregate(new GlobalAggregator())
                .doOnNext(stats -> log.info("Running stats: {} elements processed",
                        stats.totalCount()));
    }

    // Helper Classes
    public record Metric(String sourceId, long timestamp, double value) {}
    public record SensorReading(String sensorId, long timestamp, double temperature) {}
    public record UserEvent(String userId, long timestamp, String eventType) {}
    public record Record(String partitionKey, String data) {}
    public record SystemMetric(String hostname, long timestamp, double cpu, double memory) {}
    public record DataPoint(String category, double value, long timestamp) {}
    public record Event(String type, long timestamp, String payload) {}
    public record TimestampedData(String key, long timestamp, String data) {}
    public record ScoredItem(String category, String item, double score) {}
    public record Sale(String storeId, double amount, long timestamp) {}
    public record Order(String customerId, double amount, long timestamp) {}
    public record Value(double value) {}

    // Window Result Classes
    public record MetricWindow(String sourceId, long startTime, long endTime, double average, long count) {}
    public record WindowResult(String sensorId, double avgTemp, int count, long windowEnd) {}
    public record SessionData(String userId, int eventCount, long sessionStart, long sessionEnd) {}
    public record BatchResult(String partitionKey, int elementCount, double aggregateValue) {}
    public record LiveMetrics(String hostname, double cpuUsage, double memoryUsage) {}
    public record StatisticsWindow(double min, double max, double average, long count) {}
    public record PartialResult(int count, double sum) {}
    public record LateEvent(String id, long delayMs) {}
    public record CustomWindowResult(String id, String data) {}
    public record TopElementsWindow(int itemCount, List<ScoredItem> topItems) {}
    public record WindowedEvent(String id, long windowTime) {}
    public record DetailedWindowResult(String customerId, int orderCount, double totalAmount, double averageAmount) {}
    public record StreamStatistics(long totalCount, double average) {}

    // Aggregate Functions
    private static class MetricAggregator implements AggregateFunction<Metric, MetricAccumulator, MetricWindow> {
        @Override
        public MetricAccumulator createAccumulator() {
            return new MetricAccumulator();
        }

        @Override
        public MetricAccumulator add(Metric metric, MetricAccumulator acc) {
            acc.sum += metric.value();
            acc.count++;
            acc.sourceId = metric.sourceId();
            return acc;
        }

        @Override
        public MetricWindow getResult(MetricAccumulator acc) {
            return new MetricWindow(acc.sourceId, 0, 0, acc.sum / acc.count, acc.count);
        }

        @Override
        public MetricAccumulator merge(MetricAccumulator acc1, MetricAccumulator acc2) {
            acc1.sum += acc2.sum;
            acc1.count += acc2.count;
            return acc1;
        }
    }

    private static class MetricAccumulator {
        String sourceId;
        double sum = 0;
        long count = 0;
    }

    private static class SensorAggregator implements AggregateFunction<SensorReading, SensorAccumulator, WindowResult> {
        @Override
        public SensorAccumulator createAccumulator() {
            return new SensorAccumulator();
        }

        @Override
        public SensorAccumulator add(SensorReading reading, SensorAccumulator acc) {
            acc.sum += reading.temperature();
            acc.count++;
            acc.sensorId = reading.sensorId();
            return acc;
        }

        @Override
        public WindowResult getResult(SensorAccumulator acc) {
            return new WindowResult(acc.sensorId, acc.sum / acc.count, (int) acc.count, System.currentTimeMillis());
        }

        @Override
        public SensorAccumulator merge(SensorAccumulator acc1, SensorAccumulator acc2) {
            acc1.sum += acc2.sum;
            acc1.count += acc2.count;
            return acc1;
        }
    }

    private static class SensorAccumulator {
        String sensorId;
        double sum = 0;
        long count = 0;
    }

    // Window Process Functions
    private static class SessionWindowProcessFunction extends ProcessWindowFunction<UserEvent, SessionData, String, TimeWindow> {
        @Override
        public void process(String userId, ProcessWindowFunction<UserEvent, SessionData, String, TimeWindow>.Context context,
                           Iterable<UserEvent> elements, Collector<SessionData> out) {
            List<UserEvent> events = new ArrayList<>();
            elements.forEach(events::add);

            out.collect(new SessionData(
                    userId,
                    events.size(),
                    context.window().getStart(),
                    context.window().getEnd()
            ));
        }
    }

    private static class ProcessingTimeMetricsAggregator implements AggregateFunction<SystemMetric, MetricsAccumulator, LiveMetrics> {
        @Override
        public MetricsAccumulator createAccumulator() {
            return new MetricsAccumulator();
        }

        @Override
        public MetricsAccumulator add(SystemMetric metric, MetricsAccumulator acc) {
            acc.cpuSum += metric.cpu();
            acc.memorySum += metric.memory();
            acc.count++;
            acc.hostname = metric.hostname();
            return acc;
        }

        @Override
        public LiveMetrics getResult(MetricsAccumulator acc) {
            return new LiveMetrics(acc.hostname, acc.cpuSum / acc.count, acc.memorySum / acc.count);
        }

        @Override
        public MetricsAccumulator merge(MetricsAccumulator acc1, MetricsAccumulator acc2) {
            acc1.cpuSum += acc2.cpuSum;
            acc1.memorySum += acc2.memorySum;
            acc1.count += acc2.count;
            return acc1;
        }
    }

    private static class MetricsAccumulator {
        String hostname;
        double cpuSum = 0, memorySum = 0;
        long count = 0;
    }

    // Additional helper functions and implementations
    private static class StatisticsAggregator implements AggregateFunction<DataPoint, StatsAccumulator, StatisticsWindow> {
        @Override
        public StatsAccumulator createAccumulator() {
            return new StatsAccumulator();
        }

        @Override
        public StatsAccumulator add(DataPoint point, StatsAccumulator acc) {
            acc.min = Math.min(acc.min, point.value());
            acc.max = Math.max(acc.max, point.value());
            acc.sum += point.value();
            acc.count++;
            return acc;
        }

        @Override
        public StatisticsWindow getResult(StatsAccumulator acc) {
            return new StatisticsWindow(acc.min, acc.max, acc.sum / acc.count, acc.count);
        }

        @Override
        public StatsAccumulator merge(StatsAccumulator acc1, StatsAccumulator acc2) {
            acc1.min = Math.min(acc1.min, acc2.min);
            acc1.max = Math.max(acc1.max, acc2.max);
            acc1.sum += acc2.sum;
            acc1.count += acc2.count;
            return acc1;
        }
    }

    private static class StatsAccumulator {
        double min = Double.MAX_VALUE, max = Double.MIN_VALUE, sum = 0;
        long count = 0;
    }

    private static class CountAggregator implements AggregateFunction<Record, CountAccumulator, BatchResult> {
        @Override
        public CountAccumulator createAccumulator() {
            return new CountAccumulator();
        }

        @Override
        public CountAccumulator add(Record record, CountAccumulator acc) {
            acc.count++;
            acc.partitionKey = record.partitionKey();
            return acc;
        }

        @Override
        public BatchResult getResult(CountAccumulator acc) {
            return new BatchResult(acc.partitionKey, (int) acc.count, 0);
        }

        @Override
        public CountAccumulator merge(CountAccumulator acc1, CountAccumulator acc2) {
            acc1.count += acc2.count;
            return acc1;
        }
    }

    private static class CountAccumulator {
        String partitionKey;
        long count = 0;
    }

    // Placeholder implementations for brevity
    private static class EventAggregator implements AggregateFunction<Event, EventAccumulator, PartialResult> {
        @Override
        public EventAccumulator createAccumulator() { return new EventAccumulator(); }
        @Override
        public EventAccumulator add(Event event, EventAccumulator acc) { acc.count++; return acc; }
        @Override
        public PartialResult getResult(EventAccumulator acc) { return new PartialResult(acc.count, 0); }
        @Override
        public EventAccumulator merge(EventAccumulator acc1, EventAccumulator acc2) { acc1.count += acc2.count; return acc1; }
    }
    private static class EventAccumulator { long count = 0; }

    private static class SimpleAggregator implements AggregateFunction<Event, SimpleAccumulator, WindowedEvent> {
        @Override
        public SimpleAccumulator createAccumulator() { return new SimpleAccumulator(); }
        @Override
        public SimpleAccumulator add(Event event, SimpleAccumulator acc) { return acc; }
        @Override
        public WindowedEvent getResult(SimpleAccumulator acc) { return new WindowedEvent("", System.currentTimeMillis()); }
        @Override
        public SimpleAccumulator merge(SimpleAccumulator acc1, SimpleAccumulator acc2) { return acc1; }
    }
    private static class SimpleAccumulator { }

    private static class CustomAggregator implements AggregateFunction<TimestampedData, CustomAccumulator, CustomWindowResult> {
        @Override
        public CustomAccumulator createAccumulator() { return new CustomAccumulator(); }
        @Override
        public CustomAccumulator add(TimestampedData data, CustomAccumulator acc) { return acc; }
        @Override
        public CustomWindowResult getResult(CustomAccumulator acc) { return new CustomWindowResult("", ""); }
        @Override
        public CustomAccumulator merge(CustomAccumulator acc1, CustomAccumulator acc2) { return acc1; }
    }
    private static class CustomAccumulator { }

    private static class HourlyAlignedWindowAssigner extends WindowAssigner<TimestampedData, TimeWindow> {
        @Override
        public Collection<TimeWindow> assignWindows(TimestampedData element, long timestamp, WindowAssignerContext context) {
            long hourStart = (timestamp / 3600000) * 3600000;
            return Collections.singleton(new TimeWindow(hourStart, hourStart + 3600000));
        }

        @Override
        public Trigger<TimestampedData, TimeWindow> getDefaultTrigger(StreamExecutionEnvironment env) {
            return EventTimeTrigger.create();
        }

        @Override
        public String toString() { return "HourlyAlignedWindows()"; }
    }

    private static class TopKEvictor implements Evictor<ScoredItem, TimeWindow> {
        private final int k;

        TopKEvictor(int k) { this.k = k; }

        @Override
        public void evictBefore(Iterable<TimestampedValue<ScoredItem>> elements, int size, TimeWindow window, EvictorContext ctx) {}

        @Override
        public void evictAfter(Iterable<TimestampedValue<ScoredItem>> elements, int size, TimeWindow window, EvictorContext ctx) {
            List<TimestampedValue<ScoredItem>> sorted = new ArrayList<>();
            elements.forEach(sorted::add);
            sorted.sort((a, b) -> Double.compare(b.getValue().score(), a.getValue().score()));

            if (sorted.size() > k) {
                sorted.stream().skip(k).forEach(item -> {
                    // Remove lower-ranked items
                });
            }
        }
    }

    private static class ItemAggregator implements AggregateFunction<ScoredItem, ItemAccumulator, TopElementsWindow> {
        @Override
        public ItemAccumulator createAccumulator() { return new ItemAccumulator(); }
        @Override
        public ItemAccumulator add(ScoredItem item, ItemAccumulator acc) { acc.items.add(item); return acc; }
        @Override
        public TopElementsWindow getResult(ItemAccumulator acc) { return new TopElementsWindow(acc.items.size(), acc.items); }
        @Override
        public ItemAccumulator merge(ItemAccumulator acc1, ItemAccumulator acc2) { acc1.items.addAll(acc2.items); return acc1; }
    }
    private static class ItemAccumulator { List<ScoredItem> items = new ArrayList<>(); }

    private static class DetailedOrderProcessor extends ProcessWindowFunction<Order, DetailedWindowResult, String, TimeWindow> {
        @Override
        public void process(String customerId, ProcessWindowFunction<Order, DetailedWindowResult, String, TimeWindow>.Context context,
                           Iterable<Order> elements, Collector<DetailedWindowResult> out) {
            List<Order> orders = new ArrayList<>();
            double total = 0;
            int count = 0;

            for (Order order : elements) {
                orders.add(order);
                total += order.amount();
                count++;
            }

            out.collect(new DetailedWindowResult(
                    customerId,
                    count,
                    total,
                    count > 0 ? total / count : 0
            ));
        }
    }

    private static class GlobalAggregator implements AggregateFunction<Value, GlobalAccumulator, StreamStatistics> {
        @Override
        public GlobalAccumulator createAccumulator() { return new GlobalAccumulator(); }
        @Override
        public GlobalAccumulator add(Value value, GlobalAccumulator acc) { acc.count++; acc.sum += value.value(); return acc; }
        @Override
        public StreamStatistics getResult(GlobalAccumulator acc) { return new StreamStatistics(acc.count, acc.sum / acc.count); }
        @Override
        public GlobalAccumulator merge(GlobalAccumulator acc1, GlobalAccumulator acc2) { acc1.count += acc2.count; acc1.sum += acc2.sum; return acc1; }
    }
    private static class GlobalAccumulator { long count = 0; double sum = 0; }
}
```

## Window Types Summary

| Type | Use Case | Characteristics |
|------|----------|-----------------|
| **Tumbling** | Hourly reports, batch processing | Non-overlapping, fixed size |
| **Sliding** | Moving averages, trend detection | Overlapping, configurable slide |
| **Session** | User sessions, activity clustering | Gap-based, dynamic size |
| **Count** | Batch operations, fixed batches | Count-based, not time-based |

## Related Documentation

- [Flink Window API](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/operators/windows/)
- [Event Time & Watermarks](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/concepts/time/)
- [Triggers & Evictors](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/operators/windows/#triggers)
