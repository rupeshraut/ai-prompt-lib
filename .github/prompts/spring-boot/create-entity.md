# Create JPA Entity

Create a new JPA entity class following Spring Boot best practices.

## Requirements

The entity should include:
- Proper JPA annotations (@Entity, @Table, @Id, @GeneratedValue)
- All required fields with appropriate data types
- Validation constraints (@NotNull, @Size, @Email, etc.)
- Audit fields: createdAt and updatedAt with @CreationTimestamp and @UpdateTimestamp
- Proper equals() and hashCode() based on business key (not ID)
- toString() method (exclude sensitive fields)
- Relationships with other entities if needed

## Code Style
- Use Java 17 features
- Follow naming conventions: PascalCase for class names
- Use Lombok annotations: @Getter, @Setter, @NoArgsConstructor, @AllArgsConstructor
- Add JavaDoc for the class
- Table name should be plural (e.g., @Table(name = "users"))

## Example Structure
```java
@Entity
@Table(name = "entities")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class EntityName {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // Business fields with validation
    
    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;
    
    @Override
    public boolean equals(Object o) {
        // Implement based on business key
    }
    
    @Override
    public int hashCode() {
        // Implement based on business key
    }
}
```

## Security Considerations
- Never expose sensitive fields (passwords, tokens) in toString()
- Mark password fields with appropriate annotations
- Consider data encryption for sensitive fields
