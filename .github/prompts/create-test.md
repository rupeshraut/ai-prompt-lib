# Create Test Classes

Create comprehensive test classes for Spring Boot applications following testing best practices.

## Test Types

### 1. Unit Tests
- Test individual components in isolation
- Mock all dependencies
- Fast execution

### 2. Integration Tests
- Test component interactions
- Use real (or test) database
- Test API endpoints

### 3. Repository Tests
- Test data access layer
- Use embedded database or test containers

## Unit Test Template (Service Layer)

```java
package com.example.project.service;

import com.example.project.dto.*;
import com.example.project.entity.Entity;
import com.example.project.exception.EntityNotFoundException;
import com.example.project.mapper.EntityMapper;
import com.example.project.repository.EntityRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Pageable;

import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("EntityService Unit Tests")
class EntityServiceTest {

    @Mock
    private EntityRepository entityRepository;

    @Mock
    private EntityMapper entityMapper;

    @InjectMocks
    private EntityService entityService;

    private Entity entity;
    private EntityRequestDto requestDto;
    private EntityResponseDto responseDto;

    @BeforeEach
    void setUp() {
        entity = new Entity();
        entity.setId(1L);
        entity.setName("Test Entity");

        requestDto = new EntityRequestDto("Test Entity", "description");
        responseDto = new EntityResponseDto(1L, "Test Entity", "description", null, null);
    }

    @Nested
    @DisplayName("findById")
    class FindById {

        @Test
        @DisplayName("should return entity when found")
        void shouldReturnEntityWhenFound() {
            // Given
            given(entityRepository.findById(1L)).willReturn(Optional.of(entity));
            given(entityMapper.toResponseDto(entity)).willReturn(responseDto);

            // When
            EntityResponseDto result = entityService.findById(1L);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.id()).isEqualTo(1L);
            assertThat(result.name()).isEqualTo("Test Entity");

            then(entityRepository).should().findById(1L);
            then(entityMapper).should().toResponseDto(entity);
        }

        @Test
        @DisplayName("should throw exception when not found")
        void shouldThrowExceptionWhenNotFound() {
            // Given
            given(entityRepository.findById(999L)).willReturn(Optional.empty());

            // When & Then
            assertThatThrownBy(() -> entityService.findById(999L))
                .isInstanceOf(EntityNotFoundException.class)
                .hasMessageContaining("not found")
                .hasMessageContaining("999");

            then(entityRepository).should().findById(999L);
            then(entityMapper).shouldHaveNoInteractions();
        }
    }

    @Nested
    @DisplayName("create")
    class Create {

        @Test
        @DisplayName("should create entity successfully")
        void shouldCreateEntitySuccessfully() {
            // Given
            given(entityMapper.toEntity(requestDto)).willReturn(entity);
            given(entityRepository.save(any(Entity.class))).willReturn(entity);
            given(entityMapper.toResponseDto(entity)).willReturn(responseDto);

            // When
            EntityResponseDto result = entityService.create(requestDto);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.name()).isEqualTo("Test Entity");

            then(entityMapper).should().toEntity(requestDto);
            then(entityRepository).should().save(any(Entity.class));
            then(entityMapper).should().toResponseDto(entity);
        }

        @Test
        @DisplayName("should throw exception when duplicate name exists")
        void shouldThrowExceptionWhenDuplicateNameExists() {
            // Given
            given(entityRepository.existsByName(requestDto.name())).willReturn(true);

            // When & Then
            assertThatThrownBy(() -> entityService.create(requestDto))
                .isInstanceOf(DuplicateEntityException.class)
                .hasMessageContaining("already exists");

            then(entityRepository).should().existsByName(requestDto.name());
            then(entityRepository).should(never()).save(any());
        }
    }

    @Nested
    @DisplayName("update")
    class Update {

        @Test
        @DisplayName("should update entity successfully")
        void shouldUpdateEntitySuccessfully() {
            // Given
            EntityUpdateDto updateDto = new EntityUpdateDto("Updated Name", "Updated description");

            given(entityRepository.findById(1L)).willReturn(Optional.of(entity));
            given(entityRepository.save(any(Entity.class))).willReturn(entity);
            given(entityMapper.toResponseDto(entity)).willReturn(responseDto);

            // When
            EntityResponseDto result = entityService.update(1L, updateDto);

            // Then
            assertThat(result).isNotNull();

            then(entityRepository).should().findById(1L);
            then(entityMapper).should().updateEntity(entity, updateDto);
            then(entityRepository).should().save(entity);
        }

        @Test
        @DisplayName("should throw exception when entity not found")
        void shouldThrowExceptionWhenEntityNotFound() {
            // Given
            EntityUpdateDto updateDto = new EntityUpdateDto("Updated", "description");
            given(entityRepository.findById(999L)).willReturn(Optional.empty());

            // When & Then
            assertThatThrownBy(() -> entityService.update(999L, updateDto))
                .isInstanceOf(EntityNotFoundException.class);

            then(entityRepository).should(never()).save(any());
        }
    }

    @Nested
    @DisplayName("delete")
    class Delete {

        @Test
        @DisplayName("should delete entity successfully")
        void shouldDeleteEntitySuccessfully() {
            // Given
            given(entityRepository.findById(1L)).willReturn(Optional.of(entity));

            // When
            entityService.delete(1L);

            // Then
            then(entityRepository).should().findById(1L);
            then(entityRepository).should().delete(entity);
        }

        @Test
        @DisplayName("should throw exception when entity not found")
        void shouldThrowExceptionWhenEntityNotFound() {
            // Given
            given(entityRepository.findById(999L)).willReturn(Optional.empty());

            // When & Then
            assertThatThrownBy(() -> entityService.delete(999L))
                .isInstanceOf(EntityNotFoundException.class);

            then(entityRepository).should(never()).delete(any());
        }
    }

    @Nested
    @DisplayName("findAll")
    class FindAll {

        @Test
        @DisplayName("should return paginated results")
        void shouldReturnPaginatedResults() {
            // Given
            Pageable pageable = PageRequest.of(0, 10);
            Page<Entity> entityPage = new PageImpl<>(List.of(entity));

            given(entityRepository.findAll(pageable)).willReturn(entityPage);
            given(entityMapper.toResponseDto(entity)).willReturn(responseDto);

            // When
            Page<EntityResponseDto> result = entityService.findAll(pageable);

            // Then
            assertThat(result).isNotNull();
            assertThat(result.getContent()).hasSize(1);
            assertThat(result.getContent().get(0).name()).isEqualTo("Test Entity");
        }

        @Test
        @DisplayName("should return empty page when no entities exist")
        void shouldReturnEmptyPageWhenNoEntitiesExist() {
            // Given
            Pageable pageable = PageRequest.of(0, 10);
            Page<Entity> emptyPage = Page.empty();

            given(entityRepository.findAll(pageable)).willReturn(emptyPage);

            // When
            Page<EntityResponseDto> result = entityService.findAll(pageable);

            // Then
            assertThat(result).isEmpty();
        }
    }
}
```

## Integration Test Template (Controller Layer)

```java
package com.example.project.controller;

import com.example.project.dto.*;
import com.example.project.exception.EntityNotFoundException;
import com.example.project.service.EntityService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageImpl;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import java.time.LocalDateTime;
import java.util.List;

import static org.hamcrest.Matchers.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.BDDMockito.*;
import static org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultHandlers.print;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(EntityController.class)
@DisplayName("EntityController Integration Tests")
class EntityControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private EntityService entityService;

    private EntityResponseDto responseDto;
    private EntityRequestDto requestDto;

    @BeforeEach
    void setUp() {
        responseDto = new EntityResponseDto(
            1L, "Test Entity", "description",
            LocalDateTime.now(), LocalDateTime.now()
        );
        requestDto = new EntityRequestDto("Test Entity", "description");
    }

    @Nested
    @DisplayName("GET /api/v1/entities/{id}")
    class GetById {

        @Test
        @DisplayName("should return entity when found")
        @WithMockUser
        void shouldReturnEntityWhenFound() throws Exception {
            // Given
            given(entityService.findById(1L)).willReturn(responseDto);

            // When & Then
            mockMvc.perform(get("/api/v1/entities/{id}", 1L))
                .andDo(print())
                .andExpect(status().isOk())
                .andExpect(content().contentType(MediaType.APPLICATION_JSON))
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Test Entity"))
                .andExpect(jsonPath("$.description").value("description"));

            then(entityService).should().findById(1L);
        }

        @Test
        @DisplayName("should return 404 when entity not found")
        @WithMockUser
        void shouldReturn404WhenNotFound() throws Exception {
            // Given
            given(entityService.findById(999L))
                .willThrow(new EntityNotFoundException("Entity not found with id: 999"));

            // When & Then
            mockMvc.perform(get("/api/v1/entities/{id}", 999L))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.status").value(404))
                .andExpect(jsonPath("$.message").value(containsString("not found")));
        }

        @Test
        @DisplayName("should return 401 when not authenticated")
        void shouldReturn401WhenNotAuthenticated() throws Exception {
            mockMvc.perform(get("/api/v1/entities/{id}", 1L))
                .andExpect(status().isUnauthorized());
        }
    }

    @Nested
    @DisplayName("POST /api/v1/entities")
    class Create {

        @Test
        @DisplayName("should create entity and return 201")
        @WithMockUser
        void shouldCreateEntityAndReturn201() throws Exception {
            // Given
            given(entityService.create(any(EntityRequestDto.class))).willReturn(responseDto);

            // When & Then
            mockMvc.perform(post("/api/v1/entities")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(requestDto)))
                .andExpect(status().isCreated())
                .andExpect(jsonPath("$.id").value(1))
                .andExpect(jsonPath("$.name").value("Test Entity"));

            then(entityService).should().create(any(EntityRequestDto.class));
        }

        @Test
        @DisplayName("should return 400 when validation fails")
        @WithMockUser
        void shouldReturn400WhenValidationFails() throws Exception {
            // Given - Invalid request (empty name)
            EntityRequestDto invalidRequest = new EntityRequestDto("", "description");

            // When & Then
            mockMvc.perform(post("/api/v1/entities")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(invalidRequest)))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.fieldErrors").isArray())
                .andExpect(jsonPath("$.fieldErrors[0].field").value("name"));

            then(entityService).shouldHaveNoInteractions();
        }

        @Test
        @DisplayName("should return 409 when duplicate entity exists")
        @WithMockUser
        void shouldReturn409WhenDuplicateExists() throws Exception {
            // Given
            given(entityService.create(any(EntityRequestDto.class)))
                .willThrow(new DuplicateEntityException("Entity already exists"));

            // When & Then
            mockMvc.perform(post("/api/v1/entities")
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(requestDto)))
                .andExpect(status().isConflict());
        }
    }

    @Nested
    @DisplayName("PUT /api/v1/entities/{id}")
    class Update {

        @Test
        @DisplayName("should update entity and return 200")
        @WithMockUser
        void shouldUpdateEntityAndReturn200() throws Exception {
            // Given
            EntityUpdateDto updateDto = new EntityUpdateDto("Updated", "new description");
            given(entityService.update(eq(1L), any(EntityUpdateDto.class))).willReturn(responseDto);

            // When & Then
            mockMvc.perform(put("/api/v1/entities/{id}", 1L)
                    .with(csrf())
                    .contentType(MediaType.APPLICATION_JSON)
                    .content(objectMapper.writeValueAsString(updateDto)))
                .andExpect(status().isOk());

            then(entityService).should().update(eq(1L), any(EntityUpdateDto.class));
        }
    }

    @Nested
    @DisplayName("DELETE /api/v1/entities/{id}")
    class Delete {

        @Test
        @DisplayName("should delete entity and return 204")
        @WithMockUser(roles = "ADMIN")
        void shouldDeleteEntityAndReturn204() throws Exception {
            // Given
            willDoNothing().given(entityService).delete(1L);

            // When & Then
            mockMvc.perform(delete("/api/v1/entities/{id}", 1L)
                    .with(csrf()))
                .andExpect(status().isNoContent());

            then(entityService).should().delete(1L);
        }

        @Test
        @DisplayName("should return 403 when user is not admin")
        @WithMockUser(roles = "USER")
        void shouldReturn403WhenNotAdmin() throws Exception {
            mockMvc.perform(delete("/api/v1/entities/{id}", 1L)
                    .with(csrf()))
                .andExpect(status().isForbidden());

            then(entityService).shouldHaveNoInteractions();
        }
    }

    @Nested
    @DisplayName("GET /api/v1/entities")
    class GetAll {

        @Test
        @DisplayName("should return paginated list")
        @WithMockUser
        void shouldReturnPaginatedList() throws Exception {
            // Given
            Page<EntityResponseDto> page = new PageImpl<>(List.of(responseDto));
            given(entityService.findAll(any())).willReturn(page);

            // When & Then
            mockMvc.perform(get("/api/v1/entities")
                    .param("page", "0")
                    .param("size", "10"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.content").isArray())
                .andExpect(jsonPath("$.content", hasSize(1)))
                .andExpect(jsonPath("$.content[0].id").value(1));
        }
    }
}
```

## Repository Test Template

```java
package com.example.project.repository;

import com.example.project.entity.Entity;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;

import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@DisplayName("EntityRepository Tests")
class EntityRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private EntityRepository entityRepository;

    private Entity testEntity;

    @BeforeEach
    void setUp() {
        testEntity = new Entity();
        testEntity.setName("Test Entity");
        testEntity.setDescription("Test description");
        testEntity.setEnabled(true);

        entityManager.persistAndFlush(testEntity);
    }

    @Test
    @DisplayName("should find entity by name")
    void shouldFindEntityByName() {
        // When
        Optional<Entity> found = entityRepository.findByName("Test Entity");

        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("Test Entity");
    }

    @Test
    @DisplayName("should return empty when entity not found by name")
    void shouldReturnEmptyWhenNotFoundByName() {
        // When
        Optional<Entity> found = entityRepository.findByName("Nonexistent");

        // Then
        assertThat(found).isEmpty();
    }

    @Test
    @DisplayName("should check if entity exists by name")
    void shouldCheckIfEntityExistsByName() {
        // When & Then
        assertThat(entityRepository.existsByName("Test Entity")).isTrue();
        assertThat(entityRepository.existsByName("Nonexistent")).isFalse();
    }

    @Test
    @DisplayName("should find enabled entities with pagination")
    void shouldFindEnabledEntitiesWithPagination() {
        // Given
        Entity disabledEntity = new Entity();
        disabledEntity.setName("Disabled Entity");
        disabledEntity.setEnabled(false);
        entityManager.persistAndFlush(disabledEntity);

        // When
        Page<Entity> result = entityRepository.findByEnabledTrue(PageRequest.of(0, 10));

        // Then
        assertThat(result.getContent()).hasSize(1);
        assertThat(result.getContent().get(0).getName()).isEqualTo("Test Entity");
    }

    @Test
    @DisplayName("should search entities by keyword")
    void shouldSearchEntitiesByKeyword() {
        // When
        Page<Entity> result = entityRepository.searchByKeyword("Test", PageRequest.of(0, 10));

        // Then
        assertThat(result.getContent()).hasSize(1);
        assertThat(result.getContent().get(0).getName()).isEqualTo("Test Entity");
    }

    @Test
    @DisplayName("should return empty when no match for keyword")
    void shouldReturnEmptyWhenNoMatchForKeyword() {
        // When
        Page<Entity> result = entityRepository.searchByKeyword("xyz", PageRequest.of(0, 10));

        // Then
        assertThat(result.getContent()).isEmpty();
    }
}
```

## Test Data Builders Pattern

```java
package com.example.project.testutil;

import com.example.project.dto.*;
import com.example.project.entity.Entity;

import java.time.LocalDateTime;

/**
 * Builder for creating test data.
 */
public class TestDataBuilder {

    public static EntityBuilder anEntity() {
        return new EntityBuilder();
    }

    public static class EntityBuilder {
        private Long id = 1L;
        private String name = "Test Entity";
        private String description = "Test description";
        private Boolean enabled = true;
        private LocalDateTime createdAt = LocalDateTime.now();
        private LocalDateTime updatedAt = LocalDateTime.now();

        public EntityBuilder withId(Long id) {
            this.id = id;
            return this;
        }

        public EntityBuilder withName(String name) {
            this.name = name;
            return this;
        }

        public EntityBuilder withDescription(String description) {
            this.description = description;
            return this;
        }

        public EntityBuilder disabled() {
            this.enabled = false;
            return this;
        }

        public Entity build() {
            Entity entity = new Entity();
            entity.setId(id);
            entity.setName(name);
            entity.setDescription(description);
            entity.setEnabled(enabled);
            entity.setCreatedAt(createdAt);
            entity.setUpdatedAt(updatedAt);
            return entity;
        }

        public EntityRequestDto buildRequestDto() {
            return new EntityRequestDto(name, description);
        }

        public EntityResponseDto buildResponseDto() {
            return new EntityResponseDto(id, name, description, createdAt, updatedAt);
        }
    }
}

// Usage in tests:
// Entity entity = TestDataBuilder.anEntity().withName("Custom").build();
// EntityRequestDto dto = TestDataBuilder.anEntity().buildRequestDto();
```

## Best Practices

### Test Naming
- Use descriptive names: `shouldReturnEntityWhenFound`
- Use `@DisplayName` for readable test reports
- Group related tests with `@Nested`

### Test Structure (Given-When-Then)
```java
@Test
void shouldDoSomething() {
    // Given - Set up test data and mocks

    // When - Execute the action being tested

    // Then - Verify the results
}
```

### Assertions
- Use AssertJ for fluent assertions
- Assert specific values, not just "not null"
- Verify mock interactions with `then().should()`

### Mocking
- Use `@Mock` for dependencies
- Use `@InjectMocks` for the class under test
- Use BDD style: `given()`, `when()`, `then()`

### Test Coverage
- Test happy paths
- Test error scenarios
- Test edge cases
- Test validation
- Aim for 80%+ code coverage

### Test Independence
- Each test should be independent
- Use `@BeforeEach` for common setup
- Clean up test data after tests
