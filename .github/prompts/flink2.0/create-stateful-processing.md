# Create Stateful Processing

Create stateful stream processing patterns using keyed state, broadcast state, and state backends for maintaining and querying application state.

## Requirements

Your stateful processing implementation should include:

- **Keyed State**: ValueState, ListState, MapState, ReducingState usage
- **State Descriptors**: Proper configuration of state with TTL
- **Broadcast State**: Share configuration/reference data with all tasks  
- **State Backends**: Configure RocksDB, in-memory, or external state storage
- **State Serialization**: Custom serializers for complex state objects
- **State Cleanup**: TTL and manual state eviction patterns
- **State Snapshots**: Understand state checkpointing behavior
- **State Versioning**: Handle state schema evolution
- **Scaling**: Key redistribution during rescaling
- **Documentation**: Clear explanation of state lifecycle

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Apache Flink 2.0
- **API**: Modern DataStream API with state interfaces
- **Annotations**: Use for configuration and documentation

## Template Example

```java
import org.apache.flink.api.common.state.*;
import org.apache.flink.api.common.typeinfo.TypeHint;
import org.apache.flink.api.common.typeinfo.TypeInformation;
import org.apache.flink.streaming.api.datastream.BroadcastStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.functions.KeyedProcessFunction;
import org.apache.flink.streaming.api.functions.co.KeyedBroadcastProcessFunction;
import org.apache.flink.util.Collector;
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.time.Time;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

/**
 * Stateful processing patterns in Apache Flink 2.0.
 */
@Slf4j
public class StatefulProcessing {

    /**
     * Keyed Value State: Maintain single value per key.
     */
    public static DataStream<UserStatus> keyedValueState(DataStream<UserEvent> events) {
        return events
                .keyBy(UserEvent::userId)
                .process(new ValueStateFunction())
                .doOnNext(status -> log.debug("User status: {}", status.userId()));
    }

    /**
     * Keyed List State: Maintain list of values per key.
     */
    public static DataStream<OrderHistory> keyedListState(DataStream<OrderEvent> orders) {
        return orders
                .keyBy(OrderEvent::customerId)
                .process(new ListStateFunction())
                .doOnNext(history -> log.info("Customer {} has {} orders",
                        history.customerId(), history.orderCount()));
    }

    /**
     * Keyed Map State: Key-value mappings per stream key.
     */
    public static DataStream<AccountBalance> keyedMapState(DataStream<Transaction> transactions) {
        return transactions
                .keyBy(Transaction::accountId)
                .process(new MapStateFunction())
                .doOnNext(balance -> log.info("Account {} balance: ${}", 
                        balance.accountId(), balance.totalBalance()));
    }

    /**
     * Reducing State: Efficiently aggregate values.
     */
    public static DataStream<SalesSummary> reducingState(DataStream<Sale> sales) {
        return sales
                .keyBy(Sale::storeId)
                .process(new ReducingStateFunction())
                .doOnNext(summary -> log.info("Store {} total sales: ${}", 
                        summary.storeId(), summary.totalSales()));
    }

    /**
     * Broadcast State: Share configuration with all tasks.
     */
    public static DataStream<EnrichedEvent> broadcastConfiguration(
            DataStream<Event> events,
            DataStream<Configuration> configs) {

        MapStateDescriptor<String, Configuration> configDescriptor =
                new MapStateDescriptor<>("configs", String.class, Configuration.class);

        BroadcastStream<Configuration> broadcastConfigs = configs
                .broadcast(configDescriptor);

        return events
                .connect(broadcastConfigs)
                .process(new BroadcastEnrichmentFunction(configDescriptor))
                .doOnNext(enriched -> log.debug("Enriched event: {}", enriched.eventId()));
    }

    /**
     * State TTL: Automatically expire old state.
     */
    public static DataStream<SessionData> stateTTL(DataStream<UserEvent> events) {
        return events
                .keyBy(UserEvent::userId)
                .process(new TTLStateFunction())
                .doOnNext(session -> log.info("Session {} has {} events", 
                        session.sessionId(), session.eventCount()));
    }

    // Helper classes
    public record UserEvent(String userId, long timestamp, String eventType) {}
    public record OrderEvent(String customerId, long timestamp, String orderId, double amount) {}
    public record Transaction(String accountId, double amount, String type) {}
    public record Sale(String storeId, double amount, long timestamp) {}
    public record Event(String eventId, String type, String data) {}
    public record Configuration(String configId, Map<String, String> settings) {}
    public record EnrichedEvent(String eventId, String enrichedData) {}
    public record UserStatus(String userId, String status) {}
    public record OrderHistory(String customerId, int orderCount, List<String> orders) {}
    public record AccountBalance(String accountId, double totalBalance) {}
    public record SalesSummary(String storeId, double totalSales) {}
    public record SessionData(String sessionId, int eventCount) {}

    // Process Functions
    @Slf4j
    private static class ValueStateFunction extends KeyedProcessFunction<String, UserEvent, UserStatus> {
        private ValueState<String> lastStatusState;

        @Override
        public void open(Configuration parameters) throws Exception {
            ValueStateDescriptor<String> descriptor =
                    new ValueStateDescriptor<>("lastStatus", String.class);
            descriptor.enableTimeToLive(StateTtlConfig.newBuilder(Time.hours(1))
                    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
                    .build());
            lastStatusState = getRuntimeContext().getState(descriptor);
        }

        @Override
        public void processElement(UserEvent event, Context ctx, Collector<UserStatus> out) throws Exception {
            String lastStatus = lastStatusState.value();
            lastStatusState.update(event.eventType());
            out.collect(new UserStatus(event.userId(), event.eventType()));
        }
    }

    @Slf4j
    private static class ListStateFunction extends KeyedProcessFunction<String, OrderEvent, OrderHistory> {
        private ListState<String> ordersState;

        @Override
        public void open(Configuration parameters) throws Exception {
            ListStateDescriptor<String> descriptor =
                    new ListStateDescriptor<>("orders", String.class);
            ordersState = getRuntimeContext().getListState(descriptor);
        }

        @Override
        public void processElement(OrderEvent order, Context ctx, Collector<OrderHistory> out) throws Exception {
            ordersState.add(order.orderId());
            List<String> orders = new ArrayList<>();
            ordersState.get().forEach(orders::add);
            out.collect(new OrderHistory(order.customerId(), orders.size(), orders));
        }
    }

    @Slf4j
    private static class MapStateFunction extends KeyedProcessFunction<String, Transaction, AccountBalance> {
        private MapState<String, Double> balanceState;

        @Override
        public void open(Configuration parameters) throws Exception {
            MapStateDescriptor<String, Double> descriptor =
                    new MapStateDescriptor<>("balances", String.class, Double.class);
            balanceState = getRuntimeContext().getMapState(descriptor);
        }

        @Override
        public void processElement(Transaction transaction, Context ctx, Collector<AccountBalance> out) throws Exception {
            String currency = "USD";
            double current = balanceState.get(currency) != null ? balanceState.get(currency) : 0.0;
            double updated = current + transaction.amount();
            balanceState.put(currency, updated);
            out.collect(new AccountBalance(transaction.accountId(), updated));
        }
    }

    @Slf4j
    private static class ReducingStateFunction extends KeyedProcessFunction<String, Sale, SalesSummary> {
        private ReducingState<Double> totalSalesState;

        @Override
        public void open(Configuration parameters) throws Exception {
            ReducingStateDescriptor<Double> descriptor =
                    new ReducingStateDescriptor<>("totalSales", (a, b) -> a + b, Double.class);
            totalSalesState = getRuntimeContext().getReducingState(descriptor);
        }

        @Override
        public void processElement(Sale sale, Context ctx, Collector<SalesSummary> out) throws Exception {
            totalSalesState.add(sale.amount());
            out.collect(new SalesSummary(sale.storeId(), totalSalesState.get()));
        }
    }

    @Slf4j
    private static class BroadcastEnrichmentFunction extends KeyedBroadcastProcessFunction<String, Event, Configuration, EnrichedEvent> {
        private final MapStateDescriptor<String, Configuration> configDescriptor;

        BroadcastEnrichmentFunction(MapStateDescriptor<String, Configuration> descriptor) {
            this.configDescriptor = descriptor;
        }

        @Override
        public void processElement(Event event, ReadOnlyContext ctx, Collector<EnrichedEvent> out) throws Exception {
            ReadOnlyBroadcastState<String, Configuration> configs = ctx.getBroadcastState(configDescriptor);
            if (configs.contains("default")) {
                Configuration config = configs.get("default");
                out.collect(new EnrichedEvent(event.eventId(), event.data()));
            }
        }

        @Override
        public void processBroadcastElement(Configuration config, Context ctx, Collector<EnrichedEvent> out) throws Exception {
            BroadcastState<String, Configuration> configs = ctx.getBroadcastState(configDescriptor);
            configs.put("default", config);
            log.info("Configuration updated");
        }
    }

    @Slf4j
    private static class TTLStateFunction extends KeyedProcessFunction<String, UserEvent, SessionData> {
        private ValueState<Integer> sessionCountState;

        @Override
        public void open(Configuration parameters) throws Exception {
            ValueStateDescriptor<Integer> descriptor = new ValueStateDescriptor<>("sessionCount", Integer.class);
            descriptor.enableTimeToLive(StateTtlConfig.newBuilder(Time.minutes(30))
                    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
                    .setStateVisibility(StateTtlConfig.StateVisibility.ReturnExpiredIfNotCleanedUp)
                    .build());
            sessionCountState = getRuntimeContext().getState(descriptor);
        }

        @Override
        public void processElement(UserEvent event, Context ctx, Collector<SessionData> out) throws Exception {
            Integer count = sessionCountState.value() != null ? sessionCountState.value() : 0;
            sessionCountState.update(count + 1);
            out.collect(new SessionData(event.userId(), count + 1));
        }
    }
}
```

## Related Documentation

- [Keyed State](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/fault-tolerance/state/)
- [State Backends](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/ops/state/state_backends/)
