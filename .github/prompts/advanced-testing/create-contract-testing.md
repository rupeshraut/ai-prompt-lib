# Create Contract Testing

Implement consumer-driven contract testing to ensure API compatibility between microservices. Use Pact framework to define, verify, and manage contracts that guarantee provider APIs meet consumer expectations without requiring end-to-end integration testing.

## Requirements

Your contract testing implementation should include:

- **Consumer Contract Definition**: Define expected API interactions from consumer perspective
- **Provider Contract Verification**: Verify provider implements consumer contracts
- **Pact Broker Integration**: Centralized contract storage and versioning
- **CI/CD Integration**: Automated contract verification in build pipelines
- **Breaking Change Detection**: Identify API changes that break consumer contracts
- **Version Compatibility**: Test multiple provider versions against consumer contracts
- **State Management**: Provider state setup for stateful contract tests
- **Webhook Verification**: Async contract testing for webhooks and callbacks
- **GraphQL Support**: Contract testing for GraphQL APIs
- **Can-I-Deploy**: Verify deployment compatibility before release

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Spring Boot, Pact JVM, JUnit 5
- **Testing**: Pact Framework, Spring Boot Test
- **Infrastructure**: Pact Broker, Docker

## Template: Consumer Contract Test

```java
import au.com.dius.pact.consumer.dsl.PactDslWithProvider;
import au.com.dius.pact.consumer.junit5.PactConsumerTestExt;
import au.com.dius.pact.consumer.junit5.PactTestFor;
import au.com.dius.pact.core.model.RequestResponsePact;
import au.com.dius.pact.core.model.annotations.Pact;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import lombok.extern.slf4j.Slf4j;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static au.com.dius.pact.consumer.dsl.LambdaDsl.newJsonBody;

/**
 * Consumer contract test for Order Service calling User Service.
 * Defines expectations for User Service API from Order Service perspective.
 */
@Slf4j
@SpringBootTest
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "user-service", port = "8081")
public class UserServiceContractTest {

    @Autowired
    private UserServiceClient userServiceClient;

    /**
     * Contract: Get user by ID returns user details.
     */
    @Pact(consumer = "order-service", provider = "user-service")
    public RequestResponsePact getUserById(PactDslWithProvider builder) {
        return builder
            .given("user 123 exists")
            .uponReceiving("a request to get user by ID")
                .path("/api/users/123")
                .method("GET")
                .headers(Map.of(
                    "Accept", "application/json",
                    "Authorization", "Bearer token123"
                ))
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(newJsonBody(body -> {
                    body.stringType("id", "123");
                    body.stringType("email", "user@example.com");
                    body.stringType("name", "John Doe");
                    body.stringMatcher("status", "ACTIVE|INACTIVE", "ACTIVE");
                    body.datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss");
                }).build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getUserById")
    void testGetUserById() {
        // Act: Call the actual client
        UserDTO user = userServiceClient.getUserById("123", "Bearer token123");

        // Assert: Verify response matches contract
        assertThat(user).isNotNull();
        assertThat(user.id()).isEqualTo("123");
        assertThat(user.email()).isEqualTo("user@example.com");
        assertThat(user.name()).isEqualTo("John Doe");
        assertThat(user.status()).isIn("ACTIVE", "INACTIVE");
        assertThat(user.createdAt()).isNotNull();

        log.info("Contract test passed: getUserById");
    }

    /**
     * Contract: Get user by ID returns 404 when user not found.
     */
    @Pact(consumer = "order-service", provider = "user-service")
    public RequestResponsePact getUserByIdNotFound(PactDslWithProvider builder) {
        return builder
            .given("user 999 does not exist")
            .uponReceiving("a request to get non-existent user")
                .path("/api/users/999")
                .method("GET")
                .headers(Map.of(
                    "Accept", "application/json",
                    "Authorization", "Bearer token123"
                ))
            .willRespondWith()
                .status(404)
                .headers(Map.of("Content-Type", "application/json"))
                .body(newJsonBody(body -> {
                    body.stringType("error", "USER_NOT_FOUND");
                    body.stringType("message", "User not found");
                }).build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getUserByIdNotFound")
    void testGetUserByIdNotFound() {
        // Act & Assert
        assertThatThrownBy(() ->
            userServiceClient.getUserById("999", "Bearer token123")
        )
        .isInstanceOf(UserNotFoundException.class)
        .hasMessageContaining("User not found");

        log.info("Contract test passed: getUserByIdNotFound");
    }

    /**
     * Contract: Create user returns created user with generated ID.
     */
    @Pact(consumer = "order-service", provider = "user-service")
    public RequestResponsePact createUser(PactDslWithProvider builder) {
        return builder
            .given("valid user creation request")
            .uponReceiving("a request to create user")
                .path("/api/users")
                .method("POST")
                .headers(Map.of(
                    "Content-Type", "application/json",
                    "Accept", "application/json",
                    "Authorization", "Bearer token123"
                ))
                .body(newJsonBody(body -> {
                    body.stringType("email", "newuser@example.com");
                    body.stringType("name", "Jane Smith");
                }).build())
            .willRespondWith()
                .status(201)
                .headers(Map.of(
                    "Content-Type", "application/json",
                    "Location", "http://localhost:8081/api/users/456"
                ))
                .body(newJsonBody(body -> {
                    body.stringType("id", "456");
                    body.stringType("email", "newuser@example.com");
                    body.stringType("name", "Jane Smith");
                    body.stringType("status", "ACTIVE");
                    body.datetime("createdAt", "yyyy-MM-dd'T'HH:mm:ss");
                }).build())
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "createUser")
    void testCreateUser() {
        // Arrange
        CreateUserRequest request = new CreateUserRequest(
            "newuser@example.com",
            "Jane Smith"
        );

        // Act
        UserDTO user = userServiceClient.createUser(request, "Bearer token123");

        // Assert
        assertThat(user).isNotNull();
        assertThat(user.id()).isEqualTo("456");
        assertThat(user.email()).isEqualTo("newuser@example.com");
        assertThat(user.name()).isEqualTo("Jane Smith");
        assertThat(user.status()).isEqualTo("ACTIVE");

        log.info("Contract test passed: createUser");
    }
}

/**
 * User service client implementation.
 */
@Component
@RequiredArgsConstructor
@Slf4j
public class UserServiceClient {
    private final WebClient webClient;

    @Value("${user-service.base-url}")
    private String baseUrl;

    public UserDTO getUserById(String userId, String authToken) {
        return webClient.get()
            .uri(baseUrl + "/api/users/{id}", userId)
            .header("Authorization", authToken)
            .retrieve()
            .onStatus(
                status -> status == HttpStatus.NOT_FOUND,
                response -> Mono.error(new UserNotFoundException("User not found"))
            )
            .bodyToMono(UserDTO.class)
            .block();
    }

    public UserDTO createUser(CreateUserRequest request, String authToken) {
        return webClient.post()
            .uri(baseUrl + "/api/users")
            .header("Authorization", authToken)
            .bodyValue(request)
            .retrieve()
            .bodyToMono(UserDTO.class)
            .block();
    }
}
```

## Template: Provider Contract Verification

```java
import au.com.dius.pact.provider.junit5.HttpTestTarget;
import au.com.dius.pact.provider.junit5.PactVerificationContext;
import au.com.dius.pact.provider.junit5.PactVerificationInvocationContextProvider;
import au.com.dius.pact.provider.junitsupport.Provider;
import au.com.dius.pact.provider.junitsupport.State;
import au.com.dius.pact.provider.junitsupport.loader.PactBroker;
import au.com.dius.pact.provider.junitsupport.loader.PactBrokerAuth;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.TestTemplate;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.server.LocalServerPort;
import org.springframework.beans.factory.annotation.Autowired;
import lombok.extern.slf4j.Slf4j;

/**
 * Provider contract verification test for User Service.
 * Verifies User Service implements contracts expected by consumers.
 */
@Slf4j
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Provider("user-service")
@PactBroker(
    host = "${PACT_BROKER_HOST:localhost}",
    port = "${PACT_BROKER_PORT:9292}",
    authentication = @PactBrokerAuth(
        token = "${PACT_BROKER_TOKEN}"
    )
)
public class UserServiceProviderContractTest {

    @LocalServerPort
    private int port;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private UserService userService;

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
     * Provider state: user 123 exists.
     */
    @State("user 123 exists")
    void userExists() {
        // Setup: Create user in database
        User user = new User();
        user.setId("123");
        user.setEmail("user@example.com");
        user.setName("John Doe");
        user.setStatus(UserStatus.ACTIVE);
        user.setCreatedAt(LocalDateTime.now());

        userRepository.save(user);
        log.info("Provider state setup: user 123 exists");
    }

    /**
     * Provider state: user 999 does not exist.
     */
    @State("user 999 does not exist")
    void userDoesNotExist() {
        // Setup: Ensure user doesn't exist
        userRepository.deleteById("999");
        log.info("Provider state setup: user 999 does not exist");
    }

    /**
     * Provider state: valid user creation request.
     */
    @State("valid user creation request")
    void validUserCreationRequest() {
        // Setup: Clean state for user creation
        log.info("Provider state setup: ready for user creation");
    }

    /**
     * Clean up after each verification.
     */
    @AfterEach
    void cleanUp() {
        // Clean up test data
        userRepository.deleteAll();
        log.info("Provider state cleanup completed");
    }
}
```

## Pact Broker Configuration

```yaml
# docker-compose.yml for Pact Broker
version: '3'
services:
  pact-broker:
    image: pactfoundation/pact-broker:latest
    ports:
      - "9292:9292"
    environment:
      PACT_BROKER_DATABASE_URL: "postgres://pactbroker:pactbroker@postgres:5432/pactbroker"
      PACT_BROKER_BASIC_AUTH_USERNAME: pact
      PACT_BROKER_BASIC_AUTH_PASSWORD: pact
      PACT_BROKER_PUBLIC_HEARTBEAT: "true"
    depends_on:
      - postgres

  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: pactbroker
      POSTGRES_PASSWORD: pactbroker
      POSTGRES_DB: pactbroker
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

## CI/CD Pipeline Integration

```yaml
# .github/workflows/contract-testing.yml
name: Contract Testing

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  consumer-contract-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Run Consumer Contract Tests
        run: ./gradlew clean pactTest

      - name: Publish Pacts to Broker
        if: github.ref == 'refs/heads/main'
        run: ./gradlew pactPublish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          GIT_COMMIT: ${{ github.sha }}
          GIT_BRANCH: ${{ github.ref_name }}

  provider-contract-verification:
    runs-on: ubuntu-latest
    needs: consumer-contract-test
    steps:
      - uses: actions/checkout@v3

      - name: Set up JDK 17
        uses: actions/setup-java@v3
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Start Test Database
        run: docker-compose up -d postgres

      - name: Verify Provider Contracts
        run: ./gradlew clean pactVerify
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}

      - name: Publish Verification Results
        if: always()
        run: ./gradlew pactPublish
        env:
          PACT_BROKER_URL: ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          GIT_COMMIT: ${{ github.sha }}

  can-i-deploy:
    runs-on: ubuntu-latest
    needs: [consumer-contract-test, provider-contract-verification]
    steps:
      - name: Check Deployment Compatibility
        run: |
          docker run --rm \
            -e PACT_BROKER_BASE_URL=${{ secrets.PACT_BROKER_URL }} \
            -e PACT_BROKER_TOKEN=${{ secrets.PACT_BROKER_TOKEN }} \
            pactfoundation/pact-cli:latest \
            broker can-i-deploy \
            --pacticipant order-service \
            --version ${{ github.sha }} \
            --to-environment production
```

## Gradle Build Configuration

```groovy
// build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.1.0'
    id 'io.spring.dependency-management' version '1.1.0'
    id 'au.com.dius.pact' version '4.5.0'
}

dependencies {
    // Pact dependencies
    testImplementation 'au.com.dius.pact.consumer:junit5:4.5.0'
    testImplementation 'au.com.dius.pact.provider:junit5:4.5.0'
    testImplementation 'au.com.dius.pact.provider:junit5spring:4.5.0'

    // Testing
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.junit.jupiter:junit-jupiter:5.9.3'
}

pact {
    broker {
        pactBrokerUrl = System.getenv('PACT_BROKER_URL') ?: 'http://localhost:9292'
        pactBrokerToken = System.getenv('PACT_BROKER_TOKEN')
    }

    publish {
        pactDirectory = 'build/pacts'
        tags = [System.getenv('GIT_BRANCH') ?: 'local']
        version = System.getenv('GIT_COMMIT') ?: 'local'
    }
}
```

## Configuration

```yaml
# application-test.yml
pact:
  verifier:
    publishResults: true

user-service:
  base-url: http://localhost:8081

spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
```

## GraphQL Contract Testing

```java
/**
 * Consumer contract test for GraphQL API.
 */
@Slf4j
@ExtendWith(PactConsumerTestExt.class)
@PactTestFor(providerName = "product-service", port = "8082")
public class ProductServiceGraphQLContractTest {

    @Pact(consumer = "storefront-service", provider = "product-service")
    public RequestResponsePact getProductById(PactDslWithProvider builder) {
        String query = """
            query GetProduct($id: ID!) {
              product(id: $id) {
                id
                name
                price
                inStock
              }
            }
            """;

        String expectedResponse = """
            {
              "data": {
                "product": {
                  "id": "123",
                  "name": "Laptop",
                  "price": 999.99,
                  "inStock": true
                }
              }
            }
            """;

        return builder
            .given("product 123 exists")
            .uponReceiving("GraphQL query for product by ID")
                .path("/graphql")
                .method("POST")
                .headers(Map.of("Content-Type", "application/json"))
                .body(newJsonBody(body -> {
                    body.stringValue("query", query);
                    body.object("variables", variables -> {
                        variables.stringValue("id", "123");
                    });
                }).build())
            .willRespondWith()
                .status(200)
                .headers(Map.of("Content-Type", "application/json"))
                .body(expectedResponse)
            .toPact();
    }

    @Test
    @PactTestFor(pactMethod = "getProductById")
    void testGetProductById() {
        // Test GraphQL client against contract
        ProductDTO product = productClient.getProductById("123");

        assertThat(product).isNotNull();
        assertThat(product.id()).isEqualTo("123");
        assertThat(product.name()).isEqualTo("Laptop");
        assertThat(product.price()).isEqualTo(999.99);
        assertThat(product.inStock()).isTrue();
    }
}
```

## Best Practices

1. **Consumer-Driven**: Let consumers define contracts based on their actual needs, not provider capabilities
2. **Version Tagging**: Tag pacts with Git branch/commit for version tracking
3. **Use Pact Broker**: Centralize contract storage and enable can-i-deploy checks
4. **Provider States**: Use @State annotations to setup test data for specific contract scenarios
5. **Publish on Main**: Only publish pacts from main/master branch to avoid contract pollution
6. **Verify Before Deploy**: Run can-i-deploy before production deployments to ensure compatibility
7. **Test Error Cases**: Include contracts for 4xx and 5xx responses, not just happy paths
8. **Use Matchers**: Use type matchers (stringType, numberType) instead of exact values for flexible contracts
9. **Webhook Verification**: Test async webhooks by recording expected callbacks
10. **Keep Contracts Minimal**: Only include fields consumers actually use to avoid brittle tests

## Related Documentation

- [Pact Documentation](https://docs.pact.io/)
- [Pact JVM](https://github.com/pact-foundation/pact-jvm)
- [Pact Broker](https://docs.pact.io/pact_broker)
- [Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html)
