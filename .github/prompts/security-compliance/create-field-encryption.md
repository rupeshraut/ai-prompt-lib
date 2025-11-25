# Create Field Encryption

Implement field-level encryption for sensitive data in Spring Boot applications. Encrypt personally identifiable information (PII), payment details, and other sensitive fields at rest and in transit.

## Requirements

Your field encryption implementation should include:

- **Transparent Encryption**: Automatic encryption/decryption using annotations
- **Encryption Algorithms**: AES-256 and other standards-based algorithms
- **Key Management**: Secure key storage and rotation strategies
- **JPA Integration**: Seamless integration with Spring Data JPA
- **Selective Encryption**: Configure which fields are encrypted
- **Serialization Support**: Handle encryption in JSON serialization
- **Performance Optimization**: Minimize encryption overhead
- **Key Rotation**: Support changing encryption keys without data loss
- **Audit Logging**: Track encryption/decryption operations
- **Compliance Support**: GDPR, CCPA, HIPAA considerations

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot, Spring Data JPA
- **Encryption**: Jasypt, Tink, or standard javax.crypto
- **Configuration**: Environment variables and configuration classes

## Template Example

```java
import org.springframework.security.crypto.encrypt.Encryptors;
import org.springframework.security.crypto.encrypt.StandardCrypto;
import javax.crypto.Cipher;
import javax.crypto.KeyGenerator;
import javax.crypto.SecretKey;
import javax.crypto.spec.SecretKeySpec;
import java.util.Base64;
import java.security.SecureRandom;
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Value;
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import com.fasterxml.jackson.annotation.JsonProperty;
import lombok.extern.slf4j.Slf4j;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.ElementType;
import java.lang.annotation.Target;

/**
 * Field-level encryption for sensitive data.
 * Encrypts PII, payment details, and other sensitive information.
 */
@Slf4j
public class FieldEncryption {

    /**
     * Custom annotation to mark fields for automatic encryption.
     */
    @Target(ElementType.FIELD)
    @Retention(RetentionPolicy.RUNTIME)
    public @interface Encrypted {
        /**
         * Encryption algorithm to use.
         * Default: AES/GCM/NoPadding
         */
        String algorithm() default "AES/GCM/NoPadding";

        /**
         * Whether to log encryption operations.
         */
        boolean auditLog() default true;

        /**
         * Key identifier for key rotation support.
         */
        String keyId() default "default";
    }

    /**
     * Encryption service for field-level encryption/decryption.
     */
    @Service
    @Slf4j
    public static class EncryptionService {
        private final SecretKey encryptionKey;
        private final SecureRandom random = new SecureRandom();

        @Value("${encryption.key:}")
        private String encryptionKeyString;

        @Value("${encryption.algorithm:AES/GCM/NoPadding}")
        private String algorithm;

        public EncryptionService() {
            // Initialize with default key or external KMS
            this.encryptionKey = generateOrLoadKey();
        }

        /**
         * Encrypt plaintext value using AES-GCM.
         */
        public String encrypt(String plaintext) throws Exception {
            if (plaintext == null) {
                return null;
            }

            Cipher cipher = Cipher.getInstance(algorithm);
            byte[] iv = new byte[12];  // GCM IV is typically 12 bytes
            random.nextBytes(iv);

            javax.crypto.spec.GCMParameterSpec spec =
                    new javax.crypto.spec.GCMParameterSpec(128, iv);

            cipher.init(Cipher.ENCRYPT_MODE, encryptionKey, spec);
            byte[] ciphertext = cipher.doFinal(plaintext.getBytes(java.nio.charset.StandardCharsets.UTF_8));

            // Combine IV + ciphertext
            byte[] combined = new byte[iv.length + ciphertext.length];
            System.arraycopy(iv, 0, combined, 0, iv.length);
            System.arraycopy(ciphertext, 0, combined, iv.length, ciphertext.length);

            String encrypted = Base64.getEncoder().encodeToString(combined);
            log.debug("Field encrypted successfully");

            return encrypted;
        }

        /**
         * Decrypt ciphertext using AES-GCM.
         */
        public String decrypt(String encryptedValue) throws Exception {
            if (encryptedValue == null) {
                return null;
            }

            byte[] combined = Base64.getDecoder().decode(encryptedValue);

            // Extract IV and ciphertext
            byte[] iv = new byte[12];
            byte[] ciphertext = new byte[combined.length - 12];
            System.arraycopy(combined, 0, iv, 0, 12);
            System.arraycopy(combined, 12, ciphertext, 0, ciphertext.length);

            Cipher cipher = Cipher.getInstance(algorithm);
            javax.crypto.spec.GCMParameterSpec spec =
                    new javax.crypto.spec.GCMParameterSpec(128, iv);

            cipher.init(Cipher.DECRYPT_MODE, encryptionKey, spec);
            byte[] plaintext = cipher.doFinal(ciphertext);

            log.debug("Field decrypted successfully");

            return new String(plaintext, java.nio.charset.StandardCharsets.UTF_8);
        }

        private SecretKey generateOrLoadKey() {
            try {
                if (encryptionKeyString != null && !encryptionKeyString.isEmpty()) {
                    // Load from configuration
                    byte[] decodedKey = Base64.getDecoder().decode(encryptionKeyString);
                    return new SecretKeySpec(decodedKey, 0, decodedKey.length, "AES");
                } else {
                    // Generate new key
                    KeyGenerator keyGen = KeyGenerator.getInstance("AES");
                    keyGen.init(256);
                    return keyGen.generateKey();
                }
            } catch (Exception e) {
                log.error("Failed to generate/load encryption key", e);
                throw new RuntimeException("Encryption key initialization failed", e);
            }
        }
    }

    /**
     * JPA attribute converter for automatic encryption/decryption.
     * Applied to fields marked with @Encrypted annotation.
     */
    @jakarta.persistence.Converter(autoApply = false)
    @Slf4j
    public static class EncryptedConverter implements AttributeConverter<String, String> {
        private final EncryptionService encryptionService;

        public EncryptedConverter(EncryptionService encryptionService) {
            this.encryptionService = encryptionService;
        }

        @Override
        public String convertToDatabaseColumn(String attribute) {
            if (attribute == null) {
                return null;
            }

            try {
                String encrypted = encryptionService.encrypt(attribute);
                log.debug("Converted plaintext to database column (encrypted)");
                return encrypted;
            } catch (Exception e) {
                log.error("Encryption failed", e);
                throw new RuntimeException("Failed to encrypt field", e);
            }
        }

        @Override
        public String convertToEntityAttribute(String dbData) {
            if (dbData == null) {
                return null;
            }

            try {
                String decrypted = encryptionService.decrypt(dbData);
                log.debug("Converted database column to entity attribute (decrypted)");
                return decrypted;
            } catch (Exception e) {
                log.error("Decryption failed", e);
                throw new RuntimeException("Failed to decrypt field", e);
            }
        }
    }

    /**
     * Customer entity with encrypted sensitive fields.
     */
    @Entity
    @Table(name = "customers")
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    @Slf4j
    public static class Customer {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String name;

        // Encrypt email (PII)
        @Column(nullable = false)
        @Convert(converter = EncryptedConverter.class)
        @Encrypted(auditLog = true, keyId = "pii-key")
        private String email;

        // Encrypt phone (PII)
        @Column
        @Convert(converter = EncryptedConverter.class)
        @Encrypted(auditLog = true, keyId = "pii-key")
        private String phone;

        // Encrypt SSN (highly sensitive)
        @Column
        @Convert(converter = EncryptedConverter.class)
        @Encrypted(auditLog = true, keyId = "ssn-key")
        private String socialSecurityNumber;

        @Column(nullable = false, updatable = false)
        private java.time.LocalDateTime createdAt = java.time.LocalDateTime.now();

        @Column
        private java.time.LocalDateTime updatedAt;

        @PreUpdate
        public void preUpdate() {
            this.updatedAt = java.time.LocalDateTime.now();
        }
    }

    /**
     * Customer DTO with masked encrypted fields in responses.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class CustomerDTO {
        private Long id;
        private String name;

        @JsonProperty("email")
        private String maskedEmail;

        @JsonProperty("phone")
        private String maskedPhone;

        @JsonProperty("ssn")
        private String maskedSSN;

        /**
         * Create DTO from entity with field masking.
         */
        public static CustomerDTO fromEntity(Customer customer) {
            CustomerDTO dto = new CustomerDTO();
            dto.setId(customer.getId());
            dto.setName(customer.getName());
            dto.setMaskedEmail(maskEmail(customer.getEmail()));
            dto.setMaskedPhone(maskPhone(customer.getPhone()));
            dto.setMaskedSSN(maskSSN(customer.getSocialSecurityNumber()));
            return dto;
        }

        /**
         * Mask email for display: user***@example.com
         */
        private static String maskEmail(String email) {
            if (email == null || email.isEmpty()) {
                return email;
            }
            int atIndex = email.indexOf('@');
            if (atIndex <= 0) {
                return email;
            }
            String localPart = email.substring(0, Math.min(2, atIndex));
            String domain = email.substring(atIndex);
            return localPart + "***" + domain;
        }

        /**
         * Mask phone: (XXX) XXX-1234
         */
        private static String maskPhone(String phone) {
            if (phone == null || phone.length() < 4) {
                return phone;
            }
            String lastFour = phone.substring(phone.length() - 4);
            return "(XXX) XXX-" + lastFour;
        }

        /**
         * Mask SSN: XXX-XX-5678
         */
        private static String maskSSN(String ssn) {
            if (ssn == null || ssn.length() < 4) {
                return ssn;
            }
            String lastFour = ssn.substring(ssn.length() - 4);
            return "XXX-XX-" + lastFour;
        }
    }

    /**
     * Customer repository for encrypted data access.
     */
    public interface CustomerRepository extends JpaRepository<Customer, Long> {
        // Note: Filtering on encrypted fields is not recommended
        // Use unencrypted hash or secondary index instead
    }

    /**
     * Customer service with encryption support.
     */
    @Service
    @Slf4j
    @AllArgsConstructor
    public static class CustomerService {
        private final CustomerRepository customerRepository;
        private final EncryptionService encryptionService;

        /**
         * Create customer with automatic field encryption.
         */
        public CustomerDTO createCustomer(CreateCustomerRequest request) {
            Customer customer = new Customer();
            customer.setName(request.getName());
            customer.setEmail(request.getEmail());
            customer.setPhone(request.getPhone());
            customer.setSocialSecurityNumber(request.getSocialSecurityNumber());

            // Encrypt happens automatically via JPA converter
            Customer saved = customerRepository.save(customer);

            log.info("Customer created with ID: {} (sensitive fields encrypted)", saved.getId());
            return CustomerDTO.fromEntity(saved);
        }

        /**
         * Retrieve customer - fields decrypt automatically.
         */
        public CustomerDTO getCustomer(Long customerId) {
            Customer customer = customerRepository.findById(customerId)
                    .orElseThrow(() -> new RuntimeException("Customer not found"));

            // Decryption happens automatically via JPA converter
            log.info("Customer retrieved: {} (sensitive fields decrypted)", customerId);
            return CustomerDTO.fromEntity(customer);
        }

        /**
         * Update customer with field re-encryption.
         */
        public CustomerDTO updateCustomer(Long customerId, UpdateCustomerRequest request) {
            Customer customer = customerRepository.findById(customerId)
                    .orElseThrow(() -> new RuntimeException("Customer not found"));

            if (request.getEmail() != null) {
                customer.setEmail(request.getEmail());
            }
            if (request.getPhone() != null) {
                customer.setPhone(request.getPhone());
            }

            Customer updated = customerRepository.save(customer);
            log.info("Customer updated: {} (sensitive fields re-encrypted)", customerId);
            return CustomerDTO.fromEntity(updated);
        }
    }

    /**
     * Request DTOs for customer creation/update.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class CreateCustomerRequest {
        private String name;
        private String email;
        private String phone;
        private String socialSecurityNumber;
    }

    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class UpdateCustomerRequest {
        private String email;
        private String phone;
    }

    /**
     * Key management for encryption key rotation.
     */
    @Service
    @Slf4j
    public static class EncryptionKeyManager {

        /**
         * Rotate encryption key.
         * Re-encrypts all data with new key.
         */
        public void rotateKey(String newKeyString) {
            try {
                byte[] decodedKey = Base64.getDecoder().decode(newKeyString);
                SecretKey newKey = new SecretKeySpec(decodedKey, 0, decodedKey.length, "AES");

                log.warn("Starting encryption key rotation...");
                // In production, iterate through all encrypted records and re-encrypt
                log.info("Encryption key rotation completed");
            } catch (Exception e) {
                log.error("Key rotation failed", e);
                throw new RuntimeException("Encryption key rotation failed", e);
            }
        }

        /**
         * Generate new encryption key.
         */
        public String generateNewKey() throws Exception {
            KeyGenerator keyGen = KeyGenerator.getInstance("AES");
            keyGen.init(256);
            SecretKey secretKey = keyGen.generateKey();
            return Base64.getEncoder().encodeToString(secretKey.getEncoded());
        }
    }

    /**
     * Best practices for field encryption.
     */
    public static class BestPractices {
        /**
         * 1. Encrypt at rest with AES-256
         *    - Use authenticated encryption (GCM mode)
         *    - Never use ECB mode
         *    - Use strong key management
         */

        /**
         * 2. Use TLS for data in transit
         *    - Encrypt all network communication
         *    - Validate TLS certificates
         *    - Disable old TLS versions
         */

        /**
         * 3. Encrypt only sensitive fields
         *    - PII (name, email, phone, SSN)
         *    - Payment info (card numbers, bank account)
         *    - Health information (HIPAA-regulated)
         *    - Authentication secrets
         */

        /**
         * 4. Separate encryption from authentication
         *    - Encryption protects confidentiality
         *    - Use hash or HMAC for authentication
         *    - Example: encrypt + HMAC = authenticated encryption
         */

        /**
         * 5. Implement field masking in responses
         *    - Don't expose encrypted values in API
         *    - Show masked values: email@example.com -> e***@example.com
         *    - Help users identify their records
         */

        /**
         * 6. Store encryption keys securely
         *    - Use AWS KMS, Azure Key Vault, or HashiCorp Vault
         *    - Never commit keys to version control
         *    - Rotate keys regularly (annual minimum)
         *    - Separate key from encrypted data
         */

        /**
         * 7. Support key rotation without downtime
         *    - Use key versioning
         *    - Decrypt with old key, encrypt with new key
         *    - Background job for gradual re-encryption
         */

        /**
         * 8. Log encryption operations for audit
         *    - Track who accessed sensitive fields
         *    - Log encryption/decryption events
         *    - Don't log plaintext values
         */

        /**
         * 9. Test encryption thoroughly
         *    - Verify encryption is applied
         *    - Test key rotation
         *    - Verify decryption accuracy
         *    - Test with null/empty values
         */

        /**
         * 10. Consider performance implications
         *     - Encryption overhead ~5-10% latency
         *     - Use connection pooling
         *     - Cache decrypted values carefully
         */
    }
}
```

## Configuration Examples

```properties
# application.properties

# Encryption Configuration
encryption.key=${ENCRYPTION_KEY}
encryption.algorithm=AES/GCM/NoPadding

# Key rotation schedule (cron format)
encryption.key-rotation.enabled=true
encryption.key-rotation.schedule=0 0 0 1 * ?  # Monthly

# Logging
logging.level.encryption=INFO
```

```yaml
# application.yml

encryption:
  key: ${ENCRYPTION_KEY}
  algorithm: AES/GCM/NoPadding
  key-rotation:
    enabled: true
    schedule: "0 0 0 1 * ?"

spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 20
```

## Compliance Considerations

| Regulation | Requirements | Implementation |
|------------|--------------|-----------------|
| **GDPR** | Encrypt PII, support right to deletion | Field encryption, secure deletion |
| **CCPA** | Encrypt sensitive data, audit logging | Encrypted fields, comprehensive logging |
| **HIPAA** | Encrypt health information, access control | Field encryption, role-based access |
| **PCI DSS** | Encrypt payment data, key management | Separate key storage, regular rotation |

## Related Documentation

- [OWASP Cryptographic Practices](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [Jasypt Spring Boot](http://www.jasypt.org/)
- [AWS KMS Integration](https://docs.aws.amazon.com/kms/)
- [Spring Data JPA Converters](https://spring.io/projects/spring-data-jpa)
```
