# GitHub Copilot & Prompt Engineering Cheat Sheet

> **Quick reference for building Java 17 Spring Boot APIs with GitHub Copilot**

---

## 📁 File Structure Overview

```
your-project/
├── .github/
│   ├── copilot-instructions.md    # Auto-loaded by Copilot (project standards)
│   └── prompts/                    # Reusable prompt templates
│       ├── create-entity.md
│       ├── create-dto.md
│       ├── create-repository.md
│       ├── create-service.md
│       ├── create-controller.md
│       └── create-exception-handler.md
├── META_PROMPT.md                  # Reference documentation
├── PROMPT_CHAINING.md              # Chaining concepts & patterns
└── PRACTICAL_EXAMPLE_USER_API.md   # Step-by-step real example
```

---

## 🎯 Quick Start

### Method 1: Use Custom Prompts
```
@workspace Use .github/prompts/create-entity.md to create a User entity
```

### Method 2: Reference Instructions
```
@workspace Following the guidelines in .github/copilot-instructions.md,
create a UserService with CRUD operations
```

### Method 3: Chain Multiple Steps
```
@workspace Create a complete feature:
1. Entity using create-entity.md
2. Repository using create-repository.md
3. Service using create-service.md
```

---

## 🔗 Prompt Chaining Pattern

### Simple 3-Step Chain
```
Step 1: ANALYZE
"Analyze requirements for [feature]. List entities, fields, operations."

Step 2: DESIGN
"Based on analysis: [paste Step 1 output]
Design [component] with [specifications]"

Step 3: IMPLEMENT
"Using this design: [paste Step 2 output]
Implement [component] following [standards]"
```

### Complete Feature Chain (11 Steps)
```
1. Requirements Analysis    → What to build
2. Entity Design            → User.java
3. Repository Design        → UserRepository.java
4. DTO Creation             → Request/Response DTOs
5. Custom Exceptions        → Domain exceptions
6. Mapper Creation          → Entity ↔ DTO
7. Service Layer            → Business logic
8. Controller Layer         → REST endpoints
9. Exception Handler        → Error handling
10. Unit Tests              → Service tests
11. Integration Tests       → API tests
```

---

## ✅ Best Practices

### 🎨 Prompting Best Practices

#### DO ✓
```
✓ Be specific and explicit
  "@workspace Use create-entity.md to create User entity with 
   username, email, password, firstName, lastName"

✓ Include context from previous steps
  "Using this User entity: [paste entity]
   Create UserRepository with findByUsername and findByEmail"

✓ Reference your custom files
  "@workspace Follow .github/copilot-instructions.md"

✓ Break complex tasks into steps
  Step 1: Analyze → Step 2: Design → Step 3: Implement

✓ Validate each step before proceeding
  Review output, then move to next step

✓ Use @workspace for file access
  "@workspace Read the User entity and create a service"
```

#### DON'T ✗
```
✗ Be vague
  "Create a user thing"

✗ Ask for everything at once
  "Create complete app with all features"

✗ Skip context
  "Now create the service" (AI might forget context)

✗ Ignore validation
  Move to next step without checking output

✗ Use prompt chaining for simple tasks
  "Fix this typo" doesn't need a chain
```

---

### 🏗️ Java Spring Boot Best Practices

#### Layered Architecture
```
Controller  → Handle HTTP, no business logic
Service     → Business logic, transactions
Repository  → Data access only
Entity      → Domain models
DTO         → API data transfer
```

#### Naming Conventions
```
Classes:     PascalCase          (UserService, UserController)
Methods:     camelCase           (findUserById, createUser)
Constants:   UPPER_SNAKE_CASE    (MAX_LOGIN_ATTEMPTS)
Packages:    lowercase           (com.example.userapi.service)
```

#### Dependency Injection
```java
✓ Constructor injection (preferred)
@Service
@RequiredArgsConstructor  // Lombok generates constructor
public class UserService {
    private final UserRepository userRepository;  // final = required
}

✗ Field injection (avoid)
@Autowired
private UserRepository userRepository;
```

#### REST Endpoint Design
```
Base path:    /api/v1
Resources:    Plural nouns       /users (not /user)
Multi-word:   Kebab-case         /user-profiles
No verbs:     Use HTTP methods   GET /users (not /getUsers)

Examples:
POST   /api/v1/users              → Create (201)
GET    /api/v1/users              → List (200)
GET    /api/v1/users/{id}         → Retrieve (200, 404)
PUT    /api/v1/users/{id}         → Full update (200, 404)
PATCH  /api/v1/users/{id}         → Partial update (200, 404)
DELETE /api/v1/users/{id}         → Delete (204, 404)
GET    /api/v1/users/search?q=x   → Search (200)
```

#### HTTP Status Codes
```
200 OK                  → Successful GET, PUT, PATCH
201 Created             → Successful POST (resource created)
204 No Content          → Successful DELETE
400 Bad Request         → Validation errors, invalid input
401 Unauthorized        → Authentication required/failed
403 Forbidden           → Insufficient permissions
404 Not Found           → Resource doesn't exist
409 Conflict            → Duplicate resource
500 Internal Error      → Unexpected server error
```

#### DTO Pattern (Always!)
```java
✓ Use DTOs for API requests/responses
public record UserRequestDto(
    @NotBlank String username,
    @Email String email
) {}

public record UserResponseDto(
    Long id,
    String username,
    String email,
    LocalDateTime createdAt
) {}

✗ Never expose entities in REST responses
// Don't return User entity directly!
```

#### Validation
```java
// At DTO level
public record UserRequestDto(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    String username,
    
    @Email(message = "Invalid email format")
    String email,
    
    @Size(min = 8)
    String password
) {}

// In controller
@PostMapping
public ResponseEntity<UserResponseDto> create(
    @Valid @RequestBody UserRequestDto dto) {  // @Valid triggers validation
    // ...
}
```

#### Exception Handling
```java
// Custom exceptions
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// Global handler
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handle(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            ex.getMessage(),
            request.getPath()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

#### Transactions
```java
✓ Use @Transactional for write operations
@Transactional
public UserResponseDto create(UserRequestDto dto) {
    // Multiple DB operations - all or nothing
}

✓ Read-only for queries (optional but good)
@Transactional(readOnly = true)
public UserResponseDto findById(Long id) {
    // ...
}
```

#### Security
```java
// Password hashing
@Service
public class UserService {
    private final PasswordEncoder passwordEncoder;
    
    public void register(UserRequestDto dto) {
        String hashed = passwordEncoder.encode(dto.password());
        user.setPassword(hashed);  // Store hashed, never plain!
    }
}

// Role-based access
@DeleteMapping("/{id}")
@PreAuthorize("hasRole('ADMIN')")  // Only admins
public ResponseEntity<Void> delete(@PathVariable Long id) {
    // ...
}

@PutMapping("/{id}")
@PreAuthorize("#id == authentication.principal.id or hasRole('ADMIN')")
public ResponseEntity<UserResponseDto> update(@PathVariable Long id) {
    // Owner or admin only
}
```

#### Logging
```java
@Service
@Slf4j  // Lombok annotation
public class UserService {
    
    public UserResponseDto create(UserRequestDto dto) {
        log.info("Creating user: {}", dto.username());      // INFO for important operations
        
        try {
            // business logic
            log.debug("User created with id: {}", id);      // DEBUG for details
        } catch (Exception e) {
            log.error("Error creating user", e);            // ERROR with exception
            throw new UserCreationException("Failed", e);
        }
    }
}

// Never log sensitive data!
✗ log.info("Password: {}", password);  // NO!
✓ log.info("User registered: {}", username);  // YES
```

#### Testing
```java
// Unit test (service layer)
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock private UserRepository repository;
    @InjectMocks private UserService service;
    
    @Test
    void shouldCreateUserWhenDataIsValid() {
        // Given
        UserRequestDto dto = new UserRequestDto(...);
        when(repository.save(any())).thenReturn(savedUser);
        
        // When
        UserResponseDto result = service.create(dto);
        
        // Then
        assertThat(result).isNotNull();
        assertThat(result.username()).isEqualTo("john");
        verify(repository).save(any());
    }
}

// Integration test (controller)
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired private MockMvc mockMvc;
    @MockBean private UserService service;
    
    @Test
    void shouldReturnUserWhenExists() throws Exception {
        when(service.findById(1L)).thenReturn(responseDto);
        
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.username").value("john"));
    }
}
```

---

## 🎨 Code Templates

### Entity Template
```java
@Entity
@Table(name = "entities")
@Getter @Setter
@NoArgsConstructor @AllArgsConstructor
public class Entity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @NotBlank
    @Column(nullable = false, unique = true)
    private String field;
    
    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Entity entity = (Entity) o;
        return Objects.equals(field, entity.field);  // Business key!
    }
    
    @Override
    public int hashCode() {
        return Objects.hash(field);
    }
}
```

### Repository Template
```java
@Repository
public interface EntityRepository extends JpaRepository<Entity, Long> {
    Optional<Entity> findByField(String field);
    boolean existsByField(String field);
    Page<Entity> findByEnabledTrue(Pageable pageable);
    
    @Query("SELECT e FROM Entity e WHERE LOWER(e.field) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Page<Entity> search(@Param("keyword") String keyword, Pageable pageable);
}
```

### Service Template
```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EntityService {
    private final EntityRepository repository;
    private final EntityMapper mapper;
    
    @Transactional
    public EntityResponseDto create(EntityRequestDto dto) {
        log.info("Creating entity");
        
        // Validation
        if (repository.existsByField(dto.field())) {
            throw new DuplicateEntityException("Already exists");
        }
        
        // Create and save
        Entity entity = mapper.toEntity(dto);
        Entity saved = repository.save(entity);
        
        log.debug("Entity created with id: {}", saved.getId());
        return mapper.toResponseDto(saved);
    }
    
    public EntityResponseDto findById(Long id) {
        Entity entity = repository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Not found: " + id));
        return mapper.toResponseDto(entity);
    }
}
```

### Controller Template
```java
@RestController
@RequestMapping("/api/v1/entities")
@RequiredArgsConstructor
@Validated
@Tag(name = "Entity Management")
public class EntityController {
    private final EntityService service;
    
    @PostMapping
    @Operation(summary = "Create entity")
    public ResponseEntity<EntityResponseDto> create(
            @Valid @RequestBody EntityRequestDto dto) {
        EntityResponseDto created = service.create(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
    
    @GetMapping("/{id}")
    @Operation(summary = "Get entity by ID")
    public ResponseEntity<EntityResponseDto> getById(
            @PathVariable @Positive Long id) {
        EntityResponseDto entity = service.findById(id);
        return ResponseEntity.ok(entity);
    }
    
    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Page<EntitySummaryDto>> getAll(
            @PageableDefault(size = 20) Pageable pageable) {
        Page<EntitySummaryDto> entities = service.findAll(pageable);
        return ResponseEntity.ok(entities);
    }
}
```

---

## 🚀 Common Workflows

### Workflow 1: Create New Entity Feature
```
1. Define requirements
   "@workspace Analyze requirements for Product entity"

2. Create entity
   "@workspace Use create-entity.md for Product entity"

3. Create repository
   "@workspace Use create-repository.md for ProductRepository"

4. Create DTOs
   "@workspace Use create-dto.md for Product DTOs"

5. Create service
   "@workspace Use create-service.md for ProductService"

6. Create controller
   "@workspace Use create-controller.md for ProductController"

7. Add tests
   "Generate unit tests for ProductService"
```

### Workflow 2: Add New Endpoint
```
1. Identify requirements
   "I need an endpoint to search products by category"

2. Update repository
   "Add findByCategory method to ProductRepository"

3. Update service
   "Add searchByCategory method to ProductService"

4. Update controller
   "Add GET /api/v1/products/search/category endpoint"

5. Test
   "Generate integration test for category search"
```

### Workflow 3: Refactor Existing Code
```
1. Analyze current code
   "Review this service for issues: [paste code]"

2. Prioritize issues
   "Rank these issues by severity"

3. Design solution
   "Suggest refactoring for top 3 issues"

4. Implement fix
   "Refactor method X following solution"

5. Validate
   "Generate tests to verify the refactoring"
```

### Workflow 4: Add Security
```
1. Design auth strategy
   "Design JWT authentication for User API"

2. Create SecurityConfig
   "Create Spring Security configuration with JWT"

3. Add JWT utilities
   "Create JwtTokenProvider for token generation/validation"

4. Add authentication endpoints
   "Add login and register endpoints"

5. Secure existing endpoints
   "Add @PreAuthorize annotations to UserController"
```

---

## 💡 Pro Tips

### 1. **Use @workspace Liberally**
```
Always include @workspace when referencing files:
"@workspace Use .github/prompts/create-entity.md"
```

### 2. **Save Outputs Between Steps**
```
Keep a scratch file with outputs from each step.
You'll need to reference them in later prompts.
```

### 3. **Validate Before Proceeding**
```
After each step:
- Does the code compile?
- Does it follow standards?
- Are there any obvious issues?
If yes to all, proceed. If no, fix first.
```

### 4. **Be Specific About Java 17**
```
"Use Java 17 features like records for DTOs"
"Use pattern matching where appropriate"
"Use text blocks for multi-line strings"
```

### 5. **Request Specific Annotations**
```
"Include Swagger/OpenAPI annotations"
"Add Lombok annotations to reduce boilerplate"
"Use Jakarta validation annotations"
```

### 6. **Ask for Complete Code**
```
Bad:  "Create a service"
Good: "Create UserService with constructor injection, 
       logging, transactions, and exception handling"
```

### 7. **Reference Standards Explicitly**
```
"Following .github/copilot-instructions.md standards"
"Using the patterns from META_PROMPT.md"
"As shown in PRACTICAL_EXAMPLE_USER_API.md Step 7"
```

### 8. **Use Prompt Templates for Consistency**
```
Instead of writing prompts from scratch each time,
reference your custom prompts:
"@workspace Use create-service.md for OrderService"
```

### 9. **Chain Related Operations**
```
For a complete feature, chain all steps:
Requirements → Entity → Repository → DTO → 
Service → Controller → Tests
```

### 10. **Test As You Go**
```
Don't wait until the end to write tests.
After each component, generate tests:
"Generate unit tests for the UserService we just created"
```

---

## 📊 Decision Tree: When to Use What

```
Need to build something?
│
├─ Is it a single file/component?
│  ├─ Yes → Use specific prompt template
│  │        "@workspace Use create-entity.md for User"
│  │
│  └─ No → Is it a complete feature?
│          └─ Yes → Use prompt chain
│                   Follow 11-step chain from practical example
│
├─ Need to refactor/fix existing code?
│  ├─ Simple fix → Direct prompt
│  │              "Fix validation in UserController"
│  │
│  └─ Complex refactoring → Use chain
│                          Analyze → Design → Implement → Test
│
├─ Need to understand existing code?
│  └─ Direct analysis prompt
│     "@workspace Explain how UserService handles authentication"
│
└─ Adding new feature to existing code?
   ├─ Single endpoint → 3-step chain
   │                    Repository → Service → Controller
   │
   └─ Multiple related changes → Full chain
                                  Requirements → Design → Implement → Test
```

---

## 🎯 Quality Checklist

Before moving to the next step in a chain, verify:

### Code Quality
- [ ] Follows naming conventions
- [ ] Uses appropriate design patterns
- [ ] Has proper error handling
- [ ] Includes logging
- [ ] No code smells (long methods, god classes, etc.)

### Spring Boot Standards
- [ ] Uses constructor injection
- [ ] DTOs for API layer (not entities)
- [ ] @Transactional on write operations
- [ ] Proper HTTP status codes
- [ ] RESTful URL design

### Security
- [ ] Passwords are hashed
- [ ] Sensitive data not logged
- [ ] Input validation present
- [ ] Authorization checks in place
- [ ] No SQL injection vulnerabilities

### Testing
- [ ] Unit tests for business logic
- [ ] Integration tests for endpoints
- [ ] Edge cases covered
- [ ] Mocks used appropriately
- [ ] Test names are descriptive

### Documentation
- [ ] JavaDoc for public APIs
- [ ] OpenAPI/Swagger annotations
- [ ] Meaningful comments where needed
- [ ] README updated if needed

---

## 🔑 Keyboard Shortcuts (VS Code)

```
Cmd/Ctrl + I        → Open Copilot Chat
Alt/Option + \      → Toggle inline Copilot
Tab                 → Accept suggestion
Esc                 → Reject suggestion
Alt/Option + ]      → Next suggestion
Alt/Option + [      → Previous suggestion
```

---

## 📚 Quick Reference Commands

### Start a New Feature
```
@workspace Create a complete [Entity] management feature:
1. Entity using create-entity.md
2. Repository using create-repository.md
3. DTOs using create-dto.md
4. Service using create-service.md
5. Controller using create-controller.md
6. Exception handling
7. Tests
```

### Add to Existing Feature
```
@workspace For the existing User entity:
Add a new endpoint GET /api/v1/users/active that returns 
all active users with pagination. Update repository, service, 
and controller.
```

### Refactor Code
```
@workspace Review this code for:
- Code smells
- Performance issues
- Security vulnerabilities
- Best practice violations
[paste code]

Then suggest refactoring with examples.
```

### Generate Tests
```
@workspace Generate unit tests for this service:
[paste service class]

Include tests for:
- Happy paths
- Edge cases
- Exception scenarios
Use JUnit 5, Mockito, and AssertJ.
```

### Add Security
```
@workspace Add Spring Security with JWT:
1. SecurityConfig with SecurityFilterChain
2. JwtTokenProvider for token management
3. Authentication filter
4. Login/register endpoints
5. Secure existing endpoints by role
```

---

## 🆘 Troubleshooting

### Copilot Not Finding Custom Prompts
```
✗ "Use create-entity.md"
✓ "@workspace Use .github/prompts/create-entity.md"
   Always use @workspace and full path
```

### Copilot Forgetting Context
```
✗ "Now create the service"
✓ "Using this User entity: [paste entity]
   Create UserService with..."
   Always include relevant context in each prompt
```

### Code Doesn't Follow Standards
```
✗ Generic prompt
✓ "Following .github/copilot-instructions.md, create..."
   Reference your standards file explicitly
```

### Overwhelming Response
```
✗ "Create entire app"
✓ Break into steps:
   Step 1: Entity
   Step 2: Repository
   Step 3: Service
   One component at a time
```

### Tests Not Comprehensive
```
✗ "Generate tests"
✓ "Generate tests for UserService:
   - Success cases
   - Validation errors
   - Not found exceptions
   - Duplicate handling
   Use JUnit 5, Mockito, AssertJ"
   Be specific about what to test
```

---

## 🎓 Learning Path

### Week 1: Basics
- [ ] Set up .github/copilot-instructions.md
- [ ] Create your first entity with Copilot
- [ ] Practice simple prompts
- [ ] Get comfortable with @workspace

### Week 2: Prompts
- [ ] Create custom prompt templates
- [ ] Build one complete feature manually
- [ ] Document your own patterns

### Week 3: Chaining
- [ ] Try 3-step prompt chain
- [ ] Build feature with full 11-step chain
- [ ] Create your own chain template

### Week 4: Mastery
- [ ] Adapt chains for different domains
- [ ] Create project-specific prompts
- [ ] Teach prompt chaining to team

---

## 🔗 Quick Links to Your Files

```bash
# View instructions
code .github/copilot-instructions.md

# View prompt templates
code .github/prompts/create-entity.md
code .github/prompts/create-service.md
code .github/prompts/create-controller.md

# View documentation
code META_PROMPT.md
code PROMPT_CHAINING.md
code PRACTICAL_EXAMPLE_USER_API.md
```

---

## 📝 Customization Template

Copy and customize for your team:

```markdown
# [Your Company] Copilot Standards

## Tech Stack
- Java [version]
- Spring Boot [version]
- Database: [PostgreSQL/MySQL/etc]
- Build Tool: [Maven/Gradle]

## Naming Conventions
- Controllers: [pattern]
- Services: [pattern]
- Repositories: [pattern]

## Custom Patterns
[Your specific patterns]

## Security Requirements
[Your security standards]

## Testing Requirements
[Your testing standards]
```

---

## 🎉 Summary

### Remember:
1. ✅ Use @workspace for file access
2. ✅ Reference custom prompts explicitly
3. ✅ Chain complex tasks into steps
4. ✅ Validate each step before proceeding
5. ✅ Include context in every prompt
6. ✅ Follow Spring Boot best practices
7. ✅ Test as you go
8. ✅ Document your patterns

### The Golden Rule:
> **Be specific, provide context, validate often, and chain when complex.**

---

**Happy Coding! 🚀**

*Last updated: November 21, 2025*
