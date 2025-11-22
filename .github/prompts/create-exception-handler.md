# Create Exception Handler

Create a global exception handler using @ControllerAdvice to handle all exceptions consistently.

## Requirements

The exception handler should:
- Use `@ControllerAdvice` annotation
- Handle custom business exceptions
- Handle Spring validation exceptions
- Handle standard HTTP exceptions
- Return consistent error response format
- Use appropriate HTTP status codes
- Log errors appropriately
- Never expose sensitive information

## Code Style
- Create custom exception classes extending RuntimeException
- Use `@ExceptionHandler` for each exception type
- Return `ResponseEntity<ErrorResponse>` or similar DTO
- Log errors with context
- Include timestamp, status, error, message, and path in response

## Example Structure

### Custom Exception Classes

```java
/**
 * Base exception for business logic errors.
 */
public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
    
    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Exception thrown when an entity is not found.
 */
public class EntityNotFoundException extends BusinessException {
    public EntityNotFoundException(String message) {
        super(message);
    }
}

/**
 * Exception thrown when attempting to create a duplicate entity.
 */
public class DuplicateEntityException extends BusinessException {
    public DuplicateEntityException(String message) {
        super(message);
    }
}

/**
 * Exception thrown for validation errors.
 */
public class ValidationException extends BusinessException {
    public ValidationException(String message) {
        super(message);
    }
}
```

### Error Response DTOs

```java
/**
 * Standard error response structure.
 */
public record ErrorResponse(
    LocalDateTime timestamp,
    int status,
    String error,
    String message,
    String path
) {}

/**
 * Validation error response with field details.
 */
public record ValidationErrorResponse(
    LocalDateTime timestamp,
    int status,
    String error,
    String message,
    String path,
    List<FieldError> fieldErrors
) {}

/**
 * Individual field error.
 */
public record FieldError(
    String field,
    String rejectedValue,
    String message
) {}
```

### Global Exception Handler

```java
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    /**
     * Handle entity not found exceptions.
     */
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleEntityNotFound(
            EntityNotFoundException ex,
            WebRequest request) {
        
        log.warn("Entity not found: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            ex.getMessage(),
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    /**
     * Handle duplicate entity exceptions.
     */
    @ExceptionHandler(DuplicateEntityException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateEntity(
            DuplicateEntityException ex,
            WebRequest request) {
        
        log.warn("Duplicate entity: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.CONFLICT.value(),
            "Conflict",
            ex.getMessage(),
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }
    
    /**
     * Handle validation errors from @Valid.
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex,
            WebRequest request) {
        
        log.warn("Validation failed: {}", ex.getMessage());
        
        List<FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new FieldError(
                error.getField(),
                error.getRejectedValue() != null ? error.getRejectedValue().toString() : "null",
                error.getDefaultMessage()
            ))
            .toList();
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Invalid input data",
            getPath(request),
            fieldErrors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
    
    /**
     * Handle constraint violation exceptions.
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ValidationErrorResponse> handleConstraintViolation(
            ConstraintViolationException ex,
            WebRequest request) {
        
        log.warn("Constraint violation: {}", ex.getMessage());
        
        List<FieldError> fieldErrors = ex.getConstraintViolations()
            .stream()
            .map(violation -> new FieldError(
                violation.getPropertyPath().toString(),
                violation.getInvalidValue() != null ? violation.getInvalidValue().toString() : "null",
                violation.getMessage()
            ))
            .toList();
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Constraint violation",
            getPath(request),
            fieldErrors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
    
    /**
     * Handle missing request parameters.
     */
    @ExceptionHandler(MissingServletRequestParameterException.class)
    public ResponseEntity<ErrorResponse> handleMissingParams(
            MissingServletRequestParameterException ex,
            WebRequest request) {
        
        log.warn("Missing parameter: {}", ex.getParameterName());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Bad Request",
            "Required parameter '" + ex.getParameterName() + "' is missing",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
    
    /**
     * Handle HTTP message not readable (malformed JSON).
     */
    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResponse> handleHttpMessageNotReadable(
            HttpMessageNotReadableException ex,
            WebRequest request) {
        
        log.warn("Malformed JSON request: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Bad Request",
            "Malformed JSON request",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }
    
    /**
     * Handle authentication exceptions.
     */
    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthenticationException(
            AuthenticationException ex,
            WebRequest request) {
        
        log.warn("Authentication failed: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            "Authentication failed",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }
    
    /**
     * Handle access denied exceptions.
     */
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException ex,
            WebRequest request) {
        
        log.warn("Access denied: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.FORBIDDEN.value(),
            "Forbidden",
            "You don't have permission to access this resource",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }
    
    /**
     * Handle data integrity violations (e.g., foreign key constraints).
     */
    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorResponse> handleDataIntegrityViolation(
            DataIntegrityViolationException ex,
            WebRequest request) {
        
        log.error("Data integrity violation: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.CONFLICT.value(),
            "Conflict",
            "Database constraint violation",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }
    
    /**
     * Handle all other exceptions.
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAllExceptions(
            Exception ex,
            WebRequest request) {
        
        log.error("Unexpected error occurred", ex);
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal Server Error",
            "An unexpected error occurred",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
    
    /**
     * Extract request path from WebRequest.
     */
    private String getPath(WebRequest request) {
        return ((ServletWebRequest) request).getRequest().getRequestURI();
    }
}
```

## HTTP Status Codes for Exceptions
- **400 Bad Request** - Validation errors, malformed requests
- **401 Unauthorized** - Authentication required/failed
- **403 Forbidden** - Insufficient permissions
- **404 Not Found** - Resource doesn't exist
- **409 Conflict** - Duplicate resource, constraint violation
- **500 Internal Server Error** - Unexpected errors

## Best Practices
- Never expose stack traces or sensitive information in error responses
- Log errors appropriately (WARN for expected errors, ERROR for unexpected)
- Return consistent error response format across all endpoints
- Include helpful error messages for clients
- Use custom exception classes for different error scenarios
- Handle Spring's built-in exceptions (validation, authentication, etc.)
- Consider internationalization (i18n) for error messages
- Include correlation IDs for tracking errors across services

## Security Considerations
- Don't expose internal implementation details
- Don't leak database information (table names, column names)
- Sanitize error messages to prevent information disclosure
- Log full details internally, but return generic messages externally
- Be careful with validation error messages (don't reveal system information)

## Testing
- Test exception handling in integration tests
- Verify correct HTTP status codes
- Verify error response structure
- Test security-related exceptions
