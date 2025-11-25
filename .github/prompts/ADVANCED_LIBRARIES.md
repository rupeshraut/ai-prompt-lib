# Advanced Production Prompts Library

Comprehensive collection of advanced production-use-case prompts for enterprise Java applications. These libraries extend the core Spring Boot, Project Reactor, and Apache Flink prompts with critical production requirements.

## Library Overview

### 1. Distributed Tracing Library
End-to-end request tracing across microservices using OpenTelemetry and Jaeger.

**Prompts:**
- `create-opentelemetry-setup.md` - Configure OpenTelemetry SDK with Jaeger exporter
- `create-context-propagation.md` - W3C Trace Context and Jaeger header propagation
- `create-trace-sampling.md` - Adaptive and tail-based trace sampling strategies

**Key Features:**
- Spring Boot, Project Reactor, Apache Flink integration
- Automatic and manual span creation
- Context propagation across service boundaries
- Sampling for cost optimization
- Compliance with W3C Trace Context standard

### 2. Security & Compliance Library
Implement security controls and regulatory compliance (GDPR, CCPA, HIPAA, PCI DSS).

**Prompts:**
- `create-field-encryption.md` - AES-256 field encryption with key rotation
- `create-audit-logging.md` - Comprehensive audit trail for compliance
- `create-compliance-frameworks.md` - GDPR, CCPA, HIPAA, PCI DSS support
- `create-data-masking.md` - PII masking based on user roles

**Key Features:**
- Transparent field-level encryption
- Immutable audit logs
- Consent management (GDPR/CCPA)
- Data subject rights (access, erasure, portability)
- Role-based field masking

### 3. Resilience & Operations Library
Production-ready resilience patterns for high-availability systems.

**Prompts:**
- `create-graceful-shutdown.md` - Graceful shutdown with request draining
- `create-health-checks.md` - Liveness/readiness probes with dependency health
- `create-circuit-breaker.md` - Resilience4j circuit breaker patterns
- `create-rate-limiting.md` - Distributed rate limiting and traffic shaping

**Key Features:**
- Kubernetes-compatible health checks
- Circuit breaker state management
- Distributed rate limiting with Redis
- Graceful shutdown procedures
- Dependency health monitoring

### 4. Caching & Optimization Library
Multi-layer caching strategies for performance optimization.

**Prompts:**
- `create-distributed-caching.md` - Redis with Caffeine L1 cache
- `create-cache-invalidation.md` - Event-driven and pattern-based invalidation

**Key Features:**
- Multi-level caching (local L1 + distributed L2)
- Cache-aside, read-through, write-through patterns
- Event-based invalidation
- Cache warming strategies
- TTL management

### 5. Data Quality & Validation Library
Ensure data integrity and prevent duplicate processing.

**Prompts:**
- `create-schema-validation.md` - JSR-380 Bean Validation and JSON Schema
- `create-idempotency-deduplication.md` - Idempotent operations and deduplication

**Key Features:**
- Comprehensive validation frameworks
- Custom validation annotations
- Idempotency key management
- Duplicate detection (Redis, database, Flink)
- Cross-field validation

### 6. Advanced Testing Library
Production-focused testing strategies beyond unit tests.

**Prompts:**
- `create-contract-testing.md` - Consumer-driven contract testing with Pact
- `create-chaos-engineering.md` - Chaos Monkey fault injection and experiments

**Key Features:**
- Contract-driven API development
- Consumer verification
- Chaos engineering methodologies
- Fault injection testing
- Resilience validation

### 7. Observability Library
Comprehensive monitoring, logging, and metrics collection.

**Prompts:**
- `create-structured-logging.md` - JSON structured logging for ELK integration
- `create-metrics-monitoring.md` - Micrometer/Prometheus metrics and Grafana dashboards

**Key Features:**
- Structured JSON logging
- Context propagation in logs
- Business and JVM metrics
- Prometheus scraping
- Grafana dashboard templates

## Architectural Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                    Spring Boot Application                       │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  API Layer with Health Checks & Graceful Shutdown       │    │
│  │  - Liveness/Readiness Probes                            │    │
│  │  - Shutdown Hooks & Request Draining                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Authentication & Authorization Layer                   │    │
│  │  - Audit Logging (All user actions)                     │    │
│  │  - Compliance Checks (GDPR, CCPA, etc.)               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Business Logic Layer                                   │    │
│  │  - Circuit Breakers (Resilience4j)                      │    │
│  │  - Rate Limiting (Distributed)                          │    │
│  │  - Data Validation (JSR-380)                            │    │
│  │  - Idempotency Handling                                 │    │
│  └─────────────────────────────────────────────────────────┘    │
│                         ↓                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Data Access Layer                                      │    │
│  │  - Field Encryption (AES-256)                           │    │
│  │  - Caching Layer (Caffeine + Redis)                    │    │
│  │  - Cache Invalidation Events                            │    │
│  │  - Data Masking on Retrieval                            │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│  Cross-Cutting Concerns:                                         │
│  - Distributed Tracing (OpenTelemetry + Jaeger)                 │
│  - Structured Logging (JSON to ELK Stack)                       │
│  - Metrics (Micrometer to Prometheus)                           │
│  - Contract Testing (Pact verification)                         │
│  - Chaos Engineering (Fault injection)                          │
└──────────────────────────────────────────────────────────────────┘
```

## Quick Start by Use Case

### Building a Secure Microservice
1. Start with distributed-tracing: `create-opentelemetry-setup.md`
2. Add security: `create-field-encryption.md` + `create-audit-logging.md`
3. Ensure compliance: `create-compliance-frameworks.md`
4. Add resilience: `create-circuit-breaker.md` + `create-rate-limiting.md`
5. Optimize: `create-distributed-caching.md`
6. Monitor: `create-structured-logging.md` + `create-metrics-monitoring.md`

### Building a Resilient API
1. Start with health checks: `create-health-checks.md`
2. Add circuit breakers: `create-circuit-breaker.md`
3. Add rate limiting: `create-rate-limiting.md`
4. Enable graceful shutdown: `create-graceful-shutdown.md`
5. Add tracing: `create-opentelemetry-setup.md`
6. Validate data: `create-schema-validation.md`
7. Test with chaos: `create-chaos-engineering.md`

### Building a High-Performance System
1. Start with caching: `create-distributed-caching.md`
2. Add invalidation: `create-cache-invalidation.md`
3. Optimize queries: `create-schema-validation.md`
4. Monitor performance: `create-metrics-monitoring.md`
5. Ensure reliability: `create-idempotency-deduplication.md`
6. Test thoroughly: `create-contract-testing.md`

## Integration Matrix

| Library | Spring Boot | Project Reactor | Apache Flink | Kafka | Redis |
|---------|------------|-----------------|--------------|-------|-------|
| **Distributed Tracing** | ✅ | ✅ | ✅ | - | - |
| **Security & Compliance** | ✅ | ✅ | - | - | ✅ |
| **Resilience** | ✅ | ✅ | - | - | ✅ |
| **Caching** | ✅ | ✅ | - | - | ✅ |
| **Data Quality** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Advanced Testing** | ✅ | ✅ | ✅ | - | - |
| **Observability** | ✅ | ✅ | - | - | - |

## Feature Comparison Table

| Feature | Implementation | Use Case |
|---------|-----------------|----------|
| **Encryption** | Field-level AES-256 | PII, payment data protection |
| **Audit Logging** | Immutable database logs | Compliance, forensics |
| **Health Checks** | Liveness/Readiness probes | Kubernetes orchestration |
| **Circuit Breakers** | Resilience4j with fallbacks | Fault tolerance |
| **Rate Limiting** | Distributed token bucket | Traffic shaping, abuse prevention |
| **Caching** | Caffeine L1 + Redis L2 | Performance optimization |
| **Validation** | JSR-380 Bean Validation | Data integrity |
| **Idempotency** | Request deduplication | Exactly-once processing |
| **Contract Testing** | Pact framework | API contract enforcement |
| **Chaos Engineering** | Chaos Monkey + custom aspects | Resilience validation |
| **Structured Logging** | JSON/Logstash format | Log aggregation (ELK) |
| **Metrics** | Micrometer/Prometheus | Monitoring & alerting |
| **Distributed Tracing** | OpenTelemetry + Jaeger | End-to-end request tracking |

## Best Practices Across Libraries

### Security First
1. Encrypt sensitive data at field level
2. Audit all data modifications
3. Enforce compliance checks before operations
4. Mask PII in responses and logs
5. Rotate encryption keys regularly

### Resilience by Design
1. Implement health checks for all dependencies
2. Use circuit breakers for external calls
3. Apply rate limiting for traffic control
4. Graceful shutdown with request draining
5. Monitor and alert on anomalies

### Performance Optimization
1. Multi-layer caching strategy
2. Event-driven cache invalidation
3. Request deduplication for idempotency
4. Batch database operations
5. Async processing where possible

### Observability Excellence
1. Distributed tracing across services
2. Structured logging in JSON format
3. Comprehensive metrics collection
4. Correlated logs, metrics, traces
5. Real-time alerting on anomalies

### Testing & Quality
1. Contract-driven API development
2. Chaos engineering for resilience
3. Comprehensive validation rules
4. Integration testing with real dependencies
5. Regular chaos experiments in production

## Compliance Mapping

| Regulation | Relevant Libraries | Key Features |
|-----------|-------------------|--------------|
| **GDPR** | Security & Compliance, Observability | Encryption, audit logging, consent, data rights |
| **CCPA** | Security & Compliance | Consent management, opt-out, data access |
| **HIPAA** | Security & Compliance | Encryption, access control, audit trails |
| **PCI DSS** | Security & Compliance, Resilience | Card data encryption, rate limiting, monitoring |
| **SOX** | Observability, Security & Compliance | Immutable audit logs, financial transaction tracking |

## Technology Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| **Framework** | Spring Boot | 3.x |
| **Reactive** | Project Reactor | 2022.x |
| **Streaming** | Apache Flink | 2.0 |
| **Caching** | Redis + Caffeine | 7.0 + 3.1 |
| **Encryption** | Java Crypto + Tink | JDK 17 |
| **Monitoring** | Micrometer/Prometheus | Latest |
| **Logging** | SLF4J/Logback/Logstash | Latest |
| **Testing** | Pact/JUnit5/Chaos Monkey | Latest |
| **Tracing** | OpenTelemetry/Jaeger | 1.10+ |
| **Resilience** | Resilience4j | 2.1+ |

## Troubleshooting Guide

### Distributed Tracing Issues
- **Traces not appearing**: Verify Jaeger backend running, check sampling rate
- **Missing context**: Ensure W3C Trace Context headers propagated
- **Performance impact**: Reduce sampling rate, use tail-based sampling

### Security Concerns
- **Encryption overhead**: Monitor performance, use batch operations
- **Key rotation failures**: Test key rotation in staging first
- **Audit log bloat**: Implement retention policies

### Resilience Problems
- **Circuit breaker stuck**: Adjust failure threshold and timeout
- **Rate limiting too strict**: Review rate limit configuration per endpoint
- **Health check failures**: Add dependency-specific health indicators

### Performance Bottlenecks
- **Cache hit ratio low**: Increase TTL, pre-warm cache
- **Database contention**: Add indexes, use connection pooling
- **Memory pressure**: Monitor GC, adjust heap size

## Migration Guide

### From Legacy Applications
1. Add structured logging first (`create-structured-logging.md`)
2. Implement health checks (`create-health-checks.md`)
3. Add distributed tracing (`create-opentelemetry-setup.md`)
4. Implement caching (`create-distributed-caching.md`)
5. Add resilience patterns (`create-circuit-breaker.md`)
6. Implement security (`create-field-encryption.md`)

### Incremental Adoption
- Start with observability (logging, metrics, tracing)
- Add resilience patterns (health checks, circuit breakers)
- Implement security (encryption, audit logging)
- Optimize performance (caching, rate limiting)
- Ensure quality (testing, validation, idempotency)

## Contributing & Extending

Each library is designed to be independent but complementary. To extend:

1. Follow the same markdown format
2. Include production-ready Java 17 code
3. Add configuration examples
4. Document best practices
5. Provide related documentation links
6. Test thoroughly with real dependencies

## Related Core Libraries

These advanced libraries complement:
- `spring-boot/` - Core Spring Boot patterns
- `project-reactor/` - Reactive programming patterns
- `flink2.0/` - Streaming data processing

## Support & Questions

For questions or issues:
1. Check the relevant library README
2. Review prompt code examples
3. Consult best practices section
4. Check troubleshooting guide
5. Refer to related documentation

---

**Last Updated**: 2025-11-25
**Versions**: Spring Boot 3.x, Java 17 LTS, Flink 2.0
