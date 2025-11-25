# Create Monitoring Metrics

Create custom metrics in Apache Flink 2.0 for comprehensive job monitoring with Prometheus, Grafana, and alerting. Custom metrics enable real-time visibility into job performance and business KPIs.

## Requirements

Your monitoring metrics implementation should include:

- **Custom Metrics**: Implement Counter, Gauge, Histogram, Meter for job-specific metrics
- **Metric Groups**: Organize metrics into logical groups (job, operator, task)
- **Metric Reporters**: Configure Prometheus, InfluxDB, Datadog reporters
- **Business Metrics**: Track business KPIs alongside technical metrics
- **Latency Tracking**: Measure end-to-end and per-operator latency
- **Throughput Metrics**: Track records/sec, bytes/sec, batches/sec
- **Error Metrics**: Count errors, exceptions, retries by type
- **State Metrics**: Monitor state size, checkpoint duration, recovery time
- **Custom Histograms**: Track value distributions with percentiles
- **Alert Rules**: Define Prometheus alert rules for critical conditions
- **Grafana Dashboards**: Pre-built dashboards for common metrics
- **Documentation**: Clear metric naming and tagging conventions

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **Approach**: Use Flink MetricGroup API for custom metrics
- **Naming**: Follow Prometheus naming conventions (snake_case)
- **Tags/Labels**: Use consistent tagging for filtering and aggregation
- **Immutability**: Use records for metric value objects

## Template Example

```java
import org.apache.flink.metrics.*;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.functions.ProcessFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.util.Collector;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Demonstrates custom metrics implementation in Apache Flink 2.0.
 * Covers counters, gauges, histograms, meters, and integration with Prometheus/Grafana.
 */
@Slf4j
public class FlinkMetricsImplementations {

    /**
     * Basic counter metrics: Count processed records.
     * Use case: Track total events processed per event type.
     */
    public static DataStream<Event> eventCounterMetrics(DataStream<Event> events) {
        return events
                .process(new EventCounterFunction())
                .doOnNext(event -> log.trace("Event processed: {}", event.id()));
    }

    /**
     * Gauge metrics: Current state size.
     * Use case: Monitor queue size, buffer fill level.
     */
    public static DataStream<QueueItem> queueGaugeMetrics(DataStream<QueueItem> items) {
        return items
                .process(new QueueMonitorFunction())
                .doOnNext(item -> log.trace("Queue item processed: {}", item.id()));
    }

    /**
     * Histogram metrics: Value distributions.
     * Use case: Track request latency distribution with percentiles.
     */
    public static DataStream<Request> latencyHistogramMetrics(DataStream<Request> requests) {
        return requests
                .process(new LatencyHistogramFunction())
                .doOnNext(req -> log.trace("Request latency recorded: {}ms", req.latencyMs()));
    }

    /**
     * Meter metrics: Rate measurements.
     * Use case: Measure throughput (events/sec).
     */
    public static DataStream<Message> throughputMeterMetrics(DataStream<Message> messages) {
        return messages
                .process(new ThroughputMeterFunction())
                .doOnNext(msg -> log.trace("Message throughput tracked: {}", msg.id()));
    }

    /**
     * Business metrics: Domain-specific KPIs.
     * Use case: Track revenue, order count, active users.
     */
    public static DataStream<Order> businessMetrics(DataStream<Order> orders) {
        return orders
                .process(new BusinessMetricsFunction())
                .doOnNext(order -> log.info("Order metrics updated: ${}", order.amount()));
    }

    /**
     * Error metrics: Track failures and retries.
     * Use case: Monitor error rates by type and severity.
     */
    public static DataStream<ProcessingResult> errorMetrics(DataStream<InputData> data) {
        return data
                .process(new ErrorTrackingFunction())
                .doOnNext(result -> {
                    if (!result.success()) {
                        log.warn("Processing error: {}", result.error());
                    }
                });
    }

    // ==================== Metric Process Functions ====================

    /**
     * Event counter with per-type metrics.
     */
    @Slf4j
    private static class EventCounterFunction extends ProcessFunction<Event, Event> {
        private transient Counter totalEventsCounter;
        private transient Map<String, Counter> eventTypeCounters;
        private transient MetricGroup metricGroup;

        @Override
        public void open(Configuration parameters) {
            metricGroup = getRuntimeContext().getMetricGroup()
                    .addGroup("events");

            // Total events counter
            totalEventsCounter = metricGroup.counter("total_events_processed");

            // Per-type counters
            eventTypeCounters = new ConcurrentHashMap<>();
        }

        @Override
        public void processElement(Event event, Context ctx, Collector<Event> out) {
            // Increment total counter
            totalEventsCounter.inc();

            // Increment type-specific counter
            eventTypeCounters.computeIfAbsent(
                event.type(),
                type -> metricGroup
                        .addGroup("type", type)
                        .counter("events_by_type")
            ).inc();

            out.collect(event);
        }
    }

    /**
     * Queue monitor with gauge metrics.
     */
    @Slf4j
    private static class QueueMonitorFunction extends ProcessFunction<QueueItem, QueueItem> {
        private final Queue<QueueItem> buffer = new LinkedList<>();
        private transient Gauge<Integer> queueSizeGauge;
        private transient Gauge<Long> oldestItemAgeGauge;
        private transient Counter itemsEnqueued;
        private transient Counter itemsDequeued;

        @Override
        public void open(Configuration parameters) {
            MetricGroup queueMetrics = getRuntimeContext().getMetricGroup()
                    .addGroup("queue");

            // Queue size gauge
            queueSizeGauge = queueMetrics.gauge("current_size", () -> buffer.size());

            // Oldest item age gauge
            oldestItemAgeGauge = queueMetrics.gauge("oldest_item_age_ms", () -> {
                QueueItem oldest = buffer.peek();
                if (oldest != null) {
                    return System.currentTimeMillis() - oldest.timestamp();
                }
                return 0L;
            });

            // Enqueue/dequeue counters
            itemsEnqueued = queueMetrics.counter("items_enqueued");
            itemsDequeued = queueMetrics.counter("items_dequeued");
        }

        @Override
        public void processElement(QueueItem item, Context ctx, Collector<QueueItem> out) {
            buffer.add(item);
            itemsEnqueued.inc();

            // Process if buffer full or timeout
            if (buffer.size() >= 100) {
                QueueItem processed = buffer.poll();
                if (processed != null) {
                    itemsDequeued.inc();
                    out.collect(processed);
                }
            }
        }
    }

    /**
     * Latency tracking with histogram.
     */
    @Slf4j
    private static class LatencyHistogramFunction extends ProcessFunction<Request, Request> {
        private transient Histogram latencyHistogram;
        private transient Counter totalRequests;
        private transient Gauge<Double> avgLatencyGauge;
        private double runningAvgLatency = 0.0;
        private long requestCount = 0;

        @Override
        public void open(Configuration parameters) {
            MetricGroup latencyMetrics = getRuntimeContext().getMetricGroup()
                    .addGroup("latency");

            // Histogram for latency distribution
            latencyHistogram = latencyMetrics.histogram(
                    "request_latency_ms",
                    new DescriptiveStatisticsHistogram(1000) // 1000 samples
            );

            // Total requests counter
            totalRequests = latencyMetrics.counter("total_requests");

            // Average latency gauge
            avgLatencyGauge = latencyMetrics.gauge(
                    "avg_latency_ms",
                    () -> runningAvgLatency
            );
        }

        @Override
        public void processElement(Request request, Context ctx, Collector<Request> out) {
            long latency = request.latencyMs();

            // Update histogram
            latencyHistogram.update(latency);

            // Update counters and gauges
            totalRequests.inc();
            requestCount++;
            runningAvgLatency = ((runningAvgLatency * (requestCount - 1)) + latency) / requestCount;

            out.collect(request);
        }
    }

    /**
     * Throughput measurement with meter.
     */
    @Slf4j
    private static class ThroughputMeterFunction extends ProcessFunction<Message, Message> {
        private transient Meter messageRate;
        private transient Meter byteRate;
        private transient Counter totalMessages;
        private transient Counter totalBytes;

        @Override
        public void open(Configuration parameters) {
            MetricGroup throughputMetrics = getRuntimeContext().getMetricGroup()
                    .addGroup("throughput");

            // Messages per second
            messageRate = throughputMetrics.meter("messages_per_sec", new MeterView(60));

            // Bytes per second
            byteRate = throughputMetrics.meter("bytes_per_sec", new MeterView(60));

            // Total counters
            totalMessages = throughputMetrics.counter("total_messages");
            totalBytes = throughputMetrics.counter("total_bytes");
        }

        @Override
        public void processElement(Message message, Context ctx, Collector<Message> out) {
            int messageSize = message.payload().length();

            // Mark events in meters
            messageRate.markEvent();
            byteRate.markEvent(messageSize);

            // Increment counters
            totalMessages.inc();
            totalBytes.inc(messageSize);

            out.collect(message);
        }
    }

    /**
     * Business KPI metrics.
     */
    @Slf4j
    private static class BusinessMetricsFunction extends ProcessFunction<Order, Order> {
        private transient Counter orderCounter;
        private transient Counter revenueCounter;
        private transient Gauge<Double> avgOrderValue;
        private transient Histogram orderValueDistribution;
        private double totalRevenue = 0.0;
        private long totalOrders = 0;

        @Override
        public void open(Configuration parameters) {
            MetricGroup businessMetrics = getRuntimeContext().getMetricGroup()
                    .addGroup("business");

            // Order count
            orderCounter = businessMetrics.counter("total_orders");

            // Revenue (in cents to avoid floating point)
            revenueCounter = businessMetrics.counter("total_revenue_cents");

            // Average order value
            avgOrderValue = businessMetrics.gauge(
                    "avg_order_value",
                    () -> totalOrders > 0 ? totalRevenue / totalOrders : 0.0
            );

            // Order value distribution
            orderValueDistribution = businessMetrics.histogram(
                    "order_value_distribution",
                    new DescriptiveStatisticsHistogram(1000)
            );
        }

        @Override
        public void processElement(Order order, Context ctx, Collector<Order> out) {
            // Update metrics
            orderCounter.inc();
            revenueCounter.inc((long) (order.amount() * 100)); // Convert to cents
            orderValueDistribution.update((long) order.amount());

            totalOrders++;
            totalRevenue += order.amount();

            // Track by status
            if (order.status().equals("completed")) {
                getRuntimeContext().getMetricGroup()
                        .addGroup("business")
                        .addGroup("status", "completed")
                        .counter("orders")
                        .inc();
            }

            out.collect(order);
        }
    }

    /**
     * Error tracking metrics.
     */
    @Slf4j
    private static class ErrorTrackingFunction extends ProcessFunction<InputData, ProcessingResult> {
        private transient Counter totalProcessed;
        private transient Counter totalErrors;
        private transient Counter totalRetries;
        private transient Map<String, Counter> errorsByType;
        private transient Gauge<Double> errorRate;
        private long processedCount = 0;
        private long errorCount = 0;

        @Override
        public void open(Configuration parameters) {
            MetricGroup errorMetrics = getRuntimeContext().getMetricGroup()
                    .addGroup("errors");

            totalProcessed = errorMetrics.counter("total_processed");
            totalErrors = errorMetrics.counter("total_errors");
            totalRetries = errorMetrics.counter("total_retries");

            errorsByType = new ConcurrentHashMap<>();

            errorRate = errorMetrics.gauge(
                    "error_rate",
                    () -> processedCount > 0 ? (double) errorCount / processedCount : 0.0
            );
        }

        @Override
        public void processElement(InputData data, Context ctx, Collector<ProcessingResult> out) {
            totalProcessed.inc();
            processedCount++;

            try {
                // Simulate processing
                String result = processData(data);
                out.collect(new ProcessingResult(data.id(), true, result, null));

            } catch (Exception e) {
                totalErrors.inc();
                errorCount++;

                // Track by error type
                String errorType = e.getClass().getSimpleName();
                errorsByType.computeIfAbsent(
                        errorType,
                        type -> getRuntimeContext().getMetricGroup()
                                .addGroup("errors")
                                .addGroup("type", type)
                                .counter("count")
                ).inc();

                log.error("Processing error for {}: {}", data.id(), e.getMessage());
                out.collect(new ProcessingResult(data.id(), false, null, e.getMessage()));
            }
        }

        private String processData(InputData data) {
            // Processing logic
            return "processed";
        }
    }

    // ==================== Custom Histogram Implementation ====================

    /**
     * Histogram using Apache Commons Math.
     */
    public static class DescriptiveStatisticsHistogram implements Histogram {
        private final org.apache.commons.math3.stat.descriptive.DescriptiveStatistics stats;

        public DescriptiveStatisticsHistogram(int windowSize) {
            this.stats = new org.apache.commons.math3.stat.descriptive.DescriptiveStatistics(windowSize);
        }

        @Override
        public void update(long value) {
            stats.addValue(value);
        }

        @Override
        public long getCount() {
            return stats.getN();
        }

        @Override
        public HistogramStatistics getStatistics() {
            return new HistogramStatistics() {
                @Override
                public double getQuantile(double quantile) {
                    return stats.getPercentile(quantile * 100);
                }

                @Override
                public long[] getValues() {
                    return Arrays.stream(stats.getValues())
                            .mapToLong(d -> (long) d)
                            .toArray();
                }

                @Override
                public int size() {
                    return (int) stats.getN();
                }

                @Override
                public double getMean() {
                    return stats.getMean();
                }

                @Override
                public double getStdDev() {
                    return stats.getStandardDeviation();
                }

                @Override
                public long getMax() {
                    return (long) stats.getMax();
                }

                @Override
                public long getMin() {
                    return (long) stats.getMin();
                }
            };
        }
    }

    // ==================== Metric Reporter Configuration ====================

    /**
     * Example: Configure Prometheus reporter programmatically.
     */
    public static void configurePrometheusReporter(Configuration config) {
        config.setString("metrics.reporter.prom.class",
                "org.apache.flink.metrics.prometheus.PrometheusReporter");
        config.setString("metrics.reporter.prom.port", "9249-9250");
    }

    /**
     * Example: Configure InfluxDB reporter.
     */
    public static void configureInfluxDBReporter(Configuration config) {
        config.setString("metrics.reporter.influx.class",
                "org.apache.flink.metrics.influxdb.InfluxdbReporter");
        config.setString("metrics.reporter.influx.host", "localhost");
        config.setString("metrics.reporter.influx.port", "8086");
        config.setString("metrics.reporter.influx.db", "flink_metrics");
    }

    // ==================== Data Models ====================

    public record Event(String id, String type, String payload, long timestamp) {}

    public record QueueItem(String id, String data, long timestamp) {}

    public record Request(String id, long latencyMs, long timestamp) {}

    public record Message(String id, String payload, long timestamp) {}

    public record Order(String id, String customerId, double amount, String status, long timestamp) {}

    public record InputData(String id, String data) {}

    public record ProcessingResult(String id, boolean success, String result, String error) {}
}
```

## Configuration Example

```yaml
# Flink metrics configuration
metrics:
  reporters:
    # Prometheus reporter
    - name: prom
      class: org.apache.flink.metrics.prometheus.PrometheusReporter
      port: 9249-9250

    # InfluxDB reporter
    - name: influx
      class: org.apache.flink.metrics.influxdb.InfluxdbReporter
      host: localhost
      port: 8086
      db: flink_metrics
      username: flink
      password: password
      interval: 10s

  # Metric scopes
  scope:
    jm: "flink.<job_manager_id>"
    jm.job: "flink.<job_manager_id>.<job_name>"
    tm: "flink.<task_manager_id>"
    tm.job: "flink.<task_manager_id>.<job_name>"
    task: "flink.<task_manager_id>.<job_name>.<task_name>.<subtask_index>"
    operator: "flink.<task_manager_id>.<job_name>.<operator_name>.<subtask_index>"

  # Histogram settings
  histogram:
    implementation: apache-commons-math3
    window-size: 1000

# Job configuration
execution:
  parallelism: 4

# State backend
state:
  backend: rocksdb
  checkpoints-dir: s3://bucket/checkpoints
```

## Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'flink'
    static_configs:
      - targets: ['localhost:9249']
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'flink_.*'
        action: keep
```

## Alert Rules

```yaml
# flink_alerts.yml
groups:
  - name: flink_job_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighErrorRate
        expr: flink_errors_error_rate > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate in Flink job"
          description: "Error rate is {{ $value }} (threshold: 0.05)"

      # Checkpoint failures
      - alert: CheckpointFailures
        expr: increase(flink_checkpoint_failed[5m]) > 3
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Multiple checkpoint failures"
          description: "{{ $value }} checkpoint failures in last 5 minutes"

      # High latency
      - alert: HighLatency
        expr: flink_latency_avg_latency_ms > 1000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High processing latency"
          description: "Average latency is {{ $value }}ms (threshold: 1000ms)"

      # Low throughput
      - alert: LowThroughput
        expr: rate(flink_throughput_messages_per_sec[5m]) < 100
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Low message throughput"
          description: "Throughput is {{ $value }} msg/sec (threshold: 100)"

      # Queue backlog
      - alert: QueueBacklog
        expr: flink_queue_current_size > 10000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Large queue backlog"
          description: "Queue size is {{ $value }} (threshold: 10000)"
```

## Grafana Dashboard JSON

```json
{
  "dashboard": {
    "title": "Flink Job Metrics",
    "panels": [
      {
        "title": "Events Processed",
        "targets": [
          {
            "expr": "rate(flink_events_total_events_processed[5m])",
            "legendFormat": "Events/sec"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Error Rate",
        "targets": [
          {
            "expr": "flink_errors_error_rate",
            "legendFormat": "Error Rate"
          }
        ],
        "type": "graph",
        "alert": {
          "conditions": [
            {
              "evaluator": {"params": [0.05], "type": "gt"},
              "query": {"params": ["A", "5m", "now"]}
            }
          ]
        }
      },
      {
        "title": "Latency Percentiles",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, flink_latency_request_latency_ms)",
            "legendFormat": "p50"
          },
          {
            "expr": "histogram_quantile(0.95, flink_latency_request_latency_ms)",
            "legendFormat": "p95"
          },
          {
            "expr": "histogram_quantile(0.99, flink_latency_request_latency_ms)",
            "legendFormat": "p99"
          }
        ],
        "type": "graph"
      }
    ]
  }
}
```

## Best Practices

1. **Naming Conventions**: Use snake_case for metric names. Include units in names (e.g., latency_ms, size_bytes).

2. **Labels/Tags**: Use consistent tags for filtering (job_name, operator_name, subtask_index). Avoid high-cardinality tags.

3. **Metric Types**: Use Counter for monotonically increasing values, Gauge for current values, Histogram for distributions, Meter for rates.

4. **Histogram Window**: Set histogram window size based on expected traffic (1000-10000 samples). Larger windows smooth out spikes.

5. **Reporting Interval**: Set reporter interval to 10-30 seconds. Too frequent increases overhead, too infrequent misses spikes.

6. **Business Metrics**: Always track business KPIs alongside technical metrics. Revenue, orders, active users are more important than CPU.

7. **Error Tracking**: Track errors by type and severity. Separate transient errors from permanent failures.

8. **Alerting**: Set alerts on SLIs (error rate, latency, throughput). Use multi-window alerts to avoid flapping.

9. **Dashboard Design**: Create role-specific dashboards (operator, developer, business). Use red-yellow-green for at-a-glance status.

10. **Performance**: Metrics have minimal overhead but avoid excessive label cardinality. Limit custom metrics to <100 per operator.

## Related Documentation

- [Flink Metrics](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/ops/metrics/)
- [Prometheus Reporter](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/deployment/metric_reporters/#prometheus)
- [Metric Types](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/ops/metrics/#metric-types)
- [System Metrics](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/ops/metrics/#system-metrics)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards)
