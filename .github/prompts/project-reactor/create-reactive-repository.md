# Create Reactive Repository

Create a reactive data access layer using Spring Data R2DBC for non-blocking database operations. Reactive repositories enable asynchronous database queries without thread blocking.

## Requirements

Your reactive repository should include:

- **Interface Definition**: Extend `ReactiveCrudRepository<T, ID>` or `ReactiveSortingRepository<T, ID>`
- **Custom Query Methods**: Define custom queries using `@Query` with parameter binding
- **Return Types**: Use `Mono<T>`, `Mono<Boolean>`, `Flux<T>` for async results
- **Pagination**: Support pagination with `Pageable` parameter
- **Sorting**: Support sorting with `Sort` parameter
- **Named Parameters**: Use `@Param` for query parameter binding
- **Modifying Queries**: Use `@Modifying` for update/delete operations
- **Error Handling**: Handle database errors appropriately
- **Implementation**: Example custom implementation if needed (rare in modern Spring)
- **Documentation**: JavaDoc comments for each method

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Data R2DBC, Project Reactor
- **Database**: PostgreSQL, MySQL, H2 (R2DBC drivers)
- **Naming Conventions**:
  - Interface name follows `<Entity>Repository` pattern
  - Query methods use domain-driven naming (e.g., `findByUsername()`, `findByStatusOrderByCreatedAtDesc()`)
  - Update methods use `update*` prefix
- **Query Syntax**: Native SQL or derived query methods

## Template Example

```java
import org.springframework.data.r2dbc.repository.Query;
import org.springframework.data.r2dbc.repository.Modifying;
import org.springframework.data.repository.query.Param;
import org.springframework.data.r2dbc.repository.R2dbcRepository;
import org.springframework.stereotype.Repository;
import reactor.core.publisher.Mono;
import reactor.core.publisher.Flux;
import java.time.LocalDateTime;
import java.util.List;

/**
 * Reactive repository for Order entity using Spring Data R2DBC.
 * Provides non-blocking database access for order operations.
 *
 * Database: PostgreSQL with R2DBC driver
 * Table: orders
 */
@Repository
public interface OrderRepository extends R2dbcRepository<Order, Long> {

    /**
     * Finds an order by ID (reactive version of findById).
     */
    @Override
    Mono<Order> findById(Long id);

    /**
     * Finds all orders for a specific user.
     */
    Flux<Order> findByUserId(Long userId);

    /**
     * Finds orders by status with optional sorting.
     */
    Flux<Order> findByStatus(OrderStatus status);

    /**
     * Finds orders created after a specific date.
     */
    Flux<Order> findByCreatedAtAfter(LocalDateTime createdAt);

    /**
     * Finds orders within an amount range.
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE o.total >= :minAmount AND o.total <= :maxAmount
            ORDER BY o.created_at DESC
            """)
    Flux<Order> findByAmountRange(
            @Param("minAmount") Double minAmount,
            @Param("maxAmount") Double maxAmount);

    /**
     * Finds active orders for a user (not shipped or delivered).
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE o.user_id = :userId
            AND o.status NOT IN ('SHIPPED', 'DELIVERED', 'CANCELLED')
            ORDER BY o.created_at DESC
            """)
    Flux<Order> findActiveOrdersByUserId(@Param("userId") Long userId);

    /**
     * Counts orders for a specific user.
     */
    Mono<Long> countByUserId(Long userId);

    /**
     * Counts orders by status.
     */
    Mono<Long> countByStatus(OrderStatus status);

    /**
     * Checks if an order exists.
     */
    Mono<Boolean> existsByIdAndUserId(Long id, Long userId);

    /**
     * Finds the most recent order for a user.
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE o.user_id = :userId
            ORDER BY o.created_at DESC
            LIMIT 1
            """)
    Mono<Order> findLatestOrderByUserId(@Param("userId") Long userId);

    /**
     * Finds orders with pending status for processing.
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE o.status = 'PENDING'
            AND o.created_at < :cutoffTime
            ORDER BY o.created_at ASC
            LIMIT :limit
            """)
    Flux<Order> findPendingOrdersOlderThan(
            @Param("cutoffTime") LocalDateTime cutoffTime,
            @Param("limit") int limit);

    /**
     * Finds orders grouped by user ID (for aggregation).
     */
    @Query("""
            SELECT o.user_id, COUNT(*) as order_count, SUM(o.total) as total_amount
            FROM orders o
            GROUP BY o.user_id
            ORDER BY total_amount DESC
            """)
    Flux<UserOrderStats> findUserOrderStatistics();

    /**
     * Updates order status.
     */
    @Modifying
    @Query("""
            UPDATE orders
            SET status = :status, updated_at = :updatedAt
            WHERE id = :orderId
            """)
    Mono<Integer> updateOrderStatus(
            @Param("orderId") Long orderId,
            @Param("status") String status,
            @Param("updatedAt") LocalDateTime updatedAt);

    /**
     * Marks orders as shipped.
     */
    @Modifying
    @Query("""
            UPDATE orders
            SET status = 'SHIPPED', shipped_at = :shippedAt, updated_at = :updatedAt
            WHERE id = :orderId AND status = 'CONFIRMED'
            """)
    Mono<Integer> markAsShipped(
            @Param("orderId") Long orderId,
            @Param("shippedAt") LocalDateTime shippedAt,
            @Param("updatedAt") LocalDateTime updatedAt);

    /**
     * Cancels multiple orders.
     */
    @Modifying
    @Query("""
            UPDATE orders
            SET status = 'CANCELLED', cancelled_at = :cancelledAt, updated_at = :updatedAt
            WHERE user_id = :userId AND status = 'PENDING'
            """)
    Mono<Integer> cancelPendingOrdersByUser(
            @Param("userId") Long userId,
            @Param("cancelledAt") LocalDateTime cancelledAt,
            @Param("updatedAt") LocalDateTime updatedAt);

    /**
     * Deletes old cancelled orders (data cleanup).
     */
    @Modifying
    @Query("""
            DELETE FROM orders
            WHERE status = 'CANCELLED' AND cancelled_at < :cutoffDate
            """)
    Mono<Integer> deleteOldCancelledOrders(@Param("cutoffDate") LocalDateTime cutoffDate);

    /**
     * Finds orders by custom criteria using SQL.
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE 1=1
            AND (:userId IS NULL OR o.user_id = :userId)
            AND (:status IS NULL OR o.status = :status)
            AND (:minAmount IS NULL OR o.total >= :minAmount)
            AND (:maxAmount IS NULL OR o.total <= :maxAmount)
            ORDER BY o.created_at DESC
            """)
    Flux<Order> findByDynamicCriteria(
            @Param("userId") Long userId,
            @Param("status") String status,
            @Param("minAmount") Double minAmount,
            @Param("maxAmount") Double maxAmount);

    /**
     * Finds orders using text search on notes field.
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE o.notes ILIKE :searchTerm
            ORDER BY o.created_at DESC
            """)
    Flux<Order> searchByNotes(@Param("searchTerm") String searchTerm);

    /**
     * Finds top N orders by amount.
     */
    @Query("""
            SELECT o.* FROM orders o
            ORDER BY o.total DESC
            LIMIT :limit
            """)
    Flux<Order> findTopOrdersByAmount(@Param("limit") int limit);

    /**
     * Finds orders created on a specific date.
     */
    @Query("""
            SELECT o.* FROM orders o
            WHERE DATE(o.created_at) = :date
            ORDER BY o.created_at DESC
            """)
    Flux<Order> findOrdersByCreatedDate(@Param("date") java.time.LocalDate date);

    /**
     * Complex query: Find users with total orders exceeding threshold.
     */
    @Query("""
            SELECT DISTINCT o.user_id FROM orders o
            GROUP BY o.user_id
            HAVING SUM(o.total) > :threshold
            ORDER BY SUM(o.total) DESC
            """)
    Flux<Long> findUserIdsWithTotalOrdersExceeding(@Param("threshold") Double threshold);
}

/**
 * Projection for user order statistics.
 */
public interface UserOrderStats {
    Long getUserId();
    Integer getOrderCount();
    Double getTotalAmount();
}

/**
 * Example domain model for Order.
 */
public class Order {
    private Long id;
    private Long userId;
    private Double total;
    private OrderStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private LocalDateTime shippedAt;
    private LocalDateTime cancelledAt;
    private String notes;

    // Constructors, getters, setters
    public Order() {}

    public Order(Long userId, Double total, OrderStatus status) {
        this.userId = userId;
        this.total = total;
        this.status = status;
        this.createdAt = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public Long getUserId() { return userId; }
    public void setUserId(Long userId) { this.userId = userId; }

    public Double getTotal() { return total; }
    public void setTotal(Double total) { this.total = total; }

    public OrderStatus getStatus() { return status; }
    public void setStatus(OrderStatus status) { this.status = status; }

    public LocalDateTime getCreatedAt() { return createdAt; }
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }

    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }

    public LocalDateTime getShippedAt() { return shippedAt; }
    public void setShippedAt(LocalDateTime shippedAt) { this.shippedAt = shippedAt; }

    public LocalDateTime getCancelledAt() { return cancelledAt; }
    public void setCancelledAt(LocalDateTime cancelledAt) { this.cancelledAt = cancelledAt; }

    public String getNotes() { return notes; }
    public void setNotes(String notes) { this.notes = notes; }
}

/**
 * Order status enum.
 */
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

## Key Patterns

### 1. Derived Query Methods
```java
// Spring automatically derives these queries
Flux<Order> findByUserId(Long userId);
Mono<Order> findByIdAndUserId(Long id, Long userId);
Flux<Order> findByStatusOrderByCreatedAtDesc(OrderStatus status);
```

### 2. Custom SQL Queries
```java
@Query("SELECT o.* FROM orders o WHERE o.user_id = :userId")
Flux<Order> findByUserId(@Param("userId") Long userId);
```

### 3. Update Operations
```java
@Modifying
@Query("UPDATE orders SET status = :status WHERE id = :id")
Mono<Integer> updateStatus(@Param("id") Long id, @Param("status") String status);
```

### 4. Projections
```java
@Query("SELECT o.user_id, COUNT(*) FROM orders o GROUP BY o.user_id")
Flux<UserStats> getUserStats();
```

## Testing

```java
@DataR2dbcTest
@ExtendWith(SpringExtension.class)
class OrderRepositoryTest {

    @Autowired
    private OrderRepository repository;

    @Test
    void testFindByUserId() {
        StepVerifier.create(repository.findByUserId(1L))
            .expectNextCount(1)
            .expectComplete()
            .verify(Duration.ofSeconds(5));
    }
}
```

## Related Documentation

- [Spring Data R2DBC](https://spring.io/projects/spring-data-r2dbc)
- [R2DBC Specification](https://r2dbc.io/)
- [Reactive SQL Access](https://docs.spring.io/spring-data/r2dbc/reference/)
