# Create Compliance Frameworks

Implement compliance framework support for GDPR, CCPA, HIPAA, and PCI DSS. Ensure your application meets regulatory requirements through systematic compliance checks, consent management, and privacy-by-design patterns.

## Requirements

Your compliance framework implementation should include:

- **Consent Management**: GDPR and CCPA consent tracking
- **Data Subject Rights**: Right to access, right to erasure, right to portability
- **Purpose Limitation**: Restrict data use to stated purposes
- **Retention Policies**: Automatic data deletion based on policies
- **Privacy Impact Assessment**: Document data processing activities
- **Breach Notification**: Track and report data breaches
- **Data Inventory**: Catalog all data collection and usage
- **Compliance Checks**: Automated compliance validation
- **Policy Enforcement**: Prevent non-compliant operations
- **Documentation**: Generate compliance reports and artifacts

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot
- **Annotations**: Custom compliance annotations
- **Configuration**: Policy-based configuration

## Template Example

```java
import org.springframework.stereotype.Service;
import org.springframework.stereotype.Component;
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.security.core.context.SecurityContextHolder;
import lombok.extern.slf4j.Slf4j;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Compliance framework for GDPR, CCPA, HIPAA, and PCI DSS.
 * Ensures regulatory requirements are met systematically.
 */
@Slf4j
public class ComplianceFrameworks {

    /**
     * Supported compliance regulations.
     */
    public enum Regulation {
        GDPR("GDPR", "EU General Data Protection Regulation"),
        CCPA("CCPA", "California Consumer Privacy Act"),
        HIPAA("HIPAA", "Health Insurance Portability and Accountability Act"),
        PCI_DSS("PCI-DSS", "Payment Card Industry Data Security Standard"),
        SOX("SOX", "Sarbanes-Oxley Act"),
        GLBA("GLBA", "Gramm-Leach-Bliley Act");

        public final String code;
        public final String name;

        Regulation(String code, String name) {
            this.code = code;
            this.name = name;
        }
    }

    /**
     * Consent record for GDPR and CCPA compliance.
     */
    @Entity
    @Table(name = "user_consents")
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class UserConsent {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String userId;

        @Column(nullable = false)
        private String consentType;  // MARKETING, ANALYTICS, PROFILING, PROCESSING

        @Column(nullable = false)
        private String regulation;

        @Column(nullable = false)
        private Boolean granted;

        @Column(nullable = false)
        private LocalDateTime grantedAt;

        @Column
        private LocalDateTime revokedAt;

        @Column
        private String ipAddress;

        @Column(columnDefinition = "TEXT")
        private String consentText;  // Exact text shown to user

        @Column
        private Boolean explicit;  // Affirmative action vs. implicit

        @Column
        private String ipLocation;

        @Column
        private String userAgent;

        @Column(nullable = false, updatable = false)
        private LocalDateTime createdAt = LocalDateTime.now();

        @Column
        private LocalDateTime updatedAt;

        @PreUpdate
        public void preUpdate() {
            this.updatedAt = LocalDateTime.now();
        }
    }

    /**
     * Data retention policy.
     */
    @Entity
    @Table(name = "retention_policies")
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class RetentionPolicy {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String dataType;  // CUSTOMER, ORDER, LOG, SESSION

        @Column(nullable = false)
        private String regulation;

        @Column(nullable = false)
        private Integer retentionDays;

        @Column
        private String purpose;

        @Column
        private Boolean deleteAfterExpiry;

        @Column
        private Boolean anonymizeInstead;

        @Column
        private LocalDateTime effectiveFrom;

        @Column
        private LocalDateTime effectiveUntil;
    }

    /**
     * Consent repository.
     */
    public interface ConsentRepository extends JpaRepository<UserConsent, Long> {
        Optional<UserConsent> findByUserIdAndConsentTypeAndRegulation(
                String userId, String consentType, String regulation);

        List<UserConsent> findByUserIdAndGrantedTrue(String userId);

        List<UserConsent> findByUserIdAndRevokedAtIsNull(String userId);
    }

    /**
     * Retention policy repository.
     */
    public interface RetentionPolicyRepository extends JpaRepository<RetentionPolicy, Long> {
        Optional<RetentionPolicy> findByDataTypeAndRegulation(
                String dataType, String regulation);

        List<RetentionPolicy> findByDataType(String dataType);
    }

    /**
     * Consent management service for GDPR/CCPA.
     */
    @Service
    @Slf4j
    @AllArgsConstructor
    public static class ConsentManager {
        private final ConsentRepository consentRepository;

        /**
         * Record user consent for specific purpose.
         */
        public void grantConsent(String userId, String consentType,
                                String regulation, String ipAddress, String userAgent) {
            UserConsent consent = new UserConsent();
            consent.setUserId(userId);
            consent.setConsentType(consentType);
            consent.setRegulation(regulation);
            consent.setGranted(true);
            consent.setGrantedAt(LocalDateTime.now());
            consent.setIpAddress(ipAddress);
            consent.setUserAgent(userAgent);
            consent.setExplicit(true);

            consentRepository.save(consent);
            log.info("Consent granted: userId={}, type={}, regulation={}", userId, consentType, regulation);
        }

        /**
         * Revoke user consent.
         */
        public void revokeConsent(String userId, String consentType, String regulation) {
            Optional<UserConsent> existing = consentRepository
                    .findByUserIdAndConsentTypeAndRegulation(userId, consentType, regulation);

            if (existing.isPresent()) {
                UserConsent consent = existing.get();
                consent.setRevokedAt(LocalDateTime.now());
                consentRepository.save(consent);
                log.info("Consent revoked: userId={}, type={}", userId, consentType);
            }
        }

        /**
         * Check if user has given consent for operation.
         */
        public boolean hasConsent(String userId, String consentType, String regulation) {
            Optional<UserConsent> consent = consentRepository
                    .findByUserIdAndConsentTypeAndRegulation(userId, consentType, regulation);

            if (consent.isPresent()) {
                UserConsent c = consent.get();
                return c.isGranted() && c.getRevokedAt() == null;
            }
            return false;
        }

        /**
         * Get all active consents for user.
         */
        public List<UserConsent> getUserConsents(String userId) {
            return consentRepository.findByUserIdAndRevokedAtIsNull(userId);
        }
    }

    /**
     * Data subject rights implementation (GDPR Article 15-22).
     */
    @Service
    @Slf4j
    @AllArgsConstructor
    public static class DataSubjectRights {

        /**
         * Right to access (Article 15).
         * User can request copy of their personal data.
         */
        public DataAccessReport generateAccessReport(String userId) {
            DataAccessReport report = new DataAccessReport();
            report.setUserId(userId);
            report.setGeneratedAt(LocalDateTime.now());
            report.setRegulation("GDPR");

            // Collect all user data
            Map<String, Object> userData = new HashMap<>();
            userData.put("profile", fetchUserProfile(userId));
            userData.put("orders", fetchUserOrders(userId));
            userData.put("consents", fetchUserConsents(userId));
            userData.put("activities", fetchUserActivities(userId));

            report.setData(userData);

            log.info("Access report generated for user: {}", userId);
            return report;
        }

        /**
         * Right to erasure (Article 17).
         * User can request deletion of their data.
         */
        public void deleteUserData(String userId, String reason) {
            log.warn("User data deletion requested: userId={}, reason={}", userId, reason);

            // Delete user profile
            deleteUserProfile(userId);

            // Delete user orders
            deleteUserOrders(userId);

            // Delete personal data while keeping anonymized records
            anonymizeUserData(userId);

            log.info("User data deleted: {}", userId);
        }

        /**
         * Right to portability (Article 20).
         * User can export data in machine-readable format.
         */
        public String exportUserDataAsJson(String userId) {
            DataAccessReport report = generateAccessReport(userId);
            // Convert to JSON
            try {
                return new com.fasterxml.jackson.databind.ObjectMapper()
                        .writeValueAsString(report);
            } catch (Exception e) {
                throw new RuntimeException("Failed to export user data", e);
            }
        }

        /**
         * Right to object (Article 21).
         * User can object to processing.
         */
        public void objectToProcessing(String userId, String processingType, String reason) {
            log.info("User objected to processing: userId={}, type={}, reason={}",
                    userId, processingType, reason);

            // Stop the processing activity
            stopProcessing(userId, processingType);
        }

        private Map<String, Object> fetchUserProfile(String userId) {
            return Map.of("userId", userId, "name", "John Doe", "email", "john@example.com");
        }

        private List<Object> fetchUserOrders(String userId) {
            return new ArrayList<>();
        }

        private List<Object> fetchUserConsents(String userId) {
            return new ArrayList<>();
        }

        private List<Object> fetchUserActivities(String userId) {
            return new ArrayList<>();
        }

        private void deleteUserProfile(String userId) {
            log.info("User profile deleted: {}", userId);
        }

        private void deleteUserOrders(String userId) {
            log.info("User orders deleted: {}", userId);
        }

        private void anonymizeUserData(String userId) {
            log.info("User data anonymized: {}", userId);
        }

        private void stopProcessing(String userId, String processingType) {
            log.info("Processing stopped: userId={}, type={}", userId, processingType);
        }
    }

    /**
     * Compliance checker for automated validation.
     */
    @Component
    @Slf4j
    public static class ComplianceChecker {

        /**
         * Check if operation is GDPR compliant.
         */
        public boolean isGdprCompliant(ComplianceContext context) {
            // Check consent requirements
            if (!hasRequiredConsent(context)) {
                log.warn("GDPR compliance check failed: missing consent");
                return false;
            }

            // Check legal basis
            if (!hasLegalBasis(context)) {
                log.warn("GDPR compliance check failed: no legal basis");
                return false;
            }

            // Check data minimization
            if (!isMinimalDataCollection(context)) {
                log.warn("GDPR compliance check failed: excessive data collection");
                return false;
            }

            return true;
        }

        /**
         * Check if operation is CCPA compliant.
         */
        public boolean isCcpaCompliant(ComplianceContext context) {
            // Check opt-in for sale of personal information
            if (context.isSaleOfData() && !context.isOptedInForSale()) {
                log.warn("CCPA compliance check failed: sale without opt-in");
                return false;
            }

            // Check opt-out rights provided
            if (!context.hasOptOutOption()) {
                log.warn("CCPA compliance check failed: no opt-out option");
                return false;
            }

            return true;
        }

        /**
         * Check if operation is HIPAA compliant.
         */
        public boolean isHipaaCompliant(ComplianceContext context) {
            // Check if PHI is encrypted
            if (context.containsProtectedHealthInfo() && !context.isEncrypted()) {
                log.warn("HIPAA compliance check failed: PHI not encrypted");
                return false;
            }

            // Check access controls
            if (!context.hasAuthorization()) {
                log.warn("HIPAA compliance check failed: unauthorized access");
                return false;
            }

            return true;
        }

        /**
         * Check if operation is PCI DSS compliant.
         */
        public boolean isPciDssCompliant(ComplianceContext context) {
            // Check if payment card data is encrypted
            if (context.containsCardData() && !context.isEncrypted()) {
                log.warn("PCI DSS compliance check failed: card data not encrypted");
                return false;
            }

            // Check if only necessary card fields are stored
            if (context.storesFullCardNumber()) {
                log.warn("PCI DSS compliance check failed: full card number stored");
                return false;
            }

            return true;
        }

        private boolean hasRequiredConsent(ComplianceContext context) {
            return context.getConsent() != null && context.getConsent();
        }

        private boolean hasLegalBasis(ComplianceContext context) {
            return context.getLegalBasis() != null && !context.getLegalBasis().isEmpty();
        }

        private boolean isMinimalDataCollection(ComplianceContext context) {
            return context.getDataFields() < 20;  // Example threshold
        }
    }

    /**
     * Compliance context for checking operations.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class ComplianceContext {
        private String userId;
        private String operation;
        private String regulation;
        private Boolean consent;
        private String legalBasis;
        private Integer dataFields;
        private Boolean encrypted;
        private Boolean authorization;
        private Boolean saleOfData;
        private Boolean optedInForSale;
        private Boolean hasOptOutOption;
        private Boolean containsProtectedHealthInfo;
        private Boolean containsCardData;
        private Boolean storesFullCardNumber;
    }

    /**
     * Data access report for right to access requests.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class DataAccessReport {
        private String userId;
        private LocalDateTime generatedAt;
        private String regulation;
        private Map<String, Object> data;

        public String toJson() {
            try {
                return new com.fasterxml.jackson.databind.ObjectMapper()
                        .writeValueAsString(this);
            } catch (Exception e) {
                return "{}";
            }
        }
    }

    /**
     * Privacy impact assessment (PIA).
     */
    @Entity
    @Table(name = "privacy_impact_assessments")
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class PrivacyImpactAssessment {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String projectName;

        @Column(columnDefinition = "TEXT")
        private String description;

        @Column
        private String regulation;

        @Column
        private String dataTypes;  // CSV list

        @Column
        private String risks;

        @Column
        private String mitigations;

        @Column
        private Boolean approved;

        @Column
        private LocalDateTime completedAt;
    }

    /**
     * Best practices for compliance frameworks.
     */
    public static class BestPractices {
        /**
         * 1. Privacy by design - build compliance into requirements
         *    - Assess regulations upfront
         *    - Design systems for compliance
         *    - Document design decisions
         */

        /**
         * 2. Consent management - get explicit consent
         *    - Granular consent for different purposes
         *    - Easy consent withdrawal
         *    - Maintain audit trail of consents
         */

        /**
         * 3. Data minimization - collect only necessary data
         *    - Document purpose for each field
         *    - Regularly review data necessity
         *    - Delete unnecessary data
         */

        /**
         * 4. Implement data subject rights
         *    - Right to access (portability)
         *    - Right to erasure (deletion)
         *    - Right to rectification
         *    - Right to object
         */

        /**
         * 5. Automated compliance checks
         *    - Check before storing data
         *    - Check before sharing data
         *    - Regular compliance audits
         */

        /**
         * 6. Documentation and reporting
         *    - Privacy policies
         *    - Data processing agreements
         *    - Privacy impact assessments
         *    - Breach notification procedures
         */

        /**
         * 7. Regular training and audits
         *    - Staff compliance training
         *    - Annual compliance audits
         *    - External compliance reviews
         */

        /**
         * 8. Breach notification procedures
         *    - Rapid breach detection
         *    - 72-hour notification requirement (GDPR)
         *    - Comprehensive breach records
         */
    }
}
```

## Compliance Checklist

| Framework | Key Requirements | Spring Boot Implementation |
|-----------|------------------|---------------------------|
| **GDPR** | Consent, DPO, right to access, encryption | @ConsentRequired, DataSubjectRights |
| **CCPA** | Opt-in/out, consumer rights, disclosure | ConsentManager, DataAccessReport |
| **HIPAA** | Encryption, access control, audit logs | @EncryptionRequired, AuditLogger |
| **PCI DSS** | Card data encryption, tokenization, ACLs | FieldEncryption, SecureStorage |

## Related Documentation

- [GDPR Official Text](https://gdpr-info.eu/)
- [CCPA Information](https://oag.ca.gov/privacy/ccpa)
- [HIPAA Compliance](https://www.hhs.gov/hipaa/)
- [PCI DSS Standard](https://www.pcisecuritystandards.org/)
```
