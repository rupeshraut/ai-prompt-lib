# Create Chaos Engineering

Implement chaos engineering and fault injection to test system resilience and failure handling. Introduce controlled failures in production or staging environments to validate that services degrade gracefully, recover automatically, and maintain availability during infrastructure issues.

## Requirements

Your chaos engineering implementation should include:

- **Latency Injection**: Introduce artificial delays in service calls
- **Error Injection**: Trigger failures and exceptions at specific points
- **Resource Exhaustion**: Simulate CPU, memory, and disk pressure
- **Network Chaos**: Packet loss, bandwidth throttling, DNS failures
- **Service Failure**: Kill containers, terminate processes
- **Database Chaos**: Connection pool exhaustion, query timeouts
- **Message Queue Disruption**: Drop messages, delay processing
- **Time Travel**: Simulate clock skew and time drift
- **Chaos Schedules**: Automated recurring chaos experiments
- **Observability Integration**: Monitor and measure chaos impact

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Chaos Monkey for Spring Boot, Resilience4j
- **Tools**: Chaos Toolkit, Litmus Chaos, Gremlin
- **Testing**: JUnit 5, Spring Boot Test

## Template: Chaos Monkey for Spring Boot

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import de.codecentric.spring.boot.chaos.monkey.configuration.AssaultProperties;
import de.codecentric.spring.boot.chaos.monkey.configuration.ChaosMonkeyProperties;
import lombok.extern.slf4j.Slf4j;

/**
 * Spring Boot application with Chaos Monkey integration.
 * Enables chaos engineering experiments in controlled environments.
 */
@Slf4j
@SpringBootApplication
public class ChaosEngineeringApplication {

    public static void main(String[] args) {
        SpringApplication.run(ChaosEngineeringApplication.class, args);
    }
}

/**
 * Chaos Monkey configuration for fine-grained control.
 */
@Slf4j
@Configuration
public class ChaosMonkeyConfig {

    /**
     * Configure custom assault properties.
     */
    @Bean
    public AssaultProperties assaultProperties() {
        AssaultProperties properties = new AssaultProperties();

        // Latency assault: 50-500ms delay
        properties.setLatencyActive(true);
        properties.setLatencyRangeStart(50);
        properties.setLatencyRangeEnd(500);

        // Exception assault: throw random exceptions
        properties.setExceptionsActive(true);
        properties.getException().setType("java.lang.RuntimeException");

        // App killer assault: terminate application
        properties.setKillApplicationActive(false);  // Disabled by default

        // Memory assault: create memory pressure
        properties.setMemoryActive(false);  // Controlled via API

        // Level of chaos (1-10, 10 = max chaos)
        properties.setLevel(3);

        return properties;
    }

    /**
     * Configure watched components for chaos injection.
     */
    @Bean
    public ChaosMonkeyProperties chaosMonkeyProperties() {
        ChaosMonkeyProperties properties = new ChaosMonkeyProperties();
        properties.setEnabled(true);

        // Watch specific component types
        properties.getWatcher().setController(true);
        properties.getWatcher().setRestController(true);
        properties.getWatcher().setService(true);
        properties.getWatcher().setRepository(true);
        properties.getWatcher().setComponent(false);

        return properties;
    }
}
```

## Custom Fault Injection Aspects

```java
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import lombok.extern.slf4j.Slf4j;
import java.util.Random;
import java.util.concurrent.ThreadLocalRandom;

/**
 * Custom aspect for fault injection in service methods.
 */
@Slf4j
@Aspect
@Component
public class FaultInjectionAspect {

    private final Random random = ThreadLocalRandom.current();
    private final ChaosConfigurationProperties chaosConfig;

    public FaultInjectionAspect(ChaosConfigurationProperties chaosConfig) {
        this.chaosConfig = chaosConfig;
    }

    /**
     * Inject faults into @Service annotated beans.
     */
    @Around("@within(org.springframework.stereotype.Service)")
    public Object injectServiceFaults(ProceedingJoinPoint joinPoint) throws Throwable {
        if (!chaosConfig.isEnabled()) {
            return joinPoint.proceed();
        }

        String methodName = joinPoint.getSignature().toShortString();

        // Randomly decide whether to inject fault
        if (random.nextDouble() > chaosConfig.getFaultProbability()) {
            return joinPoint.proceed();
        }

        // Choose fault type
        FaultType faultType = selectRandomFaultType();

        log.warn("Injecting {} fault into {}", faultType, methodName);

        switch (faultType) {
            case LATENCY:
                return injectLatency(joinPoint);
            case EXCEPTION:
                return injectException(joinPoint, methodName);
            case TIMEOUT:
                return injectTimeout(joinPoint);
            default:
                return joinPoint.proceed();
        }
    }

    /**
     * Inject artificial latency.
     */
    private Object injectLatency(ProceedingJoinPoint joinPoint) throws Throwable {
        int delayMs = random.nextInt(
            chaosConfig.getLatencyMinMs(),
            chaosConfig.getLatencyMaxMs()
        );

        log.debug("Injecting {}ms latency", delayMs);
        Thread.sleep(delayMs);

        return joinPoint.proceed();
    }

    /**
     * Inject random exception.
     */
    private Object injectException(ProceedingJoinPoint joinPoint, String methodName)
            throws Throwable {

        String[] exceptions = chaosConfig.getExceptionTypes();
        String exceptionType = exceptions[random.nextInt(exceptions.length)];

        log.error("Injecting {} exception", exceptionType);

        switch (exceptionType) {
            case "RuntimeException":
                throw new RuntimeException("Chaos: Injected failure in " + methodName);
            case "TimeoutException":
                throw new TimeoutException("Chaos: Simulated timeout");
            case "ServiceUnavailableException":
                throw new ServiceUnavailableException("Chaos: Service unavailable");
            default:
                throw new ChaosException("Chaos: Unknown fault injection");
        }
    }

    /**
     * Inject timeout by sleeping longer than expected.
     */
    private Object injectTimeout(ProceedingJoinPoint joinPoint) throws Throwable {
        int timeoutMs = chaosConfig.getTimeoutMs();
        log.warn("Injecting timeout: {}ms", timeoutMs);

        Thread.sleep(timeoutMs);
        throw new TimeoutException("Chaos: Operation timed out");
    }

    private FaultType selectRandomFaultType() {
        FaultType[] types = FaultType.values();
        return types[random.nextInt(types.length)];
    }

    enum FaultType {
        LATENCY,
        EXCEPTION,
        TIMEOUT
    }
}

/**
 * Configuration properties for chaos engineering.
 */
@ConfigurationProperties(prefix = "chaos")
@Data
public class ChaosConfigurationProperties {
    private boolean enabled = false;
    private double faultProbability = 0.1;  // 10% of requests
    private int latencyMinMs = 100;
    private int latencyMaxMs = 1000;
    private int timeoutMs = 5000;
    private String[] exceptionTypes = {
        "RuntimeException",
        "TimeoutException",
        "ServiceUnavailableException"
    };
}
```

## HTTP Client Fault Injection

```java
import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import lombok.extern.slf4j.Slf4j;
import java.io.IOException;
import java.util.Random;

/**
 * Interceptor to inject faults into HTTP client calls.
 */
@Slf4j
public class HttpChaosInterceptor implements ClientHttpRequestInterceptor {

    private final Random random = new Random();
    private final ChaosConfigurationProperties chaosConfig;

    public HttpChaosInterceptor(ChaosConfigurationProperties chaosConfig) {
        this.chaosConfig = chaosConfig;
    }

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        if (!chaosConfig.isEnabled()) {
            return execution.execute(request, body);
        }

        // Randomly inject fault
        if (random.nextDouble() < chaosConfig.getFaultProbability()) {
            return injectHttpFault(request, body, execution);
        }

        return execution.execute(request, body);
    }

    private ClientHttpResponse injectHttpFault(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        HttpFaultType faultType = selectRandomHttpFault();

        log.warn("Injecting {} fault for HTTP request to {}",
                faultType, request.getURI());

        switch (faultType) {
            case LATENCY:
                injectHttpLatency();
                return execution.execute(request, body);

            case CONNECTION_TIMEOUT:
                throw new java.net.SocketTimeoutException(
                    "Chaos: Simulated connection timeout");

            case READ_TIMEOUT:
                Thread.sleep(10000);  // Force read timeout
                return execution.execute(request, body);

            case CONNECTION_REFUSED:
                throw new java.net.ConnectException(
                    "Chaos: Connection refused");

            case SERVICE_UNAVAILABLE:
                throw new IOException("Chaos: Service unavailable");

            default:
                return execution.execute(request, body);
        }
    }

    private void injectHttpLatency() {
        try {
            int delay = random.nextInt(
                chaosConfig.getLatencyMinMs(),
                chaosConfig.getLatencyMaxMs()
            );
            Thread.sleep(delay);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    private HttpFaultType selectRandomHttpFault() {
        HttpFaultType[] types = HttpFaultType.values();
        return types[random.nextInt(types.length)];
    }

    enum HttpFaultType {
        LATENCY,
        CONNECTION_TIMEOUT,
        READ_TIMEOUT,
        CONNECTION_REFUSED,
        SERVICE_UNAVAILABLE
    }
}

/**
 * RestTemplate configuration with chaos interceptor.
 */
@Configuration
public class RestTemplateConfig {

    @Bean
    public RestTemplate restTemplate(ChaosConfigurationProperties chaosConfig) {
        RestTemplate restTemplate = new RestTemplate();

        // Add chaos interceptor
        List<ClientHttpRequestInterceptor> interceptors = new ArrayList<>();
        interceptors.add(new HttpChaosInterceptor(chaosConfig));
        restTemplate.setInterceptors(interceptors);

        return restTemplate;
    }
}
```

## Database Chaos with Connection Pool Exhaustion

```java
import javax.sql.DataSource;
import org.springframework.jdbc.datasource.DelegatingDataSource;
import lombok.extern.slf4j.Slf4j;
import java.sql.Connection;
import java.sql.SQLException;
import java.util.Random;

/**
 * DataSource wrapper that injects database chaos.
 */
@Slf4j
public class ChaosDataSource extends DelegatingDataSource {

    private final Random random = new Random();
    private final ChaosConfigurationProperties chaosConfig;

    public ChaosDataSource(DataSource targetDataSource,
                           ChaosConfigurationProperties chaosConfig) {
        super(targetDataSource);
        this.chaosConfig = chaosConfig;
    }

    @Override
    public Connection getConnection() throws SQLException {
        if (chaosConfig.isEnabled() &&
            random.nextDouble() < chaosConfig.getFaultProbability()) {

            DbFaultType faultType = selectRandomDbFault();
            log.warn("Injecting database fault: {}", faultType);

            switch (faultType) {
                case SLOW_QUERY:
                    return new SlowConnectionWrapper(super.getConnection(), chaosConfig);

                case CONNECTION_TIMEOUT:
                    throw new SQLException("Chaos: Database connection timeout");

                case CONNECTION_POOL_EXHAUSTED:
                    throw new SQLException("Chaos: Connection pool exhausted");

                case TRANSACTION_DEADLOCK:
                    throw new SQLException("Chaos: Transaction deadlock detected");

                default:
                    return super.getConnection();
            }
        }

        return super.getConnection();
    }

    private DbFaultType selectRandomDbFault() {
        DbFaultType[] types = DbFaultType.values();
        return types[random.nextInt(types.length)];
    }

    enum DbFaultType {
        SLOW_QUERY,
        CONNECTION_TIMEOUT,
        CONNECTION_POOL_EXHAUSTED,
        TRANSACTION_DEADLOCK
    }
}

/**
 * Connection wrapper that adds latency to database operations.
 */
class SlowConnectionWrapper extends ConnectionWrapper {

    private final ChaosConfigurationProperties chaosConfig;

    public SlowConnectionWrapper(Connection connection,
                                 ChaosConfigurationProperties chaosConfig) {
        super(connection);
        this.chaosConfig = chaosConfig;
    }

    @Override
    public Statement createStatement() throws SQLException {
        injectLatency();
        return super.createStatement();
    }

    @Override
    public PreparedStatement prepareStatement(String sql) throws SQLException {
        injectLatency();
        return super.prepareStatement(sql);
    }

    private void injectLatency() {
        try {
            int delay = ThreadLocalRandom.current().nextInt(
                chaosConfig.getLatencyMinMs(),
                chaosConfig.getLatencyMaxMs()
            );
            Thread.sleep(delay);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

## Chaos Experiments with Resilience Validation

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import lombok.extern.slf4j.Slf4j;

import java.util.concurrent.CountDownLatch;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Chaos experiment: Validate circuit breaker behavior under failure.
 */
@Slf4j
@SpringBootTest
public class CircuitBreakerChaosTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired
    private ChaosConfigurationProperties chaosConfig;

    /**
     * Experiment: Verify circuit breaker opens after threshold failures.
     */
    @Test
    void chaosExperiment_CircuitBreakerOpensUnderFailure() throws Exception {
        // Enable chaos
        chaosConfig.setEnabled(true);
        chaosConfig.setFaultProbability(1.0);  // 100% failure rate

        CircuitBreaker circuitBreaker = circuitBreakerRegistry
            .circuitBreaker("orderService");

        // Initial state: CLOSED
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.CLOSED);

        // Generate failures to open circuit breaker
        int failureThreshold = 10;
        for (int i = 0; i < failureThreshold; i++) {
            try {
                orderService.createOrder(new CreateOrderRequest());
            } catch (Exception e) {
                log.debug("Expected failure #{}", i + 1);
            }
        }

        // Verify circuit breaker opened
        assertThat(circuitBreaker.getState())
            .isEqualTo(CircuitBreaker.State.OPEN);

        log.info("Chaos experiment passed: Circuit breaker opened after {} failures",
                failureThreshold);

        // Cleanup
        chaosConfig.setEnabled(false);
    }

    /**
     * Experiment: Validate graceful degradation under load and chaos.
     */
    @Test
    void chaosExperiment_GracefulDegradationUnderLoad() throws Exception {
        // Enable moderate chaos
        chaosConfig.setEnabled(true);
        chaosConfig.setFaultProbability(0.3);  // 30% failure rate

        int totalRequests = 1000;
        int concurrentThreads = 50;

        ExecutorService executor = Executors.newFixedThreadPool(concurrentThreads);
        CountDownLatch latch = new CountDownLatch(totalRequests);

        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger failureCount = new AtomicInteger(0);

        long startTime = System.currentTimeMillis();

        // Generate load with chaos
        for (int i = 0; i < totalRequests; i++) {
            executor.submit(() -> {
                try {
                    orderService.createOrder(new CreateOrderRequest());
                    successCount.incrementAndGet();
                } catch (Exception e) {
                    failureCount.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await(60, TimeUnit.SECONDS);
        executor.shutdown();

        long duration = System.currentTimeMillis() - startTime;

        // Validate system remained operational
        double successRate = (double) successCount.get() / totalRequests;

        log.info("Chaos experiment results:");
        log.info("  Total requests: {}", totalRequests);
        log.info("  Successful: {}", successCount.get());
        log.info("  Failed: {}", failureCount.get());
        log.info("  Success rate: {:.2f}%", successRate * 100);
        log.info("  Duration: {}ms", duration);

        // Assert minimum success rate despite chaos
        assertThat(successRate).isGreaterThan(0.5);  // 50% minimum

        // Cleanup
        chaosConfig.setEnabled(false);
    }
}
```

## Configuration

```yaml
# application.yml
chaos:
  enabled: ${CHAOS_ENABLED:false}
  fault-probability: ${CHAOS_FAULT_PROBABILITY:0.1}
  latency-min-ms: 100
  latency-max-ms: 1000
  timeout-ms: 5000
  exception-types:
    - RuntimeException
    - TimeoutException
    - ServiceUnavailableException

# Chaos Monkey for Spring Boot
chaos-monkey:
  enabled: ${CHAOS_MONKEY_ENABLED:false}
  assaults:
    level: 3
    latency-active: true
    latency-range-start: 50
    latency-range-end: 500
    exceptions-active: true
    exception-type: java.lang.RuntimeException
    kill-application-active: false
    memory-active: false
  watcher:
    controller: true
    rest-controller: true
    service: true
    repository: true
    component: false

# Environment-specific chaos settings
---
spring.config.activate.on-profile: staging

chaos:
  enabled: true
  fault-probability: 0.05  # 5% in staging

chaos-monkey:
  enabled: true
  assaults:
    level: 2

---
spring.config.activate.on-profile: production

chaos:
  enabled: false  # Never enable in production by default

chaos-monkey:
  enabled: false
```

## Chaos Monkey REST API Control

```bash
# Enable Chaos Monkey at runtime
curl -X POST http://localhost:8080/actuator/chaosmonkey/enable

# Disable Chaos Monkey
curl -X POST http://localhost:8080/actuator/chaosmonkey/disable

# Get current status
curl http://localhost:8080/actuator/chaosmonkey/status

# Configure assaults dynamically
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults \
  -H 'Content-Type: application/json' \
  -d '{
    "level": 5,
    "latencyActive": true,
    "latencyRangeStart": 100,
    "latencyRangeEnd": 1000,
    "exceptionsActive": true
  }'

# Trigger specific assault
curl -X POST http://localhost:8080/actuator/chaosmonkey/assaults/runtime/attack
```

## Best Practices

1. **Start Small**: Begin with low fault probability (1-5%) and gradually increase
2. **Test in Non-Production First**: Run chaos experiments in staging before production
3. **Define Steady State**: Establish baseline metrics before introducing chaos
4. **Hypothesis-Driven**: Form hypothesis about system behavior before each experiment
5. **Monitor Everything**: Track metrics, logs, and traces during chaos experiments
6. **Automated Rollback**: Disable chaos automatically if critical metrics degrade
7. **Schedule Chaos**: Run recurring experiments during business hours when team is available
8. **Gradual Blast Radius**: Start with single service, expand to entire system
9. **Document Experiments**: Record hypotheses, results, and learnings from each experiment
10. **Fix What Breaks**: Use chaos findings to improve resilience patterns

## Related Documentation

- [Chaos Monkey for Spring Boot](https://codecentric.github.io/chaos-monkey-spring-boot/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Netflix Chaos Engineering](https://netflixtechblog.com/tagged/chaos-engineering)
- [Chaos Toolkit](https://chaostoolkit.org/)
