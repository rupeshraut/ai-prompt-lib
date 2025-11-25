# Create Database Sink

Create JDBC and database sink connectors in Apache Flink 2.0 for writing data to relational databases with transactional semantics. Database sinks enable efficient bulk writes with connection pooling and exactly-once guarantees.

## Requirements

Your database sink implementation should include:

- **JDBC Connector**: Use Flink JDBC connector for common databases (MySQL, PostgreSQL, Oracle)
- **Connection Pooling**: HikariCP configuration for optimal connection management
- **Batch Operations**: Efficient batch inserts using PreparedStatement batching
- **Exactly-Once**: XA transactions or idempotent upserts for exactly-once semantics
- **Statement Builder**: Custom JdbcStatementBuilder for flexible SQL generation
- **Retry Logic**: Transient error retry with exponential backoff
- **Schema Evolution**: Handle schema changes and missing columns gracefully
- **Connection Recovery**: Automatic reconnection on connection failures
- **Execution Options**: Configure batch size, flush interval, max retries
- **Metrics**: Track throughput, errors, connection pool stats
- **Type Mapping**: Proper Java to SQL type mapping
- **Documentation**: Clear examples for different database systems

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **Approach**: Use flink-connector-jdbc for standard databases
- **Type Safety**: Use prepared statements to prevent SQL injection
- **Immutability**: Use records for data models
- **Connection Pooling**: Always use HikariCP for production

## Template Example

```java
import org.apache.flink.connector.jdbc.*;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.functions.sink.SinkFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import lombok.extern.slf4j.Slf4j;
import java.sql.*;
import java.util.*;
import java.time.Instant;
import javax.sql.DataSource;

/**
 * Demonstrates database sink implementations in Apache Flink 2.0.
 * Covers JDBC connector, connection pooling, batch operations, and exactly-once semantics.
 */
@Slf4j
public class DatabaseSinkImplementations {

    /**
     * Basic JDBC sink with batching.
     * Use case: Insert events into MySQL with connection pooling.
     */
    public static void basicMySQLSink(DataStream<Event> events) {
        events.addSink(
            JdbcSink.sink(
                "INSERT INTO events (id, type, payload, timestamp) VALUES (?, ?, ?, ?)",
                (statement, event) -> {
                    statement.setString(1, event.id());
                    statement.setString(2, event.type());
                    statement.setString(3, event.payload());
                    statement.setTimestamp(4, Timestamp.from(Instant.ofEpochMilli(event.timestamp())));
                },
                JdbcExecutionOptions.builder()
                    .withBatchSize(1000)
                    .withBatchIntervalMs(200)
                    .withMaxRetries(5)
                    .build(),
                new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                    .withUrl("jdbc:mysql://localhost:3306/analytics")
                    .withDriverName("com.mysql.cj.jdbc.Driver")
                    .withUsername("flink_user")
                    .withPassword("flink_password")
                    .build()
            )
        ).name("MySQL Events Sink");
    }

    /**
     * PostgreSQL sink with UPSERT for exactly-once semantics.
     * Use case: Idempotent writes using ON CONFLICT clause.
     */
    public static void postgresUpsertSink(DataStream<UserProfile> profiles) {
        profiles.addSink(
            JdbcSink.sink(
                "INSERT INTO user_profiles (user_id, name, email, updated_at) " +
                "VALUES (?, ?, ?, ?) " +
                "ON CONFLICT (user_id) " +
                "DO UPDATE SET name = EXCLUDED.name, email = EXCLUDED.email, updated_at = EXCLUDED.updated_at",
                (statement, profile) -> {
                    statement.setString(1, profile.userId());
                    statement.setString(2, profile.name());
                    statement.setString(3, profile.email());
                    statement.setTimestamp(4, new Timestamp(System.currentTimeMillis()));
                },
                JdbcExecutionOptions.builder()
                    .withBatchSize(500)
                    .withBatchIntervalMs(1000)
                    .withMaxRetries(3)
                    .build(),
                new JdbcConnectionOptions.JdbcConnectionOptionsBuilder()
                    .withUrl("jdbc:postgresql://localhost:5432/users_db")
                    .withDriverName("org.postgresql.Driver")
                    .withUsername("postgres")
                    .withPassword("postgres")
                    .build()
            )
        ).name("PostgreSQL Upsert Sink");
    }

    /**
     * Custom JDBC sink with HikariCP connection pooling.
     * Use case: Production-grade sink with optimized connection management.
     */
    public static void hikariPooledSink(DataStream<Metric> metrics) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/metrics_db");
        config.setUsername("metrics_user");
        config.setPassword("metrics_pass");
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        config.setAutoCommit(false);

        HikariDataSource dataSource = new HikariDataSource(config);

        metrics.addSink(new HikariJdbcSink<>(
            dataSource,
            "INSERT INTO metrics (name, value, tags, timestamp) VALUES (?, ?, ?, ?)",
            (statement, metric) -> {
                statement.setString(1, metric.name());
                statement.setDouble(2, metric.value());
                statement.setString(3, metric.tagsAsJson());
                statement.setTimestamp(4, new Timestamp(metric.timestamp()));
            },
            500,  // batch size
            1000  // batch interval ms
        )).name("HikariCP Metrics Sink");
    }

    /**
     * Multi-table sink routing records to different tables.
     * Use case: Route events to different tables based on event type.
     */
    public static void multiTableSink(DataStream<GenericEvent> events) {
        events.addSink(new MultiTableJdbcSink<>(
            createDataSource("jdbc:mysql://localhost:3306/events_db"),
            event -> switch (event.category()) {
                case "user" -> new TableConfig(
                    "user_events",
                    "INSERT INTO user_events (id, user_id, action, timestamp) VALUES (?, ?, ?, ?)",
                    (stmt, e) -> {
                        stmt.setString(1, e.id());
                        stmt.setString(2, e.attributes().get("user_id"));
                        stmt.setString(3, e.action());
                        stmt.setTimestamp(4, new Timestamp(e.timestamp()));
                    }
                );
                case "system" -> new TableConfig(
                    "system_events",
                    "INSERT INTO system_events (id, host, severity, message, timestamp) VALUES (?, ?, ?, ?, ?)",
                    (stmt, e) -> {
                        stmt.setString(1, e.id());
                        stmt.setString(2, e.attributes().get("host"));
                        stmt.setString(3, e.attributes().get("severity"));
                        stmt.setString(4, e.message());
                        stmt.setTimestamp(5, new Timestamp(e.timestamp()));
                    }
                );
                default -> new TableConfig(
                    "generic_events",
                    "INSERT INTO generic_events (id, category, data, timestamp) VALUES (?, ?, ?, ?)",
                    (stmt, e) -> {
                        stmt.setString(1, e.id());
                        stmt.setString(2, e.category());
                        stmt.setString(3, e.toJson());
                        stmt.setTimestamp(4, new Timestamp(e.timestamp()));
                    }
                );
            },
            1000,
            500
        )).name("Multi-Table Sink");
    }

    /**
     * Transactional sink with XA for exactly-once across multiple databases.
     * Use case: Write to multiple databases with distributed transaction.
     */
    public static void xaTransactionalSink(DataStream<Order> orders) {
        orders.addSink(new XATransactionalJdbcSink<>(
            Arrays.asList(
                createDataSource("jdbc:mysql://db1:3306/orders"),
                createDataSource("jdbc:mysql://db2:3306/inventory")
            ),
            List.of(
                new TransactionalWrite<>(
                    0,  // First datasource
                    "INSERT INTO orders (order_id, customer_id, amount, timestamp) VALUES (?, ?, ?, ?)",
                    (stmt, order) -> {
                        stmt.setString(1, order.orderId());
                        stmt.setString(2, order.customerId());
                        stmt.setDouble(3, order.amount());
                        stmt.setTimestamp(4, new Timestamp(order.timestamp()));
                    }
                ),
                new TransactionalWrite<>(
                    1,  // Second datasource
                    "UPDATE inventory SET quantity = quantity - ? WHERE product_id = ?",
                    (stmt, order) -> {
                        stmt.setInt(1, order.quantity());
                        stmt.setString(2, order.productId());
                    }
                )
            )
        )).name("XA Transactional Sink");
    }

    /**
     * Slowly Changing Dimension (SCD) Type 2 sink.
     * Use case: Track historical changes with effective dates.
     */
    public static void scdType2Sink(DataStream<DimensionRecord> dimensions) {
        dimensions.addSink(new SCD2JdbcSink<>(
            createDataSource("jdbc:postgresql://localhost:5432/warehouse"),
            "dim_products",
            "product_id",
            List.of("product_name", "category", "price")
        )).name("SCD Type 2 Sink");
    }

    /**
     * Partitioned table sink with dynamic partition routing.
     * Use case: Write to time-partitioned tables.
     */
    public static void partitionedTableSink(DataStream<TimeSeriesData> data) {
        data.addSink(new PartitionedTableSink<>(
            createDataSource("jdbc:mysql://localhost:3306/timeseries"),
            "sensor_data",
            record -> {
                // Route to partition based on timestamp
                Instant instant = Instant.ofEpochMilli(record.timestamp());
                return String.format("sensor_data_%s", instant.toString().substring(0, 10));
            },
            "INSERT INTO {table} (sensor_id, value, timestamp) VALUES (?, ?, ?)",
            (stmt, record) -> {
                stmt.setString(1, record.sensorId());
                stmt.setDouble(2, record.value());
                stmt.setTimestamp(3, new Timestamp(record.timestamp()));
            }
        )).name("Partitioned Table Sink");
    }

    // ==================== Custom Sink Implementations ====================

    /**
     * HikariCP-based JDBC sink with connection pooling.
     */
    @Slf4j
    public static class HikariJdbcSink<T> implements SinkFunction<T> {
        private final HikariDataSource dataSource;
        private final String sql;
        private final JdbcStatementBuilder<T> statementBuilder;
        private final int batchSize;
        private final long batchIntervalMs;

        private transient Connection connection;
        private transient PreparedStatement statement;
        private transient List<T> batch;
        private transient long lastBatchTime;

        public HikariJdbcSink(
                HikariDataSource dataSource,
                String sql,
                JdbcStatementBuilder<T> statementBuilder,
                int batchSize,
                long batchIntervalMs) {
            this.dataSource = dataSource;
            this.sql = sql;
            this.statementBuilder = statementBuilder;
            this.batchSize = batchSize;
            this.batchIntervalMs = batchIntervalMs;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            batch = new ArrayList<>(batchSize);
            lastBatchTime = System.currentTimeMillis();
            reconnect();
        }

        private void reconnect() throws SQLException {
            if (connection != null && !connection.isClosed()) {
                connection.close();
            }
            connection = dataSource.getConnection();
            connection.setAutoCommit(false);
            statement = connection.prepareStatement(sql);
            log.info("Database connection established");
        }

        @Override
        public void invoke(T value, Context context) throws Exception {
            batch.add(value);

            // Flush if batch size reached or timeout exceeded
            if (batch.size() >= batchSize ||
                System.currentTimeMillis() - lastBatchTime >= batchIntervalMs) {
                flush();
            }
        }

        private void flush() throws Exception {
            if (batch.isEmpty()) {
                return;
            }

            int retries = 0;
            int maxRetries = 3;

            while (retries <= maxRetries) {
                try {
                    // Build batch
                    for (T record : batch) {
                        statementBuilder.accept(statement, record);
                        statement.addBatch();
                    }

                    // Execute batch
                    int[] results = statement.executeBatch();
                    connection.commit();

                    log.debug("Flushed {} records to database", batch.size());
                    batch.clear();
                    lastBatchTime = System.currentTimeMillis();
                    statement.clearBatch();
                    return;

                } catch (SQLException e) {
                    retries++;
                    log.warn("Batch execution failed (retry {}/{}): {}", retries, maxRetries, e.getMessage());

                    if (retries > maxRetries) {
                        log.error("Failed to execute batch after {} retries", maxRetries, e);
                        connection.rollback();
                        throw e;
                    }

                    // Rollback and reconnect
                    try {
                        connection.rollback();
                        statement.clearBatch();
                        reconnect();
                    } catch (SQLException reconnectEx) {
                        log.error("Failed to reconnect", reconnectEx);
                    }

                    Thread.sleep(1000L * retries);
                }
            }
        }

        @Override
        public void close() throws Exception {
            try {
                flush();
            } finally {
                if (statement != null) {
                    statement.close();
                }
                if (connection != null) {
                    connection.close();
                }
                if (dataSource != null) {
                    dataSource.close();
                }
                log.info("Database connection closed");
            }
        }
    }

    /**
     * Multi-table JDBC sink that routes records to different tables.
     */
    @Slf4j
    public static class MultiTableJdbcSink<T> implements SinkFunction<T> {
        private final DataSource dataSource;
        private final TableRouter<T> router;
        private final int batchSize;
        private final long batchIntervalMs;

        private transient Connection connection;
        private transient Map<String, TableBatchContext> tableBatches;
        private transient long lastFlushTime;

        public MultiTableJdbcSink(
                DataSource dataSource,
                TableRouter<T> router,
                int batchSize,
                long batchIntervalMs) {
            this.dataSource = dataSource;
            this.router = router;
            this.batchSize = batchSize;
            this.batchIntervalMs = batchIntervalMs;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            connection = dataSource.getConnection();
            connection.setAutoCommit(false);
            tableBatches = new HashMap<>();
            lastFlushTime = System.currentTimeMillis();
        }

        @Override
        public void invoke(T value, Context context) throws Exception {
            TableConfig config = router.route(value);
            TableBatchContext batchCtx = tableBatches.computeIfAbsent(
                config.tableName(),
                k -> {
                    try {
                        return new TableBatchContext(
                            connection.prepareStatement(config.sql()),
                            new ArrayList<>()
                        );
                    } catch (SQLException e) {
                        throw new RuntimeException("Failed to create statement", e);
                    }
                }
            );

            batchCtx.batch().add(new Tuple2<>(value, config.statementBuilder()));

            // Flush if any batch reached size limit or timeout exceeded
            if (batchCtx.batch().size() >= batchSize ||
                System.currentTimeMillis() - lastFlushTime >= batchIntervalMs) {
                flushAll();
            }
        }

        private void flushAll() throws SQLException {
            if (tableBatches.isEmpty()) {
                return;
            }

            try {
                for (Map.Entry<String, TableBatchContext> entry : tableBatches.entrySet()) {
                    String tableName = entry.getKey();
                    TableBatchContext ctx = entry.getValue();

                    if (!ctx.batch().isEmpty()) {
                        for (Tuple2<T, JdbcStatementBuilder<T>> tuple : ctx.batch()) {
                            tuple.f1.accept(ctx.statement(), tuple.f0);
                            ctx.statement().addBatch();
                        }

                        int[] results = ctx.statement().executeBatch();
                        log.debug("Flushed {} records to table {}", ctx.batch().size(), tableName);
                        ctx.batch().clear();
                        ctx.statement().clearBatch();
                    }
                }

                connection.commit();
                lastFlushTime = System.currentTimeMillis();

            } catch (Exception e) {
                connection.rollback();
                log.error("Failed to flush batches, rolled back", e);
                throw e;
            }
        }

        @Override
        public void close() throws Exception {
            try {
                flushAll();
            } finally {
                for (TableBatchContext ctx : tableBatches.values()) {
                    ctx.statement().close();
                }
                if (connection != null) {
                    connection.close();
                }
            }
        }

        private record TableBatchContext(
            PreparedStatement statement,
            List<Tuple2<T, JdbcStatementBuilder<T>>> batch
        ) {}
    }

    /**
     * XA Transactional sink for distributed transactions.
     */
    @Slf4j
    public static class XATransactionalJdbcSink<T> implements SinkFunction<T> {
        private final List<DataSource> dataSources;
        private final List<TransactionalWrite<T>> writes;

        private transient List<Connection> connections;
        private transient List<PreparedStatement> statements;
        private transient List<T> batch;

        public XATransactionalJdbcSink(
                List<DataSource> dataSources,
                List<TransactionalWrite<T>> writes) {
            this.dataSources = dataSources;
            this.writes = writes;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            connections = new ArrayList<>();
            statements = new ArrayList<>();
            batch = new ArrayList<>();

            // Open connections and prepare statements
            for (int i = 0; i < dataSources.size(); i++) {
                Connection conn = dataSources.get(i).getConnection();
                conn.setAutoCommit(false);
                connections.add(conn);
            }

            for (TransactionalWrite<T> write : writes) {
                PreparedStatement stmt = connections.get(write.datasourceIndex()).prepareStatement(write.sql());
                statements.add(stmt);
            }
        }

        @Override
        public void invoke(T value, Context context) throws Exception {
            batch.add(value);

            if (batch.size() >= 100) {
                flush();
            }
        }

        private void flush() throws Exception {
            if (batch.isEmpty()) {
                return;
            }

            try {
                // Prepare all statements
                for (T record : batch) {
                    for (int i = 0; i < writes.size(); i++) {
                        TransactionalWrite<T> write = writes.get(i);
                        PreparedStatement stmt = statements.get(i);
                        write.statementBuilder().accept(stmt, record);
                        stmt.addBatch();
                    }
                }

                // Execute all batches
                for (PreparedStatement stmt : statements) {
                    stmt.executeBatch();
                    stmt.clearBatch();
                }

                // Commit all connections (XA commit in real implementation)
                for (Connection conn : connections) {
                    conn.commit();
                }

                log.info("XA transaction committed: {} records", batch.size());
                batch.clear();

            } catch (Exception e) {
                // Rollback all connections
                for (Connection conn : connections) {
                    try {
                        conn.rollback();
                    } catch (SQLException rollbackEx) {
                        log.error("Failed to rollback", rollbackEx);
                    }
                }
                log.error("XA transaction failed, rolled back", e);
                throw e;
            }
        }

        @Override
        public void close() throws Exception {
            try {
                flush();
            } finally {
                for (PreparedStatement stmt : statements) {
                    stmt.close();
                }
                for (Connection conn : connections) {
                    conn.close();
                }
            }
        }
    }

    /**
     * SCD Type 2 sink for slowly changing dimensions.
     */
    @Slf4j
    public static class SCD2JdbcSink<T> implements SinkFunction<T> {
        private final DataSource dataSource;
        private final String tableName;
        private final String keyColumn;
        private final List<String> valueColumns;

        private transient Connection connection;
        private transient PreparedStatement selectStmt;
        private transient PreparedStatement updateStmt;
        private transient PreparedStatement insertStmt;

        public SCD2JdbcSink(
                DataSource dataSource,
                String tableName,
                String keyColumn,
                List<String> valueColumns) {
            this.dataSource = dataSource;
            this.tableName = tableName;
            this.keyColumn = keyColumn;
            this.valueColumns = valueColumns;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            connection = dataSource.getConnection();
            connection.setAutoCommit(false);

            // Prepare statements
            selectStmt = connection.prepareStatement(
                String.format("SELECT * FROM %s WHERE %s = ? AND is_current = true",
                    tableName, keyColumn)
            );

            updateStmt = connection.prepareStatement(
                String.format("UPDATE %s SET is_current = false, end_date = ? WHERE %s = ? AND is_current = true",
                    tableName, keyColumn)
            );

            String columns = String.join(", ", valueColumns);
            String placeholders = String.join(", ", Collections.nCopies(valueColumns.size() + 4, "?"));
            insertStmt = connection.prepareStatement(
                String.format("INSERT INTO %s (%s, %s, is_current, start_date, end_date) VALUES (%s)",
                    tableName, keyColumn, columns, placeholders)
            );
        }

        @Override
        public void invoke(T value, Context context) throws Exception {
            // This is a simplified SCD2 implementation
            // In reality, you'd extract key and values from the record
            String key = extractKey(value);
            Map<String, Object> values = extractValues(value);

            try {
                // Check if current record exists
                selectStmt.setString(1, key);
                ResultSet rs = selectStmt.executeQuery();

                boolean hasChanged = false;
                if (rs.next()) {
                    // Check if values changed
                    for (String col : valueColumns) {
                        Object oldValue = rs.getObject(col);
                        Object newValue = values.get(col);
                        if (!Objects.equals(oldValue, newValue)) {
                            hasChanged = true;
                            break;
                        }
                    }
                } else {
                    hasChanged = true; // New record
                }

                if (hasChanged) {
                    // Expire current record
                    updateStmt.setTimestamp(1, new Timestamp(System.currentTimeMillis()));
                    updateStmt.setString(2, key);
                    updateStmt.executeUpdate();

                    // Insert new version
                    insertStmt.setString(1, key);
                    int idx = 2;
                    for (String col : valueColumns) {
                        insertStmt.setObject(idx++, values.get(col));
                    }
                    insertStmt.setBoolean(idx++, true);
                    insertStmt.setTimestamp(idx++, new Timestamp(System.currentTimeMillis()));
                    insertStmt.setTimestamp(idx++, Timestamp.valueOf("9999-12-31 23:59:59"));
                    insertStmt.executeUpdate();

                    connection.commit();
                    log.debug("SCD2 update: key={}", key);
                }

            } catch (SQLException e) {
                connection.rollback();
                log.error("SCD2 operation failed", e);
                throw e;
            }
        }

        private String extractKey(T value) {
            // Extract key from record
            return "";
        }

        private Map<String, Object> extractValues(T value) {
            // Extract values from record
            return new HashMap<>();
        }

        @Override
        public void close() throws Exception {
            if (selectStmt != null) selectStmt.close();
            if (updateStmt != null) updateStmt.close();
            if (insertStmt != null) insertStmt.close();
            if (connection != null) connection.close();
        }
    }

    /**
     * Partitioned table sink for time-series data.
     */
    @Slf4j
    public static class PartitionedTableSink<T> implements SinkFunction<T> {
        private final DataSource dataSource;
        private final String baseTableName;
        private final PartitionSelector<T> partitionSelector;
        private final String sqlTemplate;
        private final JdbcStatementBuilder<T> statementBuilder;

        private transient Connection connection;
        private transient Map<String, PreparedStatement> statementCache;

        public PartitionedTableSink(
                DataSource dataSource,
                String baseTableName,
                PartitionSelector<T> partitionSelector,
                String sqlTemplate,
                JdbcStatementBuilder<T> statementBuilder) {
            this.dataSource = dataSource;
            this.baseTableName = baseTableName;
            this.partitionSelector = partitionSelector;
            this.sqlTemplate = sqlTemplate;
            this.statementBuilder = statementBuilder;
        }

        @Override
        public void open(Configuration parameters) throws Exception {
            connection = dataSource.getConnection();
            connection.setAutoCommit(false);
            statementCache = new HashMap<>();
        }

        @Override
        public void invoke(T value, Context context) throws Exception {
            String tableName = partitionSelector.selectPartition(value);
            PreparedStatement stmt = statementCache.computeIfAbsent(tableName, k -> {
                try {
                    String sql = sqlTemplate.replace("{table}", k);
                    return connection.prepareStatement(sql);
                } catch (SQLException e) {
                    throw new RuntimeException("Failed to create statement", e);
                }
            });

            statementBuilder.accept(stmt, value);
            stmt.addBatch();

            // Execute batch every 100 records
            if (statementCache.values().stream().mapToInt(s -> {
                try {
                    return s.toString().split("addBatch").length - 1;
                } catch (Exception e) {
                    return 0;
                }
            }).sum() >= 100) {
                flush();
            }
        }

        private void flush() throws SQLException {
            try {
                for (Map.Entry<String, PreparedStatement> entry : statementCache.entrySet()) {
                    entry.getValue().executeBatch();
                    entry.getValue().clearBatch();
                }
                connection.commit();
            } catch (SQLException e) {
                connection.rollback();
                throw e;
            }
        }

        @Override
        public void close() throws Exception {
            try {
                flush();
            } finally {
                for (PreparedStatement stmt : statementCache.values()) {
                    stmt.close();
                }
                if (connection != null) {
                    connection.close();
                }
            }
        }
    }

    // ==================== Helper Interfaces and Classes ====================

    @FunctionalInterface
    public interface JdbcStatementBuilder<T> {
        void accept(PreparedStatement statement, T record) throws SQLException;
    }

    @FunctionalInterface
    public interface TableRouter<T> {
        TableConfig route(T record);
    }

    @FunctionalInterface
    public interface PartitionSelector<T> {
        String selectPartition(T record);
    }

    public record TableConfig(
        String tableName,
        String sql,
        JdbcStatementBuilder statementBuilder
    ) {}

    public record TransactionalWrite<T>(
        int datasourceIndex,
        String sql,
        JdbcStatementBuilder<T> statementBuilder
    ) {}

    // ==================== Data Models ====================

    public record Event(String id, String type, String payload, long timestamp) {}

    public record UserProfile(String userId, String name, String email) {}

    public record Metric(String name, double value, Map<String, String> tags, long timestamp) {
        public String tagsAsJson() {
            return tags.toString(); // Simplified
        }
    }

    public record GenericEvent(
        String id,
        String category,
        String action,
        String message,
        Map<String, String> attributes,
        long timestamp
    ) {
        public String toJson() {
            return String.format("{\"id\":\"%s\",\"category\":\"%s\",\"action\":\"%s\"}",
                id, category, action);
        }
    }

    public record Order(
        String orderId,
        String customerId,
        String productId,
        int quantity,
        double amount,
        long timestamp
    ) {}

    public record DimensionRecord(
        String key,
        Map<String, Object> attributes
    ) {}

    public record TimeSeriesData(
        String sensorId,
        double value,
        long timestamp
    ) {}

    // ==================== Utility Methods ====================

    private static DataSource createDataSource(String jdbcUrl) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername("user");
        config.setPassword("password");
        config.setMaximumPoolSize(10);
        config.setMinimumIdle(2);
        return new HikariDataSource(config);
    }

    /**
     * Example: Create MySQL table schema.
     */
    public static void createMySQLTables(Connection conn) throws SQLException {
        String[] ddl = {
            """
            CREATE TABLE IF NOT EXISTS events (
                id VARCHAR(255) PRIMARY KEY,
                type VARCHAR(100) NOT NULL,
                payload TEXT,
                timestamp TIMESTAMP NOT NULL,
                INDEX idx_type (type),
                INDEX idx_timestamp (timestamp)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            """,

            """
            CREATE TABLE IF NOT EXISTS metrics (
                id BIGINT AUTO_INCREMENT PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                value DOUBLE NOT NULL,
                tags JSON,
                timestamp TIMESTAMP NOT NULL,
                INDEX idx_name_timestamp (name, timestamp)
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
            """
        };

        try (Statement stmt = conn.createStatement()) {
            for (String sql : ddl) {
                stmt.execute(sql);
            }
            log.info("MySQL tables created successfully");
        }
    }

    /**
     * Example: Create PostgreSQL table schema.
     */
    public static void createPostgreSQLTables(Connection conn) throws SQLException {
        String[] ddl = {
            """
            CREATE TABLE IF NOT EXISTS user_profiles (
                user_id VARCHAR(255) PRIMARY KEY,
                name VARCHAR(255) NOT NULL,
                email VARCHAR(255) UNIQUE NOT NULL,
                updated_at TIMESTAMP NOT NULL
            )
            """,

            """
            CREATE TABLE IF NOT EXISTS dim_products (
                id BIGSERIAL PRIMARY KEY,
                product_id VARCHAR(255) NOT NULL,
                product_name VARCHAR(255),
                category VARCHAR(100),
                price DECIMAL(10, 2),
                is_current BOOLEAN NOT NULL DEFAULT true,
                start_date TIMESTAMP NOT NULL,
                end_date TIMESTAMP NOT NULL,
                INDEX idx_product_current (product_id, is_current)
            )
            """
        };

        try (Statement stmt = conn.createStatement()) {
            for (String sql : ddl) {
                stmt.execute(sql);
            }
            log.info("PostgreSQL tables created successfully");
        }
    }

    /**
     * Example: Optimize connection pool settings.
     */
    public static HikariConfig createOptimizedHikariConfig(String jdbcUrl, String username, String password) {
        HikariConfig config = new HikariConfig();

        // Connection settings
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(username);
        config.setPassword(password);

        // Pool sizing
        config.setMaximumPoolSize(20);           // Max connections
        config.setMinimumIdle(5);                // Min idle connections
        config.setConnectionTimeout(30000);       // 30 seconds
        config.setIdleTimeout(600000);           // 10 minutes
        config.setMaxLifetime(1800000);          // 30 minutes

        // Performance
        config.setAutoCommit(false);             // Manual commit for batching
        config.setLeakDetectionThreshold(60000); // 1 minute leak detection

        // MySQL specific optimizations
        config.addDataSourceProperty("cachePrepStmts", "true");
        config.addDataSourceProperty("prepStmtCacheSize", "250");
        config.addDataSourceProperty("prepStmtCacheSqlLimit", "2048");
        config.addDataSourceProperty("useServerPrepStmts", "true");
        config.addDataSourceProperty("rewriteBatchedStatements", "true");

        return config;
    }
}
```

## Configuration Example

```yaml
# Flink JDBC sink configuration
execution:
  parallelism: 4

# Checkpointing
checkpointing:
  interval: 60s
  mode: exactly-once
  timeout: 10min

# JDBC sink settings
jdbc:
  mysql:
    url: "jdbc:mysql://localhost:3306/analytics?rewriteBatchedStatements=true"
    driver: "com.mysql.cj.jdbc.Driver"
    username: "flink_user"
    password: "flink_password"
    batch-size: 1000
    batch-interval-ms: 200
    max-retries: 5

  postgresql:
    url: "jdbc:postgresql://localhost:5432/warehouse"
    driver: "org.postgresql.Driver"
    username: "postgres"
    password: "postgres"
    batch-size: 500
    batch-interval-ms: 1000
    max-retries: 3

# HikariCP connection pool
hikari:
  maximum-pool-size: 20
  minimum-idle: 5
  connection-timeout: 30000
  idle-timeout: 600000
  max-lifetime: 1800000
  auto-commit: false
  leak-detection-threshold: 60000

  # MySQL optimizations
  data-source-properties:
    cachePrepStmts: true
    prepStmtCacheSize: 250
    prepStmtCacheSqlLimit: 2048
    useServerPrepStmts: true
    rewriteBatchedStatements: true

# State backend
state:
  backend: rocksdb
  checkpoints-dir: s3://bucket/checkpoints
```

## Best Practices

1. **Connection Pooling**: Always use HikariCP for production. Configure pool size based on parallelism (2-5 connections per task).

2. **Batch Size**: Optimal batch size is 500-1000 records. Larger batches improve throughput but increase checkpoint size and recovery time.

3. **Batch Interval**: Set batch interval to 200-1000ms to balance latency vs throughput. Lower for real-time, higher for bulk loading.

4. **Exactly-Once**: Use UPSERT (INSERT ... ON CONFLICT) for idempotent writes in PostgreSQL. For MySQL, use INSERT ... ON DUPLICATE KEY UPDATE.

5. **Retry Logic**: Implement exponential backoff for transient errors. Separate connection errors (reconnect) from data errors (fail fast).

6. **Type Mapping**: Use proper SQL types: VARCHAR for strings, TIMESTAMP for dates, DECIMAL for money, JSON for complex objects.

7. **Indexing**: Create indexes on frequently queried columns. Include timestamp columns for time-series data. Use composite indexes wisely.

8. **Schema Evolution**: Handle missing columns gracefully. Use ALTER TABLE for schema changes or dual-write pattern during migrations.

9. **Monitoring**: Track batch size, flush latency, connection pool stats, and error rates. Set up alerts for connection pool exhaustion.

10. **Performance**: Enable prepared statement caching and batch rewriting in JDBC URL. Use connection pooling. Disable auto-commit for batching.

## Related Documentation

- [Flink JDBC Connector](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/connectors/datastream/jdbc/)
- [HikariCP Configuration](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
- [JDBC Best Practices](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/connectors/datastream/jdbc/#best-practices)
- [PostgreSQL UPSERT](https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT)
- [MySQL Batch Optimization](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-configuration-properties.html)
