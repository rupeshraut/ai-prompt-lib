# Distributed Tracing Library

Comprehensive prompts for implementing distributed tracing across microservices using OpenTelemetry, Jaeger, and modern Java frameworks.

## Available Prompts

### 1. Create OpenTelemetry Setup
Configure OpenTelemetry SDK for distributed tracing across Spring Boot, Project Reactor, and Apache Flink.

| Feature | Description |
|---------|-------------|
| SDK Configuration | Initialize OpenTelemetry with SDK and exporters |
| Trace Exporters | Jaeger and OTLP-compatible backends |
| Auto-Instrumentation | Automatic library instrumentation |
| Manual Instrumentation | Custom spans for business logic |
| Span Attributes | Business context enrichment |
| Baggage | Cross-process value propagation |
| Sampling Strategies | Cost-effective trace collection |
| Error Handling | Exception capture and recording |
| Resource Attributes | Service identity and deployment info |

### 2. Create Context Propagation
Implement context propagation across service boundaries with W3C Trace Context and Jaeger headers.

| Feature | Description |
|---------|-------------|
| W3C Trace Context | Standard traceparent and tracestate headers |
| Jaeger Headers | Legacy uber-trace-id support |
| HTTP Propagation | RestTemplate and WebClient integration |
| Message Queue Integration | Kafka and RabbitMQ trace propagation |
| Reactive Stream Propagation | Project Reactor context preservation |
| Spring Cloud Sleuth | Automatic context management |
| Correlation IDs | Business-level request tracking |
| MDC Integration | Diagnostic context for logging |
| Baggage | Request-scoped data passing |

### 3. Create Trace Sampling Strategies
Implement sampling strategies to manage observability costs while maintaining visibility.

| Feature | Description |
|---------|-------------|
| Static Sampling | Fixed percentage-based sampling |
| Adaptive Sampling | Dynamic rate based on traffic |
| Tail-Based Sampling | Sample based on trace properties |
| Error-Rate Sampling | Always sample error traces |
| Latency-Based Sampling | Increase rate for slow operations |
| Rate Limiting | Control maximum span throughput |
| Custom Sampling Logic | Business-specific sampling rules |
| Metrics | Track sampling decisions |
| Configuration | Externalized strategy selection |

## Quick Start

### 1. Setup OpenTelemetry

```
@workspace Use .github/prompts/distributed-tracing/create-opentelemetry-setup.md

Configure OpenTelemetry SDK with Jaeger exporter for distributed tracing across services
```

### 2. Implement Context Propagation

```
@workspace Use .github/prompts/distributed-tracing/create-context-propagation.md

Add W3C Trace Context propagation through HTTP, Kafka, and reactive streams
```

### 3. Configure Sampling

```
@workspace Use .github/prompts/distributed-tracing/create-trace-sampling.md

Implement adaptive sampling to balance observability with cost
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  User Request                            │
└────────────────┬────────────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │  API Gateway    │ ← Generates Trace ID
        │  (Start Span)   │
        └────────┬────────┘
                 │ traceparent: 00-<traceId>-<spanId>-01
                 ▼
        ┌─────────────────────┐
        │  User Service       │ ← Continue Trace
        │  (Span 1)           │
        └────────┬────────────┘
                 │ Propagate Context
                 ▼
        ┌─────────────────────────┐
        │  Order Service          │ ← Continue Trace
        │  (Span 2)               │
        └────────┬────────────────┘
                 │
         ┌───────┴───────┐
         │               │
         ▼               ▼
    ┌─────────┐   ┌──────────────┐
    │  Payment│   │  Inventory   │
    │ Service │   │   Service    │
    │(Span 3) │   │  (Span 4)    │
    └────┬────┘   └──────┬───────┘
         │                │
         └────────┬───────┘
                  │
                  ▼
         ┌─────────────────────┐
         │  Jaeger Backend     │
         │  (Trace Storage)    │
         └─────────────────────┘
                  │
                  ▼
         ┌─────────────────────┐
         │  Jaeger UI          │
         │  (Visualization)    │
         └─────────────────────┘
```

## Integration Patterns

### Spring Boot Service

```java
@Configuration
public class TracingConfig {

    @Bean
    public OpenTelemetry openTelemetry() {
        // Initialize OpenTelemetry SDK
        Resource resource = Resource.getDefault()
            .merge(Resource.create(Attributes.of(
                ResourceAttributes.SERVICE_NAME, "order-service"
            )));

        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
            .setResource(resource)
            .addSpanProcessor(BatchSpanProcessor.builder(
                JaegerThriftSpanExporter.builder().build()
            ).build())
            .build();

        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build();
    }
}

@Service
public class OrderService {
    private final Tracer tracer;

    public Order createOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("order.create")
            .setAttribute("order.customerId", request.customerId())
            .startSpan();

        try (var scope = span.makeCurrent()) {
            // Business logic
            return saveOrder(request);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Project Reactor Integration

```java
@Service
public class ReactiveOrderService {
    private final Tracer tracer;

    public Mono<Order> createOrderReactive(CreateOrderRequest request) {
        return Mono.defer(() -> {
            Span span = tracer.spanBuilder("order.create.reactive")
                .setAttribute("order.customerId", request.customerId())
                .startSpan();

            return Mono.just(request)
                .flatMap(this::saveOrderAsync)
                .doFinally(signalType -> span.end())
                .doOnError(error -> {
                    span.recordException(error);
                    span.setStatus(StatusCode.ERROR);
                });
        });
    }
}
```

### Apache Flink Integration

```java
public class TracedFlatMapFunction extends RichFlatMapFunction<Event, ProcessedEvent> {
    private transient Tracer tracer;

    @Override
    public void open(Configuration parameters) throws Exception {
        tracer = GlobalOpenTelemetry.get().getTracer("flink-job");
    }

    @Override
    public void flatMap(Event event, Collector<ProcessedEvent> out) {
        Span span = tracer.spanBuilder("event.process")
            .setAttribute("event.id", event.id())
            .startSpan();

        try (var scope = span.makeCurrent()) {
            ProcessedEvent result = processEvent(event);
            out.collect(result);
            span.setStatus(StatusCode.OK);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR);
        } finally {
            span.end();
        }
    }
}
```

## Key Concepts

### Trace
A complete request flow across all services. Contains multiple spans with parent-child relationships.

### Span
A single operation within a trace. Has name, attributes, timestamps, and status.

### Trace ID
Unique identifier for entire request flow. Propagated across all service calls.

### Span ID
Unique identifier for single operation. Used for parent-child span relationships.

### W3C Trace Context
Standard header format:
```
traceparent: 00-<trace_id>-<span_id>-<trace_flags>
tracestate: <vendor-specific-data>
```

### Sampling
Decision to record and export specific traces based on configured strategy.

## Best Practices

1. **Always Initialize OpenTelemetry Early**
   - Configure in application startup
   - Set resource attributes before first request

2. **Propagate Context Everywhere**
   - HTTP calls (RestTemplate, WebClient)
   - Message queues (Kafka, RabbitMQ)
   - Async operations (Reactor, CompletableFuture)

3. **Add Business Context to Spans**
   - User ID, customer ID, order ID
   - Feature flags, tenant information
   - Request metadata

4. **Sample Intelligently**
   - Development: 100% (always-on)
   - Staging: 20-50%
   - Production: 1-10% + 100% errors

5. **Monitor Sampling Rates**
   - Track actual sampled trace counts
   - Alert if sampling rate deviates

6. **Handle Errors Explicitly**
   - Always record exceptions in spans
   - Set span status on error
   - Include error context in attributes

7. **Test Context Propagation**
   - Verify headers in HTTP calls
   - Verify MDC in logs
   - Verify context in async chains

8. **Use Correlation IDs**
   - Independent of distributed tracing
   - Visible in application logs
   - Easy for end-users to reference

## Backend Deployment

### Docker Compose (Jaeger)

```yaml
version: '3'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "6831:6831/udp"  # Agent
      - "16686:16686"    # UI
    environment:
      COLLECTOR_ZIPKIN_HOST_PORT: ":9411"
```

### Kubernetes (OpenTelemetry Collector)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
    exporters:
      jaeger:
        endpoint: jaeger:14250
    service:
      pipelines:
        traces:
          receivers: [otlp]
          exporters: [jaeger]
```

## Troubleshooting

### Traces Not Appearing
1. Verify OpenTelemetry is initialized
2. Check Jaeger backend is running
3. Verify exporter configuration
4. Check sampling is enabled (rate > 0)

### Missing Context in Child Spans
1. Ensure context propagation is configured
2. Use W3C Trace Context headers
3. For async: verify Reactor context propagation

### High Sampling Overhead
1. Reduce sampling rate in production
2. Implement adaptive sampling
3. Use tail-based sampling for selective traces

### Backend Storage Issues
1. Monitor Jaeger storage size
2. Configure retention policies
3. Implement trace sampling (cost reduction)

## Related Documentation

- [OpenTelemetry Java Documentation](https://opentelemetry.io/docs/instrumentation/java/)
- [Jaeger Documentation](https://www.jaegertracing.io/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth)

## Library Navigation

This library focuses on distributed tracing implementation. For related capabilities:
- See **observability** library for metrics and logging
- See **resilience-operations** library for circuit breakers and health checks
- See **security-compliance** library for audit logging with traces
