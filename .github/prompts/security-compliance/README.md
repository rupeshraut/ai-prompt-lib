# Security & Compliance Library

Comprehensive prompts for implementing security controls and regulatory compliance in Spring Boot applications. Covers encryption, audit logging, compliance frameworks, and data protection.

## Available Prompts

### 1. Create Field Encryption
Implement field-level encryption for sensitive data using AES-256 and secure key management.

| Feature | Description |
|---------|-------------|
| Transparent Encryption | Automatic encryption/decryption via JPA |
| AES-256 Encryption | Industry-standard strong encryption |
| Key Management | Secure key storage and rotation |
| Selective Encryption | Mark fields for encryption |
| Field Masking | Display masked values in responses |
| Compliance Support | GDPR, CCPA, HIPAA, PCI DSS |
| Key Rotation | Change keys without data loss |
| Performance | Minimal encryption overhead |

### 2. Create Audit Logging
Implement comprehensive audit trails for compliance and forensic analysis.

| Feature | Description |
|---------|-------------|
| User Action Tracking | Who did what, when, where |
| Data Change Tracking | Before/after audit trail |
| Authentication Events | Login, logout, failed attempts |
| Immutable Logs | Prevent tampering with records |
| Structured Logging | JSON format for SIEM integration |
| Retention Policies | Compliance-driven retention |
| Search Capability | Query by user, resource, action |
| Performance | Minimal impact on throughput |

### 3. Create Compliance Frameworks
Support for GDPR, CCPA, HIPAA, and PCI DSS compliance requirements.

| Feature | Description |
|---------|-------------|
| Consent Management | GDPR and CCPA consent tracking |
| Data Subject Rights | Access, erasure, portability |
| Purpose Limitation | Restrict data usage |
| Retention Policies | Automatic data deletion |
| Breach Notification | Track and report breaches |
| Compliance Checks | Automated validation |
| Documentation | Privacy impact assessments |

### 4. Create Data Masking
Hide sensitive information in responses and logs based on user role.

| Feature | Description |
|---------|-------------|
| Multiple Strategies | Redaction, tokenization, substitution |
| PII Detection | Automatic sensitive field detection |
| Role-Based Masking | Different masking per user role |
| API Response Masking | Hide sensitive fields in JSON |
| Log Masking | Prevent sensitive data in logs |
| Database Views | Masked views for non-admin users |
| Performance | Efficient masking with caching |

## Quick Start

### 1. Encrypt Sensitive Fields

```
@workspace Use .github/prompts/security-compliance/create-field-encryption.md

Implement AES-256 encryption for PII, payment details, and other sensitive data
```

### 2. Setup Audit Logging

```
@workspace Use .github/prompts/security-compliance/create-audit-logging.md

Configure comprehensive audit trail for all data modifications and user actions
```

### 3. Implement Compliance Framework

```
@workspace Use .github/prompts/security-compliance/create-compliance-frameworks.md

Add GDPR/CCPA consent management and data subject rights support
```

### 4. Configure Data Masking

```
@workspace Use .github/prompts/security-compliance/create-data-masking.md

Mask sensitive fields in API responses based on user roles
```

## Architecture Overview

```
┌─────────────────────────────────────────────────────┐
│            Client Request (Unauthenticated)          │
└────────────────┬────────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────────┐
        │  Authentication     │ ← Audit: Login event
        │  & Authorization    │   Log: User, IP, time
        └────────┬────────────┘
                 │
                 ▼
        ┌─────────────────────────────────────────┐
        │  Data Access Layer                      │
        │  (JPA Converters Apply Decryption)      │
        │  - Fields marked @Encrypted             │
        │  - AES-256-GCM automatic decryption     │
        │  - Audit logged: Data access            │
        └────────┬────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────────────────────────────┐
        │  Business Logic                         │
        │  (Apply Compliance Rules)               │
        │  - Check consent (GDPR/CCPA)           │
        │  - Enforce data minimization            │
        │  - Apply retention policies             │
        └────────┬────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────────────────────────────┐
        │  Response Serialization                 │
        │  (Apply Field Masking)                  │
        │  - Mask based on user role              │
        │  - Redact PII fields                    │
        │  - Hide sensitive information           │
        └────────┬────────────────────────────────┘
                 │
                 ▼
        ┌─────────────────────┐
        │  JSON Response      │
        │  (Masked PII)       │
        │  {                  │
        │    "email": "j***@  │
        │              example│
        │              .com", │
        │    "ssn": "XXX-XX-  │
        │            1234"    │
        │  }                  │
        └─────────────────────┘
                 │
                 ▼ Audit: Response sent
        ┌─────────────────────┐
        │  Audit Log Store    │
        │  (Immutable)        │
        │  (Compliance proof) │
        └─────────────────────┘
```

## Integration Patterns

### Spring Boot Entity with Encryption

```java
@Entity
@Table(name = "customers")
public class Customer {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    // Encrypted field - automatic encryption/decryption
    @Column(nullable = false)
    @Convert(converter = EncryptedConverter.class)
    @Encrypted(auditLog = true)
    private String email;

    // Highly sensitive - always encrypted
    @Column
    @Convert(converter = EncryptedConverter.class)
    @Encrypted(auditLog = true)
    private String socialSecurityNumber;
}
```

### API Response with Masking

```java
@RestController
@RequestMapping("/api/customers")
public class CustomerController {
    private final CustomerService service;

    @GetMapping("/{id}")
    public CustomerDTO getCustomer(@PathVariable Long id) {
        Customer customer = service.getCustomer(id);
        // Automatic masking applied in serialization
        return CustomerDTO.fromEntity(customer);
    }
}

// DTO with Jackson serializers for masking
@Getter
@Setter
public class CustomerDTO {
    private Long id;
    private String name;

    @JsonSerialize(using = MaskedEmailSerializer.class)
    private String email;  // Masked: j***@example.com

    @JsonSerialize(using = MaskedSSNSerializer.class)
    private String ssn;    // Masked: XXX-XX-1234
}
```

### Consent & Compliance Check

```java
@Service
public class OrderService {
    private final ConsentManager consentManager;
    private final ComplianceChecker complianceChecker;

    public Order createOrder(CreateOrderRequest request) {
        // Check GDPR consent before processing
        if (!consentManager.hasConsent(userId, "PROCESSING", "GDPR")) {
            throw new ConsentRequiredException("GDPR processing consent required");
        }

        // Check compliance rules
        ComplianceContext context = new ComplianceContext();
        context.setUserId(userId);
        context.setRegulation("GDPR");
        context.setConsent(true);

        if (!complianceChecker.isGdprCompliant(context)) {
            throw new ComplianceViolationException("Operation violates GDPR");
        }

        return order;
    }
}
```

### Audit Logging for Data Modifications

```java
@Service
public class CustomerService {
    private final AuditLogger auditLogger;

    public void updateCustomer(String customerId, UpdateRequest request) {
        Customer before = fetchCustomer(customerId);

        // Update customer
        Customer after = applyChanges(before, request);
        save(after);

        // Log with before/after values
        AuditEvent event = new AuditEvent();
        event.setUserId(getCurrentUserId());
        event.setAction("UPDATE");
        event.setResourceType("CUSTOMER");
        event.setResourceId(customerId);
        event.setBeforeValue(serialize(before));
        event.setAfterValue(serialize(after));

        auditLogger.logAction(event);
    }
}
```

## Key Concepts

### Field Encryption
Encrypt sensitive data at rest using AES-256-GCM. Transparent via JPA converters. Supports key rotation.

### Audit Logging
Immutable record of all data access and modifications. Used for compliance, forensics, and anomaly detection.

### Compliance Frameworks
Systematic support for GDPR, CCPA, HIPAA, PCI DSS. Consent management, data subject rights, retention policies.

### Data Masking
Display masked versions of PII in responses and logs. Role-based masking control. Automatic PII detection.

## Best Practices

1. **Encrypt Sensitive Data**
   - Use AES-256-GCM for encryption
   - Store encryption keys in KMS (AWS, Azure, Vault)
   - Rotate keys annually

2. **Comprehensive Audit Logging**
   - Log all data access (especially PII)
   - Log authentication events
   - Store audit logs immutably
   - Retain per regulatory requirements

3. **Implement Compliance**
   - Get explicit user consent
   - Support data subject rights (access, deletion)
   - Enforce data minimization
   - Document compliance measures

4. **Mask in Responses**
   - Role-based field masking
   - Never expose full SSN, credit card
   - Mask in logs automatically
   - Test masking thoroughly

5. **Secure Key Management**
   - Use external key management service
   - Never hardcode keys
   - Implement key rotation
   - Audit key access

6. **Regular Testing**
   - Penetration testing
   - Compliance audits
   - Encryption verification
   - Audit log integrity checks

## Regulatory Requirements

| Regulation | Encryption | Audit Logging | Consent | Data Rights |
|-----------|-----------|---------------|---------|-------------|
| **GDPR** | PII required | Yes (30-day) | Explicit | Access, erasure, portability |
| **CCPA** | Recommended | Yes | Opt-in/out | Access, deletion, portability |
| **HIPAA** | PHI required | Yes (6 years) | Patient auth | Access, amendment, accounting |
| **PCI DSS** | Card data required | Yes (1 year) | N/A | Not applicable |

## Configuration Examples

```properties
# Security & Compliance Configuration

# Encryption
encryption.key=${ENCRYPTION_KEY}
encryption.algorithm=AES/GCM/NoPadding

# Audit Logging
audit.enabled=true
audit.log-sensitive-data=true
audit.retention-days=2555  # 7 years

# Compliance
compliance.frameworks=GDPR,CCPA,PCI_DSS
compliance.enforce-consent=true
compliance.data-retention-months=36

# Masking
masking.enabled=true
masking.admin-roles=ADMIN,SUPERUSER
```

## Related Documentation

- [OWASP Cryptographic Storage](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html)
- [GDPR Compliance Guide](https://gdpr-info.eu/)
- [Spring Security](https://spring.io/projects/spring-security)
- [Spring Data JPA](https://spring.io/projects/spring-data-jpa)

## Library Navigation

This library focuses on security and compliance. For related capabilities:
- See **distributed-tracing** library for audit trace logging
- See **observability** library for security metrics
- See **data-quality-validation** library for data integrity
