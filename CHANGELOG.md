# Changelog

All notable changes to the GitHub Copilot Prompt Engineering Reference Repository will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned

- Sample project with generated code from prompts
- TypeScript/Node.js equivalent prompt templates
- Python/FastAPI equivalent prompt templates
- VS Code snippets for common prompts
- GitHub Actions workflow templates
- Apache Kafka Streams prompt templates
- Apache Spark Streaming prompt templates

---

## [1.2.1] - 2025-11-22

### Added

- **Two-Job Architecture for Sanctions Processing**
  - `create-sanctions-request-job.md` - Job 1: Request flow with JMS sender
  - `create-sanctions-response-job.md` - Job 2: Response flow with correlation
  - JMS-to-Kafka bridge for decoupled response handling
  - Independent scaling and fault isolation

### Changed

- Updated `META_PROMPT_FLINK_FEDNOW.md` with two-job architecture diagrams
- Updated `PRACTICAL_EXAMPLE_FEDNOW_PAYMENT.md` to 11-step prompt chain
- Updated `.github/prompts/flink-fednow/README.md` with two-job quick start

### Why Two Jobs?

| Benefit | Description |
|---------|-------------|
| Independent Scaling | Scale JMS consumer separately from request processing |
| Fault Isolation | FircoSoft issues don't block payment ingestion |
| Operational Flexibility | Deploy/restart jobs independently |
| Kafka as Buffer | Durable storage for pending payments during outages |

---

## [1.2.0] - 2025-11-22

### Added

- **Apache Flink 2.0 FedNow Payment Processing Framework**
  - Complete multi-framework support architecture
  - New `flink-fednow/` subdirectory in `.github/prompts/`

- **FedNow Meta Prompt**
  - `META_PROMPT_FLINK_FEDNOW.md` - Comprehensive FedNow payment processing context
  - ISO 20022 message standards (pacs.008, pacs.002, camt.056)
  - External service contracts (Account Validation, RCS, FircoSoft, TMS)
  - Processing pipeline architecture and flow diagrams

- **Flink Prompt Templates** (`.github/prompts/flink-fednow/`)
  - `create-flink-job.md` - Main Flink job with checkpointing, state backend, pipeline
  - `create-async-function.md` - RichAsyncFunction for non-blocking HTTP calls
  - `create-sanctions-processor.md` - KeyedProcessFunction for JMS-based sanctions screening
  - `create-iso20022-parser.md` - ISO 20022 XML parsers and serializers
  - `create-payment-model.md` - Payment domain model with status enums
  - `create-flink-test.md` - Flink test harness patterns and integration tests

- **FedNow Practical Example**
  - `PRACTICAL_EXAMPLE_FEDNOW_PAYMENT.md` - Complete payment processing chain
  - Account validation (HTTP async)
  - Risk control debit/credit (HTTP async)
  - Sanctions screening via JMS to/from FircoSoft
  - Account posting (HTTP async)
  - Response generation and Kafka sink

### Features

- **Flink 2.0 Patterns**
  - Async I/O for non-blocking HTTP service calls
  - KeyedProcessFunction for stateful JMS correlation
  - RocksDB state backend with exactly-once checkpointing
  - Side outputs for error handling and DLQ routing
  - Timer-based timeout handling

- **FedNow Integration**
  - ISO 20022 message parsing (pacs.008 Credit Transfer)
  - ISO 20022 response serialization (pacs.002 Status Report)
  - JMS integration with FircoSoft sanctions screening
  - HTTP integration with Account Validation, RCS, TMS

- **Production-Ready Features**
  - Retry logic with exponential backoff
  - Comprehensive metrics collection
  - Structured logging with transaction context
  - Dead letter queue routing for failures

### Compatibility

- Apache Flink 2.0
- Java 17 LTS
- Apache Kafka (source/sink)
- IBM MQ / ActiveMQ (JMS)
- RocksDB state backend
- Jackson XML for ISO 20022

---

## [1.1.0] - 2025-11-22

### Added

- **New Prompt Templates**
  - `create-mapper.md` - Manual and MapStruct mapper generation
  - `create-test.md` - Unit tests, integration tests, and repository tests
  - `create-security-config.md` - Complete JWT authentication setup
  - `create-migration.md` - Flyway and Liquibase database migrations

- **Role Entity Chain** in `PRACTICAL_EXAMPLE_USER_API.md`
  - Complete 9-step prompt chain for Role entity
  - Role entity, repository, DTOs, mapper, service, controller
  - Role-specific exceptions (RoleNotFoundException, DuplicateRoleException)
  - Role assignment and removal endpoints
  - Database migration examples for roles

- **Security Configuration Templates**
  - SecurityConfig with JWT filter chain
  - JwtTokenProvider for token generation/validation
  - JwtAuthenticationFilter for request processing
  - CustomUserDetailsService implementation
  - AuthController with login, register, refresh, logout
  - AuthService with complete authentication logic
  - Authentication DTOs (LoginRequest, RegisterRequest, AuthResponse)

- **CHANGELOG.md** for tracking version history

### Changed

- Updated `PRACTICAL_EXAMPLE_USER_API.md` with bonus Role entity section
- Enhanced documentation with references to new prompt templates

### Compatibility

- Java 17 LTS
- Spring Boot 3.x
- Spring Security 6.x
- JWT (jjwt 0.11.5)

---

## [1.0.0] - 2025-11-21

### Added

- **Core Documentation**
  - `META_PROMPT.md` - Complete project context and standards
  - `PROMPT_CHAINING.md` - Prompt chaining methodology and patterns
  - `CHEAT_SHEET.md` - Quick reference guide with examples
  - `PRACTICAL_EXAMPLE_USER_API.md` - 11-step User Management API walkthrough

- **GitHub Copilot Configuration**
  - `.github/copilot-instructions.md` - Auto-loaded project standards

- **Prompt Templates** (`.github/prompts/`)
  - `create-entity.md` - JPA entity generation
  - `create-dto.md` - Java 17 record DTOs
  - `create-repository.md` - Spring Data JPA repositories
  - `create-service.md` - Service layer with business logic
  - `create-controller.md` - REST controllers with OpenAPI
  - `create-exception-handler.md` - Global exception handling
  - `README.md` - Prompt usage guide

### Features

- Complete User Management API example with 11-step prompt chain
- Java 17 features (records, sealed classes, pattern matching)
- Spring Boot 3.x best practices
- RESTful API design guidelines
- Security best practices (BCrypt, JWT, RBAC)
- Testing patterns (JUnit 5, Mockito, AssertJ)
- Layered architecture (Controller, Service, Repository, Entity, DTO)

### Compatibility

- Java 17 LTS
- Spring Boot 3.x
- Spring Data JPA
- Spring Security
- PostgreSQL/MySQL
- Maven/Gradle

---

## Version History Summary

| Version | Date | Highlights |
|---------|------|------------|
| 1.2.1 | 2025-11-22 | Two-job architecture for sanctions processing |
| 1.2.0 | 2025-11-22 | Apache Flink 2.0 FedNow payment processing framework |
| 1.1.0 | 2025-11-22 | Added mapper, test, security, migration templates; Role entity chain |
| 1.0.0 | 2025-11-21 | Initial release with core documentation and 6 prompt templates |

---

## Upgrade Guide

### From 1.0.0 to 1.1.0

No breaking changes. New features are additive.

**New templates available:**

```bash
.github/prompts/create-mapper.md
.github/prompts/create-test.md
.github/prompts/create-security-config.md
.github/prompts/create-migration.md
```

**To use new security templates:**

1. Add JWT dependencies to your project (see `create-security-config.md`)
2. Configure JWT properties in `application.yml`
3. Follow the 8-component security setup

**To implement Role entity:**

1. Follow the Role entity chain in `PRACTICAL_EXAMPLE_USER_API.md`
2. Create database migrations using `create-migration.md`
3. Update GlobalExceptionHandler with new role exceptions

---

## Contributing

When contributing to this repository:

1. Update the `[Unreleased]` section with your changes
2. Follow the format: Added, Changed, Deprecated, Removed, Fixed, Security
3. Reference any related issues or PRs
4. Ensure all prompt templates follow the established structure

---

## Links

- [Repository](https://github.com/your-org/prompt-engg)
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot)
- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Java 17 Documentation](https://docs.oracle.com/en/java/javase/17/)
