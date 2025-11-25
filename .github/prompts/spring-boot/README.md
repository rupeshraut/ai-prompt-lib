# Spring Boot Application Development Prompts

This directory contains custom prompts for generating Spring Boot application components and infrastructure.

## Available Prompts

### Core Components

| Template | Description |
|----------|-------------|
| `create-entity.md` | JPA entity classes with annotations and relationships |
| `create-dto.md` | Data Transfer Objects with validation and mapping annotations |
| `create-repository.md` | Spring Data JPA repository interfaces with custom queries |
| `create-service.md` | Business logic service classes with transaction management |
| `create-controller.md` | REST controller endpoints with proper HTTP methods and status codes |

### Infrastructure & Configuration

| Template | Description |
|----------|-------------|
| `create-security-config.md` | Spring Security configuration with authentication and authorization |
| `create-migration.md` | Database migration scripts for Liquibase or Flyway |

### Supporting Components

| Template | Description |
|----------|-------------|
| `create-mapper.md` | Entity-to-DTO mappers (MapStruct patterns) |
| `create-exception-handler.md` | Global exception handling with custom error responses |
| `create-test.md` | Unit and integration tests with JUnit 5 and Mockito |

## Quick Start

### 1. Create Entity

```
@workspace Use .github/prompts/spring-boot/create-entity.md

Create entity with JPA annotations and relationships
```

### 2. Create DTO

```
@workspace Use .github/prompts/spring-boot/create-dto.md

Create DTO with validation annotations for REST API
```

### 3. Create Repository

```
@workspace Use .github/prompts/spring-boot/create-repository.md

Create JPA repository with custom query methods
```

### 4. Create Service

```
@workspace Use .github/prompts/spring-boot/create-service.md

Create business logic service with transaction management
```

### 5. Create Controller

```
@workspace Use .github/prompts/spring-boot/create-controller.md

Create REST API endpoints with proper HTTP handling
```

### 6. Create Mapper

```
@workspace Use .github/prompts/spring-boot/create-mapper.md

Create entity-to-DTO mapper for data transformation
```

### 7. Create Exception Handler

```
@workspace Use .github/prompts/spring-boot/create-exception-handler.md

Create global exception handler for consistent error responses
```

### 8. Create Migrations

```
@workspace Use .github/prompts/spring-boot/create-migration.md

Create database migrations for schema changes
```

### 9. Create Tests

```
@workspace Use .github/prompts/spring-boot/create-test.md

Create unit and integration tests for components
```

## Development Layers

### Presentation Layer
- **Controller** - REST endpoints
- **DTO** - Data transfer objects

### Business Layer
- **Service** - Business logic and orchestration
- **Mapper** - Data transformation

### Data Access Layer
- **Entity** - Domain model with JPA annotations
- **Repository** - Database access

### Cross-Cutting Concerns
- **Exception Handler** - Centralized error handling
- **Security Config** - Authentication and authorization
- **Migration** - Database versioning

## Technologies

- Spring Boot 3.x
- Spring Data JPA
- Spring Security
- Spring Web (REST)
- JUnit 5
- Mockito
- MapStruct
- Liquibase / Flyway
- H2 / PostgreSQL (Database)
