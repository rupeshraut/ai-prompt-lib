# GitHub Copilot Instructions - Java 17 Spring Boot User API

## Project Overview
This is a Java 17 Spring Boot REST API for user management. Follow these instructions for all code suggestions and completions.

## Java & Spring Boot Standards

### Java Version
- Use Java 17 features: records, sealed classes, pattern matching, text blocks
- Target Java 17 LTS compatibility
- Use `var` for local variables when type is obvious

### Code Style
- Follow Google Java Style Guide
- Use 4 spaces for indentation
- Maximum line length: 120 characters
- Always use braces for if/else/for/while blocks
- Place opening brace on same line

### Naming Conventions
- Classes: `PascalCase` (e.g., `UserService`, `UserController`)
- Methods/Variables: `camelCase` (e.g., `findUserById`, `userRepository`)
- Constants: `UPPER_SNAKE_CASE` (e.g., `MAX_LOGIN_ATTEMPTS`)
- Packages: lowercase (e.g., `com.example.userapi.service`)

## Spring Boot Architecture

### Layered Structure
Always follow this layered architecture:
1. **Controller** - Handle HTTP requests/responses, no business logic
2. **Service** - Business logic, transaction management
3. **Repository** - Data access only
4. **Entity** - JPA entities with proper relationships
5. **DTO** - Data transfer objects for API requests/responses

### Dependency Injection
- Use constructor injection (preferred over field injection)
- Mark injected fields as `final`
- Use `@RequiredArgsConstructor` from Lombok when appropriate

```java
@Service
@RequiredArgsConstructor
public class UserService {
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    // ...
}
```

## REST API Guidelines

### Endpoint Naming
- Base path: `/api/v1`
- Use plural nouns: `/users`, not `/user`
- Use kebab-case for multi-word resources: `/user-profiles`
- Avoid verbs in URLs; use HTTP methods

### HTTP Methods
- `GET` - Retrieve resources (idempotent)
- `POST` - Create new resources
- `PUT` - Full update of existing resources
- `PATCH` - Partial update of existing resources
- `DELETE` - Remove resources (idempotent)

### Response Status Codes
Use appropriate HTTP status codes:
- `200 OK` - Successful GET, PUT, PATCH
- `201 Created` - Successful POST
- `204 No Content` - Successful DELETE
- `400 Bad Request` - Validation errors
- `401 Unauthorized` - Authentication required
- `403 Forbidden` - Insufficient permissions
- `404 Not Found` - Resource doesn't exist
- `409 Conflict` - Duplicate resource
- `500 Internal Server Error` - Unexpected errors

### Controller Template
```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
public class UserController {
    private final UserService userService;

    @GetMapping("/{id}")
    public ResponseEntity<UserResponseDto> getUserById(@PathVariable Long id) {
        UserResponseDto user = userService.findById(id);
        return ResponseEntity.ok(user);
    }

    @PostMapping
    public ResponseEntity<UserResponseDto> createUser(@Valid @RequestBody UserRequestDto request) {
        UserResponseDto user = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}
```

## Data Handling

### Entity Guidelines
- Use `@Entity` and `@Table(name = "users")`
- Always include audit fields: `createdAt`, `updatedAt`
- Use `@GeneratedValue(strategy = GenerationType.IDENTITY)` for IDs
- Implement `equals()` and `hashCode()` based on business key (not ID)
- Never expose entities in REST responses

### DTO Pattern
- Always use DTOs for API requests and responses
- Use Java records for immutable DTOs (Java 17+)
- Separate request and response DTOs
- Add validation annotations to request DTOs

```java
public record UserRequestDto(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    String username,
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password
) {}

public record UserResponseDto(
    Long id,
    String username,
    String email,
    String firstName,
    String lastName,
    LocalDateTime createdAt
) {}
```

### Repository Pattern
- Extend `JpaRepository<Entity, Long>`
- Use Spring Data JPA query methods
- Add `@Query` for complex queries
- Return `Optional<T>` for single results

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    boolean existsByUsername(String username);
    Page<User> findByEnabledTrue(Pageable pageable);
}
```

## Validation & Error Handling

### Validation
- Use Bean Validation API (`jakarta.validation.constraints`)
- Validate at the DTO level, not entity level
- Use `@Valid` in controller methods
- Create custom validators for complex business rules

### Exception Handling
- Create custom exceptions for business logic errors
- Implement global exception handler with `@ControllerAdvice`
- Return consistent error response format

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            ex.getMessage(),
            "/api/v1/users"
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidationErrors(
        MethodArgumentNotValidException ex) {
        // Handle validation errors
    }
}
```

## Security Best Practices

### Password Handling
- Always hash passwords using BCrypt
- Never log or expose passwords
- Use `PasswordEncoder` from Spring Security

### Authentication
- Implement JWT-based authentication
- Store tokens securely (HttpOnly cookies or secure storage)
- Implement token refresh mechanism
- Set appropriate token expiration

### Authorization
- Use `@PreAuthorize` for method-level security
- Implement role-based access control (RBAC)
- Check permissions in service layer

```java
@PreAuthorize("hasRole('ADMIN')")
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.deleteUser(id);
    return ResponseEntity.noContent().build();
}
```

### Input Sanitization
- Validate all user inputs
- Sanitize data before processing
- Protect against SQL injection (use JPA/JPQL)
- Protect against XSS attacks

## Database Best Practices

### Transactions
- Use `@Transactional` on service methods that modify data
- Set appropriate isolation levels
- Keep transactions short and focused

### Query Optimization
- Use pagination for list queries (`Pageable`)
- Implement projection for read-only queries
- Use `@EntityGraph` to avoid N+1 queries
- Add database indexes on frequently queried columns

### Migrations
- Use Flyway or Liquibase for database migrations
- Version all schema changes
- Never modify existing migrations

## Testing Requirements

### Unit Tests
- Test service layer with Mockito
- Mock repository dependencies
- Test business logic thoroughly
- Aim for 80%+ code coverage

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;
    
    @Mock
    private PasswordEncoder passwordEncoder;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void shouldCreateUserSuccessfully() {
        // Given
        UserRequestDto request = new UserRequestDto("john", "john@example.com", "password");
        // When & Then
    }
}
```

### Integration Tests
- Use `@SpringBootTest` for full integration tests
- Use `@WebMvcTest` for controller tests
- Use `@DataJpaTest` for repository tests
- Use test containers for database tests

## Configuration

### Application Properties
- Use `application.yml` (preferred over .properties)
- Define profiles: `dev`, `test`, `prod`
- Externalize secrets (use environment variables)
- Never commit sensitive data

```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/userdb}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:password}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
    properties:
      hibernate:
        format_sql: true
```

## Logging

### Logging Standards
- Use SLF4J with Logback
- Use appropriate log levels: ERROR, WARN, INFO, DEBUG, TRACE
- Never log sensitive information (passwords, tokens, PII)
- Use structured logging for production

```java
@Slf4j
@Service
public class UserService {
    public UserResponseDto createUser(UserRequestDto request) {
        log.info("Creating new user with username: {}", request.username());
        try {
            // business logic
            log.debug("User created successfully with id: {}", user.getId());
        } catch (Exception e) {
            log.error("Error creating user: {}", e.getMessage(), e);
            throw new UserCreationException("Failed to create user", e);
        }
    }
}
```

## Code Quality Rules

### General
- Write self-documenting code with clear names
- Keep methods small (< 20 lines when possible)
- Follow Single Responsibility Principle
- Prefer composition over inheritance
- Use `Optional<T>` instead of returning null

### Comments
- Use JavaDoc for public APIs
- Avoid obvious comments
- Explain "why", not "what"
- Keep comments up-to-date

### Immutability
- Prefer immutable objects (use records, final fields)
- Use unmodifiable collections where possible
- Avoid mutable static state

## Performance Considerations

- Use lazy loading for entity relationships
- Implement caching with Spring Cache where appropriate
- Use batch operations for bulk updates
- Monitor performance with Spring Actuator
- Use async processing for long-running tasks (`@Async`)

## Documentation

- Document all public APIs with Swagger/OpenAPI annotations
- Keep README.md updated with setup instructions
- Document configuration options
- Provide examples for API usage

---

## Quick Reference

**When creating a new feature:**
1. Create entity (if needed)
2. Create repository interface
3. Create request/response DTOs
4. Implement service with business logic
5. Create controller endpoint
6. Add exception handling
7. Write unit tests
8. Write integration tests
9. Add API documentation

**Always remember:**
- Security first (validate, sanitize, authenticate, authorize)
- DTOs for API, never expose entities
- Constructor injection for dependencies
- Return appropriate HTTP status codes
- Handle exceptions gracefully
- Test your code
- Log appropriately (no sensitive data)
