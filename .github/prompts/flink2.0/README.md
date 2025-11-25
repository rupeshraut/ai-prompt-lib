# Apache Flink 2.0 General-Purpose Prompt Library

This directory contains general-purpose prompts for building Apache Flink 2.0 streaming applications. Unlike the flink-fednow library which focuses on FedNow payment processing, these prompts cover reusable Flink patterns and building blocks applicable to any streaming use case.

## Available Prompts

### Core Window & Event-Time Patterns

| Template | Description |
|----------|-------------|
| `create-window-operations.md` | Tumbling, sliding, session, and count windows with aggregations |
| `create-event-time-processing.md` | Watermarks, late data handling, event-time semantics |

### Stateful Processing

| Template | Description |
|----------|-------------|
| `create-stateful-processing.md` | Keyed state (ValueState, ListState, MapState), broadcast state, TTL |

### Custom Connectors

| Template | Description |
|----------|-------------|
| `create-custom-source.md` | Build custom source connectors for non-standard data sources |
| `create-custom-sink.md` | Build custom sink connectors for external systems |
| `create-database-sink.md` | JDBC sink with idempotent writes and transaction handling |

### Advanced Patterns

| Template | Description |
|----------|-------------|
| `create-broadcast-state.md` | Share configuration and reference data across all tasks |
| `create-queryable-state.md` | Expose Flink state via REST API for external queries |

### Operational Patterns

| Template | Description |
|----------|-------------|
| `create-monitoring-metrics.md` | Custom metrics, dashboards, alerting patterns |
| `create-flink-testing.md` | Unit and integration testing strategies |

## Quick Start

### 1. Window Operations

```
@workspace Use .github/prompts/flink2.0/create-window-operations.md

Create tumbling windows for 1-hour aggregations with AggregateFunction
```

### 2. Event-Time Processing

```
@workspace Use .github/prompts/flink2.0/create-event-time-processing.md

Create watermark strategy with 5-second bounded out-of-orderness and side outputs for late data
```

### 3. Stateful Processing

```
@workspace Use .github/prompts/flink2.0/create-stateful-processing.md

Create keyed state for session tracking with 30-minute TTL
```

## Flink Streaming Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                      Data Sources                            │
│      (Kafka, Custom, API, Database, Files)                   │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                   Event-Time Processing                      │
│   (Watermarks, Timestamp Assignment, Allowed Lateness)      │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                  Window Operations                           │
│   (Tumbling, Sliding, Session Windows with Aggregations)   │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                Stateful Processing                           │
│   (Keyed State, Broadcast State, Custom Logic)               │
└─────────────────────────────┬────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────┐
│                    Data Sinks                                │
│  (Kafka, Database, HDFS, Custom, Queryable State)            │
└──────────────────────────────────────────────────────────────┘
```

## Pattern Categories

### 1. Window Aggregations
- **Tumbling**: Fixed, non-overlapping intervals
- **Sliding**: Overlapping windows with configurable slide
- **Session**: Gap-based windows for clustering
- **Count**: Fixed number of elements

### 2. Event-Time Semantics
- **Monotonous Watermarks**: For ordered data
- **Bounded Out-of-Orderness**: For realistic networks (5-60 seconds)
- **Allowed Lateness**: Accept events after window close
- **Side Output Late Data**: Route late arrivals separately

### 3. State Management
- **Value State**: Single value per key
- **List State**: Collection per key
- **Map State**: Key-value pairs per key
- **Reducing State**: Aggregate values efficiently
- **Broadcast State**: Share data across all tasks

### 4. Custom Connectors
- **Custom Source**: Read from non-standard sources
- **Custom Sink**: Write to non-standard destinations
- **Database Sink**: JDBC with idempotency guarantees

### 5. Advanced Patterns
- **Broadcast Variables**: Configuration distribution
- **Queryable State**: External state queries via REST
- **Monitoring**: Custom metrics and alerting

## Best Practices

1. **Choose Correct Window Type**
   - Use tumbling for periodic reports
   - Use sliding for moving averages
   - Use session for user behavior analysis
   - Use count for batch operations

2. **Event-Time Processing**
   - Always use event-time for accuracy
   - Configure appropriate watermark lag
   - Handle late arrivals with allowed lateness
   - Monitor watermark lag metrics

3. **State Management**
   - Set TTL to prevent unbounded state growth
   - Use value state for simple cases
   - Use map state for complex key-value relationships
   - Configure appropriate state backend (RocksDB for large state)

4. **Connector Development**
   - Implement checkpointing for recovery
   - Handle backpressure correctly
   - Provide clear error handling
   - Support scaling (splits in sources)

5. **Monitoring & Debugging**
   - Track watermark lag
   - Monitor state size
   - Set up alerts for processing delays
   - Log key metrics at regular intervals

## Technology Stack

- **Framework**: Apache Flink 2.0
- **Language**: Java 17 LTS
- **State Backend**: RocksDB (for production), In-Memory (for testing)
- **Connectors**: Kafka, JDBC, HDFS, Custom
- **Serialization**: Kryo (default), Java serialization
- **Testing**: Flink test harness, MiniCluster
- **Monitoring**: Flink metrics, custom reporters

## Flink 2.0 API Notes

- **DataStream API**: Primary API for all use cases
- **Source/Sink API**: Modern FLIP-27 connectors
- **KeyedStream**: Key-based operations with state
- **WindowedStream**: Windowed aggregations
- **Watermark Strategy**: Modern watermark assignment

## Common Workflows

### Real-Time Aggregation
```
Source → Event-Time → Window → State → Sink
        Timestamps    Tumbling  Reduce  Kafka
```

### Session Analysis
```
Source → Parse → Session Window → Process → Queryable State → REST API
        Events   Group by user    Enrich    External queries
```

### Event Replay
```
Kafka Source → Event-Time (use savepoint) → Windowing → Deduplicated Sink
             (restore watermarks from state)
```

## Comparison: flink-fednow vs flink2.0

| Aspect | flink-fednow | flink2.0 |
|--------|-------------|---------|
| **Focus** | FedNow payment processing | General Flink patterns |
| **Use** | Reference architecture | Reusable building blocks |
| **Scope** | Two-job orchestration | Individual components |
| **Complexity** | Production-grade FedNow | Generic Flink concepts |
| **Connectors** | Kafka, JMS | Kafka, JDBC, Custom |
| **Best For** | Payment domain | Any streaming application |

Use **flink-fednow** for understanding FedNow-specific flows.
Use **flink2.0** for general Flink pattern implementations.

## Related Documentation

- [Apache Flink 2.0 Documentation](https://nightlies.apache.org/flink/flink-docs-release-2.0/)
- [Datastream API Guide](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/dev/datastream/overview/)
- [Flink Architecture](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/concepts/architecture/)
- [State Backends](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/ops/state/state_backends/)
- [Connectors](https://nightlies.apache.org/flink/flink-docs-release-2.0/docs/connectors/datastream/overview/)
