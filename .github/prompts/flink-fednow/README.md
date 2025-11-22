# Apache Flink 2.0 FedNow Payment Processing Prompts

This directory contains custom prompts for generating Apache Flink 2.0 streaming applications for FedNow payment processing with ISO 20022 message standards.

## Two-Job Architecture (Production Recommended)

The pipeline is split into two Flink jobs plus a JMS-to-Kafka bridge for operational flexibility:

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Job 1: Request Flow                                                            │
│  Kafka → Validate → RCS → Send JMS → Kafka Pending                             │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼ FircoSoft
┌─────────────────────────────────────────────────────────────────────────────────┐
│  JMS Bridge: JMS Response → Kafka Responses                                     │
└─────────────────────────────────────────────────────────────────────────────────┘
                                          │
                                          ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Job 2: Response Flow                                                           │
│  Kafka Pending + Kafka Responses → Correlate → Post → pacs.002 → Kafka         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

| Component | Template | Description |
|-----------|----------|-------------|
| **Job 1** | `create-sanctions-request-job.md` | Validate, RCS check, send JMS, write pending |
| **Bridge** | `create-sanctions-response-job.md` | JMS-to-Kafka bridge (included) |
| **Job 2** | `create-sanctions-response-job.md` | Correlate, post accounts, generate pacs.002 |

## Available Prompts

### Core Jobs

| Template | Description |
|----------|-------------|
| `create-sanctions-request-job.md` | Job 1: Request flow with JMS sender and Kafka pending sink |
| `create-sanctions-response-job.md` | Job 2: Response flow with correlation and account posting |
| `create-flink-job.md` | Single-job template (for simpler deployments) |

### Functions & Components

| Template | Description |
|----------|-------------|
| `create-async-function.md` | RichAsyncFunction for non-blocking HTTP service calls |
| `create-sanctions-processor.md` | KeyedProcessFunction for JMS-based sanctions screening |
| `create-iso20022-parser.md` | ISO 20022 XML parsers (pacs.008) and serializers (pacs.002) |
| `create-payment-model.md` | Payment domain model with status enums and Flink serialization |
| `create-flink-test.md` | Flink test harness patterns, unit tests, and integration tests |

## Quick Start (Two-Job Architecture)

### 1. Create Payment Model

```
@workspace Use .github/prompts/flink-fednow/create-payment-model.md

Create Payment domain model with FedNow fields and processing status enums
```

### 2. Create ISO 20022 Parsers

```
@workspace Use .github/prompts/flink-fednow/create-iso20022-parser.md

Create parsers for pacs.008 and pacs.002 messages with validation
```

### 3. Create Async Functions

```
@workspace Use .github/prompts/flink-fednow/create-async-function.md

Create AccountValidationAsyncFunction with retry logic and timeout handling
```

### 4. Create Job 1 (Sanctions Request)

```
@workspace Use .github/prompts/flink-fednow/create-sanctions-request-job.md

Create SanctionsRequestJob with JMS sender and Kafka pending sink
```

### 5. Create JMS-to-Kafka Bridge

```
@workspace Use .github/prompts/flink-fednow/create-sanctions-response-job.md

Create JmsToKafkaBridge that consumes JMS responses and produces to Kafka
```

### 6. Create Job 2 (Sanctions Response)

```
@workspace Use .github/prompts/flink-fednow/create-sanctions-response-job.md

Create SanctionsResponseJob with correlation and account posting
```

### 7. Create Tests

```
@workspace Use .github/prompts/flink-fednow/create-flink-test.md

Create unit tests with Flink test harness and integration tests
```

## Why Two Jobs?

| Benefit | Description |
|---------|-------------|
| **Independent Scaling** | Scale JMS consumer separately from request processing |
| **Fault Isolation** | FircoSoft issues don't block payment ingestion |
| **Operational Flexibility** | Deploy/restart jobs independently |
| **Kafka as Buffer** | Durable storage for pending payments during outages |
| **Checkpoint Efficiency** | Job 1 checkpoints quickly (no JMS wait) |

## Related Documentation

- [META_PROMPT_FLINK_FEDNOW.md](../../../META_PROMPT_FLINK_FEDNOW.md) - Complete project context
- [PRACTICAL_EXAMPLE_FEDNOW_PAYMENT.md](../../../PRACTICAL_EXAMPLE_FEDNOW_PAYMENT.md) - 11-step prompt chain example

## Technologies

- Apache Flink 2.0
- Java 17 LTS
- Apache Kafka (source/sink)
- IBM MQ / ActiveMQ (JMS)
- ISO 20022 (pacs.008, pacs.002)
- RocksDB state backend
- Jackson XML
