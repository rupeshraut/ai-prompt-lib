# Create Queryable State

Create queryable state implementations in Apache Flink 2.0 for accessing job state externally via REST API. Queryable state enables real-time state queries without interrupting stream processing.

## Requirements

Your queryable state implementation should include:

- **Queryable State Backend**: Enable queryable state on state backend
- **State Server**: Configure state server for external queries
- **State Descriptors**: Register state descriptors as queryable
- **Client Queries**: HTTP/REST client for querying state
- **Key-Value Lookups**: Query specific keys from keyed state
- **State Types**: Support ValueState, ListState, MapState queries
- **Caching Strategy**: Client-side caching to reduce query load
- **Error Handling**: Handle missing keys and connection errors
- **Authentication**: Secure state access with authentication
- **Query Patterns**: Implement common query patterns (single key, batch, scan)
- **Monitoring**: Track query latency and state size metrics
- **Documentation**: Clear API documentation for state consumers

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **Approach**: Use QueryableStateClient for external queries
- **Type Safety**: Use strongly-typed state descriptors
- **Immutability**: Use records for state value objects
- **Async Queries**: Use CompletableFuture for non-blocking queries

## Template Example

```java
import org.apache.flink.api.common.state.*;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.queryablestate.client.QueryableStateClient;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.util.Collector;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.concurrent.CompletableFuture;
import java.time.Duration;
import com.google.common.cache.*;

/**
 * Demonstrates queryable state patterns in Apache Flink 2.0.
 * Enables external applications to query Flink job state via REST API.
 */
@Slf4j
public class QueryableStateImplementations {

    /**
     * Basic queryable value state: Store user sessions.
     * Use case: Query current user session from external API.
     */
    public static DataStream<SessionUpdate> queryableSessionState(
            DataStream<UserActivity> activities) {

        return activities
                .keyBy(UserActivity::userId)
                .process(new SessionProcessor())
                .doOnNext(update -> log.debug("Session updated: {}", update.userId()));
    }

    /**
     * Queryable map state: Store product inventory.
     * Use case: Query product stock levels in real-time.
     */
    public static DataStream<InventoryUpdate> queryableInventoryState(
            DataStream<StockChange> changes) {

        return changes
                .keyBy(StockChange::warehouseId)
                .process(new InventoryProcessor())
                .doOnNext(update -> log.debug("Inventory updated: {}", update.warehouseId()));
    }

    /**
     * Queryable list state: Store recent events.
     * Use case: Query last N events for debugging/monitoring.
     */
    public static DataStream<EventUpdate> queryableEventHistory(
            DataStream<Event> events) {

        return events
                .keyBy(Event::streamId)
                .process(new EventHistoryProcessor())
                .doOnNext(update -> log.debug("Event history updated: {}", update.streamId()));
    }

    /**
     * Queryable aggregation state: Real-time metrics.
     * Use case: Query current metrics (counts, sums, averages).
     */
    public static DataStream<MetricUpdate> queryableMetricsState(
            DataStream<MetricEvent> events) {

        return events
                .keyBy(MetricEvent::metricName)
                .process(new MetricsAggregator())
                .doOnNext(update -> log.info("Metrics updated: {} = {}",
                    update.metricName(), update.value()));
    }

    // ==================== Keyed Process Functions with Queryable State ====================

    /**
     * Session processor with queryable state.
     */
    @Slf4j
    private static class SessionProcessor extends KeyedProcessFunction<String, UserActivity, SessionUpdate> {
        private static final String SESSION_STATE_NAME = "user-sessions";
        private transient ValueState<UserSession> sessionState;

        @Override
        public void open(Configuration parameters) {
            ValueStateDescriptor<UserSession> descriptor = new ValueStateDescriptor<>(
                    SESSION_STATE_NAME,
                    TypeInformation.of(UserSession.class)
            );
            // Make state queryable
            descriptor.setQueryable(SESSION_STATE_NAME);
            sessionState = getRuntimeContext().getState(descriptor);
        }

        @Override
        public void processElement(
                UserActivity activity,
                Context ctx,
                Collector<SessionUpdate> out) throws Exception {

            UserSession session = sessionState.value();
            if (session == null) {
                session = new UserSession(
                        activity.userId(),
                        activity.timestamp(),
                        activity.timestamp(),
                        1,
                        new ArrayList<>()
                );
            } else {
                session = new UserSession(
                        session.userId(),
                        session.startTime(),
                        activity.timestamp(),
                        session.activityCount() + 1,
                        session.recentActivities()
                );
            }

            // Keep last 10 activities
            session.recentActivities().add(activity.activityType());
            if (session.recentActivities().size() > 10) {
                session.recentActivities().remove(0);
            }

            sessionState.update(session);
            out.collect(new SessionUpdate(
                    activity.userId(),
                    session.activityCount(),
                    activity.timestamp()
            ));

            log.debug("Updated session for user {}: {} activities",
                    activity.userId(), session.activityCount());
        }
    }

    /**
     * Inventory processor with queryable map state.
     */
    @Slf4j
    private static class InventoryProcessor extends KeyedProcessFunction<String, StockChange, InventoryUpdate> {
        private static final String INVENTORY_STATE_NAME = "warehouse-inventory";
        private transient MapState<String, Integer> inventoryState;

        @Override
        public void open(Configuration parameters) {
            MapStateDescriptor<String, Integer> descriptor = new MapStateDescriptor<>(
                    INVENTORY_STATE_NAME,
                    TypeInformation.of(String.class),
                    TypeInformation.of(Integer.class)
            );
            // Make state queryable
            descriptor.setQueryable(INVENTORY_STATE_NAME);
            inventoryState = getRuntimeContext().getMapState(descriptor);
        }

        @Override
        public void processElement(
                StockChange change,
                Context ctx,
                Collector<InventoryUpdate> out) throws Exception {

            String productId = change.productId();
            Integer currentStock = inventoryState.get(productId);
            if (currentStock == null) {
                currentStock = 0;
            }

            int newStock = currentStock + change.quantity();
            inventoryState.put(productId, newStock);

            out.collect(new InventoryUpdate(
                    change.warehouseId(),
                    productId,
                    newStock,
                    System.currentTimeMillis()
            ));

            if (newStock < 10) {
                log.warn("Low stock alert: {} in warehouse {} has {} units",
                        productId, change.warehouseId(), newStock);
            }
        }
    }

    /**
     * Event history processor with queryable list state.
     */
    @Slf4j
    private static class EventHistoryProcessor extends KeyedProcessFunction<String, Event, EventUpdate> {
        private static final String EVENT_HISTORY_STATE_NAME = "event-history";
        private static final int MAX_HISTORY_SIZE = 100;
        private transient ListState<Event> historyState;

        @Override
        public void open(Configuration parameters) {
            ListStateDescriptor<Event> descriptor = new ListStateDescriptor<>(
                    EVENT_HISTORY_STATE_NAME,
                    TypeInformation.of(Event.class)
            );
            // Make state queryable
            descriptor.setQueryable(EVENT_HISTORY_STATE_NAME);
            historyState = getRuntimeContext().getListState(descriptor);
        }

        @Override
        public void processElement(
                Event event,
                Context ctx,
                Collector<EventUpdate> out) throws Exception {

            // Add to history
            List<Event> history = new ArrayList<>();
            historyState.get().forEach(history::add);
            history.add(event);

            // Keep only last N events
            if (history.size() > MAX_HISTORY_SIZE) {
                history = history.subList(history.size() - MAX_HISTORY_SIZE, history.size());
            }

            // Update state
            historyState.clear();
            historyState.addAll(history);

            out.collect(new EventUpdate(
                    event.streamId(),
                    history.size(),
                    event.timestamp()
            ));
        }
    }

    /**
     * Metrics aggregator with queryable state.
     */
    @Slf4j
    private static class MetricsAggregator extends KeyedProcessFunction<String, MetricEvent, MetricUpdate> {
        private static final String METRICS_STATE_NAME = "realtime-metrics";
        private transient ValueState<MetricAggregate> metricsState;

        @Override
        public void open(Configuration parameters) {
            ValueStateDescriptor<MetricAggregate> descriptor = new ValueStateDescriptor<>(
                    METRICS_STATE_NAME,
                    TypeInformation.of(MetricAggregate.class)
            );
            // Make state queryable
            descriptor.setQueryable(METRICS_STATE_NAME);
            metricsState = getRuntimeContext().getState(descriptor);
        }

        @Override
        public void processElement(
                MetricEvent event,
                Context ctx,
                Collector<MetricUpdate> out) throws Exception {

            MetricAggregate current = metricsState.value();
            if (current == null) {
                current = new MetricAggregate(0, 0.0, 0.0, Double.MAX_VALUE, Double.MIN_VALUE);
            }

            MetricAggregate updated = new MetricAggregate(
                    current.count() + 1,
                    current.sum() + event.value(),
                    (current.sum() + event.value()) / (current.count() + 1),
                    Math.min(current.min(), event.value()),
                    Math.max(current.max(), event.value())
            );

            metricsState.update(updated);

            out.collect(new MetricUpdate(
                    event.metricName(),
                    updated.average(),
                    updated.count(),
                    System.currentTimeMillis()
            ));
        }
    }

    // ==================== Queryable State Client ====================

    /**
     * Client for querying Flink state externally.
     */
    @Slf4j
    public static class FlinkStateClient implements AutoCloseable {
        private final QueryableStateClient client;
        private final String jobId;
        private final LoadingCache<String, Object> cache;

        public FlinkStateClient(String proxyHost, int proxyPort, String jobId) {
            this.client = new QueryableStateClient(proxyHost, proxyPort);
            this.jobId = jobId;

            // Create cache with 5-minute TTL
            this.cache = CacheBuilder.newBuilder()
                    .maximumSize(10000)
                    .expireAfterWrite(Duration.ofMinutes(5))
                    .recordStats()
                    .build(new CacheLoader<String, Object>() {
                        @Override
                        public Object load(String key) throws Exception {
                            throw new UnsupportedOperationException("Use LoadingCache.get with loader");
                        }
                    });
        }

        /**
         * Query user session state.
         */
        public CompletableFuture<UserSession> queryUserSession(String userId) {
            String cacheKey = "session:" + userId;

            // Check cache first
            UserSession cached = (UserSession) cache.getIfPresent(cacheKey);
            if (cached != null) {
                return CompletableFuture.completedFuture(cached);
            }

            // Query from Flink
            return client.getKvState(
                    jobId,
                    "user-sessions",
                    userId,
                    new org.apache.flink.api.common.typeinfo.TypeHint<String>() {},
                    new ValueStateDescriptor<>(
                            "user-sessions",
                            TypeInformation.of(UserSession.class)
                    )
            ).thenApply(state -> {
                try {
                    UserSession session = state.value();
                    cache.put(cacheKey, session);
                    return session;
                } catch (Exception e) {
                    log.error("Failed to read session state", e);
                    return null;
                }
            });
        }

        /**
         * Query warehouse inventory.
         */
        public CompletableFuture<Map<String, Integer>> queryWarehouseInventory(String warehouseId) {
            return client.getKvState(
                    jobId,
                    "warehouse-inventory",
                    warehouseId,
                    new org.apache.flink.api.common.typeinfo.TypeHint<String>() {},
                    new MapStateDescriptor<>(
                            "warehouse-inventory",
                            TypeInformation.of(String.class),
                            TypeInformation.of(Integer.class)
                    )
            ).thenApply(state -> {
                try {
                    Map<String, Integer> inventory = new HashMap<>();
                    for (Map.Entry<String, Integer> entry : state.entries()) {
                        inventory.put(entry.getKey(), entry.getValue());
                    }
                    return inventory;
                } catch (Exception e) {
                    log.error("Failed to read inventory state", e);
                    return Collections.emptyMap();
                }
            });
        }

        /**
         * Query product stock level.
         */
        public CompletableFuture<Integer> queryProductStock(String warehouseId, String productId) {
            String cacheKey = "stock:" + warehouseId + ":" + productId;

            Integer cached = (Integer) cache.getIfPresent(cacheKey);
            if (cached != null) {
                return CompletableFuture.completedFuture(cached);
            }

            return queryWarehouseInventory(warehouseId)
                    .thenApply(inventory -> {
                        Integer stock = inventory.getOrDefault(productId, 0);
                        cache.put(cacheKey, stock);
                        return stock;
                    });
        }

        /**
         * Query event history.
         */
        public CompletableFuture<List<Event>> queryEventHistory(String streamId) {
            return client.getKvState(
                    jobId,
                    "event-history",
                    streamId,
                    new org.apache.flink.api.common.typeinfo.TypeHint<String>() {},
                    new ListStateDescriptor<>(
                            "event-history",
                            TypeInformation.of(Event.class)
                    )
            ).thenApply(state -> {
                try {
                    List<Event> events = new ArrayList<>();
                    state.get().forEach(events::add);
                    return events;
                } catch (Exception e) {
                    log.error("Failed to read event history", e);
                    return Collections.emptyList();
                }
            });
        }

        /**
         * Query metrics.
         */
        public CompletableFuture<MetricAggregate> queryMetrics(String metricName) {
            return client.getKvState(
                    jobId,
                    "realtime-metrics",
                    metricName,
                    new org.apache.flink.api.common.typeinfo.TypeHint<String>() {},
                    new ValueStateDescriptor<>(
                            "realtime-metrics",
                            TypeInformation.of(MetricAggregate.class)
                    )
            ).thenApply(state -> {
                try {
                    return state.value();
                } catch (Exception e) {
                    log.error("Failed to read metrics state", e);
                    return null;
                }
            });
        }

        /**
         * Batch query multiple keys.
         */
        public CompletableFuture<Map<String, UserSession>> queryMultipleUsers(List<String> userIds) {
            List<CompletableFuture<Map.Entry<String, UserSession>>> futures = userIds.stream()
                    .map(userId -> queryUserSession(userId)
                            .thenApply(session -> Map.entry(userId, session)))
                    .toList();

            return CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
                    .thenApply(v -> {
                        Map<String, UserSession> results = new HashMap<>();
                        for (CompletableFuture<Map.Entry<String, UserSession>> future : futures) {
                            try {
                                Map.Entry<String, UserSession> entry = future.join();
                                if (entry.getValue() != null) {
                                    results.put(entry.getKey(), entry.getValue());
                                }
                            } catch (Exception e) {
                                log.error("Failed to get user session", e);
                            }
                        }
                        return results;
                    });
        }

        /**
         * Get cache statistics.
         */
        public CacheStats getCacheStats() {
            return cache.stats();
        }

        @Override
        public void close() throws Exception {
            client.close();
        }
    }

    // ==================== REST API Wrapper ====================

    /**
     * REST API wrapper for queryable state.
     * Example using simple HTTP server.
     */
    @Slf4j
    public static class QueryableStateRestAPI {
        private final FlinkStateClient stateClient;

        public QueryableStateRestAPI(FlinkStateClient stateClient) {
            this.stateClient = stateClient;
        }

        /**
         * GET /api/users/{userId}/session
         */
        public UserSession getUserSession(String userId) throws Exception {
            return stateClient.queryUserSession(userId)
                    .get(5, java.util.concurrent.TimeUnit.SECONDS);
        }

        /**
         * GET /api/warehouse/{warehouseId}/inventory
         */
        public Map<String, Integer> getWarehouseInventory(String warehouseId) throws Exception {
            return stateClient.queryWarehouseInventory(warehouseId)
                    .get(5, java.util.concurrent.TimeUnit.SECONDS);
        }

        /**
         * GET /api/warehouse/{warehouseId}/product/{productId}
         */
        public Integer getProductStock(String warehouseId, String productId) throws Exception {
            return stateClient.queryProductStock(warehouseId, productId)
                    .get(5, java.util.concurrent.TimeUnit.SECONDS);
        }

        /**
         * GET /api/stream/{streamId}/history
         */
        public List<Event> getEventHistory(String streamId) throws Exception {
            return stateClient.queryEventHistory(streamId)
                    .get(5, java.util.concurrent.TimeUnit.SECONDS);
        }

        /**
         * GET /api/metrics/{metricName}
         */
        public MetricAggregate getMetrics(String metricName) throws Exception {
            return stateClient.queryMetrics(metricName)
                    .get(5, java.util.concurrent.TimeUnit.SECONDS);
        }

        /**
         * POST /api/users/batch
         * Body: ["user1", "user2", "user3"]
         */
        public Map<String, UserSession> batchGetUserSessions(List<String> userIds) throws Exception {
            return stateClient.queryMultipleUsers(userIds)
                    .get(10, java.util.concurrent.TimeUnit.SECONDS);
        }

        /**
         * GET /api/cache/stats
         */
        public Map<String, Object> getCacheStats() {
            CacheStats stats = stateClient.getCacheStats();
            return Map.of(
                    "hitRate", stats.hitRate(),
                    "missRate", stats.missRate(),
                    "hitCount", stats.hitCount(),
                    "missCount", stats.missCount(),
                    "loadCount", stats.loadCount(),
                    "evictionCount", stats.evictionCount()
            );
        }
    }

    // ==================== Data Models ====================

    public record UserActivity(String userId, String activityType, long timestamp) {}

    public record UserSession(
            String userId,
            long startTime,
            long lastActivityTime,
            int activityCount,
            List<String> recentActivities
    ) {}

    public record SessionUpdate(String userId, int activityCount, long timestamp) {}

    public record StockChange(String warehouseId, String productId, int quantity) {}

    public record InventoryUpdate(String warehouseId, String productId, int stock, long timestamp) {}

    public record Event(String streamId, String eventType, String payload, long timestamp) {}

    public record EventUpdate(String streamId, int historySize, long timestamp) {}

    public record MetricEvent(String metricName, double value, long timestamp) {}

    public record MetricAggregate(long count, double sum, double average, double min, double max) {}

    public record MetricUpdate(String metricName, double value, long count, long timestamp) {}

    // ==================== Example Usage ====================

    public static void main(String[] args) throws Exception {
        // Create state client
        FlinkStateClient client = new FlinkStateClient(
                "localhost",    // Queryable state proxy host
                9069,           // Queryable state proxy port
                "job-id-here"   // Flink job ID
        );

        try {
            // Query single user session
            UserSession session = client.queryUserSession("user123").get();
            log.info("User session: {}", session);

            // Query warehouse inventory
            Map<String, Integer> inventory = client.queryWarehouseInventory("warehouse1").get();
            log.info("Inventory: {}", inventory);

            // Query product stock
            Integer stock = client.queryProductStock("warehouse1", "product456").get();
            log.info("Product stock: {}", stock);

            // Batch query multiple users
            Map<String, UserSession> sessions = client.queryMultipleUsers(
                    List.of("user1", "user2", "user3")
            ).get();
            log.info("Batch sessions: {}", sessions);

            // Get cache stats
            CacheStats stats = client.getCacheStats();
            log.info("Cache hit rate: {}", stats.hitRate());

        } finally {
            client.close();
        }
    }
}
```

## Configuration Example

```yaml
# Flink queryable state configuration
execution:
  parallelism: 4

# Queryable state settings
queryable-state:
  enable: true
  proxy:
    port-range: 9069
    network-threads: 2
    query-threads: 4
  server:
    port-range: 9067
    network-threads: 2

# State backend (must support queryable state)
state:
  backend: rocksdb
  checkpoints-dir: s3://bucket/checkpoints
  backend.rocksdb.memory.managed: true

# Checkpointing
checkpointing:
  interval: 60s
  mode: exactly-once

# Client configuration (application.yml)
flink:
  queryable-state:
    proxy-host: localhost
    proxy-port: 9069
    job-id: "your-job-id"
    cache:
      max-size: 10000
      ttl-minutes: 5

# REST API configuration
server:
  port: 8080
rest:
  api:
    timeout-seconds: 5
    max-batch-size: 100
```

## Best Practices

1. **State Size**: Keep queryable state reasonably sized (under 100MB per key). Large state increases query latency and memory usage.

2. **Caching**: Implement client-side caching with TTL to reduce query load. Cache hit rate of 80%+ significantly improves performance.

3. **Query Timeout**: Set reasonable query timeouts (1-5 seconds). Long-running queries indicate state size or network issues.

4. **Batch Queries**: Use batch queries for multiple keys to reduce network overhead. Limit batch size to 100-1000 keys.

5. **State Types**: ValueState and MapState are most efficient for queryable state. ListState can be slow for large lists.

6. **Key Design**: Use meaningful, human-readable keys (user IDs, product SKUs). Avoid composite keys when possible.

7. **Monitoring**: Track query latency, cache hit rate, and error rates. Set up alerts for high latency or errors.

8. **State Evolution**: Handle schema evolution carefully. Add new fields with defaults. Use versioned state descriptors.

9. **Security**: Implement authentication and authorization for state queries. Restrict access to sensitive state.

10. **Performance**: Enable RocksDB state backend for better queryable state performance. Configure appropriate network threads.

## Related Documentation

- [Flink Queryable State](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/fault-tolerance/queryable_state/)
- [QueryableStateClient](https://nightlies.apache.org/flink/flink-docs-release-2.0/api/java/org/apache/flink/queryablestate/client/QueryableStateClient.html)
- [State Descriptors](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/fault-tolerance/state/)
- [State Backends](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/ops/state/state_backends/)
- [Queryable State Configuration](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/deployment/config/#queryable-state)
