# Create Schema Validation

Implement data validation and schema enforcement for Spring Boot applications. Ensure data quality through Bean Validation (JSR-380), custom validators, JSON Schema validation, and database constraints.

## Requirements

Your schema validation implementation should include:

- **Bean Validation**: Use JSR-380 annotations for field-level validation
- **Custom Validators**: Implement domain-specific validation rules
- **JSON Schema Validation**: Validate JSON payloads against schemas
- **Database Constraints**: Enforce data integrity at database level
- **Cross-Field Validation**: Validate relationships between fields
- **Conditional Validation**: Apply rules based on context
- **Validation Groups**: Different validation rules for different scenarios
- **Error Messages**: Clear, actionable validation error messages
- **Performance**: Efficient validation without impacting latency
- **Testing**: Comprehensive validation test coverage

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot 3.x, Hibernate Validator
- **Validation**: JSR-380, JSON Schema Validator
- **Configuration**: Declarative validation with annotations

## Template Example

```java
import jakarta.validation.*;
import jakarta.validation.constraints.*;
import org.springframework.validation.annotation.Validated;
import org.springframework.validation.BindingResult;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.annotation.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.hibernate.validator.constraints.Range;
import org.hibernate.validator.constraints.CreditCardNumber;
import org.hibernate.validator.constraints.URL;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.networknt.schema.JsonSchema;
import com.networknt.schema.JsonSchemaFactory;
import com.networknt.schema.SpecVersion;
import com.networknt.schema.ValidationMessage;
import lombok.extern.slf4j.Slf4j;
import lombok.RequiredArgsConstructor;
import lombok.AllArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import lombok.NoArgsConstructor;
import lombok.Builder;
import jakarta.persistence.*;
import java.lang.annotation.*;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

/**
 * Schema validation and data quality enforcement.
 * Implements JSR-380 Bean Validation, custom validators, and JSON Schema validation.
 */
@Slf4j
public class SchemaValidationPatterns {

    /**
     * Validation configuration.
     */
    @Configuration
    public static class ValidationConfiguration {

        /**
         * Validator factory with custom constraints.
         */
        @Bean
        public Validator validator() {
            ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
            return factory.getValidator();
        }

        /**
         * ObjectMapper for JSON processing.
         */
        @Bean
        public ObjectMapper objectMapper() {
            return new ObjectMapper();
        }

        /**
         * JSON Schema factory.
         */
        @Bean
        public JsonSchemaFactory jsonSchemaFactory() {
            return JsonSchemaFactory.getInstance(SpecVersion.VersionFlag.V7);
        }
    }

    /**
     * Custom validation annotations.
     */
    @Target({ElementType.FIELD, ElementType.PARAMETER})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy = PhoneNumberValidator.class)
    @Documented
    public @interface ValidPhoneNumber {
        String message() default "Invalid phone number format";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }

    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy = DateRangeValidator.class)
    @Documented
    public @interface ValidDateRange {
        String message() default "End date must be after start date";
        String startDate();
        String endDate();
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }

    @Target({ElementType.FIELD})
    @Retention(RetentionPolicy.RUNTIME)
    @Constraint(validatedBy = StrongPasswordValidator.class)
    @Documented
    public @interface StrongPassword {
        String message() default "Password must be strong: min 8 chars, uppercase, lowercase, number, special char";
        Class<?>[] groups() default {};
        Class<? extends Payload>[] payload() default {};
    }

    /**
     * Phone number validator implementation.
     */
    public static class PhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {

        private static final String PHONE_PATTERN = "^\\+?[1-9]\\d{1,14}$";

        @Override
        public boolean isValid(String phoneNumber, ConstraintValidatorContext context) {
            if (phoneNumber == null || phoneNumber.isEmpty()) {
                return true; // Use @NotNull for null check
            }

            // Remove common separators
            String cleaned = phoneNumber.replaceAll("[\\s\\-\\(\\)]", "");

            return cleaned.matches(PHONE_PATTERN);
        }
    }

    /**
     * Date range validator for cross-field validation.
     */
    public static class DateRangeValidator implements ConstraintValidator<ValidDateRange, Object> {

        private String startDateField;
        private String endDateField;

        @Override
        public void initialize(ValidDateRange constraintAnnotation) {
            this.startDateField = constraintAnnotation.startDate();
            this.endDateField = constraintAnnotation.endDate();
        }

        @Override
        public boolean isValid(Object value, ConstraintValidatorContext context) {
            try {
                LocalDate startDate = (LocalDate) getFieldValue(value, startDateField);
                LocalDate endDate = (LocalDate) getFieldValue(value, endDateField);

                if (startDate == null || endDate == null) {
                    return true; // Use @NotNull for null checks
                }

                return !endDate.isBefore(startDate);

            } catch (Exception e) {
                return false;
            }
        }

        private Object getFieldValue(Object object, String fieldName) throws Exception {
            java.lang.reflect.Field field = object.getClass().getDeclaredField(fieldName);
            field.setAccessible(true);
            return field.get(object);
        }
    }

    /**
     * Strong password validator.
     */
    public static class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {

        @Override
        public boolean isValid(String password, ConstraintValidatorContext context) {
            if (password == null) {
                return true;
            }

            // Minimum 8 characters
            if (password.length() < 8) {
                return false;
            }

            // Must contain uppercase
            if (!password.matches(".*[A-Z].*")) {
                return false;
            }

            // Must contain lowercase
            if (!password.matches(".*[a-z].*")) {
                return false;
            }

            // Must contain digit
            if (!password.matches(".*\\d.*")) {
                return false;
            }

            // Must contain special character
            return password.matches(".*[!@#$%^&*()_+\\-=\\[\\]{};':\"\\\\|,.<>/?].*");
        }
    }

    /**
     * Validation groups for different scenarios.
     */
    public interface CreateValidation {}
    public interface UpdateValidation {}
    public interface AdminValidation {}

    /**
     * User registration request with comprehensive validation.
     */
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    @ValidDateRange(startDate = "birthDate", endDate = "memberSince",
            message = "Member since date must be after birth date")
    public static class UserRegistrationRequest {

        @NotBlank(message = "Username is required", groups = CreateValidation.class)
        @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
        @Pattern(regexp = "^[a-zA-Z0-9_]+$",
                message = "Username can only contain letters, numbers, and underscores")
        private String username;

        @NotBlank(message = "Email is required")
        @Email(message = "Email must be valid")
        private String email;

        @NotBlank(message = "Password is required", groups = CreateValidation.class)
        @StrongPassword
        private String password;

        @NotBlank(message = "First name is required")
        @Size(min = 1, max = 100, message = "First name must be between 1 and 100 characters")
        private String firstName;

        @NotBlank(message = "Last name is required")
        @Size(min = 1, max = 100, message = "Last name must be between 1 and 100 characters")
        private String lastName;

        @ValidPhoneNumber
        private String phoneNumber;

        @NotNull(message = "Birth date is required")
        @Past(message = "Birth date must be in the past")
        private LocalDate birthDate;

        @NotNull(message = "Member since date is required")
        @PastOrPresent(message = "Member since date cannot be in the future")
        private LocalDate memberSince;

        @Min(value = 18, message = "Age must be at least 18")
        @Max(value = 120, message = "Age must be less than 120")
        private Integer age;

        @DecimalMin(value = "0.0", inclusive = false, message = "Account balance must be positive")
        @Digits(integer = 10, fraction = 2, message = "Invalid account balance format")
        private Double accountBalance;

        @NotNull(message = "Account type is required")
        private AccountType accountType;

        @URL(message = "Website must be a valid URL")
        private String website;

        @Size(max = 500, message = "Bio cannot exceed 500 characters")
        private String bio;

        @AssertTrue(message = "Must accept terms and conditions", groups = CreateValidation.class)
        private Boolean acceptedTerms;
    }

    /**
     * Product entity with validation constraints.
     */
    @Entity
    @Table(name = "products")
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class Product {

        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @NotBlank(message = "Product name is required")
        @Size(min = 3, max = 200, message = "Product name must be between 3 and 200 characters")
        @Column(nullable = false, length = 200)
        private String name;

        @NotBlank(message = "SKU is required")
        @Pattern(regexp = "^[A-Z0-9]{8,12}$", message = "SKU must be 8-12 uppercase alphanumeric characters")
        @Column(nullable = false, unique = true, length = 12)
        private String sku;

        @NotNull(message = "Price is required")
        @DecimalMin(value = "0.01", message = "Price must be greater than 0")
        @Digits(integer = 8, fraction = 2, message = "Price must have max 8 digits and 2 decimals")
        @Column(nullable = false, precision = 10, scale = 2)
        private Double price;

        @NotNull(message = "Stock quantity is required")
        @Min(value = 0, message = "Stock cannot be negative")
        @Column(nullable = false)
        private Integer stock;

        @Size(max = 2000, message = "Description cannot exceed 2000 characters")
        @Column(length = 2000)
        private String description;

        @NotNull(message = "Category is required")
        @Column(nullable = false, length = 50)
        private String category;

        @Range(min = 0, max = 100, message = "Discount must be between 0 and 100")
        @Column
        private Integer discountPercentage;

        @NotNull(message = "Active status is required")
        @Column(nullable = false)
        private Boolean active;

        @PastOrPresent(message = "Created date cannot be in the future")
        @Column(nullable = false, updatable = false)
        private LocalDateTime createdAt;

        @PastOrPresent(message = "Updated date cannot be in the future")
        @Column
        private LocalDateTime updatedAt;

        @PrePersist
        protected void onCreate() {
            createdAt = LocalDateTime.now();
            if (active == null) {
                active = true;
            }
        }

        @PreUpdate
        protected void onUpdate() {
            updatedAt = LocalDateTime.now();
        }
    }

    /**
     * Order request with conditional validation.
     */
    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class OrderRequest {

        @NotNull(message = "Customer ID is required")
        @Positive(message = "Customer ID must be positive")
        private Long customerId;

        @NotEmpty(message = "Order must contain at least one item")
        @Size(max = 100, message = "Order cannot contain more than 100 items")
        @Valid
        private List<OrderItem> items;

        @NotNull(message = "Payment method is required")
        private PaymentMethod paymentMethod;

        @NotBlank(message = "Shipping address is required")
        @Size(min = 10, max = 500, message = "Shipping address must be between 10 and 500 characters")
        private String shippingAddress;

        @CreditCardNumber(message = "Invalid credit card number", groups = AdminValidation.class)
        private String creditCardNumber;

        @Size(min = 3, max = 4, message = "CVV must be 3 or 4 digits")
        @Pattern(regexp = "^\\d{3,4}$", message = "CVV must contain only digits")
        private String cvv;

        @Future(message = "Delivery date must be in the future")
        private LocalDate requestedDeliveryDate;

        @Size(max = 1000, message = "Notes cannot exceed 1000 characters")
        private String notes;
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class OrderItem {

        @NotNull(message = "Product ID is required")
        @Positive(message = "Product ID must be positive")
        private Long productId;

        @NotNull(message = "Quantity is required")
        @Min(value = 1, message = "Quantity must be at least 1")
        @Max(value = 1000, message = "Quantity cannot exceed 1000")
        private Integer quantity;

        @NotNull(message = "Unit price is required")
        @DecimalMin(value = "0.01", message = "Unit price must be greater than 0")
        private Double unitPrice;
    }

    /**
     * Validation service with custom business rules.
     */
    @org.springframework.stereotype.Service
    @Slf4j
    @RequiredArgsConstructor
    public static class ValidationService {
        private final Validator validator;
        private final ProductRepository productRepository;

        /**
         * Validate object with custom groups.
         */
        public <T> ValidationResult validate(T object, Class<?>... groups) {
            Set<ConstraintViolation<T>> violations = validator.validate(object, groups);

            if (violations.isEmpty()) {
                return ValidationResult.success();
            }

            List<String> errors = violations.stream()
                    .map(v -> v.getPropertyPath() + ": " + v.getMessage())
                    .collect(Collectors.toList());

            log.warn("Validation failed: {}", errors);
            return ValidationResult.failure(errors);
        }

        /**
         * Validate order request with business rules.
         */
        public ValidationResult validateOrder(OrderRequest request) {
            // Standard validation
            ValidationResult result = validate(request);
            if (!result.isValid()) {
                return result;
            }

            List<String> businessErrors = new ArrayList<>();

            // Business rule: Check product availability
            for (OrderItem item : request.getItems()) {
                Optional<Product> product = productRepository.findById(item.getProductId());

                if (product.isEmpty()) {
                    businessErrors.add("Product not found: " + item.getProductId());
                    continue;
                }

                if (!product.get().getActive()) {
                    businessErrors.add("Product is not active: " + item.getProductId());
                }

                if (product.get().getStock() < item.getQuantity()) {
                    businessErrors.add("Insufficient stock for product: " + item.getProductId() +
                            " (available: " + product.get().getStock() + ", requested: " + item.getQuantity() + ")");
                }

                // Validate price matches current price
                if (!product.get().getPrice().equals(item.getUnitPrice())) {
                    businessErrors.add("Price mismatch for product: " + item.getProductId() +
                            " (expected: " + product.get().getPrice() + ", provided: " + item.getUnitPrice() + ")");
                }
            }

            // Business rule: Minimum order value
            double totalAmount = request.getItems().stream()
                    .mapToDouble(item -> item.getQuantity() * item.getUnitPrice())
                    .sum();

            if (totalAmount < 10.0) {
                businessErrors.add("Minimum order value is $10.00");
            }

            if (!businessErrors.isEmpty()) {
                return ValidationResult.failure(businessErrors);
            }

            return ValidationResult.success();
        }

        /**
         * Validate product with conditional rules.
         */
        public ValidationResult validateProduct(Product product) {
            ValidationResult result = validate(product);
            if (!result.isValid()) {
                return result;
            }

            List<String> errors = new ArrayList<>();

            // Business rule: Discount cannot be applied without price
            if (product.getDiscountPercentage() != null &&
                    product.getDiscountPercentage() > 0 &&
                    (product.getPrice() == null || product.getPrice() <= 0)) {
                errors.add("Cannot apply discount without valid price");
            }

            // Business rule: Active products must have stock
            if (Boolean.TRUE.equals(product.getActive()) &&
                    (product.getStock() == null || product.getStock() == 0)) {
                errors.add("Active products must have stock available");
            }

            if (!errors.isEmpty()) {
                return ValidationResult.failure(errors);
            }

            return ValidationResult.success();
        }
    }

    /**
     * JSON Schema validation service.
     */
    @org.springframework.stereotype.Service
    @Slf4j
    @RequiredArgsConstructor
    public static class JsonSchemaValidationService {
        private final JsonSchemaFactory schemaFactory;
        private final ObjectMapper objectMapper;

        /**
         * Validate JSON against schema.
         */
        public ValidationResult validateJson(String jsonData, String schemaDefinition) {
            try {
                JsonNode jsonNode = objectMapper.readTree(jsonData);
                JsonSchema schema = schemaFactory.getSchema(schemaDefinition);

                Set<ValidationMessage> validationMessages = schema.validate(jsonNode);

                if (validationMessages.isEmpty()) {
                    return ValidationResult.success();
                }

                List<String> errors = validationMessages.stream()
                        .map(ValidationMessage::getMessage)
                        .collect(Collectors.toList());

                return ValidationResult.failure(errors);

            } catch (Exception e) {
                log.error("JSON schema validation error", e);
                return ValidationResult.failure(List.of("JSON validation failed: " + e.getMessage()));
            }
        }

        /**
         * Validate payment request against schema.
         */
        public ValidationResult validatePaymentRequest(String paymentJson) {
            String schema = """
                {
                  "$schema": "http://json-schema.org/draft-07/schema#",
                  "type": "object",
                  "required": ["amount", "currency", "paymentMethod"],
                  "properties": {
                    "amount": {
                      "type": "number",
                      "minimum": 0.01,
                      "maximum": 1000000
                    },
                    "currency": {
                      "type": "string",
                      "enum": ["USD", "EUR", "GBP"]
                    },
                    "paymentMethod": {
                      "type": "string",
                      "enum": ["CREDIT_CARD", "DEBIT_CARD", "PAYPAL", "BANK_TRANSFER"]
                    },
                    "cardNumber": {
                      "type": "string",
                      "pattern": "^[0-9]{13,19}$"
                    }
                  }
                }
                """;

            return validateJson(paymentJson, schema);
        }
    }

    /**
     * REST controller with validation.
     */
    @RestController
    @RequestMapping("/api/v1")
    @Slf4j
    @RequiredArgsConstructor
    @Validated
    public static class ValidatedApiController {
        private final ValidationService validationService;
        private final ProductRepository productRepository;

        /**
         * Register user with validation.
         */
        @PostMapping("/users/register")
        public ResponseEntity<?> registerUser(
                @Validated(CreateValidation.class) @RequestBody UserRegistrationRequest request) {

            log.info("User registration request: {}", request.getUsername());
            return ResponseEntity.ok(Map.of("message", "User registered successfully"));
        }

        /**
         * Create product with validation.
         */
        @PostMapping("/products")
        public ResponseEntity<?> createProduct(@Valid @RequestBody Product product) {
            ValidationResult validationResult = validationService.validateProduct(product);

            if (!validationResult.isValid()) {
                return ResponseEntity.badRequest().body(validationResult);
            }

            Product saved = productRepository.save(product);
            log.info("Product created: {}", saved.getId());
            return ResponseEntity.ok(saved);
        }

        /**
         * Create order with complex validation.
         */
        @PostMapping("/orders")
        public ResponseEntity<?> createOrder(@Valid @RequestBody OrderRequest request) {
            ValidationResult validationResult = validationService.validateOrder(request);

            if (!validationResult.isValid()) {
                return ResponseEntity.badRequest().body(validationResult);
            }

            log.info("Order created for customer: {}", request.getCustomerId());
            return ResponseEntity.ok(Map.of("message", "Order created successfully"));
        }

        /**
         * Path variable validation.
         */
        @GetMapping("/products/{id}")
        public ResponseEntity<Product> getProduct(
                @PathVariable @Min(1) @Max(Long.MAX_VALUE) Long id) {

            Product product = productRepository.findById(id)
                    .orElseThrow(() -> new RuntimeException("Product not found"));
            return ResponseEntity.ok(product);
        }

        /**
         * Request param validation.
         */
        @GetMapping("/products/search")
        public ResponseEntity<?> searchProducts(
                @RequestParam @NotBlank @Size(min = 2, max = 100) String query,
                @RequestParam(defaultValue = "0") @Min(0) Integer page,
                @RequestParam(defaultValue = "20") @Min(1) @Max(100) Integer size) {

            log.info("Searching products: query={}, page={}, size={}", query, page, size);
            return ResponseEntity.ok(List.of());
        }
    }

    /**
     * Global validation exception handler.
     */
    @org.springframework.web.bind.annotation.RestControllerAdvice
    @Slf4j
    public static class ValidationExceptionHandler {

        @org.springframework.web.bind.annotation.ExceptionHandler(MethodArgumentNotValidException.class)
        public ResponseEntity<ValidationErrorResponse> handleValidationException(
                MethodArgumentNotValidException ex) {

            List<String> errors = ex.getBindingResult().getFieldErrors().stream()
                    .map(error -> error.getField() + ": " + error.getDefaultMessage())
                    .collect(Collectors.toList());

            log.warn("Validation failed: {}", errors);

            return ResponseEntity
                    .status(HttpStatus.BAD_REQUEST)
                    .body(ValidationErrorResponse.builder()
                            .message("Validation failed")
                            .errors(errors)
                            .timestamp(LocalDateTime.now())
                            .build());
        }

        @org.springframework.web.bind.annotation.ExceptionHandler(ConstraintViolationException.class)
        public ResponseEntity<ValidationErrorResponse> handleConstraintViolation(
                ConstraintViolationException ex) {

            List<String> errors = ex.getConstraintViolations().stream()
                    .map(violation -> violation.getPropertyPath() + ": " + violation.getMessage())
                    .collect(Collectors.toList());

            log.warn("Constraint violation: {}", errors);

            return ResponseEntity
                    .status(HttpStatus.BAD_REQUEST)
                    .body(ValidationErrorResponse.builder()
                            .message("Validation constraint violation")
                            .errors(errors)
                            .timestamp(LocalDateTime.now())
                            .build());
        }
    }

    /**
     * Domain models and enums.
     */
    public enum AccountType {
        FREE, PREMIUM, ENTERPRISE
    }

    public enum PaymentMethod {
        CREDIT_CARD, DEBIT_CARD, PAYPAL, BANK_TRANSFER
    }

    public interface ProductRepository extends org.springframework.data.jpa.repository.JpaRepository<Product, Long> {
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class ValidationResult {
        private boolean valid;
        private List<String> errors;

        public static ValidationResult success() {
            return new ValidationResult(true, List.of());
        }

        public static ValidationResult failure(List<String> errors) {
            return new ValidationResult(false, errors);
        }
    }

    @Getter
    @Setter
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class ValidationErrorResponse {
        private String message;
        private List<String> errors;
        private LocalDateTime timestamp;
    }
}
```

## Configuration Examples

```yaml
# application.yml

spring:
  # JPA validation
  jpa:
    properties:
      hibernate:
        validator:
          apply_to_ddl: true
          autoregister_listeners: true
        check_nullability: true

# Validation messages
  messages:
    basename: validation-messages
    encoding: UTF-8

# Jackson validation
  jackson:
    default-property-inclusion: non_null
    deserialization:
      fail-on-unknown-properties: true

# Database constraints
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

## Best Practices

1. **Use Appropriate Validation Annotations**
   - @NotNull: Field cannot be null
   - @NotBlank: String cannot be null or empty
   - @Size: Validate string/collection size
   - @Min/@Max: Numeric range validation
   - @Pattern: Regex pattern matching
   - @Email: Email format validation

2. **Implement Custom Validators**
   - Domain-specific validation logic
   - Cross-field validation
   - Business rule enforcement
   - Reusable validation components

3. **Use Validation Groups**
   - Different rules for create vs update
   - Conditional validation
   - Role-based validation
   - Scenario-specific rules

4. **Provide Clear Error Messages**
   - User-friendly messages
   - Actionable feedback
   - Include field names
   - Suggest corrections

5. **Validate at Multiple Layers**
   - Controller: Input validation
   - Service: Business logic validation
   - Entity: Data integrity validation
   - Database: Constraint enforcement

6. **Handle Validation Failures Gracefully**
   - Return 400 Bad Request
   - Include all validation errors
   - Don't expose internal details
   - Log validation failures

7. **Test Validation Thoroughly**
   - Test valid inputs
   - Test invalid inputs
   - Test boundary conditions
   - Test custom validators

8. **Consider Performance**
   - Validate early (fail fast)
   - Use efficient validators
   - Cache validation results
   - Avoid expensive validation in loops

## Related Documentation

- [Bean Validation (JSR-380)](https://beanvalidation.org/2.0/spec/)
- [Hibernate Validator](https://hibernate.org/validator/)
- [Spring Validation](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#validation)
- [JSON Schema](https://json-schema.org/)
