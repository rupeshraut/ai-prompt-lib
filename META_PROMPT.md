# META PROMPT: Java 17 Spring Boot User API

## Project Context
This is a Java 17 Spring Boot application implementing a RESTful User API with modern best practices, security, and scalability in mind.

## Core Technologies
- **Java 17** (LTS) - Use modern Java features (records, sealed classes, pattern matching, text blocks)
- **Spring Boot 3.x** - Latest stable version
- **Spring Data JPA** - Database interactions
- **Spring Security** - Authentication & Authorization
- **PostgreSQL/MySQL** - Primary database
- **Maven/Gradle** - Build tool
- **JUnit 5 & Mockito** - Testing framework

## Architecture Patterns

### Layered Architecture
```
Controller Layer (REST endpoints)
    ↓
Service Layer (Business logic)
    ↓
Repository Layer (Data access)
    ↓
Entity Layer (Domain models)
```

### Package Structure
```
com.example.userapi
├── controller/     # REST controllers
├── service/        # Business logic
├── repository/     # JPA repositories
├── entity/         # JPA entities
├── dto/            # Data Transfer Objects
├── mapper/         # Entity ↔ DTO mappers
├── exception/      # Custom exceptions
├── config/         # Configuration classes
└── security/       # Security configuration
```

## User API Specifications

### Core User Entity
```java
- id: Long (Primary Key, Auto-generated)
- username: String (Unique, Not Null, 3-50 chars)
- email: String (Unique, Not Null, Valid email format)
- password: String (Hashed, BCrypt)
- firstName: String
- lastName: String
- roles: Set<Role> (Many-to-Many)
- enabled: Boolean
- createdAt: LocalDateTime
- updatedAt: LocalDateTime
```

### REST Endpoints
```
POST   /api/v1/users              - Create user
GET    /api/v1/users              - Get all users (paginated)
GET    /api/v1/users/{id}         - Get user by ID
PUT    /api/v1/users/{id}         - Update user
DELETE /api/v1/users/{id}         - Delete user
GET    /api/v1/users/search       - Search users by criteria
POST   /api/v1/auth/register      - Register new user
POST   /api/v1/auth/login         - Authenticate user
POST   /api/v1/auth/refresh       - Refresh JWT token
```

## Code Generation Guidelines

### 1. Entity Design
- Use `@Entity`, `@Table` with proper naming
- Always include `@Id`, `@GeneratedValue(strategy = GenerationType.IDENTITY)`
- Use `@Column` annotations with constraints (nullable, unique, length)
- Implement `equals()` and `hashCode()` based on business key
- Use `@CreationTimestamp` and `@UpdateTimestamp` for audit fields
- Never expose entities directly in REST responses

### 2. DTO Pattern
- Create separate DTOs for requests and responses
- Use Java records for immutable DTOs (Java 17)
- Implement validation annotations (`@Valid`, `@NotNull`, `@Email`, `@Size`)
- Example: `UserRequestDto`, `UserResponseDto`, `UserSummaryDto`

### 3. Service Layer
- Use `@Service` annotation
- Inject dependencies via constructor (final fields)
- Implement business logic validation
- Handle exceptions appropriately
- Use `@Transactional` where needed
- Return DTOs, not entities

### 4. Controller Layer
- Use `@RestController` and `@RequestMapping`
- Follow RESTful conventions
- Use proper HTTP status codes (200, 201, 204, 400, 404, etc.)
- Implement `ResponseEntity<T>` for responses
- Use `@Valid` for request body validation
- Add API documentation with Swagger/OpenAPI annotations

### 5. Repository Layer
- Extend `JpaRepository<Entity, ID>`
- Use Spring Data JPA query methods
- Add custom queries with `@Query` when needed
- Use `Pageable` for pagination
- Use `Specification` for dynamic queries

### 6. Exception Handling
- Create custom exceptions (`UserNotFoundException`, `DuplicateUserException`)
- Implement `@ControllerAdvice` for global exception handling
- Return consistent error responses with proper HTTP status codes
```java
{
  "timestamp": "2025-11-21T10:00:00",
  "status": 404,
  "error": "Not Found",
  "message": "User not found with id: 123",
  "path": "/api/v1/users/123"
}
```

### 7. Security
- Use Spring Security with JWT or OAuth2
- Hash passwords with BCrypt
- Implement role-based access control (RBAC)
- Secure endpoints with `@PreAuthorize` or `SecurityFilterChain`
- Never log or expose sensitive data
- Implement rate limiting for auth endpoints

### 8. Validation
- Use Bean Validation API (`jakarta.validation`)
- Apply validation at DTO level
- Custom validators for complex rules
- Return meaningful validation error messages

### 9. Testing
- Unit tests for services (Mockito)
- Integration tests for repositories (`@DataJpaTest`)
- Controller tests (`@WebMvcTest`, MockMvc)
- Minimum 80% code coverage
- Test happy paths and edge cases

### 10. Configuration
- Use `application.yml` over `application.properties`
- Externalize configuration (database, JWT secrets)
- Use profiles (dev, test, prod)
- Never commit secrets to version control

## Best Practices

### Code Quality
- Follow SOLID principles
- Keep methods small and focused (< 20 lines)
- Use meaningful variable and method names
- Add JavaDoc for public APIs
- Use Optional<T> for nullable returns
- Avoid null returns; prefer Optional or empty collections

### Database
- Use database migrations (Flyway or Liquibase)
- Index frequently queried columns
- Use connection pooling (HikariCP)
- Implement soft deletes for user data
- Add database constraints (foreign keys, unique constraints)

### API Design
- Version your APIs (/api/v1/...)
- Use plural nouns for resources
- Support pagination, sorting, and filtering
- Implement HATEOAS for discoverability (optional)
- Use proper HTTP methods (GET, POST, PUT, DELETE, PATCH)

### Performance
- Use lazy loading for relationships
- Implement caching where appropriate (Spring Cache)
- Use database projections for read-only queries
- Batch operations when possible
- Monitor with Spring Actuator

### Security Checklist
- ✓ Validate all inputs
- ✓ Sanitize outputs
- ✓ Use HTTPS in production
- ✓ Implement CSRF protection
- ✓ Set proper CORS policies
- ✓ Rate limit API endpoints
- ✓ Log security events

## Example Prompt Templates

### Creating a New Endpoint
```
Create a new endpoint to [action] for the User API:
- Endpoint: [METHOD] /api/v1/users/[path]
- Request: [describe request body/params]
- Response: [describe response]
- Validation: [list validation rules]
- Security: [authentication/authorization requirements]

Include:
1. Controller method with proper annotations
2. Service layer implementation
3. DTO classes (request/response)
4. Exception handling
5. Unit tests for service and controller
```

### Refactoring Code
```
Refactor the [component name] to:
- Follow [specific pattern/principle]
- Improve [performance/readability/maintainability]
- Add [missing functionality]
- Ensure backward compatibility
- Update tests accordingly
```

### Adding Security
```
Implement JWT-based authentication for the User API:
1. Create SecurityConfig with SecurityFilterChain
2. Implement JwtTokenProvider for token generation/validation
3. Create JwtAuthenticationFilter
4. Secure endpoints based on roles (USER, ADMIN)
5. Add login and register endpoints
6. Include refresh token mechanism
```

## Common Pitfalls to Avoid
- ❌ Returning entities directly from controllers
- ❌ Not handling exceptions properly
- ❌ Missing validation on user inputs
- ❌ Hardcoding configuration values
- ❌ Not using transactions for multi-step operations
- ❌ Ignoring N+1 query problems
- ❌ Not implementing proper logging
- ❌ Exposing sensitive information in error messages

## Documentation Requirements
- README.md with setup instructions
- API documentation (Swagger/OpenAPI)
- Architecture decision records (ADRs)
- Database schema documentation
- Deployment guide

## Monitoring & Observability
- Use Spring Boot Actuator
- Implement structured logging (JSON format)
- Add metrics (Micrometer + Prometheus)
- Set up health checks
- Monitor database performance

---

**Remember**: This meta-prompt serves as a comprehensive guide. Always adapt to specific project requirements and team conventions.
