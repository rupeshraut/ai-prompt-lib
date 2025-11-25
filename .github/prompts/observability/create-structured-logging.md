# Create Structured Logging

Implement structured JSON logging for production-grade observability, log aggregation, and analysis. Use structured logs to enable efficient querying, correlation with traces, and integration with centralized logging platforms like ELK Stack, Splunk, and Datadog.

## Requirements

Your structured logging implementation should include:

- **JSON Format**: All logs output in structured JSON
- **Standard Fields**: Timestamp, level, message, logger, thread, trace IDs
- **Context Enrichment**: Add business context to logs (user ID, tenant ID, correlation ID)
- **MDC Integration**: Propagate context across threads and service calls
- **Trace Correlation**: Include trace ID and span ID from OpenTelemetry
- **Exception Serialization**: Structured stack traces with context
- **Performance Optimization**: Async appenders for minimal latency impact
- **Log Levels**: Environment-specific log level configuration
- **Sensitive Data Masking**: Redact PII and secrets from logs
- **Centralized Configuration**: Logback/Log4j2 configuration with profiles

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Logback, SLF4J
- **Libraries**: Logstash Logback Encoder, MDC
- **Integration**: OpenTelemetry, Spring Cloud Sleuth

## Template: Structured Logging Configuration

```java
import org.springframework.context.annotation.Configuration;
import org.slf4j.MDC;
import lombok.extern.slf4j.Slf4j;
import jakarta.servlet.Filter;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.ServletRequest;
import jakarta.servlet.ServletResponse;
import jakarta.servlet.http.HttpServletRequest;
import java.io.IOException;
import java.util.UUID;

/**
 * Logging configuration with MDC context setup.
 */
@Slf4j
@Configuration
public class LoggingConfiguration {

    private static final String CORRELATION_ID_HEADER = "X-Correlation-ID";
    private static final String USER_ID_HEADER = "X-User-ID";
    private static final String TENANT_ID_HEADER = "X-Tenant-ID";

    /**
     * Filter to populate MDC with request context.
     */
    @Bean
    public Filter mdcContextFilter() {
        return new MdcContextFilter();
    }
}

/**
 * Servlet filter that populates MDC with request context.
 * Context is automatically included in all log statements.
 */
@Slf4j
class MdcContextFilter implements Filter {

    private static final String CORRELATION_ID = "correlationId";
    private static final String USER_ID = "userId";
    private static final String TENANT_ID = "tenantId";
    private static final String REQUEST_URI = "requestUri";
    private static final String REQUEST_METHOD = "requestMethod";
    private static final String CLIENT_IP = "clientIp";

    @Override
    public void doFilter(
            ServletRequest request,
            ServletResponse response,
            FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        try {
            // Extract or generate correlation ID
            String correlationId = httpRequest.getHeader("X-Correlation-ID");
            if (correlationId == null || correlationId.isBlank()) {
                correlationId = UUID.randomUUID().toString();
            }
            MDC.put(CORRELATION_ID, correlationId);

            // Extract user context
            String userId = httpRequest.getHeader("X-User-ID");
            if (userId != null && !userId.isBlank()) {
                MDC.put(USER_ID, userId);
            }

            // Extract tenant context
            String tenantId = httpRequest.getHeader("X-Tenant-ID");
            if (tenantId != null && !tenantId.isBlank()) {
                MDC.put(TENANT_ID, tenantId);
            }

            // Add request details
            MDC.put(REQUEST_URI, httpRequest.getRequestURI());
            MDC.put(REQUEST_METHOD, httpRequest.getMethod());
            MDC.put(CLIENT_IP, getClientIP(httpRequest));

            log.info("Request started");

            // Continue request processing
            chain.doFilter(request, response);

            log.info("Request completed");

        } finally {
            // Clear MDC to prevent memory leaks
            MDC.clear();
        }
    }

    private String getClientIP(HttpServletRequest request) {
        String clientIP = request.getHeader("X-Forwarded-For");
        if (clientIP == null || clientIP.isBlank()) {
            clientIP = request.getRemoteAddr();
        }
        return clientIP;
    }
}
```

## Service Layer with Structured Logging

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.slf4j.MDC;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import net.logstash.logback.argument.StructuredArguments;

import static net.logstash.logback.argument.StructuredArguments.*;

/**
 * Service with structured logging best practices.
 */
@Service
@RequiredArgsConstructor
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;

    /**
     * Create order with comprehensive structured logging.
     */
    public Order createOrder(CreateOrderRequest request) {
        // Add business context to MDC
        MDC.put("customerId", request.customerId());
        MDC.put("orderAmount", String.valueOf(request.amount()));

        log.info("Creating order for customer",
            keyValue("customerId", request.customerId()),
            keyValue("amount", request.amount()),
            keyValue("itemCount", request.items().size())
        );

        try {
            // Validate order
            validateOrder(request);

            // Create order entity
            Order order = new Order();
            order.setCustomerId(request.customerId());
            order.setAmount(request.amount());
            order.setStatus(OrderStatus.PENDING);

            // Save to database
            Order savedOrder = orderRepository.save(order);

            log.info("Order created successfully",
                keyValue("orderId", savedOrder.getId()),
                keyValue("status", savedOrder.getStatus())
            );

            // Process payment
            processPayment(savedOrder);

            // Update status
            savedOrder.setStatus(OrderStatus.CONFIRMED);
            orderRepository.save(savedOrder);

            log.info("Order confirmed",
                keyValue("orderId", savedOrder.getId()),
                keyValue("finalStatus", savedOrder.getStatus())
            );

            return savedOrder;

        } catch (ValidationException e) {
            log.warn("Order validation failed",
                keyValue("customerId", request.customerId()),
                keyValue("reason", e.getMessage()),
                keyValue("validationErrors", e.getErrors())
            );
            throw e;

        } catch (PaymentException e) {
            log.error("Payment processing failed",
                keyValue("customerId", request.customerId()),
                keyValue("amount", request.amount()),
                keyValue("errorCode", e.getErrorCode()),
                keyValue("errorMessage", e.getMessage()),
                e
            );
            throw e;

        } catch (Exception e) {
            log.error("Unexpected error creating order",
                keyValue("customerId", request.customerId()),
                keyValue("errorType", e.getClass().getSimpleName()),
                e
            );
            throw new OrderCreationException("Failed to create order", e);

        } finally {
            // Clean up MDC context
            MDC.remove("customerId");
            MDC.remove("orderAmount");
        }
    }

    private void validateOrder(CreateOrderRequest request) {
        log.debug("Validating order",
            keyValue("customerId", request.customerId()),
            keyValue("itemCount", request.items().size())
        );

        if (request.amount() <= 0) {
            throw new ValidationException("Invalid order amount");
        }

        if (request.items().isEmpty()) {
            throw new ValidationException("Order must contain items");
        }

        log.debug("Order validation passed");
    }

    private void processPayment(Order order) {
        log.info("Processing payment",
            keyValue("orderId", order.getId()),
            keyValue("amount", order.getAmount())
        );

        try {
            paymentService.processPayment(order.getId(), order.getAmount());

            log.info("Payment processed successfully",
                keyValue("orderId", order.getId())
            );

        } catch (Exception e) {
            log.error("Payment failed",
                keyValue("orderId", order.getId()),
                keyValue("amount", order.getAmount()),
                e
            );
            throw new PaymentException("Payment processing failed", e);
        }
    }
}
```

## Reactive Logging with Project Reactor

```java
import reactor.core.publisher.Mono;
import reactor.util.context.Context;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;

/**
 * Reactive service with structured logging.
 * Uses Reactor Context for MDC propagation.
 */
@Service
@RequiredArgsConstructor
public class ReactiveOrderService {

    private static final Logger log = LoggerFactory.getLogger(ReactiveOrderService.class);

    private final ReactiveOrderRepository orderRepository;
    private final ReactivePaymentService paymentService;

    /**
     * Create order reactively with logging context.
     */
    public Mono<Order> createOrder(CreateOrderRequest request) {
        return Mono.just(request)
            .doOnNext(req ->
                log.info("Creating order for customer: {}",
                    keyValue("customerId", req.customerId()))
            )
            .flatMap(this::validateOrderAsync)
            .map(this::createOrderEntity)
            .flatMap(orderRepository::save)
            .doOnNext(order ->
                log.info("Order saved: {}", keyValue("orderId", order.getId()))
            )
            .flatMap(this::processPaymentAsync)
            .doOnNext(order ->
                log.info("Order completed: {}", keyValue("orderId", order.getId()))
            )
            .doOnError(error ->
                log.error("Order creation failed: {}",
                    keyValue("customerId", request.customerId()),
                    error)
            )
            .contextWrite(Context.of(
                "customerId", request.customerId(),
                "correlationId", UUID.randomUUID().toString()
            ));
    }

    private Mono<CreateOrderRequest> validateOrderAsync(CreateOrderRequest request) {
        return Mono.defer(() -> {
            log.debug("Validating order: {}",
                keyValue("customerId", request.customerId())
            );

            if (request.amount() <= 0) {
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

    private Mono<Order> processPaymentAsync(Order order) {
        return paymentService
            .processPayment(order.getId(), order.getAmount())
            .doOnNext(payment ->
                log.info("Payment processed: {}",
                    keyValue("paymentId", payment.getId()))
            )
            .map(payment -> {
                order.setStatus(OrderStatus.CONFIRMED);
                return order;
            })
            .flatMap(orderRepository::save);
    }
}
```

## Logback Configuration (logback-spring.xml)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- Properties -->
    <property name="LOG_FILE" value="${LOG_FILE:-${LOG_PATH:-${LOG_TEMP:-${java.io.tmpdir:-/tmp}}/}spring.log}"/>
    <property name="APPLICATION_NAME" value="${spring.application.name:-application}"/>

    <!-- Console Appender with JSON formatting -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>tenantId</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
            <includeMdcKeyName>customerId</includeMdcKeyName>
            <includeMdcKeyName>orderId</includeMdcKeyName>

            <customFields>
                {
                    "application": "${APPLICATION_NAME}",
                    "environment": "${ENVIRONMENT:-development}"
                }
            </customFields>

            <!-- Field names customization -->
            <fieldNames>
                <timestamp>timestamp</timestamp>
                <message>message</message>
                <logger>logger</logger>
                <thread>thread</thread>
                <level>level</level>
                <stackTrace>stackTrace</stackTrace>
            </fieldNames>

            <!-- Shorten logger names -->
            <shortenedLoggerNameLength>36</shortenedLoggerNameLength>
        </encoder>
    </appender>

    <!-- Async Console Appender for performance -->
    <appender name="ASYNC_CONSOLE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <appender-ref ref="CONSOLE"/>
    </appender>

    <!-- File Appender with JSON formatting -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_FILE}</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>correlationId</includeMdcKeyName>
            <includeMdcKeyName>userId</includeMdcKeyName>
            <includeMdcKeyName>tenantId</includeMdcKeyName>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>

            <customFields>
                {
                    "application": "${APPLICATION_NAME}",
                    "environment": "${ENVIRONMENT:-development}"
                }
            </customFields>
        </encoder>

        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <!-- Async File Appender -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <queueSize>512</queueSize>
        <discardingThreshold>0</discardingThreshold>
        <includeCallerData>false</includeCallerData>
        <appender-ref ref="FILE"/>
    </appender>

    <!-- Root logger -->
    <root level="INFO">
        <appender-ref ref="ASYNC_CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </root>

    <!-- Application-specific loggers -->
    <logger name="com.example.orderservice" level="DEBUG" additivity="false">
        <appender-ref ref="ASYNC_CONSOLE"/>
        <appender-ref ref="ASYNC_FILE"/>
    </logger>

    <!-- Third-party libraries -->
    <logger name="org.springframework.web" level="INFO"/>
    <logger name="org.hibernate" level="INFO"/>
    <logger name="org.apache.kafka" level="WARN"/>

    <!-- Profile-specific configurations -->
    <springProfile name="development">
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <springProfile name="production">
        <root level="INFO">
            <appender-ref ref="ASYNC_CONSOLE"/>
            <appender-ref ref="ASYNC_FILE"/>
        </root>
    </springProfile>

</configuration>
```

## Sensitive Data Masking

```java
import ch.qos.logback.classic.pattern.MessageConverter;
import ch.qos.logback.classic.spi.ILoggingEvent;
import java.util.regex.Pattern;
import java.util.regex.Matcher;

/**
 * Logback converter that masks sensitive data in log messages.
 */
public class SensitiveDataMaskingConverter extends MessageConverter {

    private static final Pattern EMAIL_PATTERN =
        Pattern.compile("([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+\\.[a-zA-Z]{2,})");

    private static final Pattern SSN_PATTERN =
        Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b");

    private static final Pattern CREDIT_CARD_PATTERN =
        Pattern.compile("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b");

    private static final Pattern API_KEY_PATTERN =
        Pattern.compile("(?i)(api[_-]?key|token|password|secret)[\"']?\\s*[:=]\\s*[\"']?([^\\s\"']+)");

    @Override
    public String convert(ILoggingEvent event) {
        String message = super.convert(event);
        return maskSensitiveData(message);
    }

    private String maskSensitiveData(String message) {
        if (message == null) {
            return null;
        }

        // Mask email addresses
        message = EMAIL_PATTERN.matcher(message).replaceAll("$1***@$2");

        // Mask SSN
        message = SSN_PATTERN.matcher(message).replaceAll("XXX-XX-XXXX");

        // Mask credit cards (show last 4 digits)
        Matcher ccMatcher = CREDIT_CARD_PATTERN.matcher(message);
        StringBuffer sb = new StringBuffer();
        while (ccMatcher.find()) {
            String cc = ccMatcher.group().replaceAll("[\\s-]", "");
            String masked = "XXXX-XXXX-XXXX-" + cc.substring(cc.length() - 4);
            ccMatcher.appendReplacement(sb, masked);
        }
        ccMatcher.appendTail(sb);
        message = sb.toString();

        // Mask API keys and secrets
        Matcher apiKeyMatcher = API_KEY_PATTERN.matcher(message);
        message = apiKeyMatcher.replaceAll("$1=***REDACTED***");

        return message;
    }
}
```

## Dependencies (Maven)

```xml
<dependencies>
    <!-- SLF4J and Logback -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
    </dependency>

    <!-- Logstash Logback Encoder for JSON logging -->
    <dependency>
        <groupId>net.logstash.logback</groupId>
        <artifactId>logstash-logback-encoder</artifactId>
        <version>7.4</version>
    </dependency>

    <!-- Spring Boot Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- OpenTelemetry for trace correlation -->
    <dependency>
        <groupId>io.opentelemetry</groupId>
        <artifactId>opentelemetry-api</artifactId>
    </dependency>
</dependencies>
```

## Configuration

```yaml
# application.yml
logging:
  level:
    root: INFO
    com.example.orderservice: DEBUG
    org.springframework.web: INFO
    org.hibernate: INFO

  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"

  file:
    name: ${LOG_FILE:/var/log/order-service/application.log}
    max-size: 100MB
    max-history: 30
    total-size-cap: 10GB

spring:
  application:
    name: order-service

# Environment-specific settings
---
spring.config.activate.on-profile: development

logging:
  level:
    root: DEBUG
    com.example.orderservice: TRACE

---
spring.config.activate.on-profile: production

logging:
  level:
    root: INFO
    com.example.orderservice: INFO
```

## Example JSON Log Output

```json
{
  "timestamp": "2025-01-15T10:30:45.123Z",
  "level": "INFO",
  "message": "Order created successfully",
  "logger": "com.example.orderservice.OrderService",
  "thread": "http-nio-8080-exec-1",
  "application": "order-service",
  "environment": "production",
  "correlationId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "traceId": "4bf92f3577b34da6a3ce929d0e0e4736",
  "spanId": "00f067aa0ba902b7",
  "userId": "user-12345",
  "tenantId": "tenant-67890",
  "customerId": "cust-98765",
  "orderId": "order-11111",
  "amount": 99.99,
  "itemCount": 3,
  "requestUri": "/api/orders",
  "requestMethod": "POST",
  "clientIp": "192.168.1.100"
}
```

## Best Practices

1. **Always Use Structured Arguments**: Use keyValue() for all contextual data, not string concatenation
2. **Populate MDC Early**: Add correlation ID, user ID, tenant ID at request entry point
3. **Clean MDC After Use**: Always clear MDC in finally blocks to prevent memory leaks
4. **Use Async Appenders**: Enable async logging in production to minimize latency impact
5. **Mask Sensitive Data**: Automatically redact PII, credentials, and secrets from logs
6. **Include Trace Context**: Always log trace ID and span ID for correlation with distributed traces
7. **Log at Appropriate Levels**: ERROR for failures, WARN for degraded state, INFO for business events, DEBUG for details
8. **Avoid Logging in Loops**: Aggregate information and log once instead of logging per iteration
9. **Use Conditional Logging**: Check isDebugEnabled() before expensive log message construction
10. **Centralize Log Collection**: Ship logs to ELK, Splunk, or Datadog for analysis and alerting

## Related Documentation

- [Logstash Logback Encoder](https://github.com/logfellow/logstash-logback-encoder)
- [Logback Documentation](http://logback.qos.ch/documentation.html)
- [SLF4J Manual](http://www.slf4j.org/manual.html)
- [Spring Boot Logging](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.logging)
