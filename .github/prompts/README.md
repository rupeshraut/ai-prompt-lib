# Custom GitHub Copilot Prompts

This directory contains custom prompts that can be used with GitHub Copilot to generate consistent, high-quality code across multiple frameworks.

## Frameworks

| Framework | Directory | Description |
|-----------|-----------|-------------|
| Spring Boot | `.github/prompts/` | Java 17 REST API with JPA, Security, Testing |
| Apache Flink | `.github/prompts/flink-fednow/` | FedNow payment processing with ISO 20022 |

---

## Spring Boot Prompts

### Core Components

| Template | Description |
|----------|-------------|
| `create-entity.md` | Generate JPA entity classes with proper annotations and relationships |
| `create-dto.md` | Create request/response DTOs using Java 17 records |
| `create-repository.md` | Generate Spring Data JPA repository interfaces |
| `create-service.md` | Create service classes with business logic |
| `create-controller.md` | Generate REST controllers with proper endpoints |
| `create-exception-handler.md` | Create global exception handler with consistent error responses |

### Supporting Components

| Template | Description |
|----------|-------------|
| `create-mapper.md` | Entity-DTO mappers (manual or MapStruct) |
| `create-test.md` | Unit tests, integration tests, repository tests |

### Infrastructure

| Template | Description |
|----------|-------------|
| `create-security-config.md` | Complete JWT authentication setup (SecurityConfig, JwtTokenProvider, filters, AuthController) |
| `create-migration.md` | Database migrations with Flyway or Liquibase |

---

## Apache Flink Prompts (FedNow)

See [flink-fednow/README.md](flink-fednow/README.md) for detailed documentation.

| Template | Description |
|----------|-------------|
| `flink-fednow/create-flink-job.md` | Main Flink job with checkpointing and RocksDB state backend |
| `flink-fednow/create-async-function.md` | RichAsyncFunction for non-blocking HTTP calls |
| `flink-fednow/create-sanctions-processor.md` | KeyedProcessFunction for JMS sanctions screening |
| `flink-fednow/create-iso20022-parser.md` | ISO 20022 XML parsers and serializers |
| `flink-fednow/create-payment-model.md` | Payment domain model with status enums |
| `flink-fednow/create-flink-test.md` | Flink test harness and integration tests |

---

## How to Use Custom Prompts

### Method 1: Using Copilot Chat (Recommended)
1. Open GitHub Copilot Chat (Cmd+I or Ctrl+I)
2. Type `@workspace` to give Copilot access to workspace files
3. Reference the prompt file:
   ```
   @workspace Use .github/prompts/create-entity.md to create a User entity
   ```

### Method 2: Using #file Reference
1. Open Copilot Chat
2. Use the `#file` command to reference the prompt:
   ```
   #file:.github/prompts/create-entity.md Create a User entity
   ```

### Method 3: Copy-Paste Method
1. Open the desired prompt file from `.github/prompts/`
2. Copy the entire content
3. Paste into Copilot Chat with your specific requirements
4. Example:
   ```
   [Paste prompt content]
   
   Create a User entity with fields: id, username, email, password, 
   firstName, lastName, enabled, createdAt, updatedAt
   ```

### Method 4: Inline Comments
1. In your code file, add a comment referencing the prompt:
   ```java
   // @copilot Use .github/prompts/create-entity.md
   // Create User entity with username, email, password fields
   ```
2. Wait for Copilot suggestions

## Example Usage

### Example 1: Create User Entity
```
@workspace Use .github/prompts/create-entity.md

Create a User entity with:
- id (Long, auto-generated)
- username (unique, 3-50 chars)
- email (unique, valid email)
- password (BCrypt hashed)
- firstName, lastName
- enabled (boolean)
- roles (Many-to-Many with Role entity)
- createdAt, updatedAt timestamps
```

### Example 2: Create User Service
```
@workspace Use .github/prompts/create-service.md

Create UserService with methods:
- findById(Long id)
- findByUsername(String username)
- create(UserRequestDto)
- update(Long id, UserRequestDto)
- delete(Long id)
- findAll(Pageable)
```

### Example 3: Create User Controller
```
@workspace Use .github/prompts/create-controller.md

Create UserController with endpoints:
- GET /api/v1/users/{id}
- GET /api/v1/users (paginated)
- POST /api/v1/users
- PUT /api/v1/users/{id}
- DELETE /api/v1/users/{id} (ADMIN only)
- GET /api/v1/users/search
```

### Example 4: Create Complete Feature
```
@workspace Create a complete User management feature:

1. Use create-entity.md for User entity
2. Use create-repository.md for UserRepository
3. Use create-dto.md for UserRequestDto and UserResponseDto
4. Use create-service.md for UserService
5. Use create-controller.md for UserController

Include validation, exception handling, and proper documentation.
```

## Tips for Best Results

### Be Specific
Instead of:
```
Create a controller
```
Use:
```
@workspace Use .github/prompts/create-controller.md
Create UserController for managing users with CRUD operations
```

### Combine with Context
```
@workspace I have a User entity in src/main/java/com/example/entity/User.java
Use .github/prompts/create-service.md to create UserService
```

### Request Multiple Components
```
@workspace Create a complete feature:
1. Entity using create-entity.md
2. Repository using create-repository.md
3. DTOs using create-dto.md
4. Service using create-service.md
5. Controller using create-controller.md
```

### Customize the Prompts
```
@workspace Use .github/prompts/create-entity.md as a base
but add soft delete functionality with deletedAt timestamp
```

## Advanced Techniques

### Chain Multiple Prompts
```
@workspace First, use create-entity.md to create Product entity.
Then use create-repository.md to create ProductRepository.
Finally use create-service.md to create ProductService.
```

### Modify Existing Code
```
@workspace Look at the existing UserService.
Use create-service.md guidelines to add a new method: 
findActiveUsers(Pageable pageable)
```

### Generate Tests
```
@workspace Based on UserService and following create-service.md patterns,
generate unit tests using JUnit 5 and Mockito
```

## Keyboard Shortcuts

- **Open Copilot Chat**: `Cmd+I` (Mac) or `Ctrl+I` (Windows/Linux)
- **Toggle Copilot Inline**: `Alt+\` or `Option+\`
- **Accept Suggestion**: `Tab`
- **Reject Suggestion**: `Esc`
- **Next Suggestion**: `Alt+]` or `Option+]`
- **Previous Suggestion**: `Alt+[` or `Option+[`

## Troubleshooting

### Copilot doesn't find the prompt file
- Make sure you're using `@workspace` prefix
- Check that the file path is correct: `.github/prompts/filename.md`
- Verify the file exists in your workspace

### Suggestions don't follow the prompt
- Be more explicit in your request
- Include more context from the prompt file
- Try referencing specific sections: "Use the validation section from create-dto.md"

### Prompts not working
- Ensure GitHub Copilot is enabled and active
- Check your Copilot subscription status
- Restart VS Code if needed
- Make sure you're in a supported file type (Java, Markdown, etc.)

## Creating Your Own Prompts

You can create custom prompts for your specific needs:

1. Create a new `.md` file in `.github/prompts/`
2. Structure your prompt with:
   - Clear requirements
   - Code examples
   - Best practices
   - Common patterns
3. Reference it using `@workspace #file:.github/prompts/your-prompt.md`

## Resources

- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Java 17 Documentation](https://docs.oracle.com/en/java/javase/17/)
