# Create Audit Logging

Implement comprehensive audit logging for compliance and security. Track all data access, modifications, authentication events, and sensitive operations with detailed context for forensic analysis and regulatory compliance.

## Requirements

Your audit logging implementation should include:

- **User Action Tracking**: Log who did what, when, and where
- **Data Change Tracking**: Audit trail of modifications (before/after values)
- **Authentication Events**: Login, logout, failed attempts, privilege changes
- **Data Access Logging**: Track access to sensitive information
- **Immutable Logs**: Prevent tampering with audit records
- **Structured Logging**: JSON format for easy querying and analysis
- **Retention Policy**: Compliance-driven log retention (e.g., 7 years for financial)
- **Performance**: Minimal impact on application performance
- **Search Capability**: Query audit logs by user, resource, timestamp, action
- **Regulatory Compliance**: GDPR, SOX, HIPAA, PCI DSS support

## Code Style

- **Language**: Java 17 LTS
- **Framework**: Spring Boot, Spring AOP
- **Logging**: SLF4J, Logback
- **Storage**: Database, file-based, or external service
- **Format**: Structured JSON logging

## Template Example

```java
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.Authentication;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;
import jakarta.persistence.*;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;
import lombok.Getter;
import lombok.Setter;
import java.time.LocalDateTime;
import java.util.*;

/**
 * Comprehensive audit logging for compliance and security.
 * Tracks user actions, data modifications, and sensitive operations.
 */
@Slf4j
public class AuditLogging {

    /**
     * Audit log entry entity for persistent storage.
     */
    @Entity
    @Table(name = "audit_logs", indexes = {
            @Index(name = "idx_user_id", columnList = "user_id"),
            @Index(name = "idx_timestamp", columnList = "timestamp"),
            @Index(name = "idx_action", columnList = "action"),
            @Index(name = "idx_resource_type", columnList = "resource_type")
    })
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class AuditLog {
        @Id
        @GeneratedValue(strategy = GenerationType.IDENTITY)
        private Long id;

        @Column(nullable = false)
        private String userId;

        @Column(nullable = false)
        private String action;  // CREATE, READ, UPDATE, DELETE, LOGIN, etc.

        @Column(nullable = false)
        private String resourceType;  // Customer, Order, Invoice, etc.

        @Column
        private String resourceId;

        @Column
        private String status;  // SUCCESS, FAILURE, PARTIAL

        @Column(columnDefinition = "TEXT")
        private String details;  // JSON with additional context

        @Column(columnDefinition = "TEXT")
        private String beforeValue;  // Previous value for modifications

        @Column(columnDefinition = "TEXT")
        private String afterValue;  // New value for modifications

        @Column(nullable = false)
        private String ipAddress;

        @Column
        private String userAgent;

        @Column
        private String sessionId;

        @Column(nullable = false)
        private LocalDateTime timestamp;

        @Column
        private String changeReason;  // Why was this change made

        @Column
        private Boolean sensitive;  // Mark if this involves sensitive data

        @PrePersist
        public void prePersist() {
            this.timestamp = LocalDateTime.now();
        }
    }

    /**
     * Audit log repository for querying audit records.
     */
    @Repository
    public interface AuditLogRepository extends JpaRepository<AuditLog, Long> {
        List<AuditLog> findByUserIdAndActionAndTimestampAfter(
                String userId, String action, LocalDateTime timestamp);

        List<AuditLog> findByResourceTypeAndResourceIdOrderByTimestampDesc(
                String resourceType, String resourceId);

        List<AuditLog> findByUserIdOrderByTimestampDesc(String userId);

        List<AuditLog> findByActionAndSensitiveOrderByTimestampDesc(
                String action, Boolean sensitive);
    }

    /**
     * Audit logging service for recording events.
     */
    @Component
    @Slf4j
    @AllArgsConstructor
    public static class AuditLogger {
        private final AuditLogRepository auditLogRepository;
        private final ObjectMapper objectMapper;

        /**
         * Log user action.
         */
        public void logAction(AuditEvent event) {
            try {
                AuditLog auditLog = new AuditLog();
                auditLog.setUserId(event.getUserId());
                auditLog.setAction(event.getAction());
                auditLog.setResourceType(event.getResourceType());
                auditLog.setResourceId(event.getResourceId());
                auditLog.setStatus(event.getStatus());
                auditLog.setDetails(objectMapper.writeValueAsString(event.getDetails()));
                auditLog.setBeforeValue(event.getBeforeValue());
                auditLog.setAfterValue(event.getAfterValue());
                auditLog.setIpAddress(event.getIpAddress());
                auditLog.setUserAgent(event.getUserAgent());
                auditLog.setSessionId(event.getSessionId());
                auditLog.setChangeReason(event.getChangeReason());
                auditLog.setSensitive(event.getSensitive());

                auditLogRepository.save(auditLog);

                // Also log to structured logging system
                logStructured(event);

            } catch (Exception e) {
                log.error("Failed to record audit log", e);
            }
        }

        /**
         * Log in structured JSON format for external systems.
         */
        private void logStructured(AuditEvent event) {
            Map<String, Object> auditMap = Map.of(
                    "timestamp", System.currentTimeMillis(),
                    "userId", event.getUserId(),
                    "action", event.getAction(),
                    "resourceType", event.getResourceType(),
                    "resourceId", event.getResourceId(),
                    "status", event.getStatus(),
                    "ipAddress", event.getIpAddress(),
                    "sensitive", event.getSensitive()
            );

            try {
                String json = objectMapper.writeValueAsString(auditMap);
                log.info("AUDIT: {}", json);
            } catch (Exception e) {
                log.error("Failed to serialize audit event", e);
            }
        }

        /**
         * Query audit logs by user.
         */
        public List<AuditLog> getUserAuditTrail(String userId) {
            return auditLogRepository.findByUserIdOrderByTimestampDesc(userId);
        }

        /**
         * Query audit logs for specific resource.
         */
        public List<AuditLog> getResourceAuditTrail(String resourceType, String resourceId) {
            return auditLogRepository.findByResourceTypeAndResourceIdOrderByTimestampDesc(
                    resourceType, resourceId);
        }

        /**
         * Search sensitive data access.
         */
        public List<AuditLog> getSensitiveDataAccess() {
            return auditLogRepository.findByActionAndSensitiveOrderByTimestampDesc("READ", true);
        }
    }

    /**
     * Audit event data class.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class AuditEvent {
        private String userId;
        private String action;  // CREATE, READ, UPDATE, DELETE, LOGIN
        private String resourceType;
        private String resourceId;
        private String status;  // SUCCESS, FAILURE
        private Map<String, Object> details;
        private String beforeValue;
        private String afterValue;
        private String ipAddress;
        private String userAgent;
        private String sessionId;
        private String changeReason;
        private Boolean sensitive;
    }

    /**
     * AOP aspect for automatic audit logging of service methods.
     */
    @Aspect
    @Component
    @Slf4j
    @AllArgsConstructor
    public static class AuditAspect {
        private final AuditLogger auditLogger;

        /**
         * Aspect to log service method calls.
         */
        @Around("@annotation(auditLog)")
        public Object auditServiceCall(JoinPoint joinPoint,
                                       Auditable auditLog) throws Throwable {
            String userId = getCurrentUserId();
            String action = auditLog.action();
            String resourceType = auditLog.resourceType();
            Object[] args = joinPoint.getArgs();

            try {
                Object result = joinPoint.proceed();

                AuditEvent event = new AuditEvent();
                event.setUserId(userId);
                event.setAction(action);
                event.setResourceType(resourceType);
                event.setStatus("SUCCESS");
                event.setAfterValue(serializeValue(result));
                event.setIpAddress(getClientIpAddress());
                event.setSessionId(getSessionId());
                event.setSensitive(auditLog.sensitive());

                auditLogger.logAction(event);

                return result;
            } catch (Exception e) {
                AuditEvent event = new AuditEvent();
                event.setUserId(userId);
                event.setAction(action);
                event.setResourceType(resourceType);
                event.setStatus("FAILURE");
                event.setDetails(Map.of("error", e.getMessage()));
                event.setIpAddress(getClientIpAddress());
                event.setSessionId(getSessionId());

                auditLogger.logAction(event);
                throw e;
            }
        }

        private String getCurrentUserId() {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            return auth != null ? auth.getName() : "ANONYMOUS";
        }

        private String getClientIpAddress() {
            try {
                org.springframework.web.context.request.RequestAttributes attrs =
                        org.springframework.web.context.request.RequestContextHolder.getRequestAttributes();
                if (attrs instanceof org.springframework.web.context.request.ServletRequestAttributes) {
                    javax.servlet.http.HttpServletRequest request =
                            ((org.springframework.web.context.request.ServletRequestAttributes) attrs)
                                    .getRequest();
                    return request.getHeader("X-Forwarded-For") != null ?
                            request.getHeader("X-Forwarded-For") :
                            request.getRemoteAddr();
                }
            } catch (Exception e) {
                log.debug("Could not determine client IP", e);
            }
            return "UNKNOWN";
        }

        private String getSessionId() {
            try {
                org.springframework.web.context.request.RequestAttributes attrs =
                        org.springframework.web.context.request.RequestContextHolder.getRequestAttributes();
                if (attrs instanceof org.springframework.web.context.request.ServletRequestAttributes) {
                    javax.servlet.http.HttpSession session =
                            ((org.springframework.web.context.request.ServletRequestAttributes) attrs)
                                    .getRequest().getSession(false);
                    return session != null ? session.getId() : null;
                }
            } catch (Exception e) {
                log.debug("Could not determine session ID", e);
            }
            return null;
        }

        private String serializeValue(Object value) {
            try {
                ObjectMapper mapper = new ObjectMapper();
                return mapper.writeValueAsString(value);
            } catch (Exception e) {
                return value.toString();
            }
        }
    }

    /**
     * Annotation for methods to be audited automatically.
     */
    @java.lang.annotation.Target(java.lang.annotation.ElementType.METHOD)
    @java.lang.annotation.Retention(java.lang.annotation.RetentionPolicy.RUNTIME)
    public @interface Auditable {
        String action();  // CREATE, READ, UPDATE, DELETE

        String resourceType();

        boolean sensitive() default false;
    }

    /**
     * Authentication audit service for login/logout/failed attempts.
     */
    @Component
    @Slf4j
    @AllArgsConstructor
    public static class AuthenticationAuditService {
        private final AuditLogger auditLogger;

        /**
         * Log successful login.
         */
        public void logLoginSuccess(String userId, String ipAddress) {
            AuditEvent event = new AuditEvent();
            event.setUserId(userId);
            event.setAction("LOGIN");
            event.setResourceType("AUTHENTICATION");
            event.setStatus("SUCCESS");
            event.setIpAddress(ipAddress);
            event.setSensitive(false);

            auditLogger.logAction(event);
            log.info("User logged in: {}", userId);
        }

        /**
         * Log failed login attempt.
         */
        public void logLoginFailure(String userId, String reason, String ipAddress) {
            AuditEvent event = new AuditEvent();
            event.setUserId(userId);
            event.setAction("LOGIN");
            event.setResourceType("AUTHENTICATION");
            event.setStatus("FAILURE");
            event.setDetails(Map.of("reason", reason));
            event.setIpAddress(ipAddress);
            event.setSensitive(false);

            auditLogger.logAction(event);
            log.warn("Login failed for user: {} ({})", userId, reason);
        }

        /**
         * Log logout.
         */
        public void logLogout(String userId, String ipAddress) {
            AuditEvent event = new AuditEvent();
            event.setUserId(userId);
            event.setAction("LOGOUT");
            event.setResourceType("AUTHENTICATION");
            event.setStatus("SUCCESS");
            event.setIpAddress(ipAddress);
            event.setSensitive(false);

            auditLogger.logAction(event);
            log.info("User logged out: {}", userId);
        }

        /**
         * Log privilege change (admin action).
         */
        public void logPrivilegeChange(String userId, String adminId, String oldRole, String newRole) {
            AuditEvent event = new AuditEvent();
            event.setUserId(adminId);
            event.setAction("UPDATE_PRIVILEGES");
            event.setResourceType("USER");
            event.setResourceId(userId);
            event.setStatus("SUCCESS");
            event.setBeforeValue(oldRole);
            event.setAfterValue(newRole);
            event.setChangeReason("Administrative privilege change");
            event.setSensitive(true);

            auditLogger.logAction(event);
            log.warn("Privileges changed for {}: {} -> {} by {}", userId, oldRole, newRole, adminId);
        }
    }

    /**
     * Example service with audit logging.
     */
    @org.springframework.stereotype.Service
    @Slf4j
    @AllArgsConstructor
    public static class OrderService {
        private final AuditLogger auditLogger;

        /**
         * Create order with audit logging.
         */
        @Auditable(action = "CREATE", resourceType = "ORDER")
        public Order createOrder(CreateOrderRequest request) {
            Order order = new Order();
            order.setCustomerId(request.getCustomerId());
            order.setAmount(request.getAmount());
            order.setStatus("PENDING");

            // Log order creation
            AuditEvent event = new AuditEvent();
            event.setUserId(getCurrentUserId());
            event.setAction("CREATE");
            event.setResourceType("ORDER");
            event.setResourceId(order.getId());
            event.setStatus("SUCCESS");
            event.setAfterValue(serializeOrder(order));
            event.setSensitive(false);

            auditLogger.logAction(event);
            return order;
        }

        /**
         * Update order with before/after audit trail.
         */
        @Auditable(action = "UPDATE", resourceType = "ORDER")
        public Order updateOrder(String orderId, UpdateOrderRequest request) {
            Order order = new Order();  // Fetch existing order
            String beforeValue = serializeOrder(order);

            // Make changes
            order.setStatus(request.getStatus());

            String afterValue = serializeOrder(order);

            AuditEvent event = new AuditEvent();
            event.setUserId(getCurrentUserId());
            event.setAction("UPDATE");
            event.setResourceType("ORDER");
            event.setResourceId(orderId);
            event.setStatus("SUCCESS");
            event.setBeforeValue(beforeValue);
            event.setAfterValue(afterValue);
            event.setChangeReason("Order status update");

            auditLogger.logAction(event);
            return order;
        }

        private String getCurrentUserId() {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            return auth != null ? auth.getName() : "SYSTEM";
        }

        private String serializeOrder(Order order) {
            try {
                return new ObjectMapper().writeValueAsString(order);
            } catch (Exception e) {
                return order.toString();
            }
        }
    }

    /**
     * Supporting classes.
     */
    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class Order {
        private String id;
        private String customerId;
        private Double amount;
        private String status;
    }

    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class CreateOrderRequest {
        private String customerId;
        private Double amount;
    }

    @NoArgsConstructor
    @AllArgsConstructor
    @Getter
    @Setter
    public static class UpdateOrderRequest {
        private String status;
    }

    /**
     * Best practices for audit logging.
     */
    public static class BestPractices {
        /**
         * 1. Log immutably - audit logs must not be modifiable
         *    - Store in append-only database
         *    - Use blockchain or WORM storage
         *    - Protect with digital signatures or hashing
         */

        /**
         * 2. Log all sensitive operations
         *    - Data access and modifications
         *    - Authentication events
         *    - Privilege changes
         *    - Configuration changes
         */

        /**
         * 3. Include complete context
         *    - Who (user ID, service account)
         *    - What (action, resource, changes)
         *    - When (timestamp, timezone)
         *    - Where (IP address, location)
         *    - Why (change reason, justification)
         */

        /**
         * 4. Use structured logging (JSON)
         *    - Machine-parseable format
         *    - Easier to search and analyze
         *    - Suitable for SIEM integration
         */

        /**
         * 5. Protect audit logs
         *    - Restrict access to audit logs
         *    - Separate log storage from application
         *    - Implement log rotation and archival
         */

        /**
         * 6. Maintain appropriate retention
         *    - Financial: 7 years
         *    - Healthcare: 10 years
         *    - GDPR: Minimum necessary period
         *    - Consider local regulations
         */

        /**
         * 7. Enable real-time monitoring
         *    - Alert on suspicious patterns
         *    - Detect privilege escalation attempts
         *    - Monitor failed authentication attempts
         *    - Watch for unusual data access
         */

        /**
         * 8. Test audit logging thoroughly
         *    - Verify events are recorded
         *    - Test with various user roles
         *    - Verify data is complete and accurate
         */
    }
}
```

## Audit Log Schema

```sql
CREATE TABLE audit_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id VARCHAR(100) NOT NULL,
    action VARCHAR(50) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id VARCHAR(255),
    status VARCHAR(20),
    details LONGTEXT,
    before_value LONGTEXT,
    after_value LONGTEXT,
    ip_address VARCHAR(45),
    user_agent TEXT,
    session_id VARCHAR(255),
    timestamp DATETIME NOT NULL,
    change_reason VARCHAR(500),
    sensitive BOOLEAN,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_timestamp (timestamp),
    INDEX idx_action (action),
    INDEX idx_resource_type (resource_type)
);
```

## Compliance Mapping

| Regulation | Requirements | Audit Logging Implementation |
|------------|--------------|------------------------------|
| **GDPR** | Data access logs, 30-day breach reporting | Access logs, sensitive data tracking |
| **SOX** | Financial transaction audit trail | Detailed before/after values |
| **HIPAA** | Health information access tracking | Patient data access audit |
| **PCI DSS** | Payment card data access logging | Sensitive flag + detailed logs |

## Related Documentation

- [OWASP Audit Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html)
- [Spring Security Audit Events](https://docs.spring.io/spring-security/reference/servlet/authentication/events.html)
- [Structured Logging with SLF4J](https://www.slf4j.org/)
- [ELK Stack for Log Analysis](https://www.elastic.co/what-is/elk-stack)
```
