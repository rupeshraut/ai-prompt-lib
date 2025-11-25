# Create Data Masking

Implement data masking to hide sensitive information in non-production environments and API responses. Display masked or redacted versions of PII while maintaining data utility for testing and analysis.

## Requirements

Your data masking implementation should include:

- **Masking Strategies**: Multiple masking approaches (redaction, tokenization, substitution, shuffling)
- **PII Detection**: Automatic detection of personally identifiable information
- **Annotation-Based Masking**: Mark fields for automatic masking
- **Conditional Masking**: Mask based on user role or data context
- **API Response Masking**: Hide sensitive fields in JSON responses
- **Logging Masking**: Prevent sensitive data in logs
- **Database View Masking**: Masked views for non-admin users
- **Performance Optimization**: Minimal masking overhead
- **Configuration**: Centralized masking rules
- **Audit Trail**: Track masking decisions

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot, Spring Data JPA
- **Serialization**: Jackson annotations and filters
- **AOP**: Aspect-oriented programming for masking

## Template Example

```java
import com.fasterxml.jackson.databind.JsonSerializer;
import com.fasterxml.jackson.databind.SerializerProvider;
import com.fasterxml.jackson.core.JsonGenerator;
import com.fasterxml.jackson.databind.annotation.JsonSerialize;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.ser.FilterProvider;
import com.fasterxml.jackson.databind.ser.impl.SimpleBeanPropertyFilter;
import com.fasterxml.jackson.databind.ser.impl.SimpleFilterProvider;
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ResponseBody;
import org.springframework.http.converter.json.MappingJacksonValue;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.Authentication;
import lombok.extern.slf4j.Slf4j;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import jakarta.persistence.*;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;
import java.util.regex.Pattern;
import java.io.IOException;

/**
 * Data masking for hiding sensitive information in responses and logs.
 * Provides multiple masking strategies based on data type and context.
 */
@Slf4j
public class DataMasking {

    /**
     * Annotation to mark fields for automatic masking.
     */
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Masked {
        /**
         * Masking strategy to apply.
         */
        MaskingStrategy strategy() default MaskingStrategy.REDACT;

        /**
         * Roles that can view unmasked data.
         */
        String[] allowedRoles() default {"ADMIN"};

        /**
         * Custom mask pattern (for CUSTOM strategy).
         */
        String pattern() default "";
    }

    /**
     * Masking strategies.
     */
    public enum MaskingStrategy {
        REDACT,          // Show as X's
        HASH,            // Show first N and last N
        SUBSTITUTE,      // Replace with placeholder
        SHUFFLE,         // Shuffle characters
        TOKENIZE,        // Replace with token
        PARTIAL_EMAIL,   // user***@example.com
        PARTIAL_PHONE,   // (XXX) XXX-1234
        PARTIAL_SSN,     // XXX-XX-1234
        NULL,            // Return null
        CUSTOM           // Custom regex pattern
    }

    /**
     * Data masking service with multiple masking strategies.
     */
    @Service
    @Slf4j
    public static class MaskingService {

        /**
         * Mask value based on strategy.
         */
        public String mask(String value, MaskingStrategy strategy) {
            if (value == null) {
                return null;
            }

            return switch (strategy) {
                case REDACT -> redact(value);
                case HASH -> partialHash(value);
                case SUBSTITUTE -> substitute(value);
                case SHUFFLE -> shuffle(value);
                case TOKENIZE -> tokenize(value);
                case PARTIAL_EMAIL -> maskEmail(value);
                case PARTIAL_PHONE -> maskPhone(value);
                case PARTIAL_SSN -> maskSSN(value);
                case NULL -> null;
                default -> value;
            };
        }

        /**
         * Redact: Replace all characters with X's.
         */
        private String redact(String value) {
            return value.replaceAll(".", "X");
        }

        /**
         * Partial hash: Show first 2 and last 4 characters.
         */
        private String partialHash(String value) {
            if (value.length() <= 6) {
                return redact(value);
            }
            String first = value.substring(0, 2);
            String last = value.substring(value.length() - 4);
            return first + "***" + last;
        }

        /**
         * Substitute: Replace with placeholder.
         */
        private String substitute(String value) {
            return "[REDACTED]";
        }

        /**
         * Shuffle: Randomly shuffle characters.
         */
        private String shuffle(String value) {
            char[] chars = value.toCharArray();
            java.util.List<Character> charList = new java.util.ArrayList<>();
            for (char c : chars) {
                charList.add(c);
            }
            java.util.Collections.shuffle(charList);
            StringBuilder result = new StringBuilder();
            for (char c : charList) {
                result.append(c);
            }
            return result.toString();
        }

        /**
         * Tokenize: Replace with hash token.
         */
        private String tokenize(String value) {
            return "tok_" + Integer.toHexString(value.hashCode());
        }

        /**
         * Mask email: user***@example.com
         */
        private String maskEmail(String email) {
            if (!email.contains("@")) {
                return partialHash(email);
            }
            int atIndex = email.indexOf('@');
            String localPart = email.substring(0, Math.min(2, atIndex));
            String domain = email.substring(atIndex);
            return localPart + "***" + domain;
        }

        /**
         * Mask phone: (XXX) XXX-1234
         */
        private String maskPhone(String phone) {
            String digits = phone.replaceAll("[^0-9]", "");
            if (digits.length() < 4) {
                return redact(phone);
            }
            String lastFour = digits.substring(digits.length() - 4);
            return "(XXX) XXX-" + lastFour;
        }

        /**
         * Mask SSN: XXX-XX-1234
         */
        private String maskSSN(String ssn) {
            String digits = ssn.replaceAll("[^0-9]", "");
            if (digits.length() < 4) {
                return redact(ssn);
            }
            String lastFour = digits.substring(digits.length() - 4);
            return "XXX-XX-" + lastFour;
        }

        /**
         * Determine if user has permission to view unmasked data.
         */
        public boolean canViewUnmasked(String[] allowedRoles) {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            if (auth == null) {
                return false;
            }

            for (String role : allowedRoles) {
                if (auth.getAuthorities().stream()
                        .anyMatch(a -> a.getAuthority().equals(role))) {
                    return true;
                }
            }

            return false;
        }
    }

    /**
     * Customer entity with masked fields.
     */
    @Entity
    @Table(name = "customers")
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class Customer {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String name;

        // Email is masked in responses by default
        @Column(nullable = false)
        @Masked(strategy = MaskingStrategy.PARTIAL_EMAIL, allowedRoles = {"ADMIN", "SUPPORT"})
        @JsonSerialize(using = MaskedEmailSerializer.class)
        private String email;

        // Phone is masked with partial visibility
        @Column
        @Masked(strategy = MaskingStrategy.PARTIAL_PHONE, allowedRoles = {"ADMIN"})
        @JsonSerialize(using = MaskedPhoneSerializer.class)
        private String phone;

        // SSN is highly redacted
        @Column
        @Masked(strategy = MaskingStrategy.PARTIAL_SSN, allowedRoles = {"ADMIN"})
        @JsonSerialize(using = MaskedSSNSerializer.class)
        private String socialSecurityNumber;

        @Column(nullable = false)
        private java.time.LocalDateTime createdAt = java.time.LocalDateTime.now();
    }

    /**
     * Custom Jackson serializers for field masking.
     */
    public static class MaskedEmailSerializer extends JsonSerializer<String> {
        @Override
        public void serialize(String value, JsonGenerator gen,
                            SerializerProvider provider) throws IOException {
            if (value == null) {
                gen.writeNull();
                return;
            }

            MaskingService maskingService = new MaskingService();
            boolean canView = maskingService.canViewUnmasked(new String[]{"ADMIN", "SUPPORT"});

            if (canView) {
                gen.writeString(value);
            } else {
                String masked = maskingService.mask(value, MaskingStrategy.PARTIAL_EMAIL);
                gen.writeString(masked);
            }
        }
    }

    public static class MaskedPhoneSerializer extends JsonSerializer<String> {
        @Override
        public void serialize(String value, JsonGenerator gen,
                            SerializerProvider provider) throws IOException {
            if (value == null) {
                gen.writeNull();
                return;
            }

            MaskingService maskingService = new MaskingService();
            boolean canView = maskingService.canViewUnmasked(new String[]{"ADMIN"});

            if (canView) {
                gen.writeString(value);
            } else {
                String masked = maskingService.mask(value, MaskingStrategy.PARTIAL_PHONE);
                gen.writeString(masked);
            }
        }
    }

    public static class MaskedSSNSerializer extends JsonSerializer<String> {
        @Override
        public void serialize(String value, JsonGenerator gen,
                            SerializerProvider provider) throws IOException {
            if (value == null) {
                gen.writeNull();
                return;
            }

            MaskingService maskingService = new MaskingService();
            boolean canView = maskingService.canViewUnmasked(new String[]{"ADMIN"});

            if (canView) {
                gen.writeString(value);
            } else {
                String masked = maskingService.mask(value, MaskingStrategy.PARTIAL_SSN);
                gen.writeString(masked);
            }
        }
    }

    /**
     * DTO for API responses with masking applied.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class CustomerDTO {
        private Long id;
        private String name;

        @JsonSerialize(using = MaskedEmailSerializer.class)
        private String email;

        @JsonSerialize(using = MaskedPhoneSerializer.class)
        private String phone;

        @JsonSerialize(using = MaskedSSNSerializer.class)
        private String ssn;

        /**
         * Create DTO from entity with masking applied.
         */
        public static CustomerDTO fromEntity(Customer customer) {
            CustomerDTO dto = new CustomerDTO();
            dto.setId(customer.getId());
            dto.setName(customer.getName());
            dto.setEmail(customer.getEmail());
            dto.setPhone(customer.getPhone());
            dto.setSsn(customer.getSocialSecurityNumber());
            return dto;
        }
    }

    /**
     * Controller advice for automatic response masking.
     */
    @ControllerAdvice
    @Slf4j
    public static class MaskingControllerAdvice {

        /**
         * Wrap response with masking filters.
         */
        @org.springframework.web.bind.annotation.PostMapping
        @ResponseBody
        public MappingJacksonValue maskResponse(Object responseBody) {
            MappingJacksonValue mappingJacksonValue = new MappingJacksonValue(responseBody);

            SimpleBeanPropertyFilter filter = SimpleBeanPropertyFilter
                    .filterOutAllExcept("id", "name", "email", "phone");

            FilterProvider filterProvider = new SimpleFilterProvider()
                    .addFilter("maskingFilter", filter);

            mappingJacksonValue.setFilters(filterProvider);
            return mappingJacksonValue;
        }
    }

    /**
     * PII detector for automatic field detection.
     */
    @Component
    @Slf4j
    public static class PiiDetector {
        private static final Pattern EMAIL_PATTERN =
                Pattern.compile("[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}");

        private static final Pattern PHONE_PATTERN =
                Pattern.compile("^[+]?[(]?[0-9]{3}[)]?[-\\s.]?[0-9]{3}[-\\s.]?[0-9]{4,6}$");

        private static final Pattern SSN_PATTERN =
                Pattern.compile("\\b\\d{3}-\\d{2}-\\d{4}\\b");

        private static final Pattern CREDIT_CARD_PATTERN =
                Pattern.compile("\\b\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}[\\s-]?\\d{4}\\b");

        /**
         * Detect PII in text.
         */
        public PiiDetectionResult detectPii(String text) {
            PiiDetectionResult result = new PiiDetectionResult();

            if (EMAIL_PATTERN.matcher(text).find()) {
                result.containsEmail = true;
            }
            if (PHONE_PATTERN.matcher(text).find()) {
                result.containsPhone = true;
            }
            if (SSN_PATTERN.matcher(text).find()) {
                result.containsSSN = true;
            }
            if (CREDIT_CARD_PATTERN.matcher(text).find()) {
                result.containsCreditCard = true;
            }

            return result;
        }

        /**
         * Mask detected PII in text.
         */
        public String maskDetectedPii(String text) {
            String masked = text;

            // Mask emails
            masked = masked.replaceAll(EMAIL_PATTERN.pattern(), "[MASKED_EMAIL]");

            // Mask credit cards
            masked = masked.replaceAll(CREDIT_CARD_PATTERN.pattern(), "[MASKED_CARD]");

            // Mask SSNs
            masked = masked.replaceAll(SSN_PATTERN.pattern(), "[MASKED_SSN]");

            // Mask phones
            masked = masked.replaceAll(PHONE_PATTERN.pattern(), "[MASKED_PHONE]");

            return masked;
        }
    }

    /**
     * PII detection result.
     */
    @Getter
    @Setter
    public static class PiiDetectionResult {
        public boolean containsEmail;
        public boolean containsPhone;
        public boolean containsSSN;
        public boolean containsCreditCard;

        public boolean hasPii() {
            return containsEmail || containsPhone || containsSSN || containsCreditCard;
        }
    }

    /**
     * Logback appender for masking sensitive information in logs.
     */
    @Slf4j
    public static class MaskingLogAppender extends ch.qos.logback.core.AppenderBase<ch.qos.logback.classic.spi.ILoggingEvent> {
        private PiiDetector piiDetector = new PiiDetector();

        @Override
        protected void append(ch.qos.logback.classic.spi.ILoggingEvent event) {
            String message = event.getMessage();
            if (message != null) {
                String maskedMessage = piiDetector.maskDetectedPii(message);
                // Log masked message
                log.debug("Original message contained PII, masked before logging");
            }
        }
    }

    /**
     * Database view with masking for non-admin users.
     */
    public static class MaskedCustomerView {
        /**
         * SQL view that masks sensitive columns.
         */
        public static final String CREATE_VIEW_SQL = """
            CREATE VIEW customer_view_masked AS
            SELECT
                id,
                name,
                CONCAT(SUBSTRING(email, 1, 2), '***', SUBSTRING(email, POSITION('@' IN email))) AS email,
                CONCAT('(XXX) XXX-', SUBSTRING(phone, -4)) AS phone,
                'XXX-XX-****' AS ssn
            FROM customers;
        """;
    }

    /**
     * Best practices for data masking.
     */
    public static class BestPractices {
        /**
         * 1. Mask at multiple layers
         *    - Database views for restricted users
         *    - API responses for external calls
         *    - Logs for audit trails
         */

        /**
         * 2. Use appropriate masking strategy
         *    - Emails: Partial hash (first 2 chars + last part)
         *    - Phones: Last 4 digits visible
         *    - SSN: Last 4 digits visible
         *    - Credit cards: Last 4 digits visible
         */

        /**
         * 3. Role-based masking
         *    - Admin: Full unmasked access
         *    - Support: Partial masking
         *    - Customer: Fully masked (their own only)
         */

        /**
         * 4. Performance considerations
         *    - Cache masking results
         *    - Use view-level masking for large datasets
         *    - Avoid re-masking on every response
         */

        /**
         * 5. Audit masking decisions
         *    - Log who accessed unmasked data
         *    - Track masking rule changes
         *    - Regular masking audits
         */

        /**
         * 6. Test masking thoroughly
         *    - Verify masking is applied
         *    - Test with different user roles
         *    - Verify no sensitive data leaks
         */

        /**
         * 7. PII detection
         *    - Automatically detect PII types
         *    - Apply appropriate masking
         *    - Log detected PII for security
         */

        /**
         * 8. Maintain data utility
         *    - Balance masking with usability
         *    - Keep identifying information visible
         *    - Support business operations
         */
    }
}
```

## Masking Strategy Comparison

| Field Type | Strategy | Result | Example |
|-----------|----------|--------|---------|
| **Email** | PARTIAL_EMAIL | First 2 + domain | jo***@example.com |
| **Phone** | PARTIAL_PHONE | Last 4 visible | (XXX) XXX-1234 |
| **SSN** | PARTIAL_SSN | Last 4 visible | XXX-XX-1234 |
| **Credit Card** | PARTIAL_HASH | First 2 + last 4 | 41***5906 |
| **Name** | SUBSTITUTE | Placeholder | [REDACTED] |
| **Token** | TOKENIZE | Hash-based | tok_3a8f9c2e |

## Configuration Examples

```properties
# application.properties

# Masking Configuration
masking.enabled=true
masking.log-masked-values=true

# Role-based masking
masking.admin-roles=ADMIN,SUPERUSER
masking.support-roles=SUPPORT,CUSTOMER_SERVICE

# PII Detection
masking.detect-pii=true
masking.mask-in-logs=true
```

## Related Documentation

- [OWASP Sensitive Data Exposure](https://owasp.org/www-project-top-ten/)
- [Jackson Data Binding](https://github.com/FasterXML/jackson-databind)
- [Spring Data JPA Projections](https://spring.io/projects/spring-data-jpa)
- [Data Masking Best Practices](https://www.imperva.com/learn/data-security/data-masking/)
```
