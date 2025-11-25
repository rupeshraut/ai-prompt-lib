# Advanced Testing Library

Comprehensive prompts for implementing advanced testing strategies including consumer-driven contract testing and chaos engineering. Build confidence in system reliability through automated contract verification and controlled failure injection.

## Available Prompts

### 1. Create Contract Testing
Implement consumer-driven contract testing using Pact framework to ensure API compatibility between microservices.

| Feature | Description |
|---------|-------------|
| Consumer Contracts | Define API expectations from consumer perspective |
| Provider Verification | Verify provider implements consumer contracts |
| Pact Broker | Centralized contract storage and versioning |
| CI/CD Integration | Automated contract verification in pipelines |
| Breaking Change Detection | Identify API changes that break consumers |
| Version Compatibility | Test multiple provider versions |
| State Management | Provider state setup for contracts |
| Webhook Verification | Async contract testing for webhooks |
| GraphQL Support | Contract testing for GraphQL APIs |
| Can-I-Deploy | Pre-deployment compatibility checks |

### 2. Create Chaos Engineering
Implement chaos engineering experiments to test system resilience and failure handling through controlled fault injection.

| Feature | Description |
|---------|-------------|
| Latency Injection | Introduce artificial delays |
| Error Injection | Trigger failures and exceptions |
| Resource Exhaustion | Simulate CPU, memory pressure |
| Network Chaos | Packet loss, bandwidth throttling |
| Service Failure | Kill containers, terminate processes |
| Database Chaos | Connection pool exhaustion, timeouts |
| Message Queue Disruption | Drop messages, delay processing |
| Time Travel | Simulate clock skew and drift |
| Chaos Schedules | Automated recurring experiments |
| Observability Integration | Monitor chaos impact |

## Quick Start

### 1. Setup Contract Testing

```
@workspace Use .github/prompts/advanced-testing/create-contract-testing.md

Implement Pact consumer contracts for order-service calling user-service APIs
```

### 2. Implement Chaos Engineering

```
@workspace Use .github/prompts/advanced-testing/create-chaos-engineering.md

Configure Chaos Monkey with 10% latency injection and circuit breaker validation
```

## Architecture Overview

```
Contract Testing Flow:

┌─────────────────────────────────────────────────────────┐
│              Consumer (Order Service)                    │
│  1. Define Contract: "GET /users/{id} returns user"     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Run Consumer Test │
            │  - Mock Provider   │
            │  - Verify Client   │
            │  - Generate Pact   │
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │   Publish Pact to  │
            │   Pact Broker      │
            └────────┬───────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│              Provider (User Service)                     │
│  2. Verify Contract: Provider implements contract       │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │ Provider Verify    │
            │ - Fetch Pacts      │
            │ - Setup State      │
            │ - Run Tests        │
            │ - Publish Results  │
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │   Can-I-Deploy?    │
            │   Check if Order   │
            │   Service can      │
            │   deploy with User │
            │   Service          │
            └────────────────────┘

Chaos Engineering Flow:

┌─────────────────────────────────────────────────────────┐
│                  Steady State                            │
│  - Throughput: 1000 req/s                               │
│  - Latency P95: 100ms                                   │
│  - Error Rate: < 0.1%                                   │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Define Hypothesis │
            │  "System remains   │
            │   operational with │
            │   30% latency      │
            │   injection"       │
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Inject Chaos      │
            │  - Latency: 30%    │
            │  - 100-500ms delay │
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Observe System    │
            │  - Circuit breaker │
            │    opens?          │
            │  - Graceful        │
            │    degradation?    │
            │  - Throughput drop?│
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Analyze Results   │
            │  - Hypothesis      │
            │    confirmed?      │
            │  - Weak points     │
            │    discovered?     │
            │  - Action items    │
            └────────┬───────────┘
                     │
                     ▼
            ┌────────────────────┐
            │  Improve System    │
            │  - Add resilience  │
            │  - Tune timeouts   │
            │  - Add fallbacks   │
            └────────────────────┘

Chaos Injection Points:

    ┌──────────────┐
    │  API Gateway │
    └──────┬───────┘
           │ ← Latency Injection
           ▼
    ┌──────────────┐
    │ Order Service│
    └──────┬───────┘
           │ ← Exception Injection
           ▼
    ┌──────────────┐
    │  HTTP Client │
    └──────┬───────┘
           │ ← Connection Timeout
           ▼
    ┌──────────────┐
    │Payment Svc   │
    └──────┬───────┘
           │ ← Service Failure
           ▼
    ┌──────────────┐
    │   Database   │
    └──────────────┘
      ↑
      └── Slow Query Injection
```

## Integration Patterns

### Consumer Contract Test

```java
@SpringBootTest
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service", port = "8081")
public class UserServiceContractTest {

    @Autowired
    private UserServiceClient userServiceClient;

    /**
     * Define contract: Get user by ID.
     */
    @Pact(consumer = "order-service", provider = "user-service")
    public RequestResponsePact getUserById(PactDslWithProvider builder) {
        return builder
            .given("user 123 exists")
            .uponReceiving("request to get user by ID")
                .path("/api/users/123")
                .method("GET")
            .willRespondWith()
                .status(200)
                .body(newJsonBody(body -> {
                    body.stringType("id", "123");
                    body.stringType("email", "user@example.com");
                    body.stringType("name", "John Doe");
                }).build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getUserById")
    void testGetUserById() {
        UserDTO user = userServiceClient.getUserById("123");

        assertThat(user).isNotNull();
        assertThat(user.id()).isEqualTo("123");
        assertThat(user.email()).isEqualTo("user@example.com");
    }
}
```

### Provider Contract Verification

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("user-service")
@PactBroker(host = "localhost", port = "9292")
public class UserServiceProviderContractTest {

    @LocalServerPort
    private int port;

    @Autowired
    private UserRepository userRepository;

    @BeforeEach
    void setUp(PactVerificationContext context) {
        context.setTarget(new HttpTestTarget("localhost", port));
    }

    @TestTemplate
    @ExtendWith(PactVerificationInvocationContextProvider.class)
    void verifyPact(PactVerificationContext context) {
        context.verifyInteraction();
    }

    /**
     * Setup provider state for contract.
     */
    @State("user 123 exists")
    void userExists() {
        User user = new User();
        user.setId("123");
        user.setEmail("user@example.com");
        user.setName("John Doe");
        userRepository.save(user);
    }
}
```

### Chaos Engineering with Chaos Monkey

```java
@Configuration
public class ChaosConfig {

    @Bean
    public AssaultProperties assaultProperties() {
        AssaultProperties properties = new AssaultProperties();

        // 30% requests get 100-500ms delay
        properties.setLatencyActive(true);
        properties.setLatencyRangeStart(100);
        properties.setLatencyRangeEnd(500);

        // 10% requests throw exception
        properties.setExceptionsActive(true);

        // Chaos level (1-10)
        properties.setLevel(3);

        return properties;
    }
}

/**
 * Chaos experiment: Verify circuit breaker behavior.
 */
@SpringBootTest
public class CircuitBreakerChaosTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private CircuitBreakerRegistry circuitBreakerRegistry;

    @Autowired
    private ChaosConfigurationProperties chaosConfig;

    @Test
    void chaosExperiment_CircuitBreakerOpensUnderFailure() {
        // Enable chaos
        chaosConfig.setEnabled(true);
        chaosConfig.setFaultProbability(1.0);  // 100% failure

        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("orderService");

        // Generate failures
        for (int i = 0; i < 10; i++) {
            try {
                orderService.createOrder(new CreateOrderRequest());
            } catch (Exception e) {
                // Expected
            }
        }

        // Verify circuit breaker opened
        assertThat(cb.getState()).isEqualTo(CircuitBreaker.State.OPEN);
    }
}
```

### HTTP Client with Chaos Injection

```java
@Slf4j
public class HttpChaosInterceptor implements ClientHttpRequestInterceptor {

    private final ChaosConfigurationProperties chaosConfig;

    @Override
    public ClientHttpResponse intercept(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        if (!chaosConfig.isEnabled()) {
            return execution.execute(request, body);
        }

        // Randomly inject fault
        if (Math.random() < chaosConfig.getFaultProbability()) {
            return injectFault(request, body, execution);
        }

        return execution.execute(request, body);
    }

    private ClientHttpResponse injectFault(
            HttpRequest request,
            byte[] body,
            ClientHttpRequestExecution execution) throws IOException {

        ChaosType type = selectRandomChaosType();

        log.warn("Injecting {} chaos for request to {}",
            type, request.getURI());

        switch (type) {
            case LATENCY:
                Thread.sleep(chaosConfig.getLatencyMs());
                return execution.execute(request, body);

            case TIMEOUT:
                throw new SocketTimeoutException("Chaos: Simulated timeout");

            case CONNECTION_REFUSED:
                throw new ConnectException("Chaos: Connection refused");

            default:
                return execution.execute(request, body);
        }
    }
}
```

## Key Concepts

### Consumer-Driven Contracts
Consumers define expected API behavior. Providers verify they meet those expectations. Prevents breaking changes.

### Pact
Framework for consumer-driven contract testing. Generates and verifies contracts between services.

### Pact Broker
Centralized storage for contracts. Enables can-i-deploy checks and contract versioning.

### Provider States
Setup required for provider verification tests. Ensures test data exists for contract scenarios.

### Chaos Engineering
Discipline of experimenting on systems to build confidence in resilience. Controlled failure injection.

### Chaos Monkey
Tool that randomly terminates instances to ensure system handles failures gracefully.

### Fault Injection
Deliberately introducing errors (latency, failures) to test system behavior under stress.

### Steady State Hypothesis
Measurable definition of normal system behavior. Used to validate system during chaos experiments.

## Best Practices

### Contract Testing

1. **Consumer-Driven Approach**
   - Let consumers define contracts based on needs
   - Providers verify against all consumer contracts
   - Use Pact Broker for centralized management

2. **Test Scenarios Thoroughly**
   - Test success cases (200 OK)
   - Test error cases (404, 500)
   - Test edge cases and validation errors

3. **Use Provider States**
   - Setup test data for each contract scenario
   - Clean up after verification
   - Use descriptive state names

4. **Integrate with CI/CD**
   - Run consumer tests on every commit
   - Verify provider contracts before deploy
   - Use can-i-deploy in deployment pipeline

5. **Version Contracts**
   - Tag contracts with Git commit/branch
   - Test against multiple provider versions
   - Use semantic versioning

### Chaos Engineering

6. **Start Small**
   - Begin with 1-5% fault probability
   - Test in non-production first
   - Gradually increase chaos intensity

7. **Define Hypotheses**
   - State expected system behavior
   - Define measurable outcomes
   - Document assumptions

8. **Monitor Everything**
   - Track metrics during experiments
   - Observe circuit breaker states
   - Monitor error rates and latency

9. **Automate Rollback**
   - Disable chaos if critical metrics degrade
   - Set experiment timeouts
   - Have kill switches ready

10. **Document Learnings**
    - Record experiment results
    - Document discovered weaknesses
    - Track improvements made

## Configuration

```yaml
# Contract Testing (application-test.yml)
pact:
  broker:
    base-url: http://localhost:9292
    auth:
      token: ${PACT_BROKER_TOKEN}
  verifier:
    publish-results: true

# Chaos Engineering (application.yml)
chaos:
  enabled: ${CHAOS_ENABLED:false}
  fault-probability: 0.1  # 10%
  latency-min-ms: 100
  latency-max-ms: 1000

chaos-monkey:
  enabled: ${CHAOS_MONKEY_ENABLED:false}
  assaults:
    level: 3
    latency-active: true
    latency-range-start: 100
    latency-range-end: 500
    exceptions-active: true
  watcher:
    controller: true
    service: true
    repository: true
```

## CI/CD Pipeline

```yaml
# .github/workflows/contract-testing.yml
name: Contract Testing

on: [push, pull_request]

jobs:
  consumer-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run Consumer Tests
        run: ./gradlew pactTest
      - name: Publish Pacts
        run: ./gradlew pactPublish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}

  provider-verification:
    needs: consumer-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Verify Provider Contracts
        run: ./gradlew pactVerify
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}

  can-i-deploy:
    needs: provider-verification
    runs-on: ubuntu-latest
    steps:
      - name: Check Deployment Safety
        run: |
          docker run pactfoundation/pact-cli \
            broker can-i-deploy \
            --pacticipant order-service \
            --version ${{ github.sha }} \
            --to-environment production
```

## Troubleshooting

### Contract Tests Failing
1. Verify provider state setup is correct
2. Check contract expectations match actual API
3. Review provider logs for errors
4. Ensure test data exists

### Pact Broker Connection Issues
1. Verify broker URL and credentials
2. Check network connectivity
3. Review firewall rules
4. Verify broker is running

### Chaos Experiments Too Aggressive
1. Reduce fault probability
2. Decrease chaos level
3. Shorten experiment duration
4. Test in isolated environment first

### System Not Recovering from Chaos
1. Check circuit breaker configuration
2. Verify retry policies
3. Review timeout settings
4. Check fallback implementations

## Related Documentation

- [Pact Documentation](https://docs.pact.io/)
- [Pact JVM](https://github.com/pact-foundation/pact-jvm)
- [Chaos Monkey for Spring Boot](https://codecentric.github.io/chaos-monkey-spring-boot/)
- [Principles of Chaos Engineering](https://principlesofchaos.org/)
- [Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)

## Library Navigation

This library focuses on advanced testing strategies. For related capabilities:
- See **resilience-operations** library for circuit breakers and retry patterns
- See **observability** library for monitoring during chaos experiments
- See **distributed-tracing** library for tracing contract test calls
