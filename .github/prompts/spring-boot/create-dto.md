# Create DTO (Data Transfer Object)

Create request and response DTOs for API endpoints using Java 17 records.

## Requirements

### Request DTO
- Use Java 17 `record` for immutability
- Add validation annotations from `jakarta.validation.constraints`
- Name: `[Entity]RequestDto`
- Include only fields that clients should send
- Add custom validation messages
- Never include ID or audit fields

### Response DTO
- Use Java 17 `record` for immutability
- Name: `[Entity]ResponseDto`
- Include ID and audit fields
- Exclude sensitive data (passwords, tokens)
- Consider creating multiple response DTOs for different use cases (summary, detail)

## Code Style
- Use records (Java 17+)
- Place validation annotations on record parameters
- Add JavaDoc for the record
- Group related validation annotations
- Use meaningful validation messages

## Example Structure

```java
/**
 * Request DTO for creating/updating an entity.
 */
public record EntityRequestDto(
    @NotBlank(message = "Field is required")
    @Size(min = 3, max = 50, message = "Field must be between 3 and 50 characters")
    String field1,
    
    @NotNull(message = "Field is required")
    @Email(message = "Invalid email format")
    String email,
    
    @NotBlank(message = "Field is required")
    @Size(min = 8, message = "Field must be at least 8 characters")
    String password
) {}

/**
 * Response DTO for entity data.
 */
public record EntityResponseDto(
    Long id,
    String field1,
    String email,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}

/**
 * Summary DTO for list views.
 */
public record EntitySummaryDto(
    Long id,
    String field1,
    String email
) {}
```

## Validation Annotations Reference
- `@NotNull` - Field cannot be null
- `@NotBlank` - String cannot be null, empty, or whitespace
- `@NotEmpty` - Collection/array cannot be null or empty
- `@Size(min, max)` - String/collection size constraints
- `@Min(value)` / `@Max(value)` - Numeric constraints
- `@Email` - Valid email format
- `@Pattern(regexp)` - Regex validation
- `@Past` / `@Future` - Date/time constraints

## Best Practices
- Never expose entity classes directly in APIs
- Create separate DTOs for different operations (create, update, response)
- Exclude sensitive fields from response DTOs
- Use appropriate validation for each field
- Consider using @JsonProperty for custom JSON field names
