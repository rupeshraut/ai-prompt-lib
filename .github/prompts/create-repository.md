# Create JPA Repository

Create a Spring Data JPA repository interface for database operations.

## Requirements

The repository should:
- Extend `JpaRepository<Entity, ID>` or `JpaSpecificationExecutor<Entity>` for complex queries
- Use Spring Data JPA query methods for simple queries
- Add `@Query` for complex JPQL/native queries
- Return `Optional<T>` for single entity results
- Use `Page<T>` for paginated results
- Include `@Repository` annotation (optional but recommended)

## Code Style
- Name: `[Entity]Repository`
- Use query method naming conventions
- Add JavaDoc for custom queries
- Use `@Param` for named parameters in @Query
- Consider performance (avoid N+1 queries)

## Example Structure

```java
@Repository
public interface EntityRepository extends JpaRepository<Entity, Long> {
    
    // Query methods (Spring Data JPA will implement automatically)
    
    /**
     * Find entity by username.
     *
     * @param username the username
     * @return optional containing entity if found
     */
    Optional<Entity> findByUsername(String username);
    
    /**
     * Find entity by email.
     *
     * @param email the email address
     * @return optional containing entity if found
     */
    Optional<Entity> findByEmail(String email);
    
    /**
     * Check if entity exists by username.
     *
     * @param username the username
     * @return true if exists, false otherwise
     */
    boolean existsByUsername(String username);
    
    /**
     * Check if entity exists by email.
     *
     * @param email the email address
     * @return true if exists, false otherwise
     */
    boolean existsByEmail(String email);
    
    /**
     * Find all active entities.
     *
     * @param pageable pagination parameters
     * @return page of active entities
     */
    Page<Entity> findByEnabledTrue(Pageable pageable);
    
    /**
     * Find entities by status.
     *
     * @param status the status to filter by
     * @param pageable pagination parameters
     * @return page of entities with given status
     */
    Page<Entity> findByStatus(String status, Pageable pageable);
    
    /**
     * Find entities created after a specific date.
     *
     * @param date the date threshold
     * @return list of entities
     */
    List<Entity> findByCreatedAtAfter(LocalDateTime date);
    
    // Custom JPQL queries
    
    /**
     * Search entities by keyword in multiple fields.
     *
     * @param keyword the search keyword
     * @param pageable pagination parameters
     * @return page of matching entities
     */
    @Query("SELECT e FROM Entity e WHERE " +
           "LOWER(e.username) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(e.email) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(e.firstName) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Page<Entity> searchByKeyword(@Param("keyword") String keyword, Pageable pageable);
    
    /**
     * Find entities with specific role.
     *
     * @param roleName the role name
     * @return list of entities with the role
     */
    @Query("SELECT e FROM Entity e JOIN e.roles r WHERE r.name = :roleName")
    List<Entity> findByRoleName(@Param("roleName") String roleName);
    
    /**
     * Count entities by status.
     *
     * @param status the status
     * @return count of entities
     */
    @Query("SELECT COUNT(e) FROM Entity e WHERE e.status = :status")
    long countByStatus(@Param("status") String status);
    
    // Native SQL queries (use sparingly)
    
    /**
     * Find entities using native SQL with complex conditions.
     *
     * @param param the parameter
     * @return list of entities
     */
    @Query(value = "SELECT * FROM entities WHERE condition = ?1", nativeQuery = true)
    List<Entity> findByComplexCondition(String param);
    
    // Update/Delete queries
    
    /**
     * Disable entity by ID.
     *
     * @param id the entity ID
     */
    @Modifying
    @Query("UPDATE Entity e SET e.enabled = false WHERE e.id = :id")
    void disableById(@Param("id") Long id);
    
    /**
     * Delete old entities.
     *
     * @param date the date threshold
     */
    @Modifying
    @Query("DELETE FROM Entity e WHERE e.createdAt < :date")
    void deleteOlderThan(@Param("date") LocalDateTime date);
}
```

## Query Method Keywords
Spring Data JPA supports these keywords in method names:
- `findBy`, `getBy`, `readBy`, `queryBy`, `searchBy`
- `countBy`, `existsBy`
- `deleteBy`, `removeBy`
- `And`, `Or`
- `Is`, `Equals`
- `LessThan`, `GreaterThan`, `Between`
- `Like`, `NotLike`, `StartingWith`, `EndingWith`, `Containing`
- `OrderBy`, `Asc`, `Desc`
- `True`, `False`
- `In`, `NotIn`
- `IgnoreCase`
- `First`, `Top`, `Distinct`

## Query Method Examples
```java
// Find by single field
Optional<User> findByUsername(String username);

// Find by multiple fields with AND
Optional<User> findByUsernameAndEmail(String username, String email);

// Find by multiple fields with OR
List<User> findByUsernameOrEmail(String username, String email);

// Case insensitive search
List<User> findByUsernameIgnoreCase(String username);

// Like queries
List<User> findByUsernameContaining(String keyword);
List<User> findByUsernameStartingWith(String prefix);

// Comparisons
List<User> findByAgeLessThan(Integer age);
List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

// Sorting
List<User> findByEnabledTrueOrderByCreatedAtDesc();

// Limiting results
List<User> findTop10ByEnabledTrueOrderByCreatedAtDesc();
Optional<User> findFirstByOrderByCreatedAtDesc();

// Pagination
Page<User> findByEnabled(Boolean enabled, Pageable pageable);
```

## Best Practices
- Return `Optional<T>` for single results to avoid NullPointerException
- Use `Page<T>` for large result sets with pagination
- Use `@Query` for complex queries that can't be expressed with method names
- Use `@EntityGraph` to avoid N+1 query problems
- Use `@Modifying` with `@Query` for UPDATE/DELETE operations
- Always use `@Transactional` with `@Modifying` queries
- Prefer JPQL over native SQL for database independence
- Use named parameters (`:paramName`) for better readability
- Add indexes on frequently queried columns (in entity or migration)

## Performance Tips
- Use projections for read-only queries (return interfaces with only needed fields)
- Use `@EntityGraph` to fetch associations in one query
- Avoid `findAll()` without pagination on large tables
- Consider using database views for complex read queries
- Use batch operations for bulk updates
- Monitor query performance with logging (`show-sql: true` in dev)
