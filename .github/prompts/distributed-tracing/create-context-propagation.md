# Create Context Propagation

Implement context propagation for distributed tracing across service boundaries. Context includes trace IDs, span IDs, and correlation data that must be propagated across HTTP headers, message queues, and reactive streams.

## Requirements

Your context propagation implementation should include:

- **W3C Trace Context Standard**: Implement W3C Trace Context headers (traceparent, tracestate)
- **Legacy Jaeger Headers**: Support Jaeger trace headers for backward compatibility
- **HTTP Header Propagation**: Propagate trace context in HTTP requests/responses
- **Message Queue Integration**: Propagate context through message queues (Kafka, RabbitMQ)
- **Reactive Stream Propagation**: Maintain context across thread boundaries in Project Reactor
- **Spring Cloud Sleuth Integration**: Use Spring Cloud Sleuth for automatic context propagation
- **Correlation IDs**: Add custom correlation IDs for business tracking
- **MDC Integration**: Map Diagnostic Context integration for logging
- **Context Extraction**: Extract context from incoming requests
- **Documentation**: Clear guidance on propagation strategies

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Project Reactor, Spring Cloud Sleuth
- **Propagation**: W3C Trace Context and Jaeger headers
- **Logging**: SLF4J with MDC

## Template Example

```java
import io.opentelemetry.api.trace.SpanContext;
import io.opentelemetry.api.trace.TraceFlags;
import io.opentelemetry.api.trace.TraceState;
import io.opentelemetry.api.GlobalOpenTelemetry;
import io.opentelemetry.api.trace.Tracer;
import io.opentelemetry.context.Context;
import io.opentelemetry.context.Scope;
import io.opentelemetry.context.propagation.TextMapPropagator;
import io.opentelemetry.extension.trace.propagation.JaegerPropagator;
import org.springframework.web.filter.OncePerRequestFilter;
import org.springframework.web.client.RestTemplate;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import org.slf4j.MDC;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import lombok.extern.slf4j.Slf4j;
import lombok.RequiredArgsConstructor;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import reactor.util.context.Context;
import java.util.HashMap;
import java.util.Map;
import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * Context propagation for distributed tracing across service boundaries.
 * Implements W3C Trace Context standard and Jaeger header support.
 */
@Slf4j
public class ContextPropagation {

    /**
     * HTTP filter for extracting and propagating trace context.
     * Extracts W3C traceparent header and sets MDC for logging.
     */
    @Component
    @RequiredArgsConstructor
    public static class TraceContextFilter extends OncePerRequestFilter {
        private final Tracer tracer;

        @Override
        protected void doFilterInternal(HttpServletRequest request,
                                       HttpServletResponse response,
                                       FilterChain filterChain) throws ServletException, IOException {
            // Extract W3C Trace Context
            String traceparent = request.getHeader("traceparent");
            String tracestate = request.getHeader("tracestate");

            if (traceparent != null) {
                // Parse W3C traceparent format: version-trace_id-parent_id-trace_flags
                String[] parts = traceparent.split("-");
                if (parts.length >= 3) {
                    String traceId = parts[1];
                    String spanId = parts[2];
                    String traceFlags = parts.length > 3 ? parts[3] : "01";

                    // Set MDC for logging correlation
                    MDC.put("traceId", traceId);
                    MDC.put("spanId", spanId);
                    MDC.put("tracingEnabled", "1".equals(traceFlags));

                    log.info("Incoming request - TraceId: {}, SpanId: {}", traceId, spanId);
                }
            }

            // Extract Jaeger headers (backward compatibility)
            String jaegerTraceId = request.getHeader("uber-trace-id");
            if (jaegerTraceId != null && MDC.get("traceId") == null) {
                String[] parts = jaegerTraceId.split(":");
                if (parts.length >= 2) {
                    MDC.put("traceId", parts[0]);
                    MDC.put("spanId", parts[1]);
                    log.info("Detected Jaeger trace format - TraceId: {}", parts[0]);
                }
            }

            try {
                filterChain.doFilter(request, response);
            } finally {
                // Clear MDC
                MDC.clear();
            }
        }
    }

    /**
     * RestTemplate interceptor for propagating trace context to downstream services.
     * Adds W3C traceparent and tracestate headers to outgoing requests.
     */
    @Component
    @RequiredArgsConstructor
    public static class TraceContextClientInterceptor implements org.springframework.http.client.ClientHttpRequestInterceptor {
        private final Tracer tracer;

        @Override
        public org.springframework.http.client.ClientHttpResponse intercept(
                org.springframework.http.client.ClientHttpRequest request,
                byte[] body,
                org.springframework.http.client.ClientHttpRequestExecution execution) throws IOException {

            // Get current trace context from MDC
            String traceId = MDC.get("traceId");
            String spanId = MDC.get("spanId");

            if (traceId != null && spanId != null) {
                // Build W3C traceparent header
                String traceparent = String.format("00-%s-%s-01", traceId, spanId);
                request.getHeaders().set("traceparent", traceparent);

                // Optionally add vendor-specific tracestate
                String tracestate = MDC.get("tracestate");
                if (tracestate != null) {
                    request.getHeaders().set("tracestate", tracestate);
                }

                log.debug("Propagating trace context: {}", traceparent);
            }

            return execution.execute(request, body);
        }
    }

    /**
     * Correlation ID generator for business-level tracking.
     * Generates unique correlation IDs independent of trace IDs.
     */
    @Component
    public static class CorrelationIdGenerator {
        private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
        private static final String REQUEST_ID_MDC_KEY = "correlationId";

        public String generateCorrelationId() {
            return java.util.UUID.randomUUID().toString();
        }

        public String extractOrGenerate(HttpServletRequest request) {
            String correlationId = request.getHeader(CORRELATION_ID_HEADER);
            if (correlationId == null || correlationId.isEmpty()) {
                correlationId = generateCorrelationId();
            }
            return correlationId;
        }

        public void setToMDC(String correlationId) {
            MDC.put(REQUEST_ID_MDC_KEY, correlationId);
        }

        public String getFromMDC() {
            return MDC.get(REQUEST_ID_MDC_KEY);
        }
    }

    /**
     * Reactive context propagation for Project Reactor.
     * Maintains trace context across async boundaries.
     */
    @Component
    @RequiredArgsConstructor
    public static class ReactiveContextPropagation {
        private final Tracer tracer;

        /**
         * Extract trace context from request and create reactor context.
         */
        public reactor.util.context.Context extractToReactorContext(HttpServletRequest request) {
            String traceparent = request.getHeader("traceparent");
            String correlationId = request.getHeader("X-Correlation-ID");

            Map<String, String> contextMap = new HashMap<>();
            if (traceparent != null) {
                contextMap.put("traceparent", traceparent);
            }
            if (correlationId != null) {
                contextMap.put("correlationId", correlationId);
            }

            return reactor.util.context.Context.of("traceContext", contextMap);
        }

        /**
         * Apply trace context to Mono operation.
         */
        public <T> Mono<T> propagateContext(Mono<T> mono, reactor.util.context.Context context) {
            return mono
                    .contextWrite(context)
                    .doOnNext(value -> {
                        @SuppressWarnings("unchecked")
                        Map<String, String> traceContext =
                                context.get("traceContext");
                        String traceId = traceContext.get("traceparent");
                        if (traceId != null) {
                            log.debug("Executing with trace context: {}", traceId);
                        }
                    });
        }

        /**
         * Apply trace context to Flux operation.
         */
        public <T> Flux<T> propagateContext(Flux<T> flux, reactor.util.context.Context context) {
            return flux
                    .contextWrite(context)
                    .doOnNext(value -> {
                        @SuppressWarnings("unchecked")
                        Map<String, String> traceContext =
                                context.get("traceContext");
                        String correlationId = traceContext.get("correlationId");
                        if (correlationId != null) {
                            log.debug("Processing item with correlation: {}", correlationId);
                        }
                    });
        }
    }

    /**
     * Message queue integration for Kafka trace context propagation.
     */
    @Component
    @Slf4j
    public static class KafkaContextPropagator {

        /**
         * Extract trace context from Kafka message headers.
         */
        public Map<String, String> extractFromKafkaHeaders(
                org.apache.kafka.common.header.Headers headers) {
            Map<String, String> context = new HashMap<>();

            org.apache.kafka.common.header.Header traceparent =
                    headers.lastHeader("traceparent");
            if (traceparent != null) {
                context.put("traceparent",
                        new String(traceparent.value(), java.nio.charset.StandardCharsets.UTF_8));
            }

            org.apache.kafka.common.header.Header correlationId =
                    headers.lastHeader("X-Correlation-ID");
            if (correlationId != null) {
                context.put("correlationId",
                        new String(correlationId.value(), java.nio.charset.StandardCharsets.UTF_8));
            }

            return context;
        }

        /**
         * Add trace context to outgoing Kafka message headers.
         */
        public void propagateToKafkaHeaders(
                org.apache.kafka.common.header.Headers headers,
                Map<String, String> traceContext) {

            String traceparent = MDC.get("traceId") != null ?
                    String.format("00-%s-%s-01", MDC.get("traceId"), MDC.get("spanId")) :
                    traceContext.get("traceparent");

            String correlationId = MDC.get("correlationId") != null ?
                    MDC.get("correlationId") :
                    traceContext.get("correlationId");

            if (traceparent != null) {
                headers.add("traceparent", traceparent.getBytes(java.nio.charset.StandardCharsets.UTF_8));
            }

            if (correlationId != null) {
                headers.add("X-Correlation-ID", correlationId.getBytes(java.nio.charset.StandardCharsets.UTF_8));
            }

            log.debug("Propagated trace context to Kafka: {}", traceparent);
        }
    }

    /**
     * Spring Cloud Sleuth integration for automatic context propagation.
     * Sleuth automatically handles trace context in HTTP and messaging.
     */
    @Component
    @Slf4j
    public static class SleuthContextProvider {

        /**
         * Get current trace context from Spring Cloud Sleuth.
         * Sleuth manages tracerId and spanId automatically.
         */
        public TraceContextInfo getCurrentTraceContext(
                brave.Tracer braveTracer) {
            brave.propagation.TraceContext braveContext = braveTracer.currentSpan() != null ?
                    braveTracer.currentSpan().context() :
                    null;

            if (braveContext != null) {
                return new TraceContextInfo(
                        braveContext.traceIdString(),
                        braveContext.spanIdString(),
                        braveContext.sampled()
                );
            }

            return null;
        }

        /**
         * Extract context from HTTP headers using Sleuth's propagator.
         */
        public TraceContextInfo extractFromHttpHeaders(HttpServletRequest request) {
            String traceparent = request.getHeader("traceparent");
            if (traceparent != null) {
                String[] parts = traceparent.split("-");
                if (parts.length >= 3) {
                    return new TraceContextInfo(
                            parts[1],  // traceId
                            parts[2],  // spanId
                            parts.length > 3 && "01".equals(parts[3])  // sampled
                    );
                }
            }

            String jaegerTrace = request.getHeader("uber-trace-id");
            if (jaegerTrace != null) {
                String[] parts = jaegerTrace.split(":");
                if (parts.length >= 2) {
                    return new TraceContextInfo(
                            parts[0],  // traceId
                            parts[1],  // spanId
                            parts.length > 3 && !"0".equals(parts[3])  // sampled
                    );
                }
            }

            return null;
        }
    }

    /**
     * Baggage propagation for passing business context across service boundaries.
     * Baggage items are automatically included in span attributes.
     */
    @Component
    @Slf4j
    public static class BaggagePropagator {
        private static final String BAGGAGE_HEADER = "W3C-Baggage";

        /**
         * Add baggage item to current context.
         */
        public void setBaggageItem(String key, String value) {
            String baggage = MDC.get(BAGGAGE_HEADER);
            if (baggage == null) {
                baggage = "";
            }
            baggage += key + "=" + value + ",";
            MDC.put(BAGGAGE_HEADER, baggage);
        }

        /**
         * Get baggage item from current context.
         */
        public String getBaggageItem(String key) {
            String baggage = MDC.get(BAGGAGE_HEADER);
            if (baggage != null) {
                String[] items = baggage.split(",");
                for (String item : items) {
                    if (item.startsWith(key + "=")) {
                        return item.substring(key.length() + 1);
                    }
                }
            }
            return null;
        }

        /**
         * Extract all baggage items from header.
         */
        public Map<String, String> extractBaggageFromHeader(HttpServletRequest request) {
            Map<String, String> baggage = new HashMap<>();
            String baggageHeader = request.getHeader(BAGGAGE_HEADER);

            if (baggageHeader != null) {
                String[] items = baggageHeader.split(",");
                for (String item : items) {
                    if (item.contains("=")) {
                        String[] parts = item.split("=");
                        baggage.put(parts[0].trim(), parts[1].trim());
                    }
                }
            }

            return baggage;
        }
    }

    /**
     * Trace context information holder.
     */
    public record TraceContextInfo(String traceId, String spanId, boolean sampled) {
        @Override
        public String toString() {
            return String.format("TraceId:%s, SpanId:%s, Sampled:%s", traceId, spanId, sampled);
        }
    }

    /**
     * Best practices for context propagation.
     */
    public static class BestPractices {
        /**
         * 1. Always use W3C Trace Context format (traceparent header)
         *    - Standard format: version-trace_id-parent_id-trace_flags
         *    - Universally supported by modern observability tools
         */

        /**
         * 2. Support Jaeger headers for backward compatibility
         *    - Header: uber-trace-id
         *    - Format: trace_id:span_id:parent_span_id:sampling_decision
         */

        /**
         * 3. Propagate context through all async boundaries
         *    - HTTP requests (RestTemplate, WebClient)
         *    - Message queues (Kafka, RabbitMQ)
         *    - Reactive streams (Mono, Flux)
         */

        /**
         * 4. Use correlation IDs for business-level tracking
         *    - Independent of distributed tracing infrastructure
         *    - Visible in application logs and UI
         *    - Recommended header: X-Correlation-ID
         */

        /**
         * 5. Maintain MDC for logging context
         *    - Set MDC.put("traceId", traceId) on request entry
         *    - Use %X{traceId} in log pattern
         *    - Clear MDC in finally block or filter
         */

        /**
         * 6. Use Spring Cloud Sleuth for automatic propagation
         *    - Sleuth handles most propagation automatically
         *    - Less boilerplate code required
         *    - Works with Spring's RestTemplate and WebClient
         */

        /**
         * 7. Pass baggage for request-scoped data
         *    - User ID, tenant ID, feature flags
         *    - Automatically added to span attributes
         *    - Visible in tracing UI
         */

        /**
         * 8. Test context propagation explicitly
         *    - Verify headers in mocked HTTP calls
         *    - Verify MDC in logs
         *    - Verify context in async operations
         */
    }
}
```

## W3C Trace Context Header Examples

```
# Request with W3C Trace Context
GET /api/orders HTTP/1.1
Host: api.example.com
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: congo=t61rcZ94W243

# Request with Jaeger headers (backward compatibility)
GET /api/orders HTTP/1.1
Host: api.example.com
uber-trace-id: 4bf92f3577b34da6a3ce929d0e0e4736:00f067aa0ba902b7:0:1

# Request with correlation ID
GET /api/orders HTTP/1.1
Host: api.example.com
X-Correlation-ID: 550e8400-e29b-41d4-a716-446655440000
```

## MDC Configuration for Logging

```properties
# application.properties
logging.pattern.console=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %X{traceId} %X{correlationId} - %msg%n
logging.pattern.file=%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %X{traceId} %X{correlationId} %X{spanId} - %msg%n
```

## Context Propagation Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Incoming Request                    │
│              (with traceparent header)               │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────┐
        │ TraceContext    │
        │ Filter          │
        └────────┬────────┘
                 │ Extract W3C/Jaeger headers
                 │ Set MDC
                 ▼
        ┌─────────────────┐
        │ Service Logic   │
        │ (with context)  │
        └────────┬────────┘
                 │
        ┌────────┼─────────────┐
        │        │             │
        ▼        ▼             ▼
    Downstream RestTemplate  Kafka  Reactive
    HTTP Call  (propagate)  (propagate) (propagate)
```

## Related Documentation

- [W3C Trace Context Specification](https://www.w3.org/TR/trace-context/)
- [Jaeger Trace Propagation](https://www.jaegertracing.io/docs/latest/client-libraries/)
- [Spring Cloud Sleuth Documentation](https://spring.io/projects/spring-cloud-sleuth)
- [OpenTelemetry Context Propagation](https://opentelemetry.io/docs/instrumentation/java/manual/#context)
- [Baggage Specification](https://www.w3.org/TR/baggage/)
```
