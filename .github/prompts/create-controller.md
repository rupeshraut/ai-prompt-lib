# Create REST Controller

Create a REST controller that handles HTTP requests following Spring Boot and RESTful best practices.

## Requirements

The controller should include:
- `@RestController` and `@RequestMapping` annotations
- Constructor injection for service dependencies
- Proper HTTP method annotations (@GetMapping, @PostMapping, etc.)
- Request validation with `@Valid`
- Appropriate HTTP status codes in responses
- Exception handling (handled by global handler)
- API documentation with Swagger/OpenAPI annotations
- No business logic (delegate to service layer)

## Code Style
- Use constructor injection (final fields)
- Use `@RequiredArgsConstructor` from Lombok
- Use `ResponseEntity<T>` for all responses
- Follow RESTful URL conventions
- Add `@Validated` at class level
- Use path variables and request parameters appropriately

## Example Structure

```java
@RestController
@RequestMapping("/api/v1/entities")
@RequiredArgsConstructor
@Validated
@Tag(name = "Entity Management", description = "APIs for managing entities")
public class EntityController {
    
    private final EntityService entityService;
    
    /**
     * Get entity by ID.
     */
    @GetMapping("/{id}")
    @Operation(summary = "Get entity by ID", description = "Retrieve a single entity by its ID")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Entity found"),
        @ApiResponse(responseCode = "404", description = "Entity not found")
    })
    public ResponseEntity<EntityResponseDto> getById(
            @PathVariable @Positive Long id) {
        EntityResponseDto entity = entityService.findById(id);
        return ResponseEntity.ok(entity);
    }
    
    /**
     * Get all entities with pagination.
     */
    @GetMapping
    @Operation(summary = "Get all entities", description = "Retrieve all entities with pagination")
    public ResponseEntity<Page<EntityResponseDto>> getAll(
            @PageableDefault(size = 20, sort = "id") Pageable pageable) {
        Page<EntityResponseDto> entities = entityService.findAll(pageable);
        return ResponseEntity.ok(entities);
    }
    
    /**
     * Create new entity.
     */
    @PostMapping
    @Operation(summary = "Create entity", description = "Create a new entity")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Entity created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid input data")
    })
    public ResponseEntity<EntityResponseDto> create(
            @Valid @RequestBody EntityRequestDto requestDto) {
        EntityResponseDto created = entityService.create(requestDto);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
    
    /**
     * Update existing entity.
     */
    @PutMapping("/{id}")
    @Operation(summary = "Update entity", description = "Update an existing entity")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Entity updated successfully"),
        @ApiResponse(responseCode = "404", description = "Entity not found"),
        @ApiResponse(responseCode = "400", description = "Invalid input data")
    })
    public ResponseEntity<EntityResponseDto> update(
            @PathVariable @Positive Long id,
            @Valid @RequestBody EntityRequestDto requestDto) {
        EntityResponseDto updated = entityService.update(id, requestDto);
        return ResponseEntity.ok(updated);
    }
    
    /**
     * Delete entity.
     */
    @DeleteMapping("/{id}")
    @Operation(summary = "Delete entity", description = "Delete an entity by ID")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "204", description = "Entity deleted successfully"),
        @ApiResponse(responseCode = "404", description = "Entity not found")
    })
    @PreAuthorize("hasRole('ADMIN')")  // Example: Require ADMIN role
    public ResponseEntity<Void> delete(@PathVariable @Positive Long id) {
        entityService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    /**
     * Search entities by criteria.
     */
    @GetMapping("/search")
    @Operation(summary = "Search entities", description = "Search entities by various criteria")
    public ResponseEntity<Page<EntityResponseDto>> search(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String status,
            @PageableDefault(size = 20) Pageable pageable) {
        Page<EntityResponseDto> results = entityService.search(keyword, status, pageable);
        return ResponseEntity.ok(results);
    }
}
```

## HTTP Status Codes
Use appropriate status codes:
- **200 OK** - Successful GET, PUT, PATCH
- **201 Created** - Successful POST (resource created)
- **204 No Content** - Successful DELETE
- **400 Bad Request** - Validation errors, invalid input
- **401 Unauthorized** - Authentication required
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Resource doesn't exist
- **409 Conflict** - Duplicate resource, business rule violation
- **500 Internal Server Error** - Unexpected server errors

## RESTful URL Conventions
- Base path: `/api/v1`
- Use plural nouns: `/users`, not `/user`
- Use kebab-case for multi-word resources: `/user-profiles`
- Avoid verbs in URLs; use HTTP methods instead
- Use path variables for IDs: `/users/{id}`
- Use query parameters for filtering/searching: `/users?status=active`

## Validation
- Use `@Valid` on request bodies
- Use validation annotations on path variables and parameters
- Let Bean Validation handle constraint validation
- Global exception handler will process validation errors

## Security
- Add `@PreAuthorize` for role-based access control
- Validate all user inputs
- Never expose internal errors to clients
- Use HTTPS in production

## API Documentation
- Add Swagger/OpenAPI annotations
- Use `@Tag` for grouping endpoints
- Use `@Operation` for endpoint descriptions
- Use `@ApiResponses` to document response codes
- Use `@Parameter` for parameter descriptions

## Testing
- Use `@WebMvcTest` for controller tests
- Mock service layer dependencies
- Test HTTP status codes and response structure
- Test validation rules
- Test security constraints
