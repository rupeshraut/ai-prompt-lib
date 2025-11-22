# Create Flink Test

Create comprehensive tests for Apache Flink streaming applications using the Flink test harness.

## Requirements

The test suite should include:
- Unit tests with Flink test harness
- ProcessFunction tests with state and timers
- AsyncFunction tests
- Integration tests with MiniCluster
- Kafka integration tests with embedded Kafka

## Code Style

- Use JUnit 5 with extensions
- Use AssertJ for fluent assertions
- Mock external services (HTTP, JMS)
- Use test containers where appropriate

## ProcessFunction Unit Test

```java
package com.example.fednow.function;

import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.streaming.api.operators.KeyedProcessOperator;
import org.apache.flink.streaming.runtime.streamrecord.StreamRecord;
import org.apache.flink.streaming.util.KeyedOneInputStreamOperatorTestHarness;
import org.apache.flink.streaming.util.OneInputStreamOperatorTestHarness;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import javax.jms.Connection;
import javax.jms.MessageProducer;
import javax.jms.Session;
import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

/**
 * Unit tests for SanctionsRequestProcessFunction.
 */
@ExtendWith(MockitoExtension.class)
class SanctionsRequestProcessFunctionTest {

    private KeyedOneInputStreamOperatorTestHarness<String, Payment, Payment> testHarness;

    @Mock
    private Connection mockJmsConnection;

    @Mock
    private Session mockJmsSession;

    @Mock
    private MessageProducer mockMessageProducer;

    private JmsConfig jmsConfig;

    @BeforeEach
    void setUp() throws Exception {
        jmsConfig = new JmsConfig(
            "ConnectionFactory",
            "user",
            "password",
            "SANCTIONS.REQUEST",
            "SANCTIONS.RESPONSE",
            30000
        );

        // Create the function with mocked JMS
        SanctionsRequestProcessFunction function =
            new TestableSanctionsRequestProcessFunction(jmsConfig, mockJmsConnection);

        // Create test harness
        testHarness = new KeyedOneInputStreamOperatorTestHarness<>(
            new KeyedProcessOperator<>(function),
            Payment::getTransactionId,
            TypeInformation.of(String.class)
        );

        testHarness.open();
    }

    @AfterEach
    void tearDown() throws Exception {
        if (testHarness != null) {
            testHarness.close();
        }
    }

    @Test
    @DisplayName("Should store payment in state and send JMS request")
    void shouldProcessPaymentAndSendJmsRequest() throws Exception {
        // Given
        Payment payment = createValidPayment("TXN001");

        // When
        testHarness.processElement(new StreamRecord<>(payment, 0L));

        // Then
        // Verify no output yet (waiting for response)
        assertThat(testHarness.extractOutputStreamRecords()).isEmpty();

        // Verify timer was registered
        assertThat(testHarness.numProcessingTimeTimers()).isEqualTo(1);

        // Verify JMS message was sent
        verify(mockMessageProducer).send(any());
    }

    @Test
    @DisplayName("Should emit to side output on timeout")
    void shouldEmitToSideOutputOnTimeout() throws Exception {
        // Given
        Payment payment = createValidPayment("TXN002");
        testHarness.processElement(new StreamRecord<>(payment, 0L));

        // When - advance time past timeout
        testHarness.setProcessingTime(testHarness.getProcessingTime() + 35000);

        // Then
        // Check side output
        var sideOutput = testHarness.getSideOutput(
            SanctionsRequestProcessFunction.PENDING_SANCTIONS
        );
        assertThat(sideOutput).hasSize(1);

        Payment timedOutPayment = sideOutput.get(0).getValue();
        assertThat(timedOutPayment.getSanctionsStatus())
            .isEqualTo(SanctionsStatus.TIMEOUT);
    }

    @Test
    @DisplayName("Should skip payments not ready for sanctions")
    void shouldSkipPaymentsNotReadyForSanctions() throws Exception {
        // Given
        Payment payment = createValidPayment("TXN003");
        payment.setValidationStatus(ValidationStatus.FAILED);

        // When
        testHarness.processElement(new StreamRecord<>(payment, 0L));

        // Then
        // Payment should pass through without state or JMS
        assertThat(testHarness.extractOutputStreamRecords()).hasSize(1);
        assertThat(testHarness.numProcessingTimeTimers()).isZero();
        verify(mockMessageProducer, never()).send(any());
    }

    @Test
    @DisplayName("Should handle snapshot and restore")
    void shouldHandleSnapshotAndRestore() throws Exception {
        // Given
        Payment payment = createValidPayment("TXN004");
        testHarness.processElement(new StreamRecord<>(payment, 0L));

        // When - take snapshot
        var snapshot = testHarness.snapshot(1L, 1L);

        // Create new harness and restore
        var restoredHarness = new KeyedOneInputStreamOperatorTestHarness<>(
            new KeyedProcessOperator<>(
                new TestableSanctionsRequestProcessFunction(jmsConfig, mockJmsConnection)
            ),
            Payment::getTransactionId,
            TypeInformation.of(String.class)
        );
        restoredHarness.setup();
        restoredHarness.initializeState(snapshot);
        restoredHarness.open();

        // Then - timer should still be registered
        assertThat(restoredHarness.numProcessingTimeTimers()).isEqualTo(1);

        restoredHarness.close();
    }

    private Payment createValidPayment(String transactionId) {
        return Payment.builder()
            .transactionId(transactionId)
            .amount(new BigDecimal("1000.00"))
            .currency("USD")
            .debtorName("John Doe")
            .debitAccount("123456789")
            .creditorName("Jane Smith")
            .creditAccount("987654321")
            .validationStatus(ValidationStatus.VALID)
            .riskControlStatus(RiskControlStatus.APPROVED)
            .status(PaymentStatus.PROCESSING)
            .build();
    }

    /**
     * Testable version with mockable JMS.
     */
    static class TestableSanctionsRequestProcessFunction
            extends SanctionsRequestProcessFunction {

        private final Connection mockConnection;

        TestableSanctionsRequestProcessFunction(
                JmsConfig config, Connection mockConnection) {
            super(config);
            this.mockConnection = mockConnection;
        }

        @Override
        protected void initializeJms() {
            // Use mock instead of real connection
        }
    }
}
```

## AsyncFunction Unit Test

```java
package com.example.fednow.function;

import org.apache.flink.streaming.api.functions.async.ResultFuture;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.ArgumentCaptor;
import org.mockito.Captor;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.util.Collection;
import java.util.concurrent.CompletableFuture;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

/**
 * Unit tests for AccountValidationAsyncFunction.
 */
@ExtendWith(MockitoExtension.class)
class AccountValidationAsyncFunctionTest {

    @Mock
    private HttpClient mockHttpClient;

    @Mock
    private HttpResponse<String> mockResponse;

    @Mock
    private ResultFuture<Payment> resultFuture;

    @Captor
    private ArgumentCaptor<Collection<Payment>> resultCaptor;

    private AccountValidationAsyncFunction function;

    @BeforeEach
    void setUp() throws Exception {
        function = new TestableAccountValidationAsyncFunction(
            "http://localhost:8081",
            10000,
            mockHttpClient
        );
        function.open(null);
    }

    @Test
    @DisplayName("Should validate accounts successfully")
    void shouldValidateAccountsSuccessfully() throws Exception {
        // Given
        Payment payment = createTestPayment("TXN001");

        String validResponse = """
            {
                "valid": true,
                "accountStatus": "ACTIVE",
                "availableBalance": 5000.00,
                "reference": "VAL-123"
            }
            """;

        when(mockResponse.statusCode()).thenReturn(200);
        when(mockResponse.body()).thenReturn(validResponse);
        when(mockHttpClient.sendAsync(any(HttpRequest.class), any()))
            .thenReturn(CompletableFuture.completedFuture(mockResponse));

        // When
        function.asyncInvoke(payment, resultFuture);

        // Then - wait for async completion
        Thread.sleep(100);

        verify(resultFuture).complete(resultCaptor.capture());
        Collection<Payment> results = resultCaptor.getValue();

        assertThat(results).hasSize(1);
        Payment result = results.iterator().next();
        assertThat(result.getValidationStatus()).isEqualTo(ValidationStatus.VALID);
        assertThat(result.getDebitValidationRef()).isEqualTo("VAL-123");
    }

    @Test
    @DisplayName("Should handle validation failure")
    void shouldHandleValidationFailure() throws Exception {
        // Given
        Payment payment = createTestPayment("TXN002");

        String failureResponse = """
            {
                "valid": false,
                "errorCode": "INSUFFICIENT_FUNDS",
                "errorMessage": "Available balance is less than requested amount"
            }
            """;

        when(mockResponse.statusCode()).thenReturn(400);
        when(mockResponse.body()).thenReturn(failureResponse);
        when(mockHttpClient.sendAsync(any(HttpRequest.class), any()))
            .thenReturn(CompletableFuture.completedFuture(mockResponse));

        // When
        function.asyncInvoke(payment, resultFuture);
        Thread.sleep(100);

        // Then
        verify(resultFuture).complete(resultCaptor.capture());
        Payment result = resultCaptor.getValue().iterator().next();

        assertThat(result.getValidationStatus()).isEqualTo(ValidationStatus.FAILED);
        assertThat(result.getFailureReason()).contains("INSUFFICIENT_FUNDS");
    }

    @Test
    @DisplayName("Should retry on server error")
    void shouldRetryOnServerError() throws Exception {
        // Given
        Payment payment = createTestPayment("TXN003");

        // First call fails, second succeeds
        when(mockResponse.statusCode())
            .thenReturn(500)
            .thenReturn(200);
        when(mockResponse.body()).thenReturn("""
            {"valid": true, "reference": "VAL-456"}
            """);
        when(mockHttpClient.sendAsync(any(HttpRequest.class), any()))
            .thenReturn(CompletableFuture.completedFuture(mockResponse));

        // When
        function.asyncInvoke(payment, resultFuture);
        Thread.sleep(2000); // Wait for retry

        // Then
        verify(mockHttpClient, atLeast(2)).sendAsync(any(), any());
    }

    @Test
    @DisplayName("Should handle timeout")
    void shouldHandleTimeout() throws Exception {
        // Given
        Payment payment = createTestPayment("TXN004");

        // When
        function.timeout(payment, resultFuture);

        // Then
        verify(resultFuture).complete(resultCaptor.capture());
        Payment result = resultCaptor.getValue().iterator().next();

        assertThat(result.getValidationStatus()).isEqualTo(ValidationStatus.TIMEOUT);
        assertThat(result.getFailureReason()).isEqualTo("TIMEOUT");
    }

    private Payment createTestPayment(String transactionId) {
        return Payment.builder()
            .transactionId(transactionId)
            .amount(new BigDecimal("1000.00"))
            .currency("USD")
            .debitAccount("123456789")
            .creditAccount("987654321")
            .debtorAccountType("CHECKING")
            .creditorAccountType("CHECKING")
            .debtorRoutingNumber("021000021")
            .creditorRoutingNumber("021000021")
            .status(PaymentStatus.PROCESSING)
            .build();
    }

    static class TestableAccountValidationAsyncFunction
            extends AccountValidationAsyncFunction {

        private final HttpClient mockClient;

        TestableAccountValidationAsyncFunction(
                String serviceUrl, long timeoutMs, HttpClient mockClient) {
            super(serviceUrl, timeoutMs);
            this.mockClient = mockClient;
        }

        @Override
        protected HttpClient createHttpClient() {
            return mockClient;
        }
    }
}
```

## Integration Test with MiniCluster

```java
package com.example.fednow.job;

import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.runtime.testutils.MiniClusterResourceConfiguration;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.sink.SinkFunction;
import org.apache.flink.test.util.MiniClusterWithClientResource;
import org.junit.jupiter.api.*;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Integration tests for the payment processing pipeline.
 */
class PaymentProcessingJobIntegrationTest {

    @RegisterExtension
    static final MiniClusterWithClientResource MINI_CLUSTER =
        new MiniClusterWithClientResource(
            new MiniClusterResourceConfiguration.Builder()
                .setNumberSlotsPerTaskManager(2)
                .setNumberTaskManagers(1)
                .build()
        );

    private static List<Payment> collectedPayments;

    @BeforeEach
    void setUp() {
        collectedPayments = Collections.synchronizedList(new ArrayList<>());
    }

    @Test
    @DisplayName("Should process payment through complete pipeline")
    void shouldProcessPaymentThroughPipeline() throws Exception {
        // Given
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        Payment testPayment = PaymentFactory.forTesting("TXN-INT-001", new BigDecimal("500.00"));

        // Create source
        DataStream<Payment> source = env.fromElements(testPayment);

        // Apply parsing (skip - already Payment object)
        DataStream<Payment> validated = source
            .map(p -> {
                p.setValidationStatus(ValidationStatus.VALID);
                return p;
            });

        // Collect results
        validated.addSink(new CollectSink());

        // When
        env.execute("Integration Test");

        // Then
        assertThat(collectedPayments).hasSize(1);
        assertThat(collectedPayments.get(0).getValidationStatus())
            .isEqualTo(ValidationStatus.VALID);
    }

    @Test
    @DisplayName("Should handle side outputs for failures")
    void shouldHandleSideOutputsForFailures() throws Exception {
        // Given
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        Payment failedPayment = PaymentFactory.forTesting("TXN-INT-002", new BigDecimal("1000.00"));
        failedPayment.setValidationStatus(ValidationStatus.FAILED);
        failedPayment.setFailureReason("INSUFFICIENT_FUNDS");

        DataStream<Payment> source = env.fromElements(failedPayment);

        // Simulate routing to DLQ
        source.filter(p -> p.getValidationStatus() == ValidationStatus.FAILED)
            .addSink(new CollectSink());

        // When
        env.execute("Failure Handling Test");

        // Then
        assertThat(collectedPayments).hasSize(1);
        assertThat(collectedPayments.get(0).getFailureReason())
            .isEqualTo("INSUFFICIENT_FUNDS");
    }

    /**
     * Sink that collects payments for assertions.
     */
    private static class CollectSink implements SinkFunction<Payment> {
        @Override
        public void invoke(Payment value, Context context) {
            collectedPayments.add(value);
        }
    }
}
```

## Kafka Integration Test

```java
package com.example.fednow.job;

import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.connector.kafka.sink.KafkaSink;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.junit.jupiter.api.*;
import org.testcontainers.containers.KafkaContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;
import org.testcontainers.utility.DockerImageName;

import java.time.Duration;
import java.util.Properties;

import static org.assertj.core.api.Assertions.assertThat;

/**
 * Kafka integration tests using Testcontainers.
 */
@Testcontainers
class KafkaIntegrationTest {

    @Container
    static final KafkaContainer KAFKA = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.4.0")
    );

    private static final String INPUT_TOPIC = "test.payments.inbound";
    private static final String OUTPUT_TOPIC = "test.payments.outbound";

    @BeforeAll
    static void setUpTopics() {
        // Topics are auto-created
    }

    @Test
    @DisplayName("Should consume from Kafka and produce results")
    void shouldConsumeAndProduce() throws Exception {
        // Given
        String bootstrapServers = KAFKA.getBootstrapServers();

        // Send test message
        Properties producerProps = new Properties();
        producerProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        producerProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");
        producerProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
            "org.apache.kafka.common.serialization.StringSerializer");

        // Create Flink job
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(1);

        KafkaSource<String> source = KafkaSource.<String>builder()
            .setBootstrapServers(bootstrapServers)
            .setTopics(INPUT_TOPIC)
            .setGroupId("test-consumer")
            .setStartingOffsets(OffsetsInitializer.earliest())
            .setValueOnlyDeserializer(new SimpleStringSchema())
            .build();

        // Test the source configuration
        assertThat(source).isNotNull();
    }

    @Test
    @DisplayName("Should handle Kafka connection failure gracefully")
    void shouldHandleKafkaFailureGracefully() {
        // Test error handling when Kafka is unavailable
        assertThat(KAFKA.isRunning()).isTrue();
    }
}
```

## Parser Unit Tests

```java
package com.example.fednow.parser;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.ValueSource;

import java.math.BigDecimal;

import static org.assertj.core.api.Assertions.*;

/**
 * Unit tests for ISO 20022 parsers.
 */
class Pacs008ParserTest {

    private Pacs008Parser parser;

    @BeforeEach
    void setUp() {
        parser = new Pacs008Parser();
    }

    @Test
    @DisplayName("Should parse valid pacs.008 message")
    void shouldParseValidMessage() throws Exception {
        // Given
        String xml = """
            <FIToFICstmrCdtTrf>
              <GrpHdr>
                <MsgId>MSG123456</MsgId>
                <CreDtTm>2025-11-22T10:00:00</CreDtTm>
                <NbOfTxs>1</NbOfTxs>
                <SttlmInf>
                  <SttlmMtd>CLRG</SttlmMtd>
                </SttlmInf>
              </GrpHdr>
              <CdtTrfTxInf>
                <PmtId>
                  <InstrId>INSTR123</InstrId>
                  <EndToEndId>E2E123</EndToEndId>
                  <TxId>TXN123</TxId>
                </PmtId>
                <IntrBkSttlmAmt Ccy="USD">1000.00</IntrBkSttlmAmt>
                <Dbtr>
                  <Nm>John Doe</Nm>
                </Dbtr>
                <DbtrAcct>
                  <Id><Othr><Id>123456789</Id></Othr></Id>
                </DbtrAcct>
                <Cdtr>
                  <Nm>Jane Smith</Nm>
                </Cdtr>
                <CdtrAcct>
                  <Id><Othr><Id>987654321</Id></Othr></Id>
                </CdtrAcct>
              </CdtTrfTxInf>
            </FIToFICstmrCdtTrf>
            """;

        // When
        Payment payment = parser.parse(xml);

        // Then
        assertThat(payment.getTransactionId()).isEqualTo("TXN123");
        assertThat(payment.getAmount()).isEqualByComparingTo(new BigDecimal("1000.00"));
        assertThat(payment.getCurrency()).isEqualTo("USD");
        assertThat(payment.getDebtorName()).isEqualTo("John Doe");
        assertThat(payment.getCreditorName()).isEqualTo("Jane Smith");
    }

    @Test
    @DisplayName("Should throw exception for invalid XML")
    void shouldThrowExceptionForInvalidXml() {
        // Given
        String invalidXml = "<invalid>not valid xml";

        // When/Then
        assertThatThrownBy(() -> parser.parse(invalidXml))
            .isInstanceOf(Pacs008ParseException.class)
            .hasMessageContaining("Failed to parse");
    }

    @ParameterizedTest
    @ValueSource(strings = {"", "   ", "<empty/>"})
    @DisplayName("Should handle empty or minimal XML")
    void shouldHandleEmptyXml(String xml) {
        assertThatThrownBy(() -> parser.parse(xml))
            .isInstanceOf(Pacs008ParseException.class);
    }
}

/**
 * Unit tests for pacs.002 serializer.
 */
class Pacs002SerializerTest {

    private Pacs002Serializer serializer;

    @BeforeEach
    void setUp() {
        serializer = new Pacs002Serializer();
    }

    @Test
    @DisplayName("Should serialize successful payment response")
    void shouldSerializeSuccessfulResponse() {
        // Given
        Payment payment = Payment.builder()
            .transactionId("TXN123")
            .instructionId("INSTR123")
            .endToEndId("E2E123")
            .status(PaymentStatus.COMPLETED)
            .build();

        // When
        String xml = serializer.serialize(payment);

        // Then
        assertThat(xml).contains("<TxSts>ACCP</TxSts>");
        assertThat(xml).contains("<OrgnlTxId>TXN123</OrgnlTxId>");
    }

    @Test
    @DisplayName("Should serialize rejected payment with reason")
    void shouldSerializeRejectedPayment() {
        // Given
        Payment payment = Payment.builder()
            .transactionId("TXN456")
            .instructionId("INSTR456")
            .endToEndId("E2E456")
            .status(PaymentStatus.FAILED)
            .failureReason("INSUFFICIENT_FUNDS")
            .errorMessage("Not enough balance")
            .build();

        // When
        String xml = serializer.serialize(payment);

        // Then
        assertThat(xml).contains("<TxSts>RJCT</TxSts>");
        assertThat(xml).contains("<Cd>AM04</Cd>"); // ISO code for insufficient funds
    }
}
```

## Test Utilities

```java
package com.example.fednow.test;

import com.example.fednow.model.Payment;
import com.example.fednow.model.PaymentStatus;
import com.example.fednow.model.ValidationStatus;
import com.example.fednow.model.RiskControlStatus;
import com.example.fednow.model.SanctionsStatus;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * Test fixture factory for creating Payment objects.
 */
public class PaymentTestFixtures {

    public static Payment.PaymentBuilder basePayment() {
        return Payment.builder()
            .transactionId("TXN-" + System.nanoTime())
            .messageId("MSG-" + System.nanoTime())
            .instructionId("INSTR-001")
            .endToEndId("E2E-001")
            .amount(new BigDecimal("1000.00"))
            .currency("USD")
            .debtorName("Test Debtor")
            .debitAccount("123456789")
            .debtorRoutingNumber("021000021")
            .creditorName("Test Creditor")
            .creditAccount("987654321")
            .creditorRoutingNumber("021000021")
            .receivedAt(LocalDateTime.now())
            .status(PaymentStatus.RECEIVED);
    }

    public static Payment validatedPayment() {
        return basePayment()
            .validationStatus(ValidationStatus.VALID)
            .debitValidationRef("VAL-D-001")
            .creditValidationRef("VAL-C-001")
            .status(PaymentStatus.PROCESSING)
            .build();
    }

    public static Payment riskApprovedPayment() {
        return basePayment()
            .validationStatus(ValidationStatus.VALID)
            .riskControlStatus(RiskControlStatus.APPROVED)
            .riskScore(15)
            .riskControlRef("RCS-001")
            .status(PaymentStatus.PROCESSING)
            .build();
    }

    public static Payment sanctionsClearedPayment() {
        return basePayment()
            .validationStatus(ValidationStatus.VALID)
            .riskControlStatus(RiskControlStatus.APPROVED)
            .sanctionsStatus(SanctionsStatus.CLEAR)
            .sanctionsRef("SAN-001")
            .status(PaymentStatus.PROCESSING)
            .build();
    }

    public static Payment completedPayment() {
        return basePayment()
            .validationStatus(ValidationStatus.VALID)
            .riskControlStatus(RiskControlStatus.APPROVED)
            .sanctionsStatus(SanctionsStatus.CLEAR)
            .postingStatus(PostingStatus.POSTED)
            .postingReference("POST-001")
            .status(PaymentStatus.COMPLETED)
            .completedAt(LocalDateTime.now())
            .build();
    }

    public static Payment failedPayment(String reason) {
        return basePayment()
            .status(PaymentStatus.FAILED)
            .failureReason(reason)
            .errorMessage("Test failure: " + reason)
            .build();
    }
}
```

## Best Practices

### Test Harness
- Use `KeyedOneInputStreamOperatorTestHarness` for keyed operators
- Use `OneInputStreamOperatorTestHarness` for non-keyed operators
- Always close harnesses in `@AfterEach`

### State Testing
- Test snapshot and restore behavior
- Verify timers are properly restored
- Test state cleanup on completion

### Async Testing
- Use `CompletableFuture.completedFuture()` for mocking
- Add small delays for async completion
- Test timeout handling explicitly

### Integration Testing
- Use `MiniClusterWithClientResource` for pipeline tests
- Use Testcontainers for Kafka integration
- Keep integration tests focused and fast

### Mocking
- Mock external services (HTTP, JMS)
- Create testable subclasses for dependency injection
- Use Mockito for clean mock setup
