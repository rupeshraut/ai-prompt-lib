# Create Custom Sink

Create custom sink connector implementations in Apache Flink 2.0 for writing data to external systems with reliability guarantees. Custom sinks enable integration with any storage system while supporting exactly-once semantics.

## Requirements

Your custom sink implementation should include:

- **Sink API**: Use the new Sink API (SinkWriter, Committer, GlobalCommitter)
- **Legacy Support**: Understand SinkFunction API for compatibility
- **Batching**: Implement efficient batching and buffering strategies
- **Error Handling**: Robust retry logic and failure recovery
- **Exactly-Once**: Two-phase commit for exactly-once delivery guarantees
- **Idempotent Writes**: Design for safe retries and duplicate handling
- **Backpressure**: Handle slow sinks gracefully with flow control
- **State Management**: Maintain sink state for fault tolerance
- **Metrics**: Expose sink performance and error metrics
- **Configuration**: Flexible configuration for batch sizes, timeouts, retries
- **Cleanup**: Proper resource management and connection pooling
- **Documentation**: Clear documentation of delivery guarantees

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **Approach**: Use new Sink API for modern implementations
- **Type Safety**: Use strongly-typed serializers and deserializers
- **Immutability**: Use records for data transfer objects
- **Error Handling**: Comprehensive exception handling with custom exceptions

## Template Example

```java
import org.apache.flink.api.connector.sink2.*;
import org.apache.flink.api.common.serialization.SerializationSchema;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.functions.sink.SinkFunction;
import org.apache.flink.streaming.api.functions.sink.TwoPhaseCommitSinkFunction;
import lombok.extern.slf4j.Slf4j;
import java.io.IOException;
import java.io.Serializable;
import java.util.*;
import java.util.concurrent.*;

/**
 * Demonstrates custom sink implementations in Apache Flink 2.0.
 * Covers new Sink API, legacy SinkFunction, batching, error handling, and exactly-once semantics.
 */
@Slf4j
public class CustomSinkImplementations {

    /**
     * New Sink API: Modern approach with separated write/commit phases.
     * Use case: HTTP API sink with batching and exactly-once semantics.
     */
    public static void httpApiSinkExample(DataStream<Event> events) {
        events.sinkTo(new HttpApiSink(
                HttpApiSinkConfig.builder()
                        .endpoint("https://api.example.com/events")
                        .batchSize(100)
                        .batchTimeout(5000)
                        .maxRetries(3)
                        .build()
        )).name("HTTP API Sink");
    }

    /**
     * File-based sink with custom batching and rotation.
     * Use case: Write to files with hourly rotation and exactly-once guarantees.
     */
    public static void fileSinkWithRotation(DataStream<LogEntry> logs) {
        logs.sinkTo(new RotatingFileSink(
                "/data/logs",
                3600000,  // Rotate every hour
                1000      // 1000 entries per file
        )).name("Rotating File Sink");
    }

    /**
     * Custom buffered sink with backpressure handling.
     * Use case: Write to rate-limited external system.
     */
    public static void bufferedSinkWithBackpressure(DataStream<Metric> metrics) {
        metrics.sinkTo(new BufferedSink<>(
                new MetricWriter(),
                BufferConfig.builder()
                        .bufferSize(500)
                        .flushIntervalMs(1000)
                        .maxConcurrentRequests(5)
                        .build()
        )).name("Buffered Metric Sink");
    }

    /**
     * Two-phase commit sink for exactly-once semantics.
     * Use case: Transactional writes to external systems.
     */
    public static void exactlyOnceSink(DataStream<Transaction> transactions) {
        transactions.addSink(new TransactionalSink(
                TransactionalSinkConfig.builder()
                        .transactionTimeout(60000)
                        .commitRetries(5)
                        .build()
        )).name("Exactly-Once Transaction Sink");
    }

    /**
     * Idempotent sink with deduplication.
     * Use case: At-least-once with deduplication for idempotency.
     */
    public static void idempotentSink(DataStream<Message> messages) {
        messages.sinkTo(new IdempotentMessageSink(
                "kafka-broker:9092",
                "output-topic"
        )).name("Idempotent Message Sink");
    }

    /**
     * Async sink for high-throughput scenarios.
     * Use case: Non-blocking writes to fast external systems.
     */
    public static void asyncSink(DataStream<Record> records) {
        records.sinkTo(new AsyncBatchSink<>(
                new RecordAsyncWriter(),
                AsyncSinkConfig.builder()
                        .maxBatchSize(1000)
                        .maxInFlightRequests(50)
                        .maxBufferedRequests(10000)
                        .build()
        )).name("Async Batch Sink");
    }

    // ==================== New Sink API Implementation ====================

    /**
     * HTTP API Sink using new Sink API with batching and retries.
     */
    @Slf4j
    public static class HttpApiSink implements Sink<Event> {
        private final HttpApiSinkConfig config;

        public HttpApiSink(HttpApiSinkConfig config) {
            this.config = config;
        }

        @Override
        public SinkWriter<Event> createWriter(InitContext context) throws IOException {
            return new HttpApiSinkWriter(config, context);
        }

        @Override
        public Optional<Committer<HttpCommittable>> createCommitter() throws IOException {
            return Optional.of(new HttpApiCommitter(config));
        }

        @Override
        public Optional<SimpleVersionedSerializer<HttpCommittable>> getCommittableSerializer() {
            return Optional.of(new HttpCommittableSerializer());
        }
    }

    /**
     * SinkWriter implementation with batching and error handling.
     */
    @Slf4j
    private static class HttpApiSinkWriter implements SinkWriter<Event> {
        private final HttpApiSinkConfig config;
        private final List<Event> buffer;
        private final HttpClient httpClient;
        private long lastFlushTime;
        private final InitContext context;

        HttpApiSinkWriter(HttpApiSinkConfig config, InitContext context) {
            this.config = config;
            this.context = context;
            this.buffer = new ArrayList<>(config.batchSize());
            this.httpClient = new HttpClient(config.endpoint());
            this.lastFlushTime = System.currentTimeMillis();
        }

        @Override
        public void write(Event event, Context context) throws IOException, InterruptedException {
            buffer.add(event);

            // Flush if batch size reached
            if (buffer.size() >= config.batchSize()) {
                flush(false);
            }
            // Flush if timeout exceeded
            else if (System.currentTimeMillis() - lastFlushTime >= config.batchTimeout()) {
                flush(false);
            }
        }

        @Override
        public void flush(boolean endOfInput) throws IOException, InterruptedException {
            if (buffer.isEmpty()) {
                return;
            }

            List<Event> toFlush = new ArrayList<>(buffer);
            buffer.clear();
            lastFlushTime = System.currentTimeMillis();

            // Retry logic
            int retries = 0;
            while (retries <= config.maxRetries()) {
                try {
                    httpClient.sendBatch(toFlush);
                    log.debug("Flushed {} events to HTTP API", toFlush.size());
                    return;
                } catch (IOException e) {
                    retries++;
                    if (retries > config.maxRetries()) {
                        log.error("Failed to flush after {} retries", config.maxRetries(), e);
                        throw e;
                    }
                    log.warn("Retry {}/{} after error: {}", retries, config.maxRetries(), e.getMessage());
                    Thread.sleep(1000L * retries); // Exponential backoff
                }
            }
        }

        @Override
        public Collection<HttpCommittable> prepareCommit() throws IOException, InterruptedException {
            // For at-least-once, we can return empty
            // For exactly-once, return committables for two-phase commit
            flush(false);
            return Collections.emptyList();
        }

        @Override
        public void close() throws Exception {
            try {
                flush(true);
            } finally {
                httpClient.close();
            }
        }
    }

    /**
     * Committer for two-phase commit protocol.
     */
    @Slf4j
    private static class HttpApiCommitter implements Committer<HttpCommittable> {
        private final HttpApiSinkConfig config;

        HttpApiCommitter(HttpApiSinkConfig config) {
            this.config = config;
        }

        @Override
        public void commit(Collection<CommitRequest<HttpCommittable>> commitRequests)
                throws IOException, InterruptedException {
            for (CommitRequest<HttpCommittable> request : commitRequests) {
                try {
                    HttpCommittable committable = request.getCommittable();
                    // Perform actual commit (e.g., send confirmation, update state)
                    log.info("Committed batch: {}", committable.batchId());
                    request.signalAlreadyCommitted();
                } catch (Exception e) {
                    log.error("Failed to commit batch", e);
                    request.retryLater();
                }
            }
        }

        @Override
        public void close() throws Exception {
            // Cleanup resources
        }
    }

    // ==================== Rotating File Sink ====================

    /**
     * Custom file sink with rotation based on time and count.
     */
    @Slf4j
    public static class RotatingFileSink implements Sink<LogEntry> {
        private final String basePath;
        private final long rotationIntervalMs;
        private final int maxEntriesPerFile;

        public RotatingFileSink(String basePath, long rotationIntervalMs, int maxEntriesPerFile) {
            this.basePath = basePath;
            this.rotationIntervalMs = rotationIntervalMs;
            this.maxEntriesPerFile = maxEntriesPerFile;
        }

        @Override
        public SinkWriter<LogEntry> createWriter(InitContext context) throws IOException {
            return new RotatingFileSinkWriter(basePath, rotationIntervalMs, maxEntriesPerFile, context);
        }
    }

    /**
     * SinkWriter that rotates files based on time and count.
     */
    @Slf4j
    private static class RotatingFileSinkWriter implements SinkWriter<LogEntry> {
        private final String basePath;
        private final long rotationIntervalMs;
        private final int maxEntriesPerFile;
        private java.io.BufferedWriter currentWriter;
        private String currentFilePath;
        private long fileStartTime;
        private int entriesInCurrentFile;
        private final int subtaskIndex;

        RotatingFileSinkWriter(String basePath, long rotationIntervalMs, int maxEntriesPerFile,
                               InitContext context) throws IOException {
            this.basePath = basePath;
            this.rotationIntervalMs = rotationIntervalMs;
            this.maxEntriesPerFile = maxEntriesPerFile;
            this.subtaskIndex = context.getSubtaskId();
            rotateFile();
        }

        @Override
        public void write(LogEntry entry, Context context) throws IOException {
            // Check if rotation needed
            if (needsRotation()) {
                rotateFile();
            }

            // Write entry
            currentWriter.write(entry.toJson());
            currentWriter.newLine();
            entriesInCurrentFile++;
        }

        private boolean needsRotation() {
            long elapsed = System.currentTimeMillis() - fileStartTime;
            return elapsed >= rotationIntervalMs || entriesInCurrentFile >= maxEntriesPerFile;
        }

        private void rotateFile() throws IOException {
            // Close current file
            if (currentWriter != null) {
                currentWriter.close();
                log.info("Rotated file: {} ({} entries)", currentFilePath, entriesInCurrentFile);
            }

            // Create new file
            long timestamp = System.currentTimeMillis();
            currentFilePath = String.format("%s/logs-%d-%d-%d.json",
                    basePath, subtaskIndex, timestamp, System.nanoTime());

            java.io.File file = new java.io.File(currentFilePath);
            file.getParentFile().mkdirs();

            currentWriter = new java.io.BufferedWriter(new java.io.FileWriter(file));
            fileStartTime = System.currentTimeMillis();
            entriesInCurrentFile = 0;
        }

        @Override
        public void flush(boolean endOfInput) throws IOException {
            if (currentWriter != null) {
                currentWriter.flush();
            }
        }

        @Override
        public void close() throws Exception {
            if (currentWriter != null) {
                currentWriter.close();
            }
        }
    }

    // ==================== Buffered Sink with Backpressure ====================

    /**
     * Generic buffered sink with configurable buffering and backpressure.
     */
    @Slf4j
    public static class BufferedSink<T> implements Sink<T> {
        private final ExternalWriter<T> writer;
        private final BufferConfig config;

        public BufferedSink(ExternalWriter<T> writer, BufferConfig config) {
            this.writer = writer;
            this.config = config;
        }

        @Override
        public SinkWriter<T> createWriter(InitContext context) throws IOException {
            return new BufferedSinkWriter<>(writer, config, context);
        }
    }

    /**
     * Buffered writer with semaphore-based backpressure.
     */
    @Slf4j
    private static class BufferedSinkWriter<T> implements SinkWriter<T> {
        private final ExternalWriter<T> writer;
        private final BufferConfig config;
        private final List<T> buffer;
        private final Semaphore inflightSemaphore;
        private final ExecutorService executorService;
        private long lastFlushTime;

        BufferedSinkWriter(ExternalWriter<T> writer, BufferConfig config, InitContext context) {
            this.writer = writer;
            this.config = config;
            this.buffer = new ArrayList<>(config.bufferSize());
            this.inflightSemaphore = new Semaphore(config.maxConcurrentRequests());
            this.executorService = Executors.newFixedThreadPool(config.maxConcurrentRequests());
            this.lastFlushTime = System.currentTimeMillis();
        }

        @Override
        public void write(T element, Context context) throws IOException, InterruptedException {
            buffer.add(element);

            // Flush if buffer full
            if (buffer.size() >= config.bufferSize()) {
                flush(false);
            }
            // Flush if timeout exceeded
            else if (System.currentTimeMillis() - lastFlushTime >= config.flushIntervalMs()) {
                flush(false);
            }
        }

        @Override
        public void flush(boolean endOfInput) throws IOException, InterruptedException {
            if (buffer.isEmpty()) {
                return;
            }

            List<T> toWrite = new ArrayList<>(buffer);
            buffer.clear();
            lastFlushTime = System.currentTimeMillis();

            // Acquire permit (blocks if too many concurrent requests)
            inflightSemaphore.acquire();

            // Async write
            executorService.submit(() -> {
                try {
                    writer.write(toWrite);
                    log.debug("Wrote {} elements", toWrite.size());
                } catch (Exception e) {
                    log.error("Failed to write batch", e);
                } finally {
                    inflightSemaphore.release();
                }
            });

            // If end of input, wait for all to complete
            if (endOfInput) {
                inflightSemaphore.acquire(config.maxConcurrentRequests());
                inflightSemaphore.release(config.maxConcurrentRequests());
            }
        }

        @Override
        public void close() throws Exception {
            flush(true);
            executorService.shutdown();
            if (!executorService.awaitTermination(30, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
            writer.close();
        }
    }

    // ==================== Two-Phase Commit Sink ====================

    /**
     * Transactional sink using legacy TwoPhaseCommitSinkFunction.
     * Provides exactly-once semantics with transaction support.
     */
    @Slf4j
    public static class TransactionalSink extends TwoPhaseCommitSinkFunction<
            Transaction, TransactionState, Void> {

        private final TransactionalSinkConfig config;

        public TransactionalSink(TransactionalSinkConfig config) {
            super(
                new org.apache.flink.api.common.typeinfo.TypeHint<TransactionState>() {}.getTypeInfo(),
                new org.apache.flink.api.common.typeinfo.TypeHint<Void>() {}.getTypeInfo()
            );
            this.config = config;
        }

        @Override
        protected void invoke(TransactionState transaction, Transaction value, Context context)
                throws Exception {
            // Write to transaction buffer
            transaction.addTransaction(value);
            log.debug("Added transaction to buffer: {}", value.id());
        }

        @Override
        protected TransactionState beginTransaction() throws Exception {
            String txnId = UUID.randomUUID().toString();
            log.info("Beginning transaction: {}", txnId);
            return new TransactionState(txnId, new ArrayList<>());
        }

        @Override
        protected void preCommit(TransactionState transaction) throws Exception {
            log.info("Pre-commit transaction: {} ({} items)",
                    transaction.transactionId(), transaction.transactions().size());

            // Prepare transaction (e.g., write to staging, validate)
            for (Transaction txn : transaction.transactions()) {
                // Write to staging area or prepare for commit
            }
        }

        @Override
        protected void commit(TransactionState transaction) {
            log.info("Committing transaction: {}", transaction.transactionId());

            int retries = 0;
            while (retries < config.commitRetries()) {
                try {
                    // Perform actual commit
                    commitToExternalSystem(transaction);
                    log.info("Transaction committed successfully: {}", transaction.transactionId());
                    return;
                } catch (Exception e) {
                    retries++;
                    log.warn("Commit retry {}/{}: {}", retries, config.commitRetries(), e.getMessage());
                    if (retries >= config.commitRetries()) {
                        log.error("Failed to commit transaction after retries", e);
                        throw new RuntimeException("Commit failed", e);
                    }
                    try {
                        Thread.sleep(1000L * retries);
                    } catch (InterruptedException ie) {
                        Thread.currentThread().interrupt();
                        throw new RuntimeException("Interrupted during retry", ie);
                    }
                }
            }
        }

        @Override
        protected void abort(TransactionState transaction) {
            log.warn("Aborting transaction: {}", transaction.transactionId());
            // Rollback or cleanup transaction
            try {
                abortExternalTransaction(transaction);
            } catch (Exception e) {
                log.error("Failed to abort transaction", e);
            }
        }

        private void commitToExternalSystem(TransactionState transaction) throws Exception {
            // Actual commit logic (e.g., database commit, file finalization)
            // This should be idempotent
        }

        private void abortExternalTransaction(TransactionState transaction) throws Exception {
            // Cleanup logic (e.g., delete staging files, rollback DB)
        }
    }

    // ==================== Idempotent Sink ====================

    /**
     * Idempotent sink that ensures duplicate writes are safe.
     * Uses deduplication key for idempotency.
     */
    @Slf4j
    public static class IdempotentMessageSink implements Sink<Message> {
        private final String brokers;
        private final String topic;

        public IdempotentMessageSink(String brokers, String topic) {
            this.brokers = brokers;
            this.topic = topic;
        }

        @Override
        public SinkWriter<Message> createWriter(InitContext context) throws IOException {
            return new IdempotentMessageWriter(brokers, topic, context);
        }
    }

    /**
     * Writer with deduplication tracking.
     */
    @Slf4j
    private static class IdempotentMessageWriter implements SinkWriter<Message> {
        private final String brokers;
        private final String topic;
        private final Set<String> writtenIds;
        private final KafkaProducerWrapper producer;

        IdempotentMessageWriter(String brokers, String topic, InitContext context) {
            this.brokers = brokers;
            this.topic = topic;
            this.writtenIds = new HashSet<>();
            this.producer = new KafkaProducerWrapper(brokers);
        }

        @Override
        public void write(Message message, Context context) throws IOException {
            // Check for duplicate
            if (writtenIds.contains(message.id())) {
                log.debug("Skipping duplicate message: {}", message.id());
                return;
            }

            // Write with idempotency key
            producer.send(topic, message.id(), message.content());
            writtenIds.add(message.id());

            // Cleanup old IDs to prevent memory growth
            if (writtenIds.size() > 100000) {
                writtenIds.clear();
                log.info("Cleared deduplication cache");
            }
        }

        @Override
        public void flush(boolean endOfInput) throws IOException {
            producer.flush();
        }

        @Override
        public void close() throws Exception {
            producer.close();
        }
    }

    // ==================== Async Batch Sink ====================

    /**
     * High-throughput async sink with batching.
     */
    @Slf4j
    public static class AsyncBatchSink<T> implements Sink<T> {
        private final AsyncWriter<T> writer;
        private final AsyncSinkConfig config;

        public AsyncBatchSink(AsyncWriter<T> writer, AsyncSinkConfig config) {
            this.writer = writer;
            this.config = config;
        }

        @Override
        public SinkWriter<T> createWriter(InitContext context) throws IOException {
            return new AsyncBatchSinkWriter<>(writer, config, context);
        }
    }

    /**
     * Async writer with concurrent request management.
     */
    @Slf4j
    private static class AsyncBatchSinkWriter<T> implements SinkWriter<T> {
        private final AsyncWriter<T> writer;
        private final AsyncSinkConfig config;
        private final List<T> buffer;
        private final Queue<CompletableFuture<Void>> inFlightRequests;

        AsyncBatchSinkWriter(AsyncWriter<T> writer, AsyncSinkConfig config, InitContext context) {
            this.writer = writer;
            this.config = config;
            this.buffer = new ArrayList<>(config.maxBatchSize());
            this.inFlightRequests = new ConcurrentLinkedQueue<>();
        }

        @Override
        public void write(T element, Context context) throws IOException, InterruptedException {
            buffer.add(element);

            // Flush if batch size reached
            if (buffer.size() >= config.maxBatchSize()) {
                flush(false);
            }

            // Wait if too many in-flight requests
            while (inFlightRequests.size() >= config.maxInFlightRequests()) {
                waitForCompletion();
            }
        }

        @Override
        public void flush(boolean endOfInput) throws IOException, InterruptedException {
            if (buffer.isEmpty()) {
                return;
            }

            List<T> toWrite = new ArrayList<>(buffer);
            buffer.clear();

            // Submit async write
            CompletableFuture<Void> future = writer.writeAsync(toWrite)
                    .whenComplete((result, error) -> {
                        if (error != null) {
                            log.error("Async write failed", error);
                        } else {
                            log.debug("Async write completed: {} elements", toWrite.size());
                        }
                    });

            inFlightRequests.add(future);

            // If end of input, wait for all
            if (endOfInput) {
                CompletableFuture.allOf(
                    inFlightRequests.toArray(new CompletableFuture[0])
                ).join();
            }
        }

        private void waitForCompletion() throws InterruptedException {
            CompletableFuture<Void> oldest = inFlightRequests.poll();
            if (oldest != null) {
                try {
                    oldest.get(10, TimeUnit.SECONDS);
                } catch (ExecutionException | TimeoutException e) {
                    log.error("Failed to wait for async write", e);
                }
            }
        }

        @Override
        public void close() throws Exception {
            flush(true);
            writer.close();
        }
    }

    // ==================== Helper Classes and Interfaces ====================

    // Configuration classes
    public record HttpApiSinkConfig(
        String endpoint,
        int batchSize,
        long batchTimeout,
        int maxRetries
    ) {
        public static Builder builder() {
            return new Builder();
        }

        public static class Builder {
            private String endpoint;
            private int batchSize = 100;
            private long batchTimeout = 5000;
            private int maxRetries = 3;

            public Builder endpoint(String endpoint) {
                this.endpoint = endpoint;
                return this;
            }

            public Builder batchSize(int batchSize) {
                this.batchSize = batchSize;
                return this;
            }

            public Builder batchTimeout(long batchTimeout) {
                this.batchTimeout = batchTimeout;
                return this;
            }

            public Builder maxRetries(int maxRetries) {
                this.maxRetries = maxRetries;
                return this;
            }

            public HttpApiSinkConfig build() {
                return new HttpApiSinkConfig(endpoint, batchSize, batchTimeout, maxRetries);
            }
        }
    }

    public record BufferConfig(
        int bufferSize,
        long flushIntervalMs,
        int maxConcurrentRequests
    ) {
        public static Builder builder() {
            return new Builder();
        }

        public static class Builder {
            private int bufferSize = 500;
            private long flushIntervalMs = 1000;
            private int maxConcurrentRequests = 5;

            public Builder bufferSize(int bufferSize) {
                this.bufferSize = bufferSize;
                return this;
            }

            public Builder flushIntervalMs(long flushIntervalMs) {
                this.flushIntervalMs = flushIntervalMs;
                return this;
            }

            public Builder maxConcurrentRequests(int maxConcurrentRequests) {
                this.maxConcurrentRequests = maxConcurrentRequests;
                return this;
            }

            public BufferConfig build() {
                return new BufferConfig(bufferSize, flushIntervalMs, maxConcurrentRequests);
            }
        }
    }

    public record TransactionalSinkConfig(
        long transactionTimeout,
        int commitRetries
    ) {
        public static Builder builder() {
            return new Builder();
        }

        public static class Builder {
            private long transactionTimeout = 60000;
            private int commitRetries = 5;

            public Builder transactionTimeout(long transactionTimeout) {
                this.transactionTimeout = transactionTimeout;
                return this;
            }

            public Builder commitRetries(int commitRetries) {
                this.commitRetries = commitRetries;
                return this;
            }

            public TransactionalSinkConfig build() {
                return new TransactionalSinkConfig(transactionTimeout, commitRetries);
            }
        }
    }

    public record AsyncSinkConfig(
        int maxBatchSize,
        int maxInFlightRequests,
        int maxBufferedRequests
    ) {
        public static Builder builder() {
            return new Builder();
        }

        public static class Builder {
            private int maxBatchSize = 1000;
            private int maxInFlightRequests = 50;
            private int maxBufferedRequests = 10000;

            public Builder maxBatchSize(int maxBatchSize) {
                this.maxBatchSize = maxBatchSize;
                return this;
            }

            public Builder maxInFlightRequests(int maxInFlightRequests) {
                this.maxInFlightRequests = maxInFlightRequests;
                return this;
            }

            public Builder maxBufferedRequests(int maxBufferedRequests) {
                this.maxBufferedRequests = maxBufferedRequests;
                return this;
            }

            public AsyncSinkConfig build() {
                return new AsyncSinkConfig(maxBatchSize, maxInFlightRequests, maxBufferedRequests);
            }
        }
    }

    // Data models
    public record Event(String id, String type, String payload, long timestamp) {}
    public record LogEntry(String level, String message, String source, long timestamp) {
        public String toJson() {
            return String.format("{\"level\":\"%s\",\"message\":\"%s\",\"source\":\"%s\",\"timestamp\":%d}",
                    level, message, source, timestamp);
        }
    }
    public record Metric(String name, double value, Map<String, String> tags, long timestamp) {}
    public record Transaction(String id, String userId, double amount, String type, long timestamp) {}
    public record Message(String id, String content, long timestamp) {}
    public record Record(String id, byte[] data) {}

    // Committable for two-phase commit
    public record HttpCommittable(String batchId, List<Event> events) implements Serializable {}

    // Transaction state
    public record TransactionState(String transactionId, List<Transaction> transactions) {
        public void addTransaction(Transaction txn) {
            transactions.add(txn);
        }
    }

    // Writer interfaces
    public interface ExternalWriter<T> extends AutoCloseable {
        void write(List<T> batch) throws Exception;
    }

    public interface AsyncWriter<T> extends AutoCloseable {
        CompletableFuture<Void> writeAsync(List<T> batch);
    }

    // Example implementations
    private static class MetricWriter implements ExternalWriter<Metric> {
        @Override
        public void write(List<Metric> batch) throws Exception {
            // Write metrics to external system
            log.info("Writing {} metrics", batch.size());
        }

        @Override
        public void close() throws Exception {
            // Cleanup
        }
    }

    private static class RecordAsyncWriter implements AsyncWriter<Record> {
        private final ExecutorService executor = Executors.newCachedThreadPool();

        @Override
        public CompletableFuture<Void> writeAsync(List<Record> batch) {
            return CompletableFuture.runAsync(() -> {
                // Async write logic
                log.info("Async writing {} records", batch.size());
            }, executor);
        }

        @Override
        public void close() throws Exception {
            executor.shutdown();
            executor.awaitTermination(10, TimeUnit.SECONDS);
        }
    }

    // HTTP Client stub
    private static class HttpClient implements AutoCloseable {
        private final String endpoint;

        HttpClient(String endpoint) {
            this.endpoint = endpoint;
        }

        void sendBatch(List<Event> events) throws IOException {
            // HTTP POST logic
            log.debug("Sending {} events to {}", events.size(), endpoint);
        }

        @Override
        public void close() throws Exception {
            // Cleanup HTTP client
        }
    }

    // Kafka Producer wrapper
    private static class KafkaProducerWrapper implements AutoCloseable {
        private final String brokers;

        KafkaProducerWrapper(String brokers) {
            this.brokers = brokers;
        }

        void send(String topic, String key, String value) throws IOException {
            // Kafka send logic
            log.debug("Sending to {}: {}", topic, key);
        }

        void flush() throws IOException {
            // Flush producer
        }

        @Override
        public void close() throws Exception {
            // Close producer
        }
    }

    // Serializer for committables
    private static class HttpCommittableSerializer implements SimpleVersionedSerializer<HttpCommittable> {
        @Override
        public int getVersion() {
            return 1;
        }

        @Override
        public byte[] serialize(HttpCommittable obj) throws IOException {
            // Serialize committable
            return new byte[0];
        }

        @Override
        public HttpCommittable deserialize(int version, byte[] serialized) throws IOException {
            // Deserialize committable
            return new HttpCommittable("", Collections.emptyList());
        }
    }
}
```

## Configuration Example

```yaml
# Flink sink configuration
execution:
  parallelism: 4
  buffer-timeout: 100ms

# Checkpointing for exactly-once
checkpointing:
  interval: 60s
  mode: exactly-once
  timeout: 10min
  min-pause-between-checkpoints: 30s

# Custom sink settings
sinks:
  http-api:
    endpoint: "https://api.example.com/events"
    batch-size: 100
    batch-timeout: 5s
    max-retries: 3

  file-rotation:
    base-path: "/data/output"
    rotation-interval: 1h
    max-entries-per-file: 1000

  buffered:
    buffer-size: 500
    flush-interval: 1s
    max-concurrent-requests: 5

  transactional:
    transaction-timeout: 60s
    commit-retries: 5

  async:
    max-batch-size: 1000
    max-inflight-requests: 50
    max-buffered-requests: 10000

# State backend
state:
  backend: rocksdb
  checkpoints-dir: s3://bucket/checkpoints
```

## Best Practices

1. **Batching**: Always batch writes to external systems. Batch size of 100-1000 provides good throughput vs latency trade-off.

2. **Backpressure**: Implement proper backpressure handling with semaphores or blocking queues to prevent overwhelming external systems.

3. **Retry Logic**: Implement exponential backoff for retries. Limit retries to avoid infinite loops. Consider dead-letter queues for persistent failures.

4. **Idempotency**: Design sinks to be idempotent. Use unique IDs for deduplication. External systems should support upsert operations.

5. **Error Handling**: Separate transient errors (retry) from permanent errors (fail fast). Log errors with sufficient context for debugging.

6. **Resource Management**: Always close resources in the close() method. Use try-with-resources where applicable. Implement proper connection pooling.

7. **Exactly-Once**: Use TwoPhaseCommitSinkFunction or the new Sink API with Committer for exactly-once semantics. Ensure external systems support transactions.

8. **Monitoring**: Expose metrics for throughput, latency, errors, and backlog. Monitor buffer sizes and in-flight requests.

9. **Testing**: Test sinks with failures, restarts, and checkpointing. Verify exactly-once guarantees with duplicate detection.

10. **Performance**: Use async writes for high throughput. Consider parallel sinks for very high throughput scenarios. Profile and optimize batch sizes.

## Related Documentation

- [Flink Sink API](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/overview/#data-sinks)
- [DataStream Connectors](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/connectors/datastream/overview/)
- [Exactly-Once Semantics](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/fault-tolerance/guarantees/)
- [TwoPhaseCommitSinkFunction](https://nightlies.apache.org/flink/flink-docs-release-2.0/api/java/org/apache/flink/streaming/api/functions/sink/TwoPhaseCommitSinkFunction.html)
- [Sink API JavaDoc](https://nightlies.apache.org/flink/flink-docs-release-2.0/api/java/org/apache/flink/api/connector/sink2/Sink.html)
