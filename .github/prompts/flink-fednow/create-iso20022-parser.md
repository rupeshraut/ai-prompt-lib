# Create ISO 20022 Parser

Create XML parsers and serializers for FedNow ISO 20022 message types.

## Requirements

The parser should include:
- JAXB or Jackson XML annotations
- Validation against ISO 20022 schema
- Parser functions for Flink map operations
- Serializer functions for response generation
- Error handling with detailed messages

## Code Style

- Use Java 17 features
- Use records for immutable message parts
- Implement proper XML namespace handling
- Add comprehensive validation

## Supported Message Types

| Message | Description | Direction |
|---------|-------------|-----------|
| pacs.008 | Credit Transfer | Inbound |
| pacs.002 | Payment Status Report | Outbound |
| camt.056 | Payment Cancellation Request | Inbound |
| camt.029 | Resolution of Investigation | Outbound |

## Pacs008 Parser (Credit Transfer)

```java
package com.example.fednow.parser;

import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlProperty;
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlRootElement;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;

import javax.xml.XMLConstants;
import javax.xml.transform.stream.StreamSource;
import javax.xml.validation.Schema;
import javax.xml.validation.SchemaFactory;
import javax.xml.validation.Validator;
import java.io.StringReader;
import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Parser for ISO 20022 pacs.008 (FIToFICustomerCreditTransfer) messages.
 */
@Slf4j
public class Pacs008Parser {

    private final XmlMapper xmlMapper;
    private final Validator schemaValidator;

    public Pacs008Parser() {
        this.xmlMapper = createXmlMapper();
        this.schemaValidator = createSchemaValidator();
    }

    private XmlMapper createXmlMapper() {
        XmlMapper mapper = new XmlMapper();
        mapper.findAndRegisterModules();
        return mapper;
    }

    private Validator createSchemaValidator() {
        try {
            SchemaFactory factory = SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI);
            Schema schema = factory.newSchema(
                getClass().getClassLoader().getResource("schema/pacs.008.001.08.xsd")
            );
            return schema.newValidator();
        } catch (Exception e) {
            log.warn("Could not load XSD schema, validation disabled", e);
            return null;
        }
    }

    /**
     * Parse pacs.008 XML message to domain object.
     */
    public Payment parse(String xml) throws Pacs008ParseException {
        try {
            // Validate against schema (optional)
            validateSchema(xml);

            // Parse XML to JAXB model
            FIToFICstmrCdtTrf message = xmlMapper.readValue(xml, FIToFICstmrCdtTrf.class);

            // Convert to domain Payment object
            return convertToPayment(message);

        } catch (Exception e) {
            log.error("Failed to parse pacs.008 message", e);
            throw new Pacs008ParseException("Failed to parse pacs.008: " + e.getMessage(), e);
        }
    }

    private void validateSchema(String xml) throws Pacs008ParseException {
        if (schemaValidator == null) {
            return;
        }

        try {
            schemaValidator.validate(new StreamSource(new StringReader(xml)));
        } catch (Exception e) {
            throw new Pacs008ParseException("Schema validation failed: " + e.getMessage(), e);
        }
    }

    private Payment convertToPayment(FIToFICstmrCdtTrf message) {
        GrpHdr header = message.getGrpHdr();
        CdtTrfTxInf txInfo = message.getCdtTrfTxInf();
        PmtId pmtId = txInfo.getPmtId();

        return Payment.builder()
            // Message identifiers
            .messageId(header.getMsgId())
            .transactionId(pmtId.getTxId())
            .instructionId(pmtId.getInstrId())
            .endToEndId(pmtId.getEndToEndId())

            // Amount and currency
            .amount(txInfo.getIntrBkSttlmAmt().getValue())
            .currency(txInfo.getIntrBkSttlmAmt().getCcy())

            // Debtor (sender) information
            .debtorName(txInfo.getDbtr().getNm())
            .debitAccount(extractAccountId(txInfo.getDbtrAcct()))
            .debtorRoutingNumber(extractBic(txInfo.getDbtrAgt()))
            .debtorCountry(extractCountry(txInfo.getDbtr()))

            // Creditor (receiver) information
            .creditorName(txInfo.getCdtr().getNm())
            .creditAccount(extractAccountId(txInfo.getCdtrAcct()))
            .creditorRoutingNumber(extractBic(txInfo.getCdtrAgt()))
            .creditorCountry(extractCountry(txInfo.getCdtr()))

            // Processing metadata
            .creationDateTime(header.getCreDtTm())
            .settlementMethod(header.getSttlmInf().getSttlmMtd())
            .status(PaymentStatus.RECEIVED)
            .receivedAt(LocalDateTime.now())

            // Store original XML for audit
            .originalMessage(null) // Set separately if needed

            .build();
    }

    private String extractAccountId(AcctId acctId) {
        if (acctId == null || acctId.getId() == null) {
            return null;
        }
        if (acctId.getId().getIBAN() != null) {
            return acctId.getId().getIBAN();
        }
        if (acctId.getId().getOthr() != null) {
            return acctId.getId().getOthr().getId();
        }
        return null;
    }

    private String extractBic(FinInstnId finInstnId) {
        if (finInstnId == null) {
            return null;
        }
        if (finInstnId.getBICFI() != null) {
            return finInstnId.getBICFI();
        }
        if (finInstnId.getClrSysMmbId() != null) {
            return finInstnId.getClrSysMmbId().getMmbId();
        }
        return null;
    }

    private String extractCountry(PartyId party) {
        if (party == null || party.getPstlAdr() == null) {
            return null;
        }
        return party.getPstlAdr().getCtry();
    }
}

/**
 * JAXB model classes for pacs.008 message structure.
 */
@Data
@JacksonXmlRootElement(localName = "FIToFICstmrCdtTrf")
class FIToFICstmrCdtTrf {
    @JacksonXmlProperty(localName = "GrpHdr")
    private GrpHdr grpHdr;

    @JacksonXmlProperty(localName = "CdtTrfTxInf")
    private CdtTrfTxInf cdtTrfTxInf;
}

@Data
class GrpHdr {
    @JacksonXmlProperty(localName = "MsgId")
    private String msgId;

    @JacksonXmlProperty(localName = "CreDtTm")
    private LocalDateTime creDtTm;

    @JacksonXmlProperty(localName = "NbOfTxs")
    private int nbOfTxs;

    @JacksonXmlProperty(localName = "SttlmInf")
    private SttlmInf sttlmInf;
}

@Data
class SttlmInf {
    @JacksonXmlProperty(localName = "SttlmMtd")
    private String sttlmMtd;
}

@Data
class CdtTrfTxInf {
    @JacksonXmlProperty(localName = "PmtId")
    private PmtId pmtId;

    @JacksonXmlProperty(localName = "IntrBkSttlmAmt")
    private Amount intrBkSttlmAmt;

    @JacksonXmlProperty(localName = "Dbtr")
    private PartyId dbtr;

    @JacksonXmlProperty(localName = "DbtrAcct")
    private AcctId dbtrAcct;

    @JacksonXmlProperty(localName = "DbtrAgt")
    private FinInstnId dbtrAgt;

    @JacksonXmlProperty(localName = "Cdtr")
    private PartyId cdtr;

    @JacksonXmlProperty(localName = "CdtrAcct")
    private AcctId cdtrAcct;

    @JacksonXmlProperty(localName = "CdtrAgt")
    private FinInstnId cdtrAgt;
}

@Data
class PmtId {
    @JacksonXmlProperty(localName = "InstrId")
    private String instrId;

    @JacksonXmlProperty(localName = "EndToEndId")
    private String endToEndId;

    @JacksonXmlProperty(localName = "TxId")
    private String txId;
}

@Data
class Amount {
    @JacksonXmlProperty(isAttribute = true, localName = "Ccy")
    private String ccy;

    @JacksonXmlProperty(localName = "")
    private BigDecimal value;
}

@Data
class PartyId {
    @JacksonXmlProperty(localName = "Nm")
    private String nm;

    @JacksonXmlProperty(localName = "PstlAdr")
    private PstlAdr pstlAdr;
}

@Data
class PstlAdr {
    @JacksonXmlProperty(localName = "Ctry")
    private String ctry;
}

@Data
class AcctId {
    @JacksonXmlProperty(localName = "Id")
    private AcctIdType id;
}

@Data
class AcctIdType {
    @JacksonXmlProperty(localName = "IBAN")
    private String iban;

    @JacksonXmlProperty(localName = "Othr")
    private OthrId othr;
}

@Data
class OthrId {
    @JacksonXmlProperty(localName = "Id")
    private String id;
}

@Data
class FinInstnId {
    @JacksonXmlProperty(localName = "BICFI")
    private String bicfi;

    @JacksonXmlProperty(localName = "ClrSysMmbId")
    private ClrSysMmbId clrSysMmbId;
}

@Data
class ClrSysMmbId {
    @JacksonXmlProperty(localName = "MmbId")
    private String mmbId;
}
```

## Pacs002 Serializer (Payment Status Report)

```java
package com.example.fednow.parser;

import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlProperty;
import com.fasterxml.jackson.dataformat.xml.annotation.JacksonXmlRootElement;
import lombok.Builder;
import lombok.Data;
import lombok.extern.slf4j.Slf4j;

import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

/**
 * Serializer for ISO 20022 pacs.002 (FIToFIPaymentStatusReport) messages.
 */
@Slf4j
public class Pacs002Serializer {

    private static final DateTimeFormatter ISO_DATETIME =
        DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss");

    private final XmlMapper xmlMapper;

    public Pacs002Serializer() {
        this.xmlMapper = new XmlMapper();
        this.xmlMapper.findAndRegisterModules();
    }

    /**
     * Generate pacs.002 response from payment processing result.
     */
    public String serialize(Payment payment) {
        try {
            FIToFIPmtStsRpt response = buildResponse(payment);
            return xmlMapper.writeValueAsString(response);
        } catch (Exception e) {
            log.error("Failed to serialize pacs.002 message: transactionId={}",
                payment.getTransactionId(), e);
            throw new Pacs002SerializeException("Failed to serialize pacs.002", e);
        }
    }

    private FIToFIPmtStsRpt buildResponse(Payment payment) {
        // Determine transaction status
        TxSts txSts = mapPaymentStatusToTxSts(payment);

        // Build response message
        return FIToFIPmtStsRpt.builder()
            .grpHdr(GrpHdr002.builder()
                .msgId(generateMessageId(payment))
                .creDtTm(LocalDateTime.now().format(ISO_DATETIME))
                .build())
            .txInfAndSts(TxInfAndSts.builder()
                .orgnlInstrId(payment.getInstructionId())
                .orgnlEndToEndId(payment.getEndToEndId())
                .orgnlTxId(payment.getTransactionId())
                .txSts(txSts.getCode())
                .stsRsnInf(buildStatusReasonInfo(payment, txSts))
                .build())
            .build();
    }

    private TxSts mapPaymentStatusToTxSts(Payment payment) {
        return switch (payment.getStatus()) {
            case COMPLETED -> TxSts.ACCP;  // Accepted
            case POSTED -> TxSts.ACSP;     // AcceptedSettlementInProcess
            case FAILED -> TxSts.RJCT;     // Rejected
            case PENDING -> TxSts.PDNG;    // Pending
            case SANCTIONS_HIT -> TxSts.RJCT;
            case TIMEOUT -> TxSts.RJCT;
            default -> TxSts.PDNG;
        };
    }

    private StsRsnInf buildStatusReasonInfo(Payment payment, TxSts txSts) {
        if (txSts != TxSts.RJCT) {
            return null;
        }

        String reasonCode = mapFailureReasonToCode(payment.getFailureReason());

        return StsRsnInf.builder()
            .rsn(Rsn.builder()
                .cd(reasonCode)
                .build())
            .addtlInf(payment.getErrorMessage())
            .build();
    }

    private String mapFailureReasonToCode(String failureReason) {
        if (failureReason == null) {
            return "NARR";  // Narrative - unspecified
        }

        return switch (failureReason) {
            case "INSUFFICIENT_FUNDS" -> "AM04";
            case "INVALID_ACCOUNT" -> "AC01";
            case "ACCOUNT_CLOSED" -> "AC04";
            case "SANCTIONS_HIT" -> "AG01";
            case "RISK_DECLINED" -> "AM05";
            case "TIMEOUT" -> "AB06";
            case "POSTING_FAILED" -> "AM21";
            default -> "NARR";
        };
    }

    private String generateMessageId(Payment payment) {
        return "RESP-" + payment.getTransactionId() + "-" +
            System.currentTimeMillis();
    }
}

/**
 * Transaction status codes per ISO 20022.
 */
enum TxSts {
    ACCP("ACCP"),  // Accepted
    ACSP("ACSP"),  // AcceptedSettlementInProcess
    ACSC("ACSC"),  // AcceptedSettlementCompleted
    RJCT("RJCT"),  // Rejected
    PDNG("PDNG");  // Pending

    private final String code;

    TxSts(String code) {
        this.code = code;
    }

    public String getCode() {
        return code;
    }
}

/**
 * JAXB model for pacs.002 message.
 */
@Data
@Builder
@JacksonXmlRootElement(localName = "FIToFIPmtStsRpt")
class FIToFIPmtStsRpt {
    @JacksonXmlProperty(localName = "GrpHdr")
    private GrpHdr002 grpHdr;

    @JacksonXmlProperty(localName = "TxInfAndSts")
    private TxInfAndSts txInfAndSts;
}

@Data
@Builder
class GrpHdr002 {
    @JacksonXmlProperty(localName = "MsgId")
    private String msgId;

    @JacksonXmlProperty(localName = "CreDtTm")
    private String creDtTm;
}

@Data
@Builder
class TxInfAndSts {
    @JacksonXmlProperty(localName = "OrgnlInstrId")
    private String orgnlInstrId;

    @JacksonXmlProperty(localName = "OrgnlEndToEndId")
    private String orgnlEndToEndId;

    @JacksonXmlProperty(localName = "OrgnlTxId")
    private String orgnlTxId;

    @JacksonXmlProperty(localName = "TxSts")
    private String txSts;

    @JacksonXmlProperty(localName = "StsRsnInf")
    private StsRsnInf stsRsnInf;
}

@Data
@Builder
class StsRsnInf {
    @JacksonXmlProperty(localName = "Rsn")
    private Rsn rsn;

    @JacksonXmlProperty(localName = "AddtlInf")
    private String addtlInf;
}

@Data
@Builder
class Rsn {
    @JacksonXmlProperty(localName = "Cd")
    private String cd;
}
```

## Flink Map Functions

```java
package com.example.fednow.function;

import org.apache.flink.api.common.functions.RichMapFunction;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.metrics.Counter;
import lombok.extern.slf4j.Slf4j;

/**
 * Flink map function to parse incoming pacs.008 messages.
 */
@Slf4j
public class Iso20022ParserFunction extends RichMapFunction<String, Payment> {

    private transient Pacs008Parser parser;
    private transient Counter parseSuccessCounter;
    private transient Counter parseFailureCounter;

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        this.parser = new Pacs008Parser();

        parseSuccessCounter = getRuntimeContext()
            .getMetricGroup()
            .counter("iso20022.parse.success");
        parseFailureCounter = getRuntimeContext()
            .getMetricGroup()
            .counter("iso20022.parse.failure");

        log.info("Iso20022ParserFunction initialized");
    }

    @Override
    public Payment map(String xml) throws Exception {
        try {
            Payment payment = parser.parse(xml);
            parseSuccessCounter.inc();

            log.debug("Parsed pacs.008 message: transactionId={}, amount={} {}",
                payment.getTransactionId(),
                payment.getAmount(),
                payment.getCurrency());

            return payment;

        } catch (Pacs008ParseException e) {
            parseFailureCounter.inc();
            log.error("Failed to parse pacs.008 message", e);

            // Return a failed payment for DLQ routing
            return Payment.builder()
                .status(PaymentStatus.PARSE_ERROR)
                .failureReason("PARSE_FAILED")
                .errorMessage(e.getMessage())
                .originalMessage(xml)
                .receivedAt(LocalDateTime.now())
                .build();
        }
    }
}

/**
 * Flink map function to generate pacs.002 response messages.
 */
@Slf4j
public class Pacs002GeneratorFunction extends RichMapFunction<Payment, String> {

    private transient Pacs002Serializer serializer;
    private transient Counter serializeCounter;

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);
        this.serializer = new Pacs002Serializer();

        serializeCounter = getRuntimeContext()
            .getMetricGroup()
            .counter("iso20022.serialize.count");

        log.info("Pacs002GeneratorFunction initialized");
    }

    @Override
    public String map(Payment payment) throws Exception {
        String xml = serializer.serialize(payment);
        serializeCounter.inc();

        log.debug("Generated pacs.002 response: transactionId={}, status={}",
            payment.getTransactionId(),
            payment.getStatus());

        return xml;
    }
}
```

## Validation Utilities

```java
package com.example.fednow.parser;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.regex.Pattern;

/**
 * Validation utilities for ISO 20022 message fields.
 */
public class Iso20022Validator {

    private static final Pattern BIC_PATTERN =
        Pattern.compile("^[A-Z]{4}[A-Z]{2}[A-Z0-9]{2}([A-Z0-9]{3})?$");
    private static final Pattern IBAN_PATTERN =
        Pattern.compile("^[A-Z]{2}[0-9]{2}[A-Z0-9]{1,30}$");
    private static final Pattern ACCOUNT_PATTERN =
        Pattern.compile("^[A-Z0-9]{1,34}$");

    private static final BigDecimal MAX_FEDNOW_AMOUNT = new BigDecimal("500000.00");
    private static final BigDecimal MIN_AMOUNT = new BigDecimal("0.01");

    /**
     * Validate payment fields according to FedNow rules.
     */
    public static ValidationResult validate(Payment payment) {
        List<String> errors = new ArrayList<>();

        // Transaction ID validation
        if (payment.getTransactionId() == null || payment.getTransactionId().isBlank()) {
            errors.add("Transaction ID is required");
        }

        // Amount validation
        if (payment.getAmount() == null) {
            errors.add("Amount is required");
        } else if (payment.getAmount().compareTo(MIN_AMOUNT) < 0) {
            errors.add("Amount must be at least " + MIN_AMOUNT);
        } else if (payment.getAmount().compareTo(MAX_FEDNOW_AMOUNT) > 0) {
            errors.add("Amount exceeds FedNow limit of " + MAX_FEDNOW_AMOUNT);
        }

        // Currency validation
        if (!"USD".equals(payment.getCurrency())) {
            errors.add("Currency must be USD for FedNow");
        }

        // Account validation
        if (!isValidAccount(payment.getDebitAccount())) {
            errors.add("Invalid debit account format");
        }
        if (!isValidAccount(payment.getCreditAccount())) {
            errors.add("Invalid credit account format");
        }

        // Routing number validation
        if (!isValidRoutingNumber(payment.getDebtorRoutingNumber())) {
            errors.add("Invalid debtor routing number");
        }
        if (!isValidRoutingNumber(payment.getCreditorRoutingNumber())) {
            errors.add("Invalid creditor routing number");
        }

        // Party name validation
        if (payment.getDebtorName() == null || payment.getDebtorName().isBlank()) {
            errors.add("Debtor name is required");
        }
        if (payment.getCreditorName() == null || payment.getCreditorName().isBlank()) {
            errors.add("Creditor name is required");
        }

        return new ValidationResult(errors.isEmpty(), errors);
    }

    private static boolean isValidAccount(String account) {
        if (account == null) {
            return false;
        }
        // Check if IBAN or standard account format
        return IBAN_PATTERN.matcher(account).matches() ||
               ACCOUNT_PATTERN.matcher(account).matches();
    }

    private static boolean isValidRoutingNumber(String routingNumber) {
        if (routingNumber == null || routingNumber.length() != 9) {
            return false;
        }
        // Validate ABA routing number checksum
        try {
            int[] digits = routingNumber.chars()
                .map(c -> c - '0')
                .toArray();

            int checksum = 3 * (digits[0] + digits[3] + digits[6]) +
                          7 * (digits[1] + digits[4] + digits[7]) +
                          1 * (digits[2] + digits[5] + digits[8]);

            return checksum % 10 == 0;
        } catch (Exception e) {
            return false;
        }
    }

    public record ValidationResult(boolean valid, List<String> errors) {
        public String getErrorMessage() {
            return String.join("; ", errors);
        }
    }
}
```

## Custom Exceptions

```java
package com.example.fednow.parser;

/**
 * Exception for pacs.008 parsing errors.
 */
public class Pacs008ParseException extends Exception {
    public Pacs008ParseException(String message) {
        super(message);
    }

    public Pacs008ParseException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Exception for pacs.002 serialization errors.
 */
public class Pacs002SerializeException extends RuntimeException {
    public Pacs002SerializeException(String message) {
        super(message);
    }

    public Pacs002SerializeException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## Best Practices

### XML Parsing
- Use streaming parsers for large messages
- Validate against XSD when possible
- Handle namespace prefixes correctly
- Use immutable records for parsed data

### Error Handling
- Preserve original message for DLQ
- Include detailed error messages
- Map errors to ISO 20022 reason codes
- Log parsing failures with context

### Performance
- Reuse parser instances (thread-local if needed)
- Cache compiled XSD validators
- Consider SAX parsing for very large messages

### Testing
- Test with sample FedNow messages
- Validate round-trip parsing/serialization
- Test edge cases (missing fields, invalid data)
- Test namespace handling variations
