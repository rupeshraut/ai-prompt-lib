# Create Metrics Collection and Monitoring

Implement comprehensive metrics collection and monitoring for production systems using Micrometer and Prometheus. Track application performance, business KPIs, and system health with custom metrics, dashboards, and alerting rules.

## Requirements

Your metrics collection implementation should include:

- **Application Metrics**: Request rates, response times, error rates (RED metrics)
- **JVM Metrics**: Memory usage, garbage collection, thread pools
- **Business Metrics**: Orders processed, payment success rate, user signups
- **Custom Metrics**: Domain-specific measurements with tags
- **Metric Types**: Counters, gauges, timers, distribution summaries
- **Prometheus Integration**: Scrape endpoint with proper labels
- **Grafana Dashboards**: Pre-built visualization templates
- **Alert Rules**: Automated alerting based on metric thresholds
- **High Cardinality Management**: Tag best practices to avoid cardinality explosion
- **Performance**: Minimal overhead on application throughput

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Micrometer, Prometheus
- **Libraries**: Spring Boot Actuator, Micrometer Core
- **Visualization**: Grafana, Prometheus

## Template: Metrics Configuration

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.Timer;
import io.micrometer.core.instrument.Gauge;
import io.micrometer.core.instrument.DistributionSummary;
import io.micrometer.core.instrument.Tag;
import io.micrometer.core.instrument.Tags;
import io.micrometer.prometheus.PrometheusConfig;
import io.micrometer.prometheus.PrometheusMeterRegistry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.boot.actuate.autoconfigure.metrics.MeterRegistryCustomizer;
import lombok.extern.slf4j.Slf4j;

/**
 * Metrics configuration for application monitoring.
 * Configures Micrometer with Prometheus registry.
 */
@Slf4j
@Configuration
public class MetricsConfiguration {

    /**
     * Customize meter registry with common tags.
     */
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags(
            @Value("${spring.application.name}") String applicationName,
            @Value("${ENVIRONMENT:development}") String environment) {

        return registry -> registry.config()
            .commonTags(
                "application", applicationName,
                "environment", environment,
                "region", System.getenv("AWS_REGION") != null ?
                    System.getenv("AWS_REGION") : "local"
            )
            .namingConvention(new PrometheusNamingConvention());
    }

    /**
     * Configure Prometheus registry.
     */
    @Bean
    public PrometheusMeterRegistry prometheusRegistry() {
        return new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
    }
}

/**
 * Custom metrics for business operations.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class BusinessMetrics {

    private final MeterRegistry meterRegistry;

    // Counters
    private Counter ordersCreatedCounter;
    private Counter orderFailuresCounter;
    private Counter paymentsProcessedCounter;
    private Counter paymentFailuresCounter;

    // Timers
    private Timer orderCreationTimer;
    private Timer paymentProcessingTimer;

    // Distribution Summary
    private DistributionSummary orderAmountDistribution;

    @PostConstruct
    public void initMetrics() {
        // Initialize counters
        ordersCreatedCounter = Counter.builder("orders.created.total")
            .description("Total number of orders created")
            .register(meterRegistry);

        orderFailuresCounter = Counter.builder("orders.failures.total")
            .description("Total number of order creation failures")
            .register(meterRegistry);

        paymentsProcessedCounter = Counter.builder("payments.processed.total")
            .description("Total number of payments processed")
            .register(meterRegistry);

        paymentFailuresCounter = Counter.builder("payments.failures.total")
            .description("Total number of payment failures")
            .register(meterRegistry);

        // Initialize timers
        orderCreationTimer = Timer.builder("orders.creation.duration")
            .description("Time taken to create an order")
            .publishPercentiles(0.5, 0.95, 0.99)
            .minimumExpectedValue(Duration.ofMillis(10))
            .maximumExpectedValue(Duration.ofSeconds(5))
            .register(meterRegistry);

        paymentProcessingTimer = Timer.builder("payments.processing.duration")
            .description("Time taken to process a payment")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);

        // Initialize distribution summary
        orderAmountDistribution = DistributionSummary.builder("orders.amount")
            .description("Distribution of order amounts")
            .baseUnit("USD")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(meterRegistry);

        log.info("Business metrics initialized");
    }

    /**
     * Record order creation.
     */
    public void recordOrderCreated(Order order) {
        ordersCreatedCounter.increment();
        orderAmountDistribution.record(order.getAmount());

        // Counter with tags for detailed tracking
        Counter.builder("orders.created.by.status")
            .tag("status", order.getStatus().toString())
            .tag("customerId", order.getCustomerId())
            .register(meterRegistry)
            .increment();
    }

    /**
     * Record order creation failure.
     */
    public void recordOrderFailure(String reason) {
        Counter.builder("orders.failures.total")
            .tag("reason", reason)
            .register(meterRegistry)
            .increment();
    }

    /**
     * Time order creation operation.
     */
    public <T> T timeOrderCreation(Supplier<T> operation) {
        return orderCreationTimer.record(operation);
    }

    /**
     * Record payment processed.
     */
    public void recordPaymentProcessed(Payment payment) {
        paymentsProcessedCounter.increment();

        Counter.builder("payments.processed.by.method")
            .tag("method", payment.getPaymentMethod())
            .register(meterRegistry)
            .increment();
    }

    /**
     * Record payment failure.
     */
    public void recordPaymentFailure(String errorCode) {
        Counter.builder("payments.failures.total")
            .tag("errorCode", errorCode)
            .register(meterRegistry)
            .increment();
    }
}
```

## Service with Metrics Integration

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.annotation.Timed;
import io.micrometer.core.annotation.Counted;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

/**
 * Order service with comprehensive metrics tracking.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final BusinessMetrics businessMetrics;
    private final MeterRegistry meterRegistry;

    /**
     * Create order with metrics tracking.
     * Timer automatically tracks duration and throughput.
     */
    @Timed(
        value = "orders.create",
        description = "Time taken to create an order",
        percentiles = {0.5, 0.95, 0.99}
    )
    @Counted(
        value = "orders.create.count",
        description = "Number of order creation attempts"
    )
    public Order createOrder(CreateOrderRequest request) {
        return businessMetrics.timeOrderCreation(() -> {
            try {
                // Validate order
                validateOrder(request);

                // Create order
                Order order = new Order();
                order.setCustomerId(request.customerId());
                order.setAmount(request.amount());
                order.setStatus(OrderStatus.PENDING);

                // Save order
                Order savedOrder = orderRepository.save(order);

                // Process payment
                processPayment(savedOrder);

                // Update status
                savedOrder.setStatus(OrderStatus.CONFIRMED);
                orderRepository.save(savedOrder);

                // Record metrics
                businessMetrics.recordOrderCreated(savedOrder);

                // Track order by category
                meterRegistry.counter(
                    "orders.created.by.category",
                    "category", request.category(),
                    "priority", request.priority()
                ).increment();

                return savedOrder;

            } catch (ValidationException e) {
                businessMetrics.recordOrderFailure("validation_failed");
                throw e;
            } catch (PaymentException e) {
                businessMetrics.recordOrderFailure("payment_failed");
                throw e;
            } catch (Exception e) {
                businessMetrics.recordOrderFailure("unknown_error");
                throw e;
            }
        });
    }

    /**
     * Get active orders count as a gauge.
     */
    @PostConstruct
    public void registerGauges() {
        // Gauge for active orders count
        Gauge.builder("orders.active.count", orderRepository, repo -> {
            return repo.countByStatus(OrderStatus.ACTIVE);
        })
        .description("Number of active orders")
        .register(meterRegistry);

        // Gauge for pending orders count
        Gauge.builder("orders.pending.count", orderRepository, repo -> {
            return repo.countByStatus(OrderStatus.PENDING);
        })
        .description("Number of pending orders")
        .register(meterRegistry);
    }

    private void validateOrder(CreateOrderRequest request) {
        meterRegistry.counter("orders.validations.total").increment();

        if (request.amount() <= 0) {
            meterRegistry.counter(
                "orders.validations.failed",
                "reason", "invalid_amount"
            ).increment();
            throw new ValidationException("Invalid amount");
        }
    }

    private void processPayment(Order order) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            paymentService.processPayment(order.getId(), order.getAmount());

            sample.stop(meterRegistry.timer(
                "payments.processing.success",
                "orderId", order.getId()
            ));

            businessMetrics.recordPaymentProcessed(new Payment(order.getId()));

        } catch (Exception e) {
            sample.stop(meterRegistry.timer(
                "payments.processing.failure",
                "errorType", e.getClass().getSimpleName()
            ));

            businessMetrics.recordPaymentFailure(e.getMessage());
            throw e;
        }
    }
}
```

## Custom Metrics Controller

```java
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.prometheus.PrometheusMeterRegistry;
import org.springframework.web.bind.annotation.*;
import lombok.RequiredArgsConstructor;

/**
 * Controller for custom metrics and Prometheus endpoint.
 */
@RestController
@RequestMapping("/api/metrics")
@RequiredArgsConstructor
public class MetricsController {

    private final MeterRegistry meterRegistry;
    private final PrometheusMeterRegistry prometheusRegistry;

    /**
     * Prometheus scrape endpoint.
     */
    @GetMapping(value = "/prometheus", produces = "text/plain")
    public String prometheus() {
        return prometheusRegistry.scrape();
    }

    /**
     * Record custom business event.
     */
    @PostMapping("/events")
    public void recordEvent(@RequestBody MetricEvent event) {
        meterRegistry.counter(
            "business.events",
            "type", event.getType(),
            "category", event.getCategory()
        ).increment(event.getCount());
    }

    /**
     * Get current metric values.
     */
    @GetMapping("/current")
    public Map<String, Double> getCurrentMetrics() {
        Map<String, Double> metrics = new HashMap<>();

        meterRegistry.getMeters().forEach(meter -> {
            meter.measure().forEach(measurement -> {
                String key = meter.getId().getName() + "_" +
                    measurement.getStatistic().name().toLowerCase();
                metrics.put(key, measurement.getValue());
            });
        });

        return metrics;
    }
}
```

## Reactive Metrics with Project Reactor

```java
import reactor.core.publisher.Mono;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;

/**
 * Reactive service with metrics integration.
 */
@Service
@RequiredArgsConstructor
public class ReactiveOrderService {

    private final ReactiveOrderRepository orderRepository;
    private final MeterRegistry meterRegistry;
    private final BusinessMetrics businessMetrics;

    /**
     * Create order reactively with metrics.
     */
    public Mono<Order> createOrder(CreateOrderRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);

        return Mono.just(request)
            .doOnNext(req ->
                meterRegistry.counter("orders.create.attempts").increment()
            )
            .flatMap(this::validateOrderAsync)
            .map(this::createOrderEntity)
            .flatMap(orderRepository::save)
            .doOnSuccess(order -> {
                sample.stop(meterRegistry.timer("orders.creation.duration"));
                businessMetrics.recordOrderCreated(order);
                meterRegistry.counter("orders.create.success").increment();
            })
            .doOnError(error -> {
                sample.stop(meterRegistry.timer(
                    "orders.creation.duration",
                    "outcome", "failure"
                ));
                meterRegistry.counter(
                    "orders.create.failures",
                    "errorType", error.getClass().getSimpleName()
                ).increment();
            });
    }

    private Mono<CreateOrderRequest> validateOrderAsync(CreateOrderRequest request) {
        return Mono.defer(() -> {
            meterRegistry.counter("orders.validations").increment();

            if (request.amount() <= 0) {
                meterRegistry.counter(
                    "orders.validations.failed",
                    "reason", "invalid_amount"
                ).increment();
                return Mono.error(new ValidationException("Invalid amount"));
            }

            return Mono.just(request);
        });
    }

    private Order createOrderEntity(CreateOrderRequest request) {
        Order order = new Order();
        order.setCustomerId(request.customerId());
        order.setAmount(request.amount());
        order.setStatus(OrderStatus.PENDING);
        return order;
    }
}
```

## Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator

  endpoint:
    health:
      show-details: always
    metrics:
      enabled: true
    prometheus:
      enabled: true

  metrics:
    export:
      prometheus:
        enabled: true
        step: 1m
        descriptions: true

    distribution:
      percentiles-histogram:
        http.server.requests: true
        orders.creation.duration: true
        payments.processing.duration: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
        orders.creation.duration: 0.5, 0.95, 0.99
      slo:
        http.server.requests: 10ms, 50ms, 100ms, 200ms, 500ms, 1s
        orders.creation.duration: 100ms, 500ms, 1s, 2s

    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:development}

spring:
  application:
    name: order-service
```

## Prometheus Configuration (prometheus.yml)

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'spring-boot-order-service'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['localhost:8080']
        labels:
          application: 'order-service'
          environment: 'production'

  - job_name: 'spring-boot-payment-service'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 10s
    static_configs:
      - targets: ['localhost:8081']
        labels:
          application: 'payment-service'
          environment: 'production'
```

## Alert Rules (alerts.yml)

```yaml
groups:
  - name: order_service_alerts
    interval: 30s
    rules:
      # High error rate
      - alert: HighOrderFailureRate
        expr: |
          (
            rate(orders_failures_total[5m])
            /
            rate(orders_created_total[5m])
          ) > 0.05
        for: 5m
        labels:
          severity: warning
          service: order-service
        annotations:
          summary: "High order failure rate"
          description: "Order failure rate is {{ $value | humanizePercentage }} (threshold: 5%)"

      # High latency
      - alert: HighOrderCreationLatency
        expr: |
          histogram_quantile(0.95,
            rate(orders_creation_duration_bucket[5m])
          ) > 2
        for: 5m
        labels:
          severity: warning
          service: order-service
        annotations:
          summary: "High order creation latency"
          description: "P95 order creation latency is {{ $value }}s (threshold: 2s)"

      # Low throughput
      - alert: LowOrderThroughput
        expr: rate(orders_created_total[5m]) < 10
        for: 10m
        labels:
          severity: info
          service: order-service
        annotations:
          summary: "Low order throughput"
          description: "Order creation rate is {{ $value }} orders/sec (threshold: 10/sec)"

      # Payment failures
      - alert: HighPaymentFailureRate
        expr: |
          (
            rate(payments_failures_total[5m])
            /
            rate(payments_processed_total[5m])
          ) > 0.1
        for: 5m
        labels:
          severity: critical
          service: payment-service
        annotations:
          summary: "High payment failure rate"
          description: "Payment failure rate is {{ $value | humanizePercentage }} (threshold: 10%)"

      # JVM memory
      - alert: HighMemoryUsage
        expr: |
          (
            jvm_memory_used_bytes{area="heap"}
            /
            jvm_memory_max_bytes{area="heap"}
          ) > 0.9
        for: 5m
        labels:
          severity: warning
          service: order-service
        annotations:
          summary: "High JVM memory usage"
          description: "JVM heap usage is {{ $value | humanizePercentage }} (threshold: 90%)"
```

## Grafana Dashboard JSON (dashboard.json)

```json
{
  "dashboard": {
    "title": "Order Service Metrics",
    "panels": [
      {
        "title": "Order Creation Rate",
        "targets": [
          {
            "expr": "rate(orders_created_total[5m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Order Creation Latency (P95)",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(orders_creation_duration_bucket[5m]))"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Order Failure Rate",
        "targets": [
          {
            "expr": "rate(orders_failures_total[5m]) / rate(orders_created_total[5m])"
          }
        ],
        "type": "graph"
      },
      {
        "title": "Active Orders",
        "targets": [
          {
            "expr": "orders_active_count"
          }
        ],
        "type": "stat"
      }
    ]
  }
}
```

## Dependencies (Maven)

```xml
<dependencies>
    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Micrometer Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Micrometer Core -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-core</artifactId>
    </dependency>
</dependencies>
```

## Best Practices

1. **Use Consistent Naming**: Follow Prometheus naming conventions (snake_case, base_unit suffix)
2. **Add Meaningful Tags**: Include service, environment, region tags on all metrics
3. **Avoid High Cardinality**: Don't use user IDs, session IDs, or timestamps as tag values
4. **Use Percentiles**: Track P50, P95, P99 for latency metrics instead of averages
5. **Set SLOs**: Define Service Level Objectives as histogram buckets for latency metrics
6. **Monitor RED Metrics**: Track Rate, Errors, Duration for all services
7. **Use Gauges for Snapshots**: Use gauges for current state (active connections, queue size)
8. **Use Counters for Totals**: Use counters for cumulative values (total requests, total errors)
9. **Minimize Overhead**: Use async metrics recording to avoid blocking application threads
10. **Test Alert Rules**: Validate alerting thresholds in staging before production deployment

## Related Documentation

- [Micrometer Documentation](https://micrometer.io/docs)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)
