# Create OpenTelemetry Setup

Configure OpenTelemetry for distributed tracing across Spring Boot, Project Reactor, Apache Flink, and other frameworks. OpenTelemetry provides vendor-neutral instrumentation for end-to-end request tracing in microservices architectures.

## Requirements

Your OpenTelemetry setup should include:

- **OpenTelemetry SDK Configuration**: Initialize and configure SDK for your framework
- **Trace Exporter**: Export traces to Jaeger, Zipkin, or OTLP-compatible backend
- **Auto-Instrumentation**: Leverage automatic instrumentation libraries
- **Manual Instrumentation**: Create custom spans for business logic
- **Span Attributes**: Enrich spans with business context (user ID, order ID, etc.)
- **Baggage**: Propagate values across process boundaries
- **Sampling Strategies**: Configure sampling for high-volume systems
- **Exporter Configuration**: Batch vs. immediate export strategies
- **Error Handling**: Capture exceptions and error details in spans
- **Resource Attributes**: Add service identity and deployment info

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Project Reactor, Apache Flink
- **Dependencies**: OpenTelemetry API, SDK, auto-instrumentation libraries
- **Configuration**: Environment variables + programmatic setup

## Template: Spring Boot with OpenTelemetry

```java
import io.opentelemetry.api.OpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.api.trace.Span;
import io.opentelemetry.api.common.Attributes;
import io.opentelemetry.api.common.AttributeKey;
import io.opentelemetry.sdk.OpenTelemetrySdk;
import io.opentelemetry.sdk.trace.SdkTracerProvider;
import io.opentelemetry.sdk.trace.export.BatchSpanProcessor;
import io.opentelemetry.sdk.resources.Resource;
import io.opentelemetry.semconv.resource.attributes.ResourceAttributes;
import io.opentelemetry.exporter.jaeger.thrift.JaegerThriftSpanExporter;
import io.opentelemetry.sdk.trace.samplers.Sampler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;

/**
 * OpenTelemetry configuration for Spring Boot applications.
 * Provides distributed tracing across microservices.
 */
@Slf4j
@Configuration
@RequiredArgsConstructor
public class OpenTelemetryConfig {

    /**
     * Configure OpenTelemetry SDK with Jaeger exporter.
     */
    @Bean
    public OpenTelemetry openTelemetry() {
        // Create resource with service metadata
        Resource resource = Resource.getDefault()
                .merge(Resource.create(Attributes.of(
                        ResourceAttributes.SERVICE_NAME, "user-service",
                        ResourceAttributes.SERVICE_VERSION, "1.0.0",
                        ResourceAttributes.DEPLOYMENT_ENVIRONMENT, getEnvironment()
                )));

        // Configure Jaeger exporter
        JaegerThriftSpanExporter jaegerExporter = JaegerThriftSpanExporter.builder()
                .setAgentHost(System.getenv("JAEGER_HOST") != null ? System.getenv("JAEGER_HOST") : "localhost")
                .setAgentPort(Integer.parseInt(System.getenv("JAEGER_PORT") != null ? System.getenv("JAEGER_PORT") : "6831"))
                .build();

        // Configure sampler (80% sampling rate in production)
        Sampler sampler = Sampler.traceIdRatioBased(getSamplingRate());

        // Create SDK with sampler and exporter
        SdkTracerProvider tracerProvider = SdkTracerProvider.builder()
                .setResource(resource)
                .addSpanProcessor(BatchSpanProcessor.builder(jaegerExporter)
                        .setMaxQueueSize(2048)
                        .setScheduleDelayMillis(5000)
                        .setMaxExportBatchSize(512)
                        .build())
                .setSampler(sampler)
                .build();

        return OpenTelemetrySdk.builder()
                .setTracerProvider(tracerProvider)
                .build();
    }

    /**
     * Tracer bean for injection into components.
     */
    @Bean
    public Tracer tracer(OpenTelemetry openTelemetry) {
        return openTelemetry.getTracer("user-service", "1.0.0");
    }

    private String getEnvironment() {
        return System.getenv("ENVIRONMENT") != null ? System.getenv("ENVIRONMENT") : "development";
    }

    private double getSamplingRate() {
        String rate = System.getenv("SAMPLING_RATE");
        return rate != null ? Double.parseDouble(rate) : 0.8;
    }
}
```

## Usage Example: Custom Span in Service

```java
@Slf4j
@Service
@RequiredArgsConstructor
public class OrderService {
    private final Tracer tracer;
    private final OrderRepository orderRepository;

    public Order createOrder(CreateOrderRequest request) {
        // Create span for order creation
        Span span = tracer.spanBuilder("order.create")
                .setAttribute("order.customerId", request.customerId())
                .setAttribute("order.amount", request.amount())
                .setAttribute("order.items", request.items().size())
                .startSpan();

        try (var scope = span.makeCurrent()) {
            log.info("Creating order for customer: {}", request.customerId());

            // Add event
            span.addEvent("order.validation.started");
            validateOrder(request);
            span.addEvent("order.validation.completed");

            // Add event with attributes
            span.addEvent("order.persistence.started",
                    Attributes.of(AttributeKey.stringKey("persistence.type"), "jdbc"));
            Order order = orderRepository.save(new Order(request));
            span.addEvent("order.persistence.completed",
                    Attributes.of(AttributeKey.longKey("order.id"), order.id()));

            span.setStatus(StatusCode.OK);
            return order;

        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, "Order creation failed: " + e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }

    private void validateOrder(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("order.validate")
                .setParent(Context.current())
                .startSpan();

        try (var scope = span.makeCurrent()) {
            if (request.amount() <= 0) {
                throw new ValidationException("Invalid order amount");
            }
            span.setStatus(StatusCode.OK);
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

## Project Reactor Integration

```java
@Slf4j
@Component
public class ReactiveOrderService {
    private final Tracer tracer;
    private final ReactiveOrderRepository repository;

    public Mono<Order> createOrderReactive(CreateOrderRequest request) {
        return Mono.defer(() -> {
            Span span = tracer.spanBuilder("order.create.reactive")
                    .setAttribute("order.customerId", request.customerId())
                    .startSpan();

            return Mono.just(request)
                    .doOnNext(req -> span.addEvent("order.validation.started"))
                    .flatMap(this::validateOrderAsync)
                    .doOnNext(req -> span.addEvent("order.validation.completed"))
                    .flatMap(repository::save)
                    .doOnNext(order -> {
                        span.setAttribute("order.id", order.id());
                        span.setStatus(StatusCode.OK);
                    })
                    .doFinally(signalType -> span.end())
                    .doOnError(error -> {
                        span.recordException(error);
                        span.setStatus(StatusCode.ERROR, error.getMessage());
                    });
        });
    }

    private Mono<CreateOrderRequest> validateOrderAsync(CreateOrderRequest request) {
        Span span = tracer.spanBuilder("order.validate")
                .setParent(Context.current())
                .startSpan();

        return Mono.just(request)
                .doOnNext(req -> {
                    if (req.amount() <= 0) {
                        throw new ValidationException("Invalid amount");
                    }
                    span.setStatus(StatusCode.OK);
                })
                .doFinally(signalType -> span.end())
                .onErrorResume(error -> {
                    span.recordException(error);
                    span.setStatus(StatusCode.ERROR);
                    return Mono.error(error);
                });
    }
}
```

## Apache Flink Integration

```java
@Slf4j
public class TracedFlatMapFunction extends RichFlatMapFunction<Event, ProcessedEvent> {
    private transient Tracer tracer;

    @Override
    public void open(Configuration parameters) throws Exception {
        // Initialize OpenTelemetry in Flink task
        tracer = OpenTelemetry.getGlobalTracer("flink-job", "1.0.0");
    }

    @Override
    public void flatMap(Event event, Collector<ProcessedEvent> out) {
        Span span = tracer.spanBuilder("event.process")
                .setAttribute("event.id", event.id())
                .setAttribute("event.type", event.type())
                .setAttribute("flink.task", getRuntimeContext().getTaskName())
                .startSpan();

        try (var scope = span.makeCurrent()) {
            ProcessedEvent processed = processEvent(event);
            span.setStatus(StatusCode.OK);
            out.collect(processed);
        } catch (Exception e) {
            span.recordException(e);
            span.setStatus(StatusCode.ERROR, e.getMessage());
            throw new RuntimeException(e);
        } finally {
            span.end();
        }
    }

    private ProcessedEvent processEvent(Event event) {
        return new ProcessedEvent(event.id(), "processed");
    }
}
```

## Configuration via Environment Variables

```bash
# Jaeger Configuration
export JAEGER_HOST=localhost
export JAEGER_PORT=6831
export JAEGER_SAMPLER_TYPE=const
export JAEGER_SAMPLER_PARAM=1

# Service Identity
export SERVICE_NAME=user-service
export SERVICE_VERSION=1.0.0
export ENVIRONMENT=production

# Sampling
export SAMPLING_RATE=0.8

# OTLP Exporter (alternative to Jaeger)
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_EXPORTER_OTLP_PROTOCOL=grpc
```

## Jaeger Backend Setup (Docker Compose)

```yaml
version: '3'
services:
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "6831:6831/udp"      # Jaeger agent
      - "16686:16686"        # UI
      - "14268:14268"        # Collector
    environment:
      COLLECTOR_ZIPKIN_HOST_PORT: ":9411"
```

## Best Practices

1. **Resource Attributes**: Always include service name, version, and environment
2. **Sampling**: Use higher rates in development (1.0), lower in production (0.1-0.5)
3. **Batch Export**: Use BatchSpanProcessor for better throughput and lower overhead
4. **Context Propagation**: Always propagate trace context across thread boundaries
5. **Span Naming**: Use consistent naming: `service.operation` format
6. **Attributes**: Add business context (user ID, customer ID) to spans
7. **Error Handling**: Record exceptions with `span.recordException()`
8. **Status**: Always set span status (OK or ERROR)

## Related Documentation

- [OpenTelemetry Java Documentation](https://opentelemetry.io/docs/instrumentation/java/)
- [Jaeger Getting Started](https://www.jaegertracing.io/docs/getting-started/)
- [Trace Exporters](https://opentelemetry.io/docs/reference/specification/protocol/exporter/)
