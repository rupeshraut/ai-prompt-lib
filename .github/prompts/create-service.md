# Create Service Class

Create a service class that implements business logic following Spring Boot best practices.

## Requirements

The service class should include:
- `@Service` annotation
- Constructor injection for dependencies (use `@RequiredArgsConstructor`)
- Business logic implementation
- Transaction management (`@Transactional` where needed)
- Proper exception handling
- Return DTOs, never entities
- Logging with SLF4J
- Input validation

## Code Style
- Use constructor injection (final fields)
- Use Lombok's `@RequiredArgsConstructor` and `@Slf4j`
- Keep methods small and focused (< 20 lines)
- Follow Single Responsibility Principle
- Use meaningful method names (verbs)
- Add JavaDoc for public methods

## Example Structure

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class EntityService {
    
    private final EntityRepository entityRepository;
    private final EntityMapper entityMapper;
    // Other dependencies
    
    /**
     * Retrieve entity by ID.
     *
     * @param id the entity ID
     * @return the entity response DTO
     * @throws EntityNotFoundException if entity not found
     */
    public EntityResponseDto findById(Long id) {
        log.debug("Finding entity with id: {}", id);
        
        Entity entity = entityRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Entity not found with id: " + id));
        
        return entityMapper.toResponseDto(entity);
    }
    
    /**
     * Create a new entity.
     *
     * @param requestDto the entity request DTO
     * @return the created entity response DTO
     */
    @Transactional
    public EntityResponseDto create(EntityRequestDto requestDto) {
        log.info("Creating new entity");
        
        // Business validation
        validateBusinessRules(requestDto);
        
        // Map and save
        Entity entity = entityMapper.toEntity(requestDto);
        Entity savedEntity = entityRepository.save(entity);
        
        log.debug("Entity created successfully with id: {}", savedEntity.getId());
        return entityMapper.toResponseDto(savedEntity);
    }
    
    /**
     * Update existing entity.
     *
     * @param id the entity ID
     * @param requestDto the update request DTO
     * @return the updated entity response DTO
     */
    @Transactional
    public EntityResponseDto update(Long id, EntityRequestDto requestDto) {
        log.info("Updating entity with id: {}", id);
        
        Entity entity = entityRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Entity not found with id: " + id));
        
        // Update fields
        entityMapper.updateEntity(entity, requestDto);
        Entity updatedEntity = entityRepository.save(entity);
        
        log.debug("Entity updated successfully");
        return entityMapper.toResponseDto(updatedEntity);
    }
    
    /**
     * Delete entity by ID.
     *
     * @param id the entity ID
     */
    @Transactional
    public void delete(Long id) {
        log.info("Deleting entity with id: {}", id);
        
        if (!entityRepository.existsById(id)) {
            throw new EntityNotFoundException("Entity not found with id: " + id);
        }
        
        entityRepository.deleteById(id);
        log.debug("Entity deleted successfully");
    }
    
    /**
     * Get all entities with pagination.
     *
     * @param pageable pagination parameters
     * @return page of entity response DTOs
     */
    public Page<EntityResponseDto> findAll(Pageable pageable) {
        log.debug("Finding all entities with pagination");
        
        return entityRepository.findAll(pageable)
            .map(entityMapper::toResponseDto);
    }
    
    private void validateBusinessRules(EntityRequestDto requestDto) {
        // Implement business validation logic
    }
}
```

## Transaction Guidelines
- Use `@Transactional` on methods that modify data (create, update, delete)
- Keep transactions short and focused
- Read-only operations don't need `@Transactional` (or use `@Transactional(readOnly = true)`)
- Consider transaction isolation levels for concurrent operations

## Exception Handling
- Throw custom exceptions for business logic errors
- Let the global exception handler catch and process exceptions
- Don't catch exceptions unless you can handle them meaningfully
- Always log errors with context

## Logging Best Practices
- Log method entry for important operations (INFO level)
- Log successful completion (DEBUG level)
- Log errors with full context (ERROR level)
- Never log sensitive data (passwords, tokens, PII)
- Use parameterized logging: `log.info("Message: {}", param)`

## Testing Considerations
- Service layer should be thoroughly unit tested
- Mock repository dependencies
- Test happy paths and error scenarios
- Verify exception handling
