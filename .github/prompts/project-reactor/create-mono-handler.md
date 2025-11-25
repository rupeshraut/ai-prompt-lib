# Create Mono Handler

Create a new Mono handler for single-value asynchronous operations using Project Reactor. Mono represents a reactive type that emits 0 or 1 element.

## Requirements

Your Mono handler should include:

- **Class Declaration**: Public class with descriptive name (e.g., `UserMono Handler`, `OrderMono Handler`)
- **Mono Operations**: Use appropriate Mono methods like `map()`, `flatMap()`, `filter()`, `switchIfEmpty()`
- **Error Handling**: Include proper error handling with `onError()` or `doOnError()`
- **Timeout Handling**: Add timeout constraints with `timeout()` method
- **Null Safety**: Use `Mono.justOrEmpty()` for nullable values
- **Subscription Handling**: Include examples of `subscribe()` with success, error, and completion handlers
- **Logging**: Add logging at key points using `doOnNext()`, `doOnError()`, `doOnComplete()`
- **Java 17 Features**: Use records for simple data holders, sealed classes where applicable
- **Documentation**: Include JavaDoc comments explaining the reactive pipeline

## Code Style

- **Language**: Java 17 LTS
- **Frameworks**: Project Reactor, Spring (if dependency injection needed)
- **Build Tool**: Maven with spring-boot-starter-webflux dependency
- **Naming Conventions**:
  - Method names should describe the operation (e.g., `getUserMono()`, `createOrderMono()`)
  - Use reactive naming patterns (suffix with `Mono` or `Flux` for return types)
- **Annotations**: Use `@Component` or `@Service` if Spring-managed
- **Best Practices**:
  - Avoid blocking operations within Mono chains
  - Use `Mono.defer()` for lazy evaluation
  - Leverage `Mono.fromCallable()` for blocking I/O wrapped in async
  - Always provide error handling context

## Template Example

```java
import org.springframework.stereotype.Component;
import reactor.core.publisher.Mono;
import reactor.util.context.Context;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import java.time.Duration;
import java.util.Optional;

/**
 * Handles single-value asynchronous operations using Mono.
 * Example: User lookup, single order retrieval, etc.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class UserMonoHandler {

    // Inject dependencies as needed (repositories, services, etc.)
    // private final UserRepository userRepository;
    // private final NotificationService notificationService;

    /**
     * Retrieves a user by ID asynchronously.
     * Returns empty Mono if user not found.
     */
    public Mono<UserResponse> getUserById(Long userId) {
        log.debug("Fetching user with ID: {}", userId);

        return Mono.defer(() -> {
            // Validate input
            if (userId == null || userId <= 0) {
                return Mono.error(new IllegalArgumentException("Invalid user ID"));
            }

            // Simulate repository call (replace with actual repository)
            return Mono.justOrEmpty(findUserInDatabase(userId))
                    .doOnNext(user -> log.info("User found: {}", user.id()))
                    .timeout(Duration.ofSeconds(5))
                    .doOnError(throwable -> log.error("Error fetching user", throwable));
        })
        .onErrorResume(IllegalArgumentException.class, e -> {
            log.warn("Invalid request: {}", e.getMessage());
            return Mono.empty();
        })
        .onErrorResume(TimeoutException.class, e -> {
            log.error("User lookup timed out");
            return Mono.error(new ServiceUnavailableException("User service timeout"));
        })
        .map(user -> new UserResponse(user.id(), user.username(), user.email()))
        .doOnSuccess(response -> log.debug("User response prepared: {}", response.id()))
        .doOnCancel(() -> log.info("User fetch cancelled"));
    }

    /**
     * Creates a new user asynchronously with validation and notification.
     */
    public Mono<UserResponse> createUser(CreateUserRequest request) {
        return Mono.fromCallable(() -> {
            // Validation
            if (request.username() == null || request.username().isBlank()) {
                throw new ValidationException("Username is required");
            }
            if (request.email() == null || !isValidEmail(request.email())) {
                throw new ValidationException("Invalid email format");
            }
            return request;
        })
        .flatMap(validRequest -> {
            log.info("Creating user: {}", validRequest.username());

            // Simulate repository save
            User newUser = new User(null, validRequest.username(), validRequest.email());
            return Mono.justOrEmpty(saveUserInDatabase(newUser));
        })
        .flatMap(savedUser -> {
            // Send notification asynchronously
            return notifyUserCreation(savedUser)
                    .then(Mono.just(savedUser));
        })
        .map(user -> new UserResponse(user.id(), user.username(), user.email()))
        .timeout(Duration.ofSeconds(10))
        .doOnSuccess(response -> log.info("User created successfully: {}", response.id()))
        .doOnError(e -> log.error("Failed to create user", e))
        .onErrorResume(ValidationException.class, e -> {
            log.warn("Validation error: {}", e.getMessage());
            return Mono.error(new BadRequestException(e.getMessage()));
        });
    }

    /**
     * Performs a conditional operation based on value existence.
     * Example: Get user or create default guest user.
     */
    public Mono<UserResponse> getUserOrDefault(Long userId) {
        return getUserById(userId)
                .switchIfEmpty(Mono.defer(() -> {
                    log.info("User not found, returning guest user");
                    return Mono.just(createGuestUser());
                }))
                .doFinally(signalType -> log.debug("Operation completed with signal: {}", signalType));
    }

    /**
     * Demonstrates context propagation in Mono chains.
     */
    public Mono<UserResponse> getUserWithContext(Long userId, String requestId) {
        return Mono.deferContextual(context -> {
            String correlationId = context.get("correlationId");
            log.debug("Processing request {} with correlation {}", requestId, correlationId);
            return getUserById(userId);
        })
        .contextWrite(Context.of("correlationId", requestId));
    }

    // Helper methods
    private Optional<User> findUserInDatabase(Long userId) {
        // Simulate database lookup
        return Optional.empty();
    }

    private Optional<User> saveUserInDatabase(User user) {
        // Simulate database save
        return Optional.of(user);
    }

    private Mono<Void> notifyUserCreation(User user) {
        return Mono.fromRunnable(() -> {
            log.info("Sending notification for user: {}", user.username());
            // Simulate async notification
        });
    }

    private boolean isValidEmail(String email) {
        return email.matches("^[A-Za-z0-9+_.-]+@(.+)$");
    }

    private UserResponse createGuestUser() {
        return new UserResponse(null, "guest", "guest@example.com");
    }

    // DTOs
    public record UserResponse(Long id, String username, String email) {}
    public record CreateUserRequest(String username, String email) {}

    // Domain model
    public record User(Long id, String username, String email) {}

    // Custom exceptions
    public static class ValidationException extends RuntimeException {
        public ValidationException(String message) {
            super(message);
        }
    }

    public static class ServiceUnavailableException extends RuntimeException {
        public ServiceUnavailableException(String message) {
            super(message);
        }
    }

    public static class BadRequestException extends RuntimeException {
        public BadRequestException(String message) {
            super(message);
        }
    }
}
```

## Key Patterns

### 1. Defer vs Lazy Evaluation
```java
// Use Mono.defer() to delay computation until subscription
Mono<User> lazyMono = Mono.defer(() -> Mono.just(findUser()));
```

### 2. Null-Safe Operations
```java
// Use justOrEmpty() for Optional values
public Mono<User> getUserSafe(Optional<User> user) {
    return Mono.justOrEmpty(user);
}
```

### 3. Chaining Operations
```java
monoValue
    .map(value -> transform(value))              // Single value transformation
    .flatMap(value -> monoOperation(value))      // Chain multiple Monos
    .filter(value -> validate(value))            // Conditional filtering
    .switchIfEmpty(Mono.just(defaultValue()))   // Fallback
    .timeout(Duration.ofSeconds(5))              // Set timeout
    .doOnNext(value -> log.info("Value: {}", value))
    .doOnError(error -> log.error("Error", error))
    .doOnComplete(() -> log.info("Completed"));
```

### 4. Error Recovery
```java
Mono<User> resilientMono = getUserMono()
    .onErrorResume(TimeoutException.class, e -> Mono.just(cachedUser))
    .onErrorResume(NotFoundException.class, e -> Mono.just(defaultUser))
    .onErrorReturn(unexpectedError);  // Last resort fallback
```

### 5. Context Propagation
```java
getUserMono()
    .contextWrite(Context.of("userId", userId))
    .contextWrite(Context.of("requestId", requestId));
```

## Testing Patterns

```java
// Use StepVerifier to test Mono
StepVerifier.create(getUserMono())
    .expectNextMatches(user -> user.id() > 0)
    .expectComplete()
    .verify(Duration.ofSeconds(10));
```

## Related Documentation

- [Project Reactor Mono Documentation](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Mono.html)
- [Spring WebFlux Guide](https://spring.io/guides/gs/reactive-rest-service/)
- [Reactive Programming Best Practices](https://www.baeldung.com/reactor-core)
