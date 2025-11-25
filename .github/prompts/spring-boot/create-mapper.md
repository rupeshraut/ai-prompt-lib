# Create Mapper Class

Create a mapper class for converting between entities and DTOs following Spring Boot best practices.

## Requirements

The mapper should include:
- `@Component` annotation for Spring dependency injection
- Methods to convert entity to response DTO
- Methods to convert request DTO to entity
- Methods to update existing entity from DTO
- Support for collections and pagination
- Null-safe implementations

## Approaches

### Option 1: Manual Mapper (Recommended for simple cases)
Use a `@Component` class with explicit mapping methods.

### Option 2: MapStruct (Recommended for complex mappings)
Use MapStruct library for compile-time code generation.

## Manual Mapper Example

```java
package com.example.project.mapper;

import com.example.project.dto.*;
import com.example.project.entity.Entity;
import org.springframework.stereotype.Component;

import java.util.List;
import java.util.stream.Collectors;

/**
 * Mapper for converting between Entity and DTOs.
 */
@Component
public class EntityMapper {

    /**
     * Convert request DTO to entity.
     * Note: Does not set ID or audit fields.
     *
     * @param dto the request DTO
     * @return the entity
     */
    public Entity toEntity(EntityRequestDto dto) {
        if (dto == null) {
            return null;
        }

        Entity entity = new Entity();
        entity.setField1(dto.field1());
        entity.setField2(dto.field2());
        // Map other fields...
        return entity;
    }

    /**
     * Convert entity to response DTO.
     *
     * @param entity the entity
     * @return the response DTO
     */
    public EntityResponseDto toResponseDto(Entity entity) {
        if (entity == null) {
            return null;
        }

        return new EntityResponseDto(
            entity.getId(),
            entity.getField1(),
            entity.getField2(),
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }

    /**
     * Convert entity to summary DTO (for list views).
     *
     * @param entity the entity
     * @return the summary DTO
     */
    public EntitySummaryDto toSummaryDto(Entity entity) {
        if (entity == null) {
            return null;
        }

        return new EntitySummaryDto(
            entity.getId(),
            entity.getField1(),
            entity.getField2()
        );
    }

    /**
     * Update existing entity with data from DTO.
     * Does not modify ID or audit fields.
     *
     * @param entity the entity to update
     * @param dto the update DTO
     */
    public void updateEntity(Entity entity, EntityUpdateDto dto) {
        if (entity == null || dto == null) {
            return;
        }

        if (dto.field1() != null) {
            entity.setField1(dto.field1());
        }
        if (dto.field2() != null) {
            entity.setField2(dto.field2());
        }
        // Update other fields...
    }

    /**
     * Convert list of entities to list of response DTOs.
     *
     * @param entities the list of entities
     * @return the list of response DTOs
     */
    public List<EntityResponseDto> toResponseDtoList(List<Entity> entities) {
        if (entities == null) {
            return List.of();
        }

        return entities.stream()
            .map(this::toResponseDto)
            .collect(Collectors.toList());
    }

    /**
     * Convert list of entities to list of summary DTOs.
     *
     * @param entities the list of entities
     * @return the list of summary DTOs
     */
    public List<EntitySummaryDto> toSummaryDtoList(List<Entity> entities) {
        if (entities == null) {
            return List.of();
        }

        return entities.stream()
            .map(this::toSummaryDto)
            .collect(Collectors.toList());
    }
}
```

## MapStruct Mapper Example

### Step 1: Add MapStruct Dependency

**Maven:**
```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.5.5.Final</version>
    <scope>provided</scope>
</dependency>
```

**Gradle:**
```groovy
implementation 'org.mapstruct:mapstruct:1.5.5.Final'
annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
```

### Step 2: Create MapStruct Mapper Interface

```java
package com.example.project.mapper;

import com.example.project.dto.*;
import com.example.project.entity.Entity;
import org.mapstruct.*;

import java.util.List;

/**
 * MapStruct mapper for Entity.
 */
@Mapper(
    componentModel = "spring",
    nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE,
    unmappedTargetPolicy = ReportingPolicy.IGNORE
)
public interface EntityMapper {

    /**
     * Convert request DTO to entity.
     */
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    Entity toEntity(EntityRequestDto dto);

    /**
     * Convert entity to response DTO.
     */
    EntityResponseDto toResponseDto(Entity entity);

    /**
     * Convert entity to summary DTO.
     */
    EntitySummaryDto toSummaryDto(Entity entity);

    /**
     * Update entity from DTO (partial update).
     */
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "updatedAt", ignore = true)
    void updateEntity(@MappingTarget Entity entity, EntityUpdateDto dto);

    /**
     * Convert list of entities to response DTOs.
     */
    List<EntityResponseDto> toResponseDtoList(List<Entity> entities);

    /**
     * Convert list of entities to summary DTOs.
     */
    List<EntitySummaryDto> toSummaryDtoList(List<Entity> entities);
}
```

### Advanced MapStruct Features

#### Custom Mapping Methods

```java
@Mapper(componentModel = "spring")
public interface UserMapper {

    @Mapping(target = "fullName", expression = "java(user.getFirstName() + \" \" + user.getLastName())")
    @Mapping(target = "roleNames", source = "roles", qualifiedByName = "rolesToNames")
    UserResponseDto toResponseDto(User user);

    @Named("rolesToNames")
    default List<String> rolesToNames(Set<Role> roles) {
        if (roles == null) {
            return List.of();
        }
        return roles.stream()
            .map(Role::getName)
            .collect(Collectors.toList());
    }
}
```

#### Nested Object Mapping

```java
@Mapper(componentModel = "spring", uses = {AddressMapper.class, RoleMapper.class})
public interface UserMapper {

    @Mapping(target = "address", source = "address")
    @Mapping(target = "roles", source = "roles")
    UserResponseDto toResponseDto(User user);
}
```

## Best Practices

### Null Safety
- Always check for null inputs
- Return empty collections instead of null
- Use Optional where appropriate

### Immutability
- DTOs should be immutable (use Java records)
- Don't expose mutable collections
- Create new DTOs instead of modifying existing ones

### Performance
- Use MapStruct for compile-time generation (faster than reflection)
- Avoid creating unnecessary intermediate objects
- Use lazy loading for nested mappings when appropriate

### Consistency
- Map all fields explicitly (avoid missing fields)
- Use consistent naming conventions
- Document complex mappings

### Testing
- Unit test all mapping methods
- Test null handling
- Test edge cases (empty collections, missing fields)
- Verify bidirectional mappings if applicable

## Testing Example

```java
@ExtendWith(MockitoExtension.class)
class EntityMapperTest {

    private EntityMapper mapper;

    @BeforeEach
    void setUp() {
        mapper = new EntityMapper();
    }

    @Test
    void shouldMapEntityToResponseDto() {
        // Given
        Entity entity = new Entity();
        entity.setId(1L);
        entity.setField1("value1");
        entity.setField2("value2");

        // When
        EntityResponseDto dto = mapper.toResponseDto(entity);

        // Then
        assertThat(dto).isNotNull();
        assertThat(dto.id()).isEqualTo(1L);
        assertThat(dto.field1()).isEqualTo("value1");
        assertThat(dto.field2()).isEqualTo("value2");
    }

    @Test
    void shouldReturnNullWhenEntityIsNull() {
        // When
        EntityResponseDto dto = mapper.toResponseDto(null);

        // Then
        assertThat(dto).isNull();
    }

    @Test
    void shouldUpdateEntityFromDto() {
        // Given
        Entity entity = new Entity();
        entity.setId(1L);
        entity.setField1("old");

        EntityUpdateDto dto = new EntityUpdateDto("new", null);

        // When
        mapper.updateEntity(entity, dto);

        // Then
        assertThat(entity.getId()).isEqualTo(1L); // Unchanged
        assertThat(entity.getField1()).isEqualTo("new");
    }
}
```

## When to Use Which Approach

| Scenario | Recommended Approach |
|----------|---------------------|
| Simple mappings (< 10 fields) | Manual mapper |
| Complex nested objects | MapStruct |
| High-performance requirements | MapStruct |
| Minimal dependencies | Manual mapper |
| Large number of entities | MapStruct |
| Custom transformation logic | Manual mapper |
