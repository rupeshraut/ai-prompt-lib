# Create Sanctions Processor

Create Flink KeyedProcessFunction for asynchronous JMS-based sanctions screening with FircoSoft.

## Requirements

The sanctions processor should include:
- `KeyedProcessFunction` for stateful processing
- Keyed state to store pending payments
- JMS message sending to FircoSoft
- Timer-based timeout handling
- JMS response correlation
- Side output for sanctions hits
- Exactly-once semantics support

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Sanctions Processing Flow                     │
└─────────────────────────────────────────────────────────────────┘

    Payment Stream                              JMS Response Stream
          │                                            │
          ▼                                            ▼
┌──────────────────┐                      ┌──────────────────┐
│  SanctionsRequest│                      │ SanctionsResponse│
│  ProcessFunction │                      │  SourceFunction  │
└────────┬─────────┘                      └────────┬─────────┘
         │                                         │
         │  Store in State                         │
         │  Send JMS Request                       │
         │  Register Timer                         │
         │                                         │
         ▼                                         │
┌──────────────────┐                               │
│   Pending State  │◀──────────────────────────────┘
│   (RocksDB)      │         Correlate by TransactionId
└────────┬─────────┘
         │
         │  On Response or Timeout
         ▼
┌──────────────────┐
│  Output Payment  │
│  (or Side Output)│
└──────────────────┘
```

## Sanctions Request Processor

```java
package com.example.fednow.function;

import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.util.OutputTag;
import org.apache.flink.metrics.Counter;
import lombok.extern.slf4j.Slf4j;

import javax.jms.*;
import javax.naming.InitialContext;

/**
 * Keyed process function for sanctions screening via JMS.
 *
 * <p>This function:
 * <ol>
 *   <li>Stores incoming payment in keyed state</li>
 *   <li>Sends sanctions request to FircoSoft via JMS</li>
 *   <li>Registers timeout timer</li>
 *   <li>Waits for response (handled by separate function)</li>
 * </ol>
 *
 * <p>The response is correlated using the transaction ID as the key.
 */
@Slf4j
public class SanctionsRequestProcessFunction
    extends KeyedProcessFunction<String, Payment, Payment> {

    private static final long SANCTIONS_TIMEOUT_MS = 30_000; // 30 seconds

    private final JmsConfig jmsConfig;

    // State
    private transient ValueState<Payment> pendingPaymentState;
    private transient ValueState<Long> timerState;

    // JMS resources
    private transient Connection jmsConnection;
    private transient Session jmsSession;
    private transient MessageProducer messageProducer;

    // Metrics
    private transient Counter requestsSent;
    private transient Counter timeouts;

    // Side output for payments waiting for response
    public static final OutputTag<Payment> PENDING_SANCTIONS =
        new OutputTag<>("pending-sanctions") {};

    public SanctionsRequestProcessFunction(JmsConfig jmsConfig) {
        this.jmsConfig = jmsConfig;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);

        // Initialize state
        ValueStateDescriptor<Payment> paymentDescriptor = new ValueStateDescriptor<>(
            "pending-payment",
            TypeInformation.of(Payment.class)
        );
        pendingPaymentState = getRuntimeContext().getState(paymentDescriptor);

        ValueStateDescriptor<Long> timerDescriptor = new ValueStateDescriptor<>(
            "timeout-timer",
            TypeInformation.of(Long.class)
        );
        timerState = getRuntimeContext().getState(timerDescriptor);

        // Initialize JMS connection
        initializeJms();

        // Register metrics
        requestsSent = getRuntimeContext()
            .getMetricGroup()
            .counter("sanctions.requestsSent");
        timeouts = getRuntimeContext()
            .getMetricGroup()
            .counter("sanctions.timeouts");

        log.info("SanctionsRequestProcessFunction initialized");
    }

    private void initializeJms() throws Exception {
        ConnectionFactory connectionFactory = (ConnectionFactory)
            new InitialContext().lookup(jmsConfig.getConnectionFactoryName());

        jmsConnection = connectionFactory.createConnection(
            jmsConfig.getUsername(),
            jmsConfig.getPassword()
        );
        jmsConnection.start();

        jmsSession = jmsConnection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        Destination requestQueue = jmsSession.createQueue(jmsConfig.getRequestQueue());
        messageProducer = jmsSession.createProducer(requestQueue);
        messageProducer.setDeliveryMode(DeliveryMode.PERSISTENT);

        log.info("JMS connection established: queue={}", jmsConfig.getRequestQueue());
    }

    @Override
    public void processElement(Payment payment, Context ctx, Collector<Payment> out)
            throws Exception {

        // Skip if previous steps failed
        if (!payment.isReadyForSanctions()) {
            out.collect(payment);
            return;
        }

        String transactionId = payment.getTransactionId();

        log.info("Starting sanctions screening: transactionId={}", transactionId);

        // Store payment in state
        pendingPaymentState.update(payment);

        // Send JMS request
        sendSanctionsRequest(payment);
        requestsSent.inc();

        // Register timeout timer
        long timeoutTime = ctx.timerService().currentProcessingTime() + SANCTIONS_TIMEOUT_MS;
        ctx.timerService().registerProcessingTimeTimer(timeoutTime);
        timerState.update(timeoutTime);

        payment.addProcessingStep("SANCTIONS_SCREENING", "REQUEST_SENT");

        log.debug("Sanctions request sent, timer registered: transactionId={}, timeout={}",
            transactionId, timeoutTime);

        // Don't emit yet - wait for response or timeout
    }

    /**
     * Send sanctions screening request to FircoSoft via JMS.
     */
    private void sendSanctionsRequest(Payment payment) throws JMSException {
        SanctionsRequest request = SanctionsRequest.builder()
            .requestId(generateRequestId())
            .transactionId(payment.getTransactionId())
            .debtorName(payment.getDebtorName())
            .debtorAccount(payment.getDebitAccount())
            .debtorCountry(payment.getDebtorCountry())
            .creditorName(payment.getCreditorName())
            .creditorAccount(payment.getCreditAccount())
            .creditorCountry(payment.getCreditorCountry())
            .amount(payment.getAmount())
            .currency(payment.getCurrency())
            .build();

        String xmlMessage = serializeToXml(request);

        TextMessage jmsMessage = jmsSession.createTextMessage(xmlMessage);
        jmsMessage.setStringProperty("TransactionId", payment.getTransactionId());
        jmsMessage.setStringProperty("RequestId", request.getRequestId());
        jmsMessage.setJMSCorrelationID(payment.getTransactionId());

        messageProducer.send(jmsMessage);

        log.debug("JMS message sent: transactionId={}, requestId={}",
            payment.getTransactionId(), request.getRequestId());
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Payment> out)
            throws Exception {

        Payment payment = pendingPaymentState.value();

        if (payment != null) {
            // Timeout occurred - no response received
            log.warn("Sanctions screening timeout: transactionId={}", payment.getTransactionId());

            payment.setSanctionsStatus(SanctionsStatus.TIMEOUT);
            payment.setFailureReason("SANCTIONS_TIMEOUT");
            payment.addProcessingStep("SANCTIONS_SCREENING", "TIMEOUT");

            timeouts.inc();

            // Clear state
            pendingPaymentState.clear();
            timerState.clear();

            // Emit to side output for timeout handling
            ctx.output(PaymentProcessingJob.SANCTIONS_HIT, payment);
        }
    }

    /**
     * Called when sanctions response is received (via co-process function).
     */
    public void onSanctionsResponse(SanctionsResponse response, Context ctx, Collector<Payment> out)
            throws Exception {

        Payment payment = pendingPaymentState.value();

        if (payment == null) {
            log.warn("Received sanctions response for unknown transaction: transactionId={}",
                response.getTransactionId());
            return;
        }

        // Cancel timeout timer
        Long timerTimestamp = timerState.value();
        if (timerTimestamp != null) {
            ctx.timerService().deleteProcessingTimeTimer(timerTimestamp);
        }

        // Process response
        processSanctionsResponse(payment, response, ctx, out);

        // Clear state
        pendingPaymentState.clear();
        timerState.clear();
    }

    private void processSanctionsResponse(
            Payment payment,
            SanctionsResponse response,
            Context ctx,
            Collector<Payment> out) {

        switch (response.getResult()) {
            case CLEAR -> {
                payment.setSanctionsStatus(SanctionsStatus.CLEAR);
                payment.setSanctionsRef(response.getRequestId());
                payment.addProcessingStep("SANCTIONS_SCREENING", "CLEAR");

                log.info("Sanctions screening clear: transactionId={}",
                    payment.getTransactionId());

                out.collect(payment);
            }
            case HIT -> {
                payment.setSanctionsStatus(SanctionsStatus.HIT);
                payment.setSanctionsHits(response.getHits());
                payment.addProcessingStep("SANCTIONS_SCREENING", "HIT: " + response.getHits().size() + " matches");

                log.warn("Sanctions hit: transactionId={}, hits={}",
                    payment.getTransactionId(), response.getHits().size());

                ctx.output(PaymentProcessingJob.SANCTIONS_HIT, payment);
            }
            case POTENTIAL_HIT -> {
                // Potential hit - may require manual review
                payment.setSanctionsStatus(SanctionsStatus.POTENTIAL_HIT);
                payment.setSanctionsHits(response.getHits());
                payment.addProcessingStep("SANCTIONS_SCREENING", "POTENTIAL_HIT");

                log.warn("Sanctions potential hit: transactionId={}",
                    payment.getTransactionId());

                ctx.output(PaymentProcessingJob.SANCTIONS_HIT, payment);
            }
        }
    }

    private String generateRequestId() {
        return "SAN-" + System.currentTimeMillis() + "-" +
            getRuntimeContext().getIndexOfThisSubtask();
    }

    private String serializeToXml(SanctionsRequest request) {
        // Use JAXB or Jackson XML to serialize
        return new SanctionsRequestSerializer().serialize(request);
    }

    @Override
    public void close() throws Exception {
        if (messageProducer != null) {
            messageProducer.close();
        }
        if (jmsSession != null) {
            jmsSession.close();
        }
        if (jmsConnection != null) {
            jmsConnection.close();
        }
        super.close();
    }
}
```

## Sanctions Response Source Function

```java
package com.example.fednow.function;

import org.apache.flink.streaming.api.functions.source.RichSourceFunction;
import org.apache.flink.configuration.Configuration;
import lombok.extern.slf4j.Slf4j;

import javax.jms.*;
import javax.naming.InitialContext;

/**
 * Source function to receive sanctions responses from FircoSoft via JMS.
 */
@Slf4j
public class SanctionsResponseSourceFunction
    extends RichSourceFunction<SanctionsResponse> {

    private final JmsConfig jmsConfig;

    private volatile boolean running = true;
    private transient Connection jmsConnection;
    private transient Session jmsSession;
    private transient MessageConsumer messageConsumer;

    public SanctionsResponseSourceFunction(JmsConfig jmsConfig) {
        this.jmsConfig = jmsConfig;
    }

    @Override
    public void open(Configuration parameters) throws Exception {
        super.open(parameters);

        ConnectionFactory connectionFactory = (ConnectionFactory)
            new InitialContext().lookup(jmsConfig.getConnectionFactoryName());

        jmsConnection = connectionFactory.createConnection(
            jmsConfig.getUsername(),
            jmsConfig.getPassword()
        );

        jmsSession = jmsConnection.createSession(false, Session.AUTO_ACKNOWLEDGE);

        Destination responseQueue = jmsSession.createQueue(jmsConfig.getResponseQueue());
        messageConsumer = jmsSession.createConsumer(responseQueue);

        jmsConnection.start();

        log.info("JMS response consumer started: queue={}", jmsConfig.getResponseQueue());
    }

    @Override
    public void run(SourceContext<SanctionsResponse> ctx) throws Exception {
        while (running) {
            Message message = messageConsumer.receive(1000); // 1 second timeout

            if (message instanceof TextMessage textMessage) {
                try {
                    String xmlContent = textMessage.getText();
                    String transactionId = textMessage.getStringProperty("TransactionId");

                    SanctionsResponse response = parseResponse(xmlContent);

                    log.debug("Received sanctions response: transactionId={}, result={}",
                        transactionId, response.getResult());

                    synchronized (ctx.getCheckpointLock()) {
                        ctx.collect(response);
                    }
                } catch (Exception e) {
                    log.error("Failed to process sanctions response", e);
                }
            }
        }
    }

    private SanctionsResponse parseResponse(String xml) {
        return new SanctionsResponseParser().parse(xml);
    }

    @Override
    public void cancel() {
        running = false;
    }

    @Override
    public void close() throws Exception {
        running = false;
        if (messageConsumer != null) {
            messageConsumer.close();
        }
        if (jmsSession != null) {
            jmsSession.close();
        }
        if (jmsConnection != null) {
            jmsConnection.close();
        }
        super.close();
    }
}
```

## Co-Process Function for Response Correlation

```java
package com.example.fednow.function;

import org.apache.flink.streaming.api.functions.co.KeyedCoProcessFunction;
import org.apache.flink.api.common.state.ValueState;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import org.apache.flink.util.Collector;

/**
 * Correlates sanctions responses with pending payments.
 */
public class SanctionsCorrelationFunction
    extends KeyedCoProcessFunction<String, Payment, SanctionsResponse, Payment> {

    private ValueState<Payment> pendingPaymentState;
    private ValueState<SanctionsResponse> pendingResponseState;
    private ValueState<Long> timerState;

    private static final long TIMEOUT_MS = 30_000;

    @Override
    public void open(Configuration parameters) throws Exception {
        pendingPaymentState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("pending-payment", Payment.class));
        pendingResponseState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("pending-response", SanctionsResponse.class));
        timerState = getRuntimeContext().getState(
            new ValueStateDescriptor<>("timer", Long.class));
    }

    @Override
    public void processElement1(Payment payment, Context ctx, Collector<Payment> out)
            throws Exception {

        // Check if response already arrived
        SanctionsResponse response = pendingResponseState.value();

        if (response != null) {
            // Response already here - process immediately
            processMatch(payment, response, ctx, out);
            pendingResponseState.clear();
        } else {
            // Store payment and wait for response
            pendingPaymentState.update(payment);

            // Send JMS request
            sendSanctionsRequest(payment);

            // Register timeout
            long timeout = ctx.timerService().currentProcessingTime() + TIMEOUT_MS;
            ctx.timerService().registerProcessingTimeTimer(timeout);
            timerState.update(timeout);
        }
    }

    @Override
    public void processElement2(SanctionsResponse response, Context ctx, Collector<Payment> out)
            throws Exception {

        // Check if payment already arrived
        Payment payment = pendingPaymentState.value();

        if (payment != null) {
            // Payment already here - process immediately
            cancelTimer(ctx);
            processMatch(payment, response, ctx, out);
            pendingPaymentState.clear();
        } else {
            // Store response and wait for payment (unlikely but handle it)
            pendingResponseState.update(response);
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Payment> out)
            throws Exception {

        Payment payment = pendingPaymentState.value();

        if (payment != null) {
            // Timeout
            payment.setSanctionsStatus(SanctionsStatus.TIMEOUT);
            ctx.output(PaymentProcessingJob.SANCTIONS_HIT, payment);
            pendingPaymentState.clear();
        }
    }

    private void processMatch(Payment payment, SanctionsResponse response,
            Context ctx, Collector<Payment> out) {
        // Process based on sanctions result
        // ... (same logic as before)
    }

    private void cancelTimer(Context ctx) throws Exception {
        Long timer = timerState.value();
        if (timer != null) {
            ctx.timerService().deleteProcessingTimeTimer(timer);
            timerState.clear();
        }
    }
}
```

## Best Practices

### State Management
- Use keyed state for payment correlation
- Store minimal data needed for correlation
- Clear state after processing or timeout

### JMS Integration
- Use persistent delivery for reliability
- Set correlation IDs for response matching
- Handle connection failures gracefully

### Timeout Handling
- Always register timeout timers
- Clean up state on timeout
- Route timeouts to appropriate DLQ

### Exactly-Once
- JMS producers should be transactional if possible
- Use Flink's checkpoint barriers
- Handle duplicate responses idempotently
