# Create Payment Model

Create domain models for FedNow payment processing with proper state management.

## Requirements

The payment model should include:
- Immutable core payment data
- Mutable processing state
- Processing step tracking
- Validation and sanctions results
- Serialization support for Flink state

## Code Style

- Use Java 17 features (records, sealed interfaces)
- Use Lombok for boilerplate reduction
- Implement Flink's serialization interfaces
- Add comprehensive JavaDoc

## Payment Domain Model

```java
package com.example.fednow.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.apache.flink.api.common.typeinfo.TypeInfo;
import org.apache.flink.api.common.typeinfo.TypeInfoFactory;

import java.io.Serializable;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

/**
 * Core domain model for FedNow payment processing.
 *
 * <p>This class holds all payment data through the processing pipeline,
 * including original message data, validation results, and processing state.
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Payment implements Serializable {

    private static final long serialVersionUID = 1L;

    // ========== Message Identifiers ==========

    /** ISO 20022 Message ID from pacs.008 */
    private String messageId;

    /** Transaction ID - primary correlation key */
    private String transactionId;

    /** Instruction ID for tracking */
    private String instructionId;

    /** End-to-end ID for customer reference */
    private String endToEndId;

    // ========== Amount Information ==========

    /** Payment amount */
    private BigDecimal amount;

    /** Currency code (always USD for FedNow) */
    private String currency;

    /** Value date for settlement */
    private LocalDate valueDate;

    // ========== Debtor (Sender) Information ==========

    /** Debtor name */
    private String debtorName;

    /** Debtor account number */
    private String debitAccount;

    /** Debtor bank routing number */
    private String debtorRoutingNumber;

    /** Debtor country code */
    private String debtorCountry;

    /** Debtor account type (CHECKING, SAVINGS) */
    private String debtorAccountType;

    // ========== Creditor (Receiver) Information ==========

    /** Creditor name */
    private String creditorName;

    /** Creditor account number */
    private String creditAccount;

    /** Creditor bank routing number */
    private String creditorRoutingNumber;

    /** Creditor country code */
    private String creditorCountry;

    /** Creditor account type */
    private String creditorAccountType;

    // ========== Processing Status ==========

    /** Current payment status */
    @Builder.Default
    private PaymentStatus status = PaymentStatus.RECEIVED;

    /** Failure reason code if rejected */
    private String failureReason;

    /** Detailed error message */
    private String errorMessage;

    // ========== Account Validation Results ==========

    /** Account validation status */
    private ValidationStatus validationStatus;

    /** Debit account validation reference */
    private String debitValidationRef;

    /** Credit account validation reference */
    private String creditValidationRef;

    // ========== Risk Control Results ==========

    /** Risk control status */
    private RiskControlStatus riskControlStatus;

    /** Risk score (0-100) */
    private Integer riskScore;

    /** Risk control reference */
    private String riskControlRef;

    /** Risk decline reason if declined */
    private String riskDeclineReason;

    // ========== Sanctions Screening Results ==========

    /** Sanctions screening status */
    private SanctionsStatus sanctionsStatus;

    /** Sanctions screening reference */
    private String sanctionsRef;

    /** List of sanctions hits if any */
    @Builder.Default
    private List<SanctionsHit> sanctionsHits = new ArrayList<>();

    // ========== Account Posting Results ==========

    /** Posting status */
    private PostingStatus postingStatus;

    /** Posting reference from TMS */
    private String postingReference;

    /** Posting error code if failed */
    private String postingErrorCode;

    // ========== Processing Metadata ==========

    /** Original message creation time */
    private LocalDateTime creationDateTime;

    /** Settlement method from message */
    private String settlementMethod;

    /** Time payment was received */
    private LocalDateTime receivedAt;

    /** Time payment completed processing */
    private LocalDateTime completedAt;

    /** Original XML message (for audit) */
    private String originalMessage;

    /** Processing steps for audit trail */
    @Builder.Default
    private List<ProcessingStep> processingSteps = new ArrayList<>();

    // ========== Helper Methods ==========

    /**
     * Add a processing step to the audit trail.
     */
    public void addProcessingStep(String step, String result) {
        if (processingSteps == null) {
            processingSteps = new ArrayList<>();
        }
        processingSteps.add(new ProcessingStep(
            step,
            result,
            LocalDateTime.now()
        ));
    }

    /**
     * Check if payment is ready for sanctions screening.
     */
    public boolean isReadyForSanctions() {
        return validationStatus == ValidationStatus.VALID &&
               riskControlStatus == RiskControlStatus.APPROVED;
    }

    /**
     * Check if payment is ready for account posting.
     */
    public boolean isReadyForPosting() {
        return isReadyForSanctions() &&
               sanctionsStatus == SanctionsStatus.CLEAR;
    }

    /**
     * Check if payment processing is complete.
     */
    public boolean isComplete() {
        return status == PaymentStatus.COMPLETED ||
               status == PaymentStatus.FAILED;
    }

    /**
     * Get account type for validation requests.
     */
    public String getAccountType(String direction) {
        return "DEBIT".equals(direction)
            ? debtorAccountType
            : creditorAccountType;
    }

    /**
     * Get routing number for validation requests.
     */
    public String getRoutingNumber(String direction) {
        return "DEBIT".equals(direction)
            ? debtorRoutingNumber
            : creditorRoutingNumber;
    }

    /**
     * Mark payment as completed successfully.
     */
    public void markCompleted() {
        this.status = PaymentStatus.COMPLETED;
        this.completedAt = LocalDateTime.now();
        addProcessingStep("COMPLETE", "SUCCESS");
    }

    /**
     * Mark payment as failed.
     */
    public void markFailed(String reason, String message) {
        this.status = PaymentStatus.FAILED;
        this.failureReason = reason;
        this.errorMessage = message;
        this.completedAt = LocalDateTime.now();
        addProcessingStep("COMPLETE", "FAILED: " + reason);
    }
}
```

## Status Enumerations

```java
package com.example.fednow.model;

/**
 * Overall payment processing status.
 */
public enum PaymentStatus {
    /** Payment received and queued */
    RECEIVED,

    /** Payment is being processed */
    PROCESSING,

    /** Parsing of ISO 20022 message failed */
    PARSE_ERROR,

    /** Account validation in progress */
    VALIDATING,

    /** Risk control check in progress */
    RISK_CHECKING,

    /** Sanctions screening in progress */
    SANCTIONS_SCREENING,

    /** Account posting in progress */
    POSTING,

    /** Payment completed successfully */
    COMPLETED,

    /** Payment failed at some step */
    FAILED,

    /** Payment rejected due to sanctions */
    SANCTIONS_HIT,

    /** Payment timed out */
    TIMEOUT,

    /** Pending manual review */
    PENDING_REVIEW
}

/**
 * Account validation status.
 */
public enum ValidationStatus {
    /** Not yet validated */
    PENDING,

    /** Validation successful */
    VALID,

    /** Validation failed */
    FAILED,

    /** Validation timed out */
    TIMEOUT,

    /** System error during validation */
    ERROR
}

/**
 * Risk control status.
 */
public enum RiskControlStatus {
    /** Not yet checked */
    PENDING,

    /** Risk check approved */
    APPROVED,

    /** Risk check declined */
    DECLINED,

    /** Risk check timed out */
    TIMEOUT,

    /** System error during risk check */
    ERROR
}

/**
 * Sanctions screening status.
 */
public enum SanctionsStatus {
    /** Not yet screened */
    PENDING,

    /** Screening complete - no hits */
    CLEAR,

    /** Definite sanctions hit */
    HIT,

    /** Potential hit requiring review */
    POTENTIAL_HIT,

    /** Screening timed out */
    TIMEOUT,

    /** System error during screening */
    ERROR
}

/**
 * Account posting status.
 */
public enum PostingStatus {
    /** Not yet posted */
    PENDING,

    /** Posted successfully */
    POSTED,

    /** Posting failed */
    FAILED,

    /** Posting timed out */
    TIMEOUT,

    /** System error during posting */
    ERROR
}
```

## Supporting Models

```java
package com.example.fednow.model;

import java.io.Serializable;
import java.time.LocalDateTime;

/**
 * Represents a processing step for audit trail.
 */
public record ProcessingStep(
    String step,
    String result,
    LocalDateTime timestamp
) implements Serializable {
    private static final long serialVersionUID = 1L;
}

/**
 * Represents a sanctions hit from FircoSoft.
 */
public record SanctionsHit(
    String listName,
    String matchedName,
    String matchedParty,
    int matchScore,
    String hitDetails
) implements Serializable {
    private static final long serialVersionUID = 1L;
}

/**
 * Configuration for JMS connections.
 */
public record JmsConfig(
    String connectionFactoryName,
    String username,
    String password,
    String requestQueue,
    String responseQueue,
    long timeoutMs
) implements Serializable {
    private static final long serialVersionUID = 1L;

    public static JmsConfig fromParams(ParameterTool params) {
        return new JmsConfig(
            params.get("jms.connection-factory", "ConnectionFactory"),
            params.get("jms.username", ""),
            params.get("jms.password", ""),
            params.get("jms.sanctions.request-queue", "SANCTIONS.REQUEST.QUEUE"),
            params.get("jms.sanctions.response-queue", "SANCTIONS.RESPONSE.QUEUE"),
            params.getLong("jms.sanctions.timeout", 30000)
        );
    }
}
```

## Sanctions Request/Response Models

```java
package com.example.fednow.model;

import lombok.Builder;
import lombok.Data;

import java.io.Serializable;
import java.math.BigDecimal;
import java.util.List;

/**
 * Request model for FircoSoft sanctions screening.
 */
@Data
@Builder
public class SanctionsRequest implements Serializable {
    private static final long serialVersionUID = 1L;

    private String requestId;
    private String transactionId;

    // Debtor information
    private String debtorName;
    private String debtorAccount;
    private String debtorCountry;

    // Creditor information
    private String creditorName;
    private String creditorAccount;
    private String creditorCountry;

    // Transaction details
    private BigDecimal amount;
    private String currency;
}

/**
 * Response model from FircoSoft sanctions screening.
 */
@Data
@Builder
public class SanctionsResponse implements Serializable {
    private static final long serialVersionUID = 1L;

    private String requestId;
    private String transactionId;
    private SanctionsResult result;
    private List<SanctionsHit> hits;
    private String responseCode;
    private String responseMessage;

    public enum SanctionsResult {
        CLEAR,
        HIT,
        POTENTIAL_HIT,
        ERROR
    }
}
```

## Flink Type Serializer

```java
package com.example.fednow.state;

import com.example.fednow.model.Payment;
import org.apache.flink.api.common.typeinfo.TypeInfoFactory;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.api.common.typeinfo.Types;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import java.lang.reflect.Type;
import java.util.Map;

/**
 * Custom TypeInfoFactory for Payment to ensure proper Flink serialization.
 */
public class PaymentTypeInfoFactory extends TypeInfoFactory<Payment> {

    @Override
    public TypeInformation<Payment> createTypeInfo(
            Type t, Map<String, TypeInformation<?>> genericParameters) {
        return TypeInformation.of(Payment.class);
    }
}

/**
 * JSON serializer for Payment state.
 *
 * <p>Use this when you need to inspect state in RocksDB or for debugging.
 */
public class PaymentJsonSerializer {

    private static final ObjectMapper MAPPER = new ObjectMapper()
        .registerModule(new JavaTimeModule());

    public static byte[] serialize(Payment payment) {
        try {
            return MAPPER.writeValueAsBytes(payment);
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize Payment", e);
        }
    }

    public static Payment deserialize(byte[] bytes) {
        try {
            return MAPPER.readValue(bytes, Payment.class);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize Payment", e);
        }
    }
}
```

## State Schema for Upgrades

```java
package com.example.fednow.state;

import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.TypeInformation;

/**
 * State descriptors for payment processing functions.
 *
 * <p>Centralized state schema definitions for consistency across operators.
 */
public class PaymentStateSchema {

    /** State for pending payment awaiting sanctions response */
    public static final ValueStateDescriptor<Payment> PENDING_PAYMENT =
        new ValueStateDescriptor<>(
            "pending-payment",
            TypeInformation.of(Payment.class)
        );

    /** State for timer timestamp */
    public static final ValueStateDescriptor<Long> TIMER_STATE =
        new ValueStateDescriptor<>(
            "timeout-timer",
            TypeInformation.of(Long.class)
        );

    /** State for pending sanctions response (if arrived before payment) */
    public static final ValueStateDescriptor<SanctionsResponse> PENDING_RESPONSE =
        new ValueStateDescriptor<>(
            "pending-response",
            TypeInformation.of(SanctionsResponse.class)
        );

    /**
     * State versioning for schema evolution.
     * Increment when making breaking changes to state schema.
     */
    public static final int STATE_VERSION = 1;
}
```

## Builder Extensions

```java
package com.example.fednow.model;

/**
 * Factory methods for creating Payment instances.
 */
public class PaymentFactory {

    /**
     * Create a failed payment from a parse error.
     */
    public static Payment parseError(String xml, String errorMessage) {
        return Payment.builder()
            .status(PaymentStatus.PARSE_ERROR)
            .failureReason("PARSE_FAILED")
            .errorMessage(errorMessage)
            .originalMessage(xml)
            .receivedAt(LocalDateTime.now())
            .build();
    }

    /**
     * Create a payment for testing.
     */
    public static Payment forTesting(String transactionId, BigDecimal amount) {
        return Payment.builder()
            .transactionId(transactionId)
            .messageId("MSG-" + transactionId)
            .instructionId("INSTR-" + transactionId)
            .endToEndId("E2E-" + transactionId)
            .amount(amount)
            .currency("USD")
            .debtorName("Test Debtor")
            .debitAccount("123456789")
            .debtorRoutingNumber("021000021")
            .debtorCountry("US")
            .creditorName("Test Creditor")
            .creditAccount("987654321")
            .creditorRoutingNumber("021000021")
            .creditorCountry("US")
            .status(PaymentStatus.RECEIVED)
            .receivedAt(LocalDateTime.now())
            .build();
    }
}
```

## Best Practices

### Serialization
- Implement `Serializable` for all model classes
- Use records for immutable value objects
- Register JavaTimeModule for LocalDateTime handling
- Consider Avro/Protobuf for production state

### State Evolution
- Version your state schema
- Use nullable fields for new additions
- Test state migration on upgrades
- Document breaking changes

### Performance
- Keep Payment objects lightweight
- Use lazy loading for large fields (originalMessage)
- Consider separate state for audit trail
- Minimize allocations in hot paths

### Testing
- Use `PaymentFactory.forTesting()` in tests
- Test serialization round-trips
- Verify state schema compatibility
