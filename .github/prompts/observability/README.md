# Observability Library

Comprehensive prompts for implementing production-grade observability with structured logging and metrics collection. Enable efficient debugging, performance monitoring, and proactive alerting through centralized logging and comprehensive metrics.

## Available Prompts

### 1. Create Structured Logging
Implement structured JSON logging for efficient log aggregation, querying, and analysis.

| Feature | Description |
|---------|-------------|
| JSON Format | All logs output in structured JSON |
| Standard Fields | Timestamp, level, message, logger, thread |
| Context Enrichment | Business context (user ID, tenant ID) |
| MDC Integration | Propagate context across threads |
| Trace Correlation | Include trace ID and span ID |
| Exception Serialization | Structured stack traces |
| Async Appenders | Minimal latency impact |
| Log Levels | Environment-specific configuration |
| Sensitive Data Masking | Redact PII and secrets |
| ELK/Splunk Integration | Centralized log aggregation |

### 2. Create Metrics Collection and Monitoring
Implement comprehensive metrics collection using Micrometer and Prometheus for monitoring and alerting.

| Feature | Description |
|---------|-------------|
| Application Metrics | Request rates, response times, error rates |
| JVM Metrics | Memory, GC, thread pools |
| Business Metrics | Orders processed, payment success rate |
| Custom Metrics | Domain-specific measurements |
| Metric Types | Counters, gauges, timers, histograms |
| Prometheus Integration | Scrape endpoint with labels |
| Grafana Dashboards | Pre-built visualizations |
| Alert Rules | Automated alerting |
| High Cardinality Management | Tag best practices |
| Performance | Sub-millisecond metric recording |

## Quick Start

### 1. Setup Structured Logging

```
@workspace Use .github/prompts/observability/create-structured-logging.md

Configure JSON logging with MDC context, trace correlation, and PII masking
```

### 2. Implement Metrics Collection

```
@workspace Use .github/prompts/observability/create-metrics-monitoring.md

Add Prometheus metrics for RED (Rate, Errors, Duration) and business KPIs
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Application Layer                         │
│  ┌────────────────────┐      ┌────────────────────┐        │
│  │  Structured Logs   │      │  Metrics           │        │
│  │  (JSON)            │      │  (Micrometer)      │        │
│  │                    │      │                    │        │
│  │  - Request logs    │      │  - Counters        │        │
│  │  - Business events │      │  - Timers          │        │
│  │  - Error logs      │      │  - Gauges          │        │
│  │  - Trace context   │      │  - Histograms      │        │
│  └────────┬───────────┘      └─────────┬──────────┘        │
└───────────┼──────────────────────────────┼──────────────────┘
            │                              │
            ▼                              ▼
   ┌────────────────────┐        ┌────────────────────┐
   │  Log Aggregation   │        │  Metrics Storage   │
   │  (ELK Stack)       │        │  (Prometheus)      │
   │                    │        │                    │
   │  Elasticsearch     │        │  Time-series DB    │
   │  Logstash          │        │  15s scrape        │
   │  Kibana            │        │  15d retention     │
   └────────┬───────────┘        └─────────┬──────────┘
            │                              │
            ▼                              ▼
   ┌────────────────────┐        ┌────────────────────┐
   │  Log Analysis      │        │  Visualization     │
   │  - Search logs     │        │  (Grafana)         │
   │  - Correlation     │        │                    │
   │  - Alerting        │        │  - Dashboards      │
   └────────────────────┘        │  - Alerts          │
                                 │  - SLO tracking    │
                                 └────────────────────┘

Structured Log Flow:

    Application Request
           │
           ▼
    ┌──────────────────┐
    │  MDC Filter      │ ← Add correlationId, userId
    │  (Servlet)       │   requestUri, clientIp
    └────────┬─────────┘
           │
           ▼
    ┌──────────────────┐
    │  Service Layer   │
    │  log.info(...)   │ ← Business context
    └────────┬─────────┘   orderId, customerId
           │
           ▼
    ┌──────────────────┐
    │  Logback Encoder │ ← Convert to JSON
    │  (Logstash)      │   Add timestamp, level
    └────────┬─────────┘
           │
           ▼
    {
      "timestamp": "2025-01-15T10:30:45.123Z",
      "level": "INFO",
      "message": "Order created",
      "logger": "OrderService",
      "thread": "http-nio-8080-exec-1",
      "correlationId": "a1b2c3d4...",
      "traceId": "4bf92f35...",
      "userId": "user-123",
      "orderId": "order-456"
    }
           │
           ▼
    ┌──────────────────┐
    │  Elasticsearch   │
    └──────────────────┘

Metrics Flow:

    Application Code
           │
           ▼
    ┌──────────────────┐
    │  @Timed          │ ← Annotation-based
    │  @Counted        │   metrics
    └────────┬─────────┘
           │
           ▼
    ┌──────────────────┐
    │  Micrometer      │ ← Record metrics
    │  MeterRegistry   │   with tags
    └────────┬─────────┘
           │
           ▼
    ┌──────────────────┐
    │  Prometheus      │ ← Expose /actuator/
    │  Scrape Endpoint │   prometheus
    └────────┬─────────┘
           │
           ▼ (Scrape every 15s)
    ┌──────────────────┐
    │  Prometheus      │ ← Store time-series
    │  Server          │   data
    └────────┬─────────┘
           │
           ▼
    ┌──────────────────┐
    │  Grafana         │ ← Visualize + alert
    │  Dashboards      │
    └──────────────────┘

Observability Integration:

    Request
       │
       ├──────────────────────────┬──────────────────────┐
       │                          │                      │
       ▼                          ▼                      ▼
    Logs                      Metrics               Traces
       │                          │                      │
  correlationId              Request Count         Trace ID
  userId                     Latency P95           Span ID
  orderId                    Error Rate            Parent Span
       │                          │                      │
       └──────────────┬───────────┴──────────────────────┘
                      │
                      ▼
              ┌────────────────┐
              │   Correlation  │ ← Link logs, metrics,
              │   & Analysis   │   traces by IDs
              └────────────────┘
```

## Integration Patterns

### Spring Boot Application with Full Observability

```java
@Configuration
public class ObservabilityConfig {

    /**
     * MDC filter for request context.
     */
    @Bean
    public FilterRegistrationBean<MdcContextFilter> mdcFilter() {
        FilterRegistrationBean<MdcContextFilter> registration =
            new FilterRegistrationBean<>();
        registration.setFilter(new MdcContextFilter());
        registration.addUrlPatterns("/*");
        return registration;
    }

    /**
     * Metrics customization.
     */
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config()
            .commonTags(
                "application", "order-service",
                "environment", getEnvironment()
            );
    }
}

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderService {

    private final OrderRepository orderRepository;
    private final MeterRegistry meterRegistry;

    /**
     * Create order with logging and metrics.
     */
    @Timed(value = "orders.create", percentiles = {0.5, 0.95, 0.99})
    @Counted(value = "orders.create.count")
    public Order createOrder(CreateOrderRequest request) {
        // Add context to MDC
        MDC.put("customerId", request.customerId());
        MDC.put("orderAmount", String.valueOf(request.amount()));

        try {
            // Structured logging with context
            log.info("Creating order",
                keyValue("customerId", request.customerId()),
                keyValue("amount", request.amount()),
                keyValue("itemCount", request.items().size())
            );

            // Business logic
            Order order = processOrder(request);

            // Record business metric
            meterRegistry.counter("orders.created.by.category",
                "category", request.category()
            ).increment();

            log.info("Order created successfully",
                keyValue("orderId", order.getId()),
                keyValue("status", order.getStatus())
            );

            return order;

        } catch (Exception e) {
            // Record error metric
            meterRegistry.counter("orders.create.failures",
                "errorType", e.getClass().getSimpleName()
            ).increment();

            // Structured error logging
            log.error("Order creation failed",
                keyValue("customerId", request.customerId()),
                keyValue("errorMessage", e.getMessage()),
                e
            );

            throw e;

        } finally {
            MDC.clear();
        }
    }
}
```

### Logback Configuration with JSON Logging

```xml
<configuration>
    <!-- JSON Console Appender -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <!-- Include MDC keys -->
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>

            <!-- Custom fields -->
            <customFields>
                {
                    "application": "order-service",
                    "environment": "${ENVIRONMENT}"
                }
            </customFields>
        </encoder>
    </appender>

    <!-- Async Appender for performance -->
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <appender-ref ref="CONSOLE"/>
    </appender>

    <root level="INFO">
        <appender-ref ref="ASYNC"/>
    </root>
</configuration>
```

### Prometheus Metrics Endpoint

```java
@RestController
@RequiredArgsConstructor
public class MetricsController {

    private final PrometheusMeterRegistry prometheusRegistry;

    /**
     * Prometheus scrape endpoint.
     */
    @GetMapping(value = "/actuator/prometheus", produces = "text/plain")
    public String prometheus() {
        return prometheusRegistry.scrape();
    }
}
```

### Grafana Dashboard Configuration

```json
{
  "dashboard": {
    "title": "Order Service Metrics",
    "panels": [
      {
        "title": "Request Rate (req/s)",
        "targets": [
          {"expr": "rate(orders_create_count_total[5m])"}
        ]
      },
      {
        "title": "Latency P95 (ms)",
        "targets": [
          {"expr": "histogram_quantile(0.95, rate(orders_create_seconds_bucket[5m])) * 1000"}
        ]
      },
      {
        "title": "Error Rate (%)",
        "targets": [
          {"expr": "rate(orders_create_failures_total[5m]) / rate(orders_create_count_total[5m]) * 100"}
        ]
      }
    ]
  }
}
```

### Alert Rules

```yaml
# prometheus-alerts.yml
groups:
  - name: order_service_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          rate(orders_create_failures_total[5m])
          /
          rate(orders_create_count_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate in order service"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95,
            rate(orders_create_seconds_bucket[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency in order creation"
          description: "P95 latency is {{ $value }}s"
```

## Key Concepts

### Structured Logging
Logs formatted as JSON with consistent fields. Enables efficient querying, filtering, and correlation.

### MDC (Mapped Diagnostic Context)
Thread-local context storage for logging. Automatically includes contextual information in all log statements.

### RED Metrics
Rate (throughput), Errors (failure rate), Duration (latency). Core metrics for service monitoring.

### Metric Types
- **Counter**: Cumulative value (requests, errors)
- **Gauge**: Current value (active connections, queue size)
- **Timer**: Duration distribution (latency percentiles)
- **Histogram**: Value distribution (request sizes)

### Percentiles
Statistical values showing distribution (P50, P95, P99). P95 means 95% of requests are faster.

### Cardinality
Number of unique metric label combinations. High cardinality causes memory issues in Prometheus.

### Log Levels
- **ERROR**: Failures requiring attention
- **WARN**: Degraded state, potential issues
- **INFO**: Business events, lifecycle
- **DEBUG**: Detailed debugging information
- **TRACE**: Very detailed debugging

## Best Practices

### Logging

1. **Always Use Structured Logging**
   - Output JSON format in production
   - Use keyValue() for context, not string concatenation
   - Include correlation IDs in all logs

2. **Populate MDC Early**
   - Add correlationId, userId, tenantId at request entry
   - Clear MDC after request completes
   - Propagate MDC across async operations

3. **Log at Appropriate Levels**
   - ERROR: Failures, exceptions
   - WARN: Degradation, retries
   - INFO: Business events, API calls
   - DEBUG: Detailed flow (dev/staging only)

4. **Mask Sensitive Data**
   - Automatically redact PII, passwords, tokens
   - Use custom converters for masking
   - Never log credit card numbers, SSNs

5. **Include Trace Context**
   - Always log traceId and spanId
   - Correlate logs with distributed traces
   - Link logs to metrics by correlation ID

### Metrics

6. **Track RED Metrics**
   - Rate: requests per second
   - Errors: error rate percentage
   - Duration: latency percentiles (P50, P95, P99)

7. **Use Common Tags**
   - application, environment, region on all metrics
   - endpoint, method, status on HTTP metrics
   - Keep tag cardinality low (<100 values)

8. **Set Appropriate SLOs**
   - Define Service Level Objectives (e.g., P95 < 100ms)
   - Create histogram buckets aligned with SLOs
   - Alert when SLOs are breached

9. **Monitor Business Metrics**
   - Track business KPIs (orders, revenue, signups)
   - Use domain-specific tags (category, priority)
   - Create business dashboards

10. **Minimize Performance Impact**
    - Use async log appenders
    - Record metrics without blocking
    - Sample high-volume metrics if needed

## Configuration

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example.orderservice: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99

spring:
  application:
    name: order-service
```

## Troubleshooting

### Logs Not Appearing in ELK
1. Verify log format is valid JSON
2. Check Logstash pipeline configuration
3. Verify network connectivity to Elasticsearch
4. Check Elasticsearch disk space

### Metrics Not Scraped by Prometheus
1. Verify /actuator/prometheus endpoint is accessible
2. Check Prometheus scrape configuration
3. Verify application is running and healthy
4. Check firewall rules

### High Cardinality Warnings
1. Review metric tag usage
2. Remove user IDs, session IDs from tags
3. Use fixed sets of tag values
4. Consider sampling high-cardinality metrics

### Missing Trace Context in Logs
1. Verify OpenTelemetry is configured
2. Check MDC propagation in async code
3. Review trace context headers
4. Ensure trace IDs are generated

## Related Documentation

- [Logback Documentation](http://logback.qos.ch/documentation.html)
- [Logstash Logback Encoder](https://github.com/logfellow/logstash-logback-encoder)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
- [ELK Stack](https://www.elastic.co/what-is/elk-stack)

## Library Navigation

This library focuses on observability through logs and metrics. For related capabilities:
- See **distributed-tracing** library for distributed tracing with OpenTelemetry
- See **resilience-operations** library for health checks and circuit breaker metrics
- See **security-compliance** library for audit logging
