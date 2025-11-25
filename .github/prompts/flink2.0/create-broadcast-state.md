# Create Broadcast State

Create broadcast state patterns in Apache Flink 2.0 for sharing configuration and reference data across parallel tasks. Broadcast state enables efficient coordination and enrichment without external lookups.

## Requirements

Your broadcast state implementation should include:

- **Broadcast Stream**: Create broadcast streams from configuration/reference data sources
- **State Descriptors**: Define MapStateDescriptor for broadcast state schema
- **Connect Operations**: Connect regular streams with broadcast streams
- **Broadcast Process Function**: Implement KeyedBroadcastProcessFunction or BroadcastProcessFunction
- **State Updates**: Handle broadcast side state updates (processBroadcastElement)
- **State Queries**: Query broadcast state from non-broadcast side (processElement)
- **Multi-Task Coordination**: Ensure consistent state across all parallel tasks
- **Dynamic Configuration**: Update rules, patterns, or configurations at runtime
- **Data Enrichment**: Join stream data with reference data without external calls
- **Version Control**: Track configuration versions and handle transitions
- **Error Handling**: Gracefully handle malformed broadcast data
- **Documentation**: Clear explanation of broadcast semantics and usage

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **Approach**: Use KeyedBroadcastProcessFunction for stateful enrichment
- **Type Safety**: Use strongly-typed state descriptors
- **Immutability**: Use records for broadcast data objects

## Template Example

```java
import org.apache.flink.api.common.state.*;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.streaming.api.datastream.BroadcastStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.functions.co.KeyedBroadcastProcessFunction;
import org.apache.flink.streaming.api.functions.co.BroadcastProcessFunction;
import org.apache.flink.util.Collector;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

/**
 * Demonstrates broadcast state patterns in Apache Flink 2.0.
 * Broadcast state allows sharing configuration and reference data across all parallel tasks.
 */
@Slf4j
public class BroadcastStatePatterns {

    /**
     * Basic broadcast state: Enrich transactions with dynamic rules.
     * Use case: Apply fraud detection rules that change at runtime.
     */
    public static DataStream<EnrichedTransaction> basicBroadcastState(
            DataStream<Transaction> transactions,
            DataStream<FraudRule> rules) {

        // Define broadcast state descriptor
        MapStateDescriptor<String, FraudRule> ruleStateDescriptor = new MapStateDescriptor<>(
                "fraud-rules",
                TypeInformation.of(String.class),
                TypeInformation.of(FraudRule.class)
        );

        // Create broadcast stream
        BroadcastStream<FraudRule> broadcastRules = rules.broadcast(ruleStateDescriptor);

        // Connect and process
        return transactions
                .keyBy(Transaction::userId)
                .connect(broadcastRules)
                .process(new FraudDetectionFunction(ruleStateDescriptor))
                .doOnNext(enriched -> log.info("Enriched transaction: {} with rules applied",
                        enriched.transactionId()));
    }

    /**
     * Configuration broadcast: Share application configuration across tasks.
     * Use case: Dynamic feature flags, thresholds, and settings.
     */
    public static DataStream<ProcessedEvent> configurationBroadcast(
            DataStream<Event> events,
            DataStream<Configuration> configs) {

        MapStateDescriptor<String, Configuration> configDescriptor = new MapStateDescriptor<>(
                "app-config",
                TypeInformation.of(String.class),
                TypeInformation.of(Configuration.class)
        );

        BroadcastStream<Configuration> broadcastConfig = configs.broadcast(configDescriptor);

        return events
                .keyBy(Event::eventType)
                .connect(broadcastConfig)
                .process(new ConfigurableEventProcessor(configDescriptor))
                .doOnNext(processed -> log.debug("Event processed with config version: {}",
                        processed.configVersion()));
    }

    /**
     * Reference data enrichment: Join stream with reference data (dimension table).
     * Use case: Enrich events with product catalog, user profiles, or location data.
     */
    public static DataStream<EnrichedOrder> referenceDataEnrichment(
            DataStream<Order> orders,
            DataStream<ProductInfo> products) {

        MapStateDescriptor<String, ProductInfo> productDescriptor = new MapStateDescriptor<>(
                "product-catalog",
                TypeInformation.of(String.class),
                TypeInformation.of(ProductInfo.class)
        );

        BroadcastStream<ProductInfo> broadcastProducts = products.broadcast(productDescriptor);

        return orders
                .keyBy(Order::customerId)
                .connect(broadcastProducts)
                .process(new ProductEnrichmentFunction(productDescriptor))
                .doOnNext(enriched -> log.info("Order {} enriched with product: {}",
                        enriched.orderId(), enriched.productName()));
    }

    /**
     * Multi-pattern matching: Apply multiple patterns/rules simultaneously.
     * Use case: CEP-like pattern matching with dynamic patterns.
     */
    public static DataStream<MatchedEvent> multiPatternMatching(
            DataStream<Event> events,
            DataStream<Pattern> patterns) {

        MapStateDescriptor<String, Pattern> patternDescriptor = new MapStateDescriptor<>(
                "patterns",
                TypeInformation.of(String.class),
                TypeInformation.of(Pattern.class)
        );

        BroadcastStream<Pattern> broadcastPatterns = patterns.broadcast(patternDescriptor);

        return events
                .keyBy(Event::userId)
                .connect(broadcastPatterns)
                .process(new PatternMatchingFunction(patternDescriptor))
                .doOnNext(match -> log.info("Pattern match: {} triggered by event {}",
                        match.patternId(), match.eventId()));
    }

    /**
     * Non-keyed broadcast: Apply broadcast without keying the main stream.
     * Use case: Global filtering, sampling, or processing decisions.
     */
    public static DataStream<FilteredData> nonKeyedBroadcast(
            DataStream<RawData> data,
            DataStream<FilterConfig> filters) {

        MapStateDescriptor<String, FilterConfig> filterDescriptor = new MapStateDescriptor<>(
                "filters",
                TypeInformation.of(String.class),
                TypeInformation.of(FilterConfig.class)
        );

        BroadcastStream<FilterConfig> broadcastFilters = filters.broadcast(filterDescriptor);

        return data
                .connect(broadcastFilters)
                .process(new GlobalFilterFunction(filterDescriptor))
                .doOnNext(filtered -> log.debug("Data filtered with config: {}", filtered.id()));
    }

    /**
     * Versioned broadcast state: Track and handle configuration versions.
     * Use case: Ensure consistent processing during configuration updates.
     */
    public static DataStream<VersionedResult> versionedBroadcastState(
            DataStream<Message> messages,
            DataStream<VersionedConfig> configs) {

        MapStateDescriptor<String, VersionedConfig> configDescriptor = new MapStateDescriptor<>(
                "versioned-config",
                TypeInformation.of(String.class),
                TypeInformation.of(VersionedConfig.class)
        );

        BroadcastStream<VersionedConfig> broadcastConfig = configs.broadcast(configDescriptor);

        return messages
                .keyBy(Message::partition)
                .connect(broadcastConfig)
                .process(new VersionAwareProcessor(configDescriptor))
                .doOnNext(result -> log.info("Processed with config version: {}", result.version()));
    }

    /**
     * Multi-broadcast state: Use multiple broadcast states in single function.
     * Use case: Complex enrichment with multiple reference data sources.
     */
    public static DataStream<ComplexEnriched> multiBroadcastState(
            DataStream<UserActivity> activities,
            DataStream<UserProfile> profiles,
            DataStream<LocationData> locations) {

        MapStateDescriptor<String, UserProfile> profileDescriptor = new MapStateDescriptor<>(
                "user-profiles",
                TypeInformation.of(String.class),
                TypeInformation.of(UserProfile.class)
        );

        MapStateDescriptor<String, LocationData> locationDescriptor = new MapStateDescriptor<>(
                "locations",
                TypeInformation.of(String.class),
                TypeInformation.of(LocationData.class)
        );

        BroadcastStream<UserProfile> broadcastProfiles = profiles.broadcast(profileDescriptor);
        BroadcastStream<LocationData> broadcastLocations = locations.broadcast(locationDescriptor);

        return activities
                .keyBy(UserActivity::userId)
                .connect(broadcastProfiles)
                .process(new ProfileEnrichmentFunction(profileDescriptor))
                .keyBy(ProfileEnrichedActivity::userId)
                .connect(broadcastLocations)
                .process(new LocationEnrichmentFunction(locationDescriptor))
                .doOnNext(enriched -> log.info("Fully enriched activity: {}", enriched.activityId()));
    }

    /**
     * Broadcast state cleanup: Remove stale entries from broadcast state.
     * Use case: Prevent memory bloat with time-based or count-based eviction.
     */
    public static DataStream<CleanedResult> broadcastStateWithCleanup(
            DataStream<DataEvent> events,
            DataStream<ReferenceEntry> references) {

        MapStateDescriptor<String, TimestampedReference> refDescriptor = new MapStateDescriptor<>(
                "references-with-ttl",
                TypeInformation.of(String.class),
                TypeInformation.of(TimestampedReference.class)
        );

        BroadcastStream<ReferenceEntry> broadcastRefs = references.broadcast(refDescriptor);

        return events
                .keyBy(DataEvent::key)
                .connect(broadcastRefs)
                .process(new CleanupAwareBroadcastFunction(refDescriptor))
                .doOnNext(result -> log.debug("Processed with cleaned state: {}", result.id()));
    }

    // Broadcast Process Functions

    /**
     * Fraud detection with dynamic rules using broadcast state.
     */
    private static class FraudDetectionFunction extends
            KeyedBroadcastProcessFunction<String, Transaction, FraudRule, EnrichedTransaction> {

        private final MapStateDescriptor<String, FraudRule> ruleDescriptor;
        private transient ValueState<TransactionHistory> historyState;

        FraudDetectionFunction(MapStateDescriptor<String, FraudRule> ruleDescriptor) {
            this.ruleDescriptor = ruleDescriptor;
        }

        @Override
        public void open(Configuration parameters) {
            ValueStateDescriptor<TransactionHistory> historyDescriptor = new ValueStateDescriptor<>(
                    "transaction-history",
                    TypeInformation.of(TransactionHistory.class)
            );
            historyState = getRuntimeContext().getState(historyDescriptor);
        }

        @Override
        public void processElement(
                Transaction transaction,
                ReadOnlyContext ctx,
                Collector<EnrichedTransaction> out) throws Exception {

            // Query broadcast state
            ReadOnlyBroadcastState<String, FraudRule> rules = ctx.getBroadcastState(ruleDescriptor);

            List<String> triggeredRules = new ArrayList<>();
            TransactionHistory history = historyState.value();
            if (history == null) {
                history = new TransactionHistory(transaction.userId(), new ArrayList<>());
            }

            // Apply all active rules
            for (Map.Entry<String, FraudRule> entry : rules.immutableEntries()) {
                FraudRule rule = entry.getValue();
                if (rule.matches(transaction, history)) {
                    triggeredRules.add(rule.ruleId());
                    log.warn("Fraud rule triggered: {} for transaction {}",
                            rule.ruleId(), transaction.transactionId());
                }
            }

            // Update history
            history.transactions().add(transaction);
            if (history.transactions().size() > 100) {
                history.transactions().remove(0);
            }
            historyState.update(history);

            // Emit enriched result
            out.collect(new EnrichedTransaction(
                    transaction.transactionId(),
                    transaction.userId(),
                    transaction.amount(),
                    transaction.timestamp(),
                    triggeredRules,
                    triggeredRules.isEmpty() ? "APPROVED" : "FLAGGED"
            ));
        }

        @Override
        public void processBroadcastElement(
                FraudRule rule,
                Context ctx,
                Collector<EnrichedTransaction> out) throws Exception {

            // Update broadcast state
            BroadcastState<String, FraudRule> state = ctx.getBroadcastState(ruleDescriptor);

            if (rule.isActive()) {
                state.put(rule.ruleId(), rule);
                log.info("Fraud rule added/updated: {} - {}", rule.ruleId(), rule.description());
            } else {
                state.remove(rule.ruleId());
                log.info("Fraud rule removed: {}", rule.ruleId());
            }
        }
    }

    /**
     * Configurable event processor with dynamic configuration.
     */
    private static class ConfigurableEventProcessor extends
            KeyedBroadcastProcessFunction<String, Event, Configuration, ProcessedEvent> {

        private final MapStateDescriptor<String, Configuration> configDescriptor;

        ConfigurableEventProcessor(MapStateDescriptor<String, Configuration> configDescriptor) {
            this.configDescriptor = configDescriptor;
        }

        @Override
        public void processElement(
                Event event,
                ReadOnlyContext ctx,
                Collector<ProcessedEvent> out) throws Exception {

            ReadOnlyBroadcastState<String, Configuration> configState =
                    ctx.getBroadcastState(configDescriptor);

            Configuration config = configState.get("current");
            if (config == null) {
                log.warn("No configuration available, using defaults");
                config = Configuration.defaultConfig();
            }

            // Process event with current configuration
            boolean shouldProcess = config.isEnabled(event.eventType());
            if (shouldProcess) {
                double threshold = config.getThreshold(event.eventType());
                String result = processWithConfig(event, config, threshold);

                out.collect(new ProcessedEvent(
                        event.eventId(),
                        event.eventType(),
                        result,
                        config.version(),
                        System.currentTimeMillis()
                ));
            }
        }

        @Override
        public void processBroadcastElement(
                Configuration config,
                Context ctx,
                Collector<ProcessedEvent> out) throws Exception {

            BroadcastState<String, Configuration> state = ctx.getBroadcastState(configDescriptor);
            state.put("current", config);
            log.info("Configuration updated to version: {}", config.version());
        }

        private String processWithConfig(Event event, Configuration config, double threshold) {
            // Apply configuration-specific processing logic
            return "processed";
        }
    }

    /**
     * Product enrichment using broadcast reference data.
     */
    private static class ProductEnrichmentFunction extends
            KeyedBroadcastProcessFunction<String, Order, ProductInfo, EnrichedOrder> {

        private final MapStateDescriptor<String, ProductInfo> productDescriptor;

        ProductEnrichmentFunction(MapStateDescriptor<String, ProductInfo> productDescriptor) {
            this.productDescriptor = productDescriptor;
        }

        @Override
        public void processElement(
                Order order,
                ReadOnlyContext ctx,
                Collector<EnrichedOrder> out) throws Exception {

            ReadOnlyBroadcastState<String, ProductInfo> products =
                    ctx.getBroadcastState(productDescriptor);

            ProductInfo product = products.get(order.productId());
            if (product == null) {
                log.warn("Product not found: {}", order.productId());
                // Could emit to side output for unmatched orders
                return;
            }

            out.collect(new EnrichedOrder(
                    order.orderId(),
                    order.customerId(),
                    order.productId(),
                    product.productName(),
                    product.category(),
                    product.price(),
                    order.quantity(),
                    order.timestamp()
            ));
        }

        @Override
        public void processBroadcastElement(
                ProductInfo product,
                Context ctx,
                Collector<EnrichedOrder> out) throws Exception {

            BroadcastState<String, ProductInfo> state = ctx.getBroadcastState(productDescriptor);
            state.put(product.productId(), product);
            log.debug("Product catalog updated: {}", product.productId());
        }
    }

    /**
     * Pattern matching with dynamic patterns.
     */
    private static class PatternMatchingFunction extends
            KeyedBroadcastProcessFunction<String, Event, Pattern, MatchedEvent> {

        private final MapStateDescriptor<String, Pattern> patternDescriptor;
        private transient ListState<Event> eventBuffer;

        PatternMatchingFunction(MapStateDescriptor<String, Pattern> patternDescriptor) {
            this.patternDescriptor = patternDescriptor;
        }

        @Override
        public void open(Configuration parameters) {
            ListStateDescriptor<Event> bufferDescriptor = new ListStateDescriptor<>(
                    "event-buffer",
                    TypeInformation.of(Event.class)
            );
            eventBuffer = getRuntimeContext().getListState(bufferDescriptor);
        }

        @Override
        public void processElement(
                Event event,
                ReadOnlyContext ctx,
                Collector<MatchedEvent> out) throws Exception {

            // Add to buffer
            eventBuffer.add(event);

            // Check against all patterns
            ReadOnlyBroadcastState<String, Pattern> patterns = ctx.getBroadcastState(patternDescriptor);
            List<Event> recentEvents = new ArrayList<>();
            eventBuffer.get().forEach(recentEvents::add);

            for (Map.Entry<String, Pattern> entry : patterns.immutableEntries()) {
                Pattern pattern = entry.getValue();
                if (pattern.matches(recentEvents)) {
                    out.collect(new MatchedEvent(
                            pattern.patternId(),
                            event.eventId(),
                            event.userId(),
                            pattern.severity(),
                            System.currentTimeMillis()
                    ));
                }
            }

            // Keep only last N events
            if (recentEvents.size() > 50) {
                eventBuffer.clear();
                recentEvents.stream().skip(recentEvents.size() - 50)
                        .forEach(e -> {
                            try {
                                eventBuffer.add(e);
                            } catch (Exception ex) {
                                log.error("Failed to update buffer", ex);
                            }
                        });
            }
        }

        @Override
        public void processBroadcastElement(
                Pattern pattern,
                Context ctx,
                Collector<MatchedEvent> out) throws Exception {

            BroadcastState<String, Pattern> state = ctx.getBroadcastState(patternDescriptor);
            if (pattern.isActive()) {
                state.put(pattern.patternId(), pattern);
                log.info("Pattern added: {}", pattern.patternId());
            } else {
                state.remove(pattern.patternId());
                log.info("Pattern removed: {}", pattern.patternId());
            }
        }
    }

    /**
     * Non-keyed global filter using broadcast state.
     */
    private static class GlobalFilterFunction extends
            BroadcastProcessFunction<RawData, FilterConfig, FilteredData> {

        private final MapStateDescriptor<String, FilterConfig> filterDescriptor;

        GlobalFilterFunction(MapStateDescriptor<String, FilterConfig> filterDescriptor) {
            this.filterDescriptor = filterDescriptor;
        }

        @Override
        public void processElement(
                RawData data,
                ReadOnlyContext ctx,
                Collector<FilteredData> out) throws Exception {

            ReadOnlyBroadcastState<String, FilterConfig> filters =
                    ctx.getBroadcastState(filterDescriptor);

            FilterConfig config = filters.get("current");
            if (config == null || config.shouldInclude(data)) {
                out.collect(new FilteredData(data.id(), data.value(), data.timestamp()));
            } else {
                log.debug("Data filtered out: {}", data.id());
            }
        }

        @Override
        public void processBroadcastElement(
                FilterConfig config,
                Context ctx,
                Collector<FilteredData> out) throws Exception {

            BroadcastState<String, FilterConfig> state = ctx.getBroadcastState(filterDescriptor);
            state.put("current", config);
            log.info("Filter configuration updated");
        }
    }

    /**
     * Version-aware processor tracking configuration versions.
     */
    private static class VersionAwareProcessor extends
            KeyedBroadcastProcessFunction<String, Message, VersionedConfig, VersionedResult> {

        private final MapStateDescriptor<String, VersionedConfig> configDescriptor;

        VersionAwareProcessor(MapStateDescriptor<String, VersionedConfig> configDescriptor) {
            this.configDescriptor = configDescriptor;
        }

        @Override
        public void processElement(
                Message message,
                ReadOnlyContext ctx,
                Collector<VersionedResult> out) throws Exception {

            ReadOnlyBroadcastState<String, VersionedConfig> configState =
                    ctx.getBroadcastState(configDescriptor);

            VersionedConfig config = configState.get("current");
            int version = config != null ? config.version() : 0;

            String processed = config != null
                    ? message.content() + "-v" + version
                    : message.content() + "-default";

            out.collect(new VersionedResult(
                    message.id(),
                    processed,
                    version,
                    System.currentTimeMillis()
            ));
        }

        @Override
        public void processBroadcastElement(
                VersionedConfig config,
                Context ctx,
                Collector<VersionedResult> out) throws Exception {

            BroadcastState<String, VersionedConfig> state = ctx.getBroadcastState(configDescriptor);
            VersionedConfig oldConfig = state.get("current");

            if (oldConfig == null || config.version() > oldConfig.version()) {
                state.put("current", config);
                log.info("Configuration upgraded: v{} -> v{}",
                        oldConfig != null ? oldConfig.version() : 0, config.version());
            } else {
                log.warn("Ignored outdated configuration: v{} (current: v{})",
                        config.version(), oldConfig.version());
            }
        }
    }

    /**
     * Profile enrichment (first stage of multi-broadcast).
     */
    private static class ProfileEnrichmentFunction extends
            KeyedBroadcastProcessFunction<String, UserActivity, UserProfile, ProfileEnrichedActivity> {

        private final MapStateDescriptor<String, UserProfile> profileDescriptor;

        ProfileEnrichmentFunction(MapStateDescriptor<String, UserProfile> profileDescriptor) {
            this.profileDescriptor = profileDescriptor;
        }

        @Override
        public void processElement(
                UserActivity activity,
                ReadOnlyContext ctx,
                Collector<ProfileEnrichedActivity> out) throws Exception {

            ReadOnlyBroadcastState<String, UserProfile> profiles =
                    ctx.getBroadcastState(profileDescriptor);

            UserProfile profile = profiles.get(activity.userId());
            out.collect(new ProfileEnrichedActivity(
                    activity.activityId(),
                    activity.userId(),
                    profile != null ? profile.name() : "Unknown",
                    profile != null ? profile.tier() : "Basic",
                    activity.locationId(),
                    activity.timestamp()
            ));
        }

        @Override
        public void processBroadcastElement(
                UserProfile profile,
                Context ctx,
                Collector<ProfileEnrichedActivity> out) throws Exception {

            BroadcastState<String, UserProfile> state = ctx.getBroadcastState(profileDescriptor);
            state.put(profile.userId(), profile);
        }
    }

    /**
     * Location enrichment (second stage of multi-broadcast).
     */
    private static class LocationEnrichmentFunction extends
            KeyedBroadcastProcessFunction<String, ProfileEnrichedActivity, LocationData, ComplexEnriched> {

        private final MapStateDescriptor<String, LocationData> locationDescriptor;

        LocationEnrichmentFunction(MapStateDescriptor<String, LocationData> locationDescriptor) {
            this.locationDescriptor = locationDescriptor;
        }

        @Override
        public void processElement(
                ProfileEnrichedActivity activity,
                ReadOnlyContext ctx,
                Collector<ComplexEnriched> out) throws Exception {

            ReadOnlyBroadcastState<String, LocationData> locations =
                    ctx.getBroadcastState(locationDescriptor);

            LocationData location = locations.get(activity.locationId());
            out.collect(new ComplexEnriched(
                    activity.activityId(),
                    activity.userId(),
                    activity.userName(),
                    activity.userTier(),
                    location != null ? location.city() : "Unknown",
                    location != null ? location.country() : "Unknown",
                    activity.timestamp()
            ));
        }

        @Override
        public void processBroadcastElement(
                LocationData location,
                Context ctx,
                Collector<ComplexEnriched> out) throws Exception {

            BroadcastState<String, LocationData> state = ctx.getBroadcastState(locationDescriptor);
            state.put(location.locationId(), location);
        }
    }

    /**
     * Broadcast function with automatic state cleanup.
     */
    private static class CleanupAwareBroadcastFunction extends
            KeyedBroadcastProcessFunction<String, DataEvent, ReferenceEntry, CleanedResult> {

        private final MapStateDescriptor<String, TimestampedReference> refDescriptor;
        private static final long TTL_MS = 3600000; // 1 hour

        CleanupAwareBroadcastFunction(MapStateDescriptor<String, TimestampedReference> refDescriptor) {
            this.refDescriptor = refDescriptor;
        }

        @Override
        public void processElement(
                DataEvent event,
                ReadOnlyContext ctx,
                Collector<CleanedResult> out) throws Exception {

            ReadOnlyBroadcastState<String, TimestampedReference> refs =
                    ctx.getBroadcastState(refDescriptor);

            TimestampedReference ref = refs.get(event.refKey());
            if (ref != null && !ref.isExpired(TTL_MS)) {
                out.collect(new CleanedResult(event.id(), ref.value(), event.timestamp()));
            }
        }

        @Override
        public void processBroadcastElement(
                ReferenceEntry entry,
                Context ctx,
                Collector<CleanedResult> out) throws Exception {

            BroadcastState<String, TimestampedReference> state = ctx.getBroadcastState(refDescriptor);

            // Add new entry
            state.put(entry.key(), new TimestampedReference(
                    entry.value(),
                    System.currentTimeMillis()
            ));

            // Cleanup expired entries
            long now = System.currentTimeMillis();
            List<String> toRemove = new ArrayList<>();
            for (Map.Entry<String, TimestampedReference> e : state.immutableEntries()) {
                if (e.getValue().isExpired(TTL_MS)) {
                    toRemove.add(e.getKey());
                }
            }
            for (String key : toRemove) {
                state.remove(key);
                log.debug("Removed expired reference: {}", key);
            }
        }
    }

    // Data Models
    public record Transaction(String transactionId, String userId, double amount, long timestamp) {}
    public record EnrichedTransaction(String transactionId, String userId, double amount,
                                     long timestamp, List<String> triggeredRules, String status) {}
    public record FraudRule(String ruleId, String description, String condition, boolean isActive) {
        public boolean matches(Transaction tx, TransactionHistory history) {
            // Rule matching logic
            return false;
        }
    }
    public record TransactionHistory(String userId, List<Transaction> transactions) {}

    public record Event(String eventId, String userId, String eventType, long timestamp) {}
    public record ProcessedEvent(String eventId, String eventType, String result,
                                 int configVersion, long processedAt) {}
    public record Configuration(int version, Map<String, Boolean> features,
                                Map<String, Double> thresholds) {
        public boolean isEnabled(String feature) {
            return features.getOrDefault(feature, false);
        }
        public double getThreshold(String key) {
            return thresholds.getOrDefault(key, 0.0);
        }
        public static Configuration defaultConfig() {
            return new Configuration(0, Map.of(), Map.of());
        }
    }

    public record Order(String orderId, String customerId, String productId, int quantity, long timestamp) {}
    public record ProductInfo(String productId, String productName, String category, double price) {}
    public record EnrichedOrder(String orderId, String customerId, String productId,
                               String productName, String category, double price,
                               int quantity, long timestamp) {}

    public record Pattern(String patternId, String description, List<String> conditions,
                         String severity, boolean isActive) {
        public boolean matches(List<Event> events) {
            // Pattern matching logic
            return false;
        }
    }
    public record MatchedEvent(String patternId, String eventId, String userId,
                              String severity, long matchedAt) {}

    public record RawData(String id, double value, long timestamp) {}
    public record FilterConfig(Set<String> allowedIds, double minValue, double maxValue) {
        public boolean shouldInclude(RawData data) {
            return data.value() >= minValue && data.value() <= maxValue;
        }
    }
    public record FilteredData(String id, double value, long timestamp) {}

    public record Message(String id, String partition, String content, long timestamp) {}
    public record VersionedConfig(int version, String settings) {}
    public record VersionedResult(String id, String processed, int version, long timestamp) {}

    public record UserActivity(String activityId, String userId, String locationId, long timestamp) {}
    public record UserProfile(String userId, String name, String tier) {}
    public record LocationData(String locationId, String city, String country) {}
    public record ProfileEnrichedActivity(String activityId, String userId, String userName,
                                         String userTier, String locationId, long timestamp) {}
    public record ComplexEnriched(String activityId, String userId, String userName,
                                 String userTier, String city, String country, long timestamp) {}

    public record DataEvent(String id, String key, String refKey, long timestamp) {}
    public record ReferenceEntry(String key, String value) {}
    public record TimestampedReference(String value, long timestamp) {
        public boolean isExpired(long ttlMs) {
            return System.currentTimeMillis() - timestamp > ttlMs;
        }
    }
    public record CleanedResult(String id, String enrichedValue, long timestamp) {}
}
```

## Configuration Example

```yaml
# Flink job configuration for broadcast state
execution:
  parallelism: 4
  checkpointing:
    interval: 60s
    mode: exactly-once

state:
  backend: rocksdb
  checkpoints-dir: s3://my-bucket/checkpoints
  backend.rocksdb.memory.managed: true

# Broadcast stream configuration
broadcast:
  buffer-timeout: 100ms
  # Broadcast state is replicated to all tasks automatically
```

## Best Practices

1. **State Size Management**: Keep broadcast state reasonably sized (under 10MB per descriptor). Large broadcast states increase checkpoint times.

2. **Update Frequency**: Avoid excessive broadcast updates. Batch configuration changes or use rate limiting on broadcast streams.

3. **Version Control**: Always include version numbers in broadcast configurations to track which version processed each event.

4. **Immutable Entries**: Use immutableEntries() when reading broadcast state to avoid ConcurrentModificationException.

5. **Cleanup Strategy**: Implement TTL or periodic cleanup for broadcast state entries to prevent unbounded growth.

6. **Error Handling**: Gracefully handle missing broadcast state (null checks) during startup or after broadcast stream restarts.

7. **Type Safety**: Use strongly-typed MapStateDescriptor with proper serializers to avoid deserialization errors.

8. **Monitoring**: Track broadcast state size and update frequency with custom metrics.

9. **Testing**: Test broadcast state with checkpoints to ensure state is properly saved and restored.

10. **Multiple Broadcasts**: When using multiple broadcast states, consider chaining instead of nested connects for better readability.

## Related Documentation

- [Flink Broadcast State](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/fault-tolerance/broadcast_state/)
- [State Descriptors](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/fault-tolerance/state/)
- [Connected Streams](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/operators/overview/#connected-streams)
- [Broadcast Process Function](https://nightlies.apache.org/flink/flink-docs-release-2.0/api/java/org/apache/flink/streaming/api/functions/co/BroadcastProcessFunction.html)
