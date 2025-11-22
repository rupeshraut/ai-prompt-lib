# Practical Prompt Chaining Example: User Management API

This document demonstrates a **real, executable prompt chain** for building a complete User Management API feature using Java 17 Spring Boot.

## Project Context

**Goal**: Build a complete User Management feature with:
- User registration and authentication
- CRUD operations
- Role-based access control
- Proper validation and error handling
- Comprehensive testing

**Tech Stack**: Java 17, Spring Boot 3.x, Spring Data JPA, Spring Security, PostgreSQL

---

## The Complete Chain: 11 Steps

### Overview of Steps
```
1. Requirements Analysis       → Define what we need
2. Entity Design               → Create User entity
3. Repository Design           → Create UserRepository
4. DTO Creation                → Create request/response DTOs
5. Custom Exceptions           → Create domain exceptions
6. Mapper Creation             → Entity ↔ DTO conversion
7. Service Layer               → Business logic implementation
8. Controller Layer            → REST endpoints
9. Exception Handler           → Global error handling
10. Unit Tests                 → Service layer tests
11. Integration Tests          → API endpoint tests
```

---

## STEP 1: Requirements Analysis

### Prompt
```
I'm building a User Management API with Spring Boot. Analyze the requirements:

Features needed:
- User registration (public endpoint)
- User login (JWT-based authentication)
- Get user by ID
- Get all users (paginated, admin only)
- Update user profile
- Delete user (soft delete, admin only)
- Search users by keyword

Security:
- Passwords must be hashed (BCrypt)
- JWT tokens for authentication
- Role-based access (USER, ADMIN)

For each feature, identify:
1. Required entities and fields
2. Validation rules
3. Business rules
4. Security requirements
5. HTTP endpoints (method, path, status codes)
```

### Expected Output
```
=== USER MANAGEMENT API REQUIREMENTS ===

ENTITIES:
1. User
   - id: Long (Primary Key, auto-generated)
   - username: String (unique, required, 3-50 chars)
   - email: String (unique, required, valid email format)
   - password: String (required, min 8 chars, BCrypt hashed)
   - firstName: String (optional, max 50 chars)
   - lastName: String (optional, max 50 chars)
   - enabled: Boolean (default true, for soft delete)
   - roles: Set<Role> (Many-to-Many)
   - createdAt: LocalDateTime (auto-generated)
   - updatedAt: LocalDateTime (auto-updated)

2. Role
   - id: Long (Primary Key)
   - name: String (USER, ADMIN)
   - users: Set<User> (Many-to-Many)

VALIDATION RULES:
- Username: required, unique, 3-50 chars, alphanumeric
- Email: required, unique, valid email format
- Password: required, min 8 chars (on registration)
- Roles: at least one role required

BUSINESS RULES:
- New users get USER role by default
- Cannot delete the last ADMIN user
- Users can only update their own profile (unless ADMIN)
- Soft delete: set enabled=false instead of actual deletion
- Password must be hashed before storage

REST ENDPOINTS:
1. POST   /api/v1/auth/register          - Register user (201, 400, 409)
2. POST   /api/v1/auth/login             - Login (200, 401)
3. GET    /api/v1/users/{id}             - Get user (200, 404) [Authenticated]
4. GET    /api/v1/users                  - List users (200) [Admin only, paginated]
5. PUT    /api/v1/users/{id}             - Update user (200, 404, 403) [Owner or Admin]
6. DELETE /api/v1/users/{id}             - Delete user (204, 404, 403) [Admin only]
7. GET    /api/v1/users/search?q=keyword - Search users (200) [Authenticated]
```

---

## STEP 2: Entity Design

### Prompt
```
@workspace Use .github/prompts/create-entity.md

Based on this requirements analysis:
[paste the output from Step 1 - User entity details]

Create the User entity with:
- All fields identified in requirements
- Proper JPA annotations (@Entity, @Table, @Column)
- Validation constraints
- Many-to-Many relationship with Role entity
- equals() and hashCode() based on username (business key)
- toString() that excludes password
- Audit fields with @CreationTimestamp and @UpdateTimestamp
```

### Expected Output
```java
package com.example.userapi.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.*;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

/**
 * User entity representing a system user.
 */
@Entity
@Table(name = "users", uniqueConstraints = {
    @UniqueConstraint(columnNames = "username"),
    @UniqueConstraint(columnNames = "email")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Column(nullable = false)
    private String password;

    @Size(max = 50, message = "First name cannot exceed 50 characters")
    @Column(length = 50)
    private String firstName;

    @Size(max = 50, message = "Last name cannot exceed 50 characters")
    @Column(length = 50)
    private String lastName;

    @Column(nullable = false)
    private Boolean enabled = true;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return Objects.equals(username, user.username);
    }

    @Override
    public int hashCode() {
        return Objects.hash(username);
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", username='" + username + '\'' +
                ", email='" + email + '\'' +
                ", firstName='" + firstName + '\'' +
                ", lastName='" + lastName + '\'' +
                ", enabled=" + enabled +
                ", createdAt=" + createdAt +
                ", updatedAt=" + updatedAt +
                '}';
    }
}
```

---

## STEP 3: Repository Design

### Prompt
```
@workspace Use .github/prompts/create-repository.md

For this User entity:
[paste the User entity from Step 2]

Create UserRepository with these methods:
1. findByUsername(String username) - for login
2. findByEmail(String email) - for duplicate check
3. existsByUsername(String username) - for registration validation
4. existsByEmail(String email) - for registration validation
5. findByEnabledTrue(Pageable pageable) - for listing active users
6. searchByKeyword(String keyword, Pageable pageable) - search in username, email, firstName, lastName

Use @Query for the search method with case-insensitive LIKE.
```

### Expected Output
```java
package com.example.userapi.repository;

import com.example.userapi.entity.User;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.Optional;

/**
 * Repository interface for User entity.
 */
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    /**
     * Find user by username.
     *
     * @param username the username
     * @return optional containing user if found
     */
    Optional<User> findByUsername(String username);

    /**
     * Find user by email.
     *
     * @param email the email address
     * @return optional containing user if found
     */
    Optional<User> findByEmail(String email);

    /**
     * Check if user exists by username.
     *
     * @param username the username
     * @return true if exists, false otherwise
     */
    boolean existsByUsername(String username);

    /**
     * Check if user exists by email.
     *
     * @param email the email address
     * @return true if exists, false otherwise
     */
    boolean existsByEmail(String email);

    /**
     * Find all enabled users with pagination.
     *
     * @param pageable pagination parameters
     * @return page of enabled users
     */
    Page<User> findByEnabledTrue(Pageable pageable);

    /**
     * Search users by keyword in username, email, first name, or last name.
     *
     * @param keyword the search keyword
     * @param pageable pagination parameters
     * @return page of matching users
     */
    @Query("SELECT u FROM User u WHERE " +
           "LOWER(u.username) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(u.email) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(u.firstName) LIKE LOWER(CONCAT('%', :keyword, '%')) OR " +
           "LOWER(u.lastName) LIKE LOWER(CONCAT('%', :keyword, '%'))")
    Page<User> searchByKeyword(@Param("keyword") String keyword, Pageable pageable);
}
```

---

## STEP 4: DTO Creation

### Prompt
```
@workspace Use .github/prompts/create-dto.md

Based on the User entity and requirements from Step 1:
[paste User entity fields]

Create these DTOs using Java 17 records:

1. UserRegistrationDto - for POST /api/v1/auth/register
   Fields: username, email, password, firstName, lastName
   Validation: All required except firstName/lastName

2. UserLoginDto - for POST /api/v1/auth/login
   Fields: username, password
   Validation: Both required

3. UserResponseDto - for API responses
   Fields: id, username, email, firstName, lastName, enabled, createdAt, updatedAt
   (Exclude password!)

4. UserUpdateDto - for PUT /api/v1/users/{id}
   Fields: email, firstName, lastName
   (Cannot update username, password handled separately)

5. UserSummaryDto - for list views and search results
   Fields: id, username, email, firstName, lastName

Include appropriate validation annotations and custom error messages.
```

### Expected Output
```java
package com.example.userapi.dto;

import jakarta.validation.constraints.*;
import java.time.LocalDateTime;

/**
 * DTO for user registration.
 */
public record UserRegistrationDto(
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be between 3 and 50 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username must be alphanumeric")
    String username,

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    String password,

    @Size(max = 50, message = "First name cannot exceed 50 characters")
    String firstName,

    @Size(max = 50, message = "Last name cannot exceed 50 characters")
    String lastName
) {}

/**
 * DTO for user login.
 */
public record UserLoginDto(
    @NotBlank(message = "Username is required")
    String username,

    @NotBlank(message = "Password is required")
    String password
) {}

/**
 * DTO for user response (full details).
 */
public record UserResponseDto(
    Long id,
    String username,
    String email,
    String firstName,
    String lastName,
    Boolean enabled,
    LocalDateTime createdAt,
    LocalDateTime updatedAt
) {}

/**
 * DTO for user update.
 */
public record UserUpdateDto(
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    String email,

    @Size(max = 50, message = "First name cannot exceed 50 characters")
    String firstName,

    @Size(max = 50, message = "Last name cannot exceed 50 characters")
    String lastName
) {}

/**
 * DTO for user summary (list views).
 */
public record UserSummaryDto(
    Long id,
    String username,
    String email,
    String firstName,
    String lastName
) {}
```

---

## STEP 5: Custom Exceptions

### Prompt
```
Create custom exception classes for the User API:

1. UserNotFoundException
   - Message: "User not found with id: {id}" or "User not found with username: {username}"
   - Should extend RuntimeException
   
2. DuplicateUserException
   - Message: "User already exists with username: {username}" or "User already exists with email: {email}"
   - Should extend RuntimeException
   
3. InvalidCredentialsException
   - Message: "Invalid username or password"
   - Should extend RuntimeException

Each exception should:
- Have constructors for message and message+cause
- Be in package com.example.userapi.exception
```

### Expected Output
```java
package com.example.userapi.exception;

/**
 * Exception thrown when a user is not found.
 */
public class UserNotFoundException extends RuntimeException {
    
    public UserNotFoundException(String message) {
        super(message);
    }
    
    public UserNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
    
    public static UserNotFoundException byId(Long id) {
        return new UserNotFoundException("User not found with id: " + id);
    }
    
    public static UserNotFoundException byUsername(String username) {
        return new UserNotFoundException("User not found with username: " + username);
    }
}

/**
 * Exception thrown when attempting to create a duplicate user.
 */
public class DuplicateUserException extends RuntimeException {
    
    public DuplicateUserException(String message) {
        super(message);
    }
    
    public DuplicateUserException(String message, Throwable cause) {
        super(message, cause);
    }
    
    public static DuplicateUserException byUsername(String username) {
        return new DuplicateUserException("User already exists with username: " + username);
    }
    
    public static DuplicateUserException byEmail(String email) {
        return new DuplicateUserException("User already exists with email: " + email);
    }
}

/**
 * Exception thrown for invalid login credentials.
 */
public class InvalidCredentialsException extends RuntimeException {
    
    public InvalidCredentialsException() {
        super("Invalid username or password");
    }
    
    public InvalidCredentialsException(String message) {
        super(message);
    }
    
    public InvalidCredentialsException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## STEP 6: Mapper Creation

### Prompt
```
Create a UserMapper class to convert between User entity and DTOs.

Context:
- User entity: [paste User entity]
- DTOs: [paste UserRegistrationDto, UserResponseDto, UserUpdateDto, UserSummaryDto]

Create a mapper with these methods:
1. toEntity(UserRegistrationDto dto) -> User
   (Don't set password here, will be hashed in service)
   
2. toResponseDto(User user) -> UserResponseDto
   
3. toSummaryDto(User user) -> UserSummaryDto
   
4. updateEntity(User user, UserUpdateDto dto) -> void
   (Update existing user with DTO data)

Use MapStruct or manual mapping. If manual, use a @Component class.
```

### Expected Output
```java
package com.example.userapi.mapper;

import com.example.userapi.dto.*;
import com.example.userapi.entity.User;
import org.springframework.stereotype.Component;

/**
 * Mapper for converting between User entity and DTOs.
 */
@Component
public class UserMapper {

    /**
     * Convert registration DTO to User entity.
     * Note: Password must be hashed separately before saving.
     *
     * @param dto the registration DTO
     * @return the user entity
     */
    public User toEntity(UserRegistrationDto dto) {
        User user = new User();
        user.setUsername(dto.username());
        user.setEmail(dto.email());
        user.setPassword(dto.password()); // Will be hashed in service
        user.setFirstName(dto.firstName());
        user.setLastName(dto.lastName());
        user.setEnabled(true);
        return user;
    }

    /**
     * Convert User entity to response DTO.
     *
     * @param user the user entity
     * @return the response DTO
     */
    public UserResponseDto toResponseDto(User user) {
        return new UserResponseDto(
            user.getId(),
            user.getUsername(),
            user.getEmail(),
            user.getFirstName(),
            user.getLastName(),
            user.getEnabled(),
            user.getCreatedAt(),
            user.getUpdatedAt()
        );
    }

    /**
     * Convert User entity to summary DTO.
     *
     * @param user the user entity
     * @return the summary DTO
     */
    public UserSummaryDto toSummaryDto(User user) {
        return new UserSummaryDto(
            user.getId(),
            user.getUsername(),
            user.getEmail(),
            user.getFirstName(),
            user.getLastName()
        );
    }

    /**
     * Update existing user entity with data from update DTO.
     *
     * @param user the user entity to update
     * @param dto the update DTO
     */
    public void updateEntity(User user, UserUpdateDto dto) {
        user.setEmail(dto.email());
        user.setFirstName(dto.firstName());
        user.setLastName(dto.lastName());
    }
}
```

---

## STEP 7: Service Layer

### Prompt
```
@workspace Use .github/prompts/create-service.md

Create UserService with the following methods:

Context:
- UserRepository: [paste repository methods]
- DTOs: [paste DTO names]
- Exceptions: [paste exception names]
- UserMapper: [paste mapper methods]

Methods needed:
1. register(UserRegistrationDto) -> UserResponseDto
   - Check if username exists (throw DuplicateUserException.byUsername)
   - Check if email exists (throw DuplicateUserException.byEmail)
   - Hash password using BCrypt
   - Assign default USER role
   - Save and return response DTO
   
2. findById(Long id) -> UserResponseDto
   - Find by ID or throw UserNotFoundException.byId
   
3. findByUsername(String username) -> UserResponseDto
   - Find by username or throw UserNotFoundException.byUsername
   
4. findAll(Pageable) -> Page<UserSummaryDto>
   - Return all enabled users with pagination
   - Map to summary DTOs
   
5. update(Long id, UserUpdateDto) -> UserResponseDto
   - Find user by ID
   - Check if new email is already taken by another user
   - Update and save
   
6. delete(Long id) -> void
   - Find user by ID
   - Soft delete (set enabled = false)
   
7. search(String keyword, Pageable) -> Page<UserSummaryDto>
   - Search using repository method
   - Map to summary DTOs

Use:
- @Service, @RequiredArgsConstructor, @Slf4j
- @Transactional for write operations
- PasswordEncoder for hashing
- Proper logging at INFO and DEBUG levels
```

### Expected Output
```java
package com.example.userapi.service;

import com.example.userapi.dto.*;
import com.example.userapi.entity.User;
import com.example.userapi.exception.*;
import com.example.userapi.mapper.UserMapper;
import com.example.userapi.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * Service for managing users.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;

    /**
     * Register a new user.
     *
     * @param dto the registration DTO
     * @return the created user response DTO
     * @throws DuplicateUserException if username or email already exists
     */
    @Transactional
    public UserResponseDto register(UserRegistrationDto dto) {
        log.info("Registering new user with username: {}", dto.username());

        // Check for duplicate username
        if (userRepository.existsByUsername(dto.username())) {
            log.warn("Registration failed: Username already exists: {}", dto.username());
            throw DuplicateUserException.byUsername(dto.username());
        }

        // Check for duplicate email
        if (userRepository.existsByEmail(dto.email())) {
            log.warn("Registration failed: Email already exists: {}", dto.email());
            throw DuplicateUserException.byEmail(dto.email());
        }

        // Create user entity
        User user = userMapper.toEntity(dto);
        
        // Hash password
        String hashedPassword = passwordEncoder.encode(dto.password());
        user.setPassword(hashedPassword);

        // TODO: Assign default USER role (requires Role entity and RoleRepository)
        // Role userRole = roleRepository.findByName("USER").orElseThrow();
        // user.getRoles().add(userRole);

        // Save user
        User savedUser = userRepository.save(user);
        log.debug("User registered successfully with id: {}", savedUser.getId());

        return userMapper.toResponseDto(savedUser);
    }

    /**
     * Find user by ID.
     *
     * @param id the user ID
     * @return the user response DTO
     * @throws UserNotFoundException if user not found
     */
    public UserResponseDto findById(Long id) {
        log.debug("Finding user with id: {}", id);

        User user = userRepository.findById(id)
            .orElseThrow(() -> UserNotFoundException.byId(id));

        return userMapper.toResponseDto(user);
    }

    /**
     * Find user by username.
     *
     * @param username the username
     * @return the user response DTO
     * @throws UserNotFoundException if user not found
     */
    public UserResponseDto findByUsername(String username) {
        log.debug("Finding user with username: {}", username);

        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> UserNotFoundException.byUsername(username));

        return userMapper.toResponseDto(user);
    }

    /**
     * Find all enabled users with pagination.
     *
     * @param pageable pagination parameters
     * @return page of user summary DTOs
     */
    public Page<UserSummaryDto> findAll(Pageable pageable) {
        log.debug("Finding all enabled users with pagination");

        return userRepository.findByEnabledTrue(pageable)
            .map(userMapper::toSummaryDto);
    }

    /**
     * Update user profile.
     *
     * @param id the user ID
     * @param dto the update DTO
     * @return the updated user response DTO
     * @throws UserNotFoundException if user not found
     * @throws DuplicateUserException if new email is already taken
     */
    @Transactional
    public UserResponseDto update(Long id, UserUpdateDto dto) {
        log.info("Updating user with id: {}", id);

        // Find existing user
        User user = userRepository.findById(id)
            .orElseThrow(() -> UserNotFoundException.byId(id));

        // Check if new email is already taken by another user
        if (!user.getEmail().equals(dto.email()) && 
            userRepository.existsByEmail(dto.email())) {
            log.warn("Update failed: Email already exists: {}", dto.email());
            throw DuplicateUserException.byEmail(dto.email());
        }

        // Update user
        userMapper.updateEntity(user, dto);
        User updatedUser = userRepository.save(user);

        log.debug("User updated successfully");
        return userMapper.toResponseDto(updatedUser);
    }

    /**
     * Delete user (soft delete).
     *
     * @param id the user ID
     * @throws UserNotFoundException if user not found
     */
    @Transactional
    public void delete(Long id) {
        log.info("Deleting user with id: {}", id);

        User user = userRepository.findById(id)
            .orElseThrow(() -> UserNotFoundException.byId(id));

        // Soft delete
        user.setEnabled(false);
        userRepository.save(user);

        log.debug("User deleted (soft) successfully");
    }

    /**
     * Search users by keyword.
     *
     * @param keyword the search keyword
     * @param pageable pagination parameters
     * @return page of user summary DTOs
     */
    public Page<UserSummaryDto> search(String keyword, Pageable pageable) {
        log.debug("Searching users with keyword: {}", keyword);

        return userRepository.searchByKeyword(keyword, pageable)
            .map(userMapper::toSummaryDto);
    }
}
```

---

## STEP 8: Controller Layer

### Prompt
```
@workspace Use .github/prompts/create-controller.md

Create UserController with these endpoints:

Context:
- UserService methods: [paste service method signatures]
- DTOs: UserRegistrationDto, UserUpdateDto, UserResponseDto, UserSummaryDto

Endpoints:
1. POST /api/v1/users/register
   - Public endpoint
   - Request body: UserRegistrationDto
   - Response: 201 CREATED with UserResponseDto
   
2. GET /api/v1/users/{id}
   - Requires authentication
   - Path variable: id
   - Response: 200 OK with UserResponseDto
   
3. GET /api/v1/users
   - Requires ADMIN role
   - Query params: page, size, sort
   - Response: 200 OK with Page<UserSummaryDto>
   
4. PUT /api/v1/users/{id}
   - Requires authentication (owner or admin)
   - Path variable: id
   - Request body: UserUpdateDto
   - Response: 200 OK with UserResponseDto
   
5. DELETE /api/v1/users/{id}
   - Requires ADMIN role
   - Path variable: id
   - Response: 204 NO CONTENT
   
6. GET /api/v1/users/search
   - Requires authentication
   - Query param: q (keyword)
   - Query params: page, size, sort
   - Response: 200 OK with Page<UserSummaryDto>

Use:
- @RestController, @RequestMapping, @RequiredArgsConstructor
- @PreAuthorize for role-based security
- @Valid for validation
- @PageableDefault for pagination
- Swagger/OpenAPI annotations
```

### Expected Output
```java
package com.example.userapi.controller;

import com.example.userapi.dto.*;
import com.example.userapi.service.UserService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import jakarta.validation.constraints.Positive;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.web.PageableDefault;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

/**
 * REST controller for user management.
 */
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
@Tag(name = "User Management", description = "APIs for managing users")
public class UserController {

    private final UserService userService;

    /**
     * Register a new user.
     */
    @PostMapping("/register")
    @Operation(summary = "Register new user", description = "Create a new user account (public endpoint)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "User registered successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid input data"),
        @ApiResponse(responseCode = "409", description = "Username or email already exists")
    })
    public ResponseEntity<UserResponseDto> register(
            @Valid @RequestBody UserRegistrationDto dto) {
        UserResponseDto user = userService.register(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }

    /**
     * Get user by ID.
     */
    @GetMapping("/{id}")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Get user by ID", description = "Retrieve user details by ID")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "User found"),
        @ApiResponse(responseCode = "404", description = "User not found"),
        @ApiResponse(responseCode = "401", description = "Unauthorized")
    })
    public ResponseEntity<UserResponseDto> getUserById(
            @PathVariable @Positive Long id) {
        UserResponseDto user = userService.findById(id);
        return ResponseEntity.ok(user);
    }

    /**
     * Get all users (admin only).
     */
    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Get all users", description = "Retrieve all users with pagination (admin only)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Users retrieved successfully"),
        @ApiResponse(responseCode = "401", description = "Unauthorized"),
        @ApiResponse(responseCode = "403", description = "Forbidden - Admin role required")
    })
    public ResponseEntity<Page<UserSummaryDto>> getAllUsers(
            @PageableDefault(size = 20, sort = "id") Pageable pageable) {
        Page<UserSummaryDto> users = userService.findAll(pageable);
        return ResponseEntity.ok(users);
    }

    /**
     * Update user profile.
     */
    @PutMapping("/{id}")
    @PreAuthorize("isAuthenticated() and (#id == authentication.principal.id or hasRole('ADMIN'))")
    @Operation(summary = "Update user", description = "Update user profile (owner or admin only)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "User updated successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid input data"),
        @ApiResponse(responseCode = "404", description = "User not found"),
        @ApiResponse(responseCode = "403", description = "Forbidden - Not authorized"),
        @ApiResponse(responseCode = "409", description = "Email already exists")
    })
    public ResponseEntity<UserResponseDto> updateUser(
            @PathVariable @Positive Long id,
            @Valid @RequestBody UserUpdateDto dto) {
        UserResponseDto user = userService.update(id, dto);
        return ResponseEntity.ok(user);
    }

    /**
     * Delete user (admin only).
     */
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Delete user", description = "Delete user (soft delete, admin only)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "204", description = "User deleted successfully"),
        @ApiResponse(responseCode = "404", description = "User not found"),
        @ApiResponse(responseCode = "403", description = "Forbidden - Admin role required")
    })
    public ResponseEntity<Void> deleteUser(@PathVariable @Positive Long id) {
        userService.delete(id);
        return ResponseEntity.noContent().build();
    }

    /**
     * Search users by keyword.
     */
    @GetMapping("/search")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Search users", description = "Search users by keyword in username, email, or name")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Search completed successfully"),
        @ApiResponse(responseCode = "401", description = "Unauthorized")
    })
    public ResponseEntity<Page<UserSummaryDto>> searchUsers(
            @RequestParam(value = "q", required = false) String keyword,
            @PageableDefault(size = 20) Pageable pageable) {
        Page<UserSummaryDto> users = userService.search(keyword != null ? keyword : "", pageable);
        return ResponseEntity.ok(users);
    }
}
```

---

## STEP 9: Exception Handler

### Prompt
```
@workspace Use .github/prompts/create-exception-handler.md

Extend the global exception handler to handle these user-specific exceptions:

Custom exceptions to handle:
1. UserNotFoundException → 404 NOT FOUND
2. DuplicateUserException → 409 CONFLICT
3. InvalidCredentialsException → 401 UNAUTHORIZED

Also handle:
4. MethodArgumentNotValidException (validation errors) → 400 BAD REQUEST
5. ConstraintViolationException → 400 BAD REQUEST
6. AccessDeniedException → 403 FORBIDDEN
7. Exception (generic) → 500 INTERNAL SERVER ERROR

Return consistent error format:
{
  "timestamp": "2025-11-21T10:00:00",
  "status": 404,
  "error": "Not Found",
  "message": "User not found with id: 123",
  "path": "/api/v1/users/123"
}

For validation errors, include field-level details.
```

### Expected Output
```java
package com.example.userapi.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.context.request.ServletWebRequest;
import org.springframework.web.context.request.WebRequest;

import jakarta.validation.ConstraintViolationException;
import java.time.LocalDateTime;
import java.util.List;

/**
 * Global exception handler for the application.
 */
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    /**
     * Handle user not found exceptions.
     */
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UserNotFoundException ex,
            WebRequest request) {
        
        log.warn("User not found: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            ex.getMessage(),
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    /**
     * Handle duplicate user exceptions.
     */
    @ExceptionHandler(DuplicateUserException.class)
    public ResponseEntity<ErrorResponse> handleDuplicateUser(
            DuplicateUserException ex,
            WebRequest request) {
        
        log.warn("Duplicate user: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.CONFLICT.value(),
            "Conflict",
            ex.getMessage(),
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    /**
     * Handle invalid credentials exceptions.
     */
    @ExceptionHandler(InvalidCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleInvalidCredentials(
            InvalidCredentialsException ex,
            WebRequest request) {
        
        log.warn("Invalid credentials attempt");
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            ex.getMessage(),
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }

    /**
     * Handle validation errors from @Valid.
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex,
            WebRequest request) {
        
        log.warn("Validation failed: {} errors", ex.getBindingResult().getErrorCount());
        
        List<FieldError> fieldErrors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> new FieldError(
                error.getField(),
                error.getRejectedValue() != null ? error.getRejectedValue().toString() : "null",
                error.getDefaultMessage()
            ))
            .toList();
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Invalid input data",
            getPath(request),
            fieldErrors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    /**
     * Handle constraint violation exceptions.
     */
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ValidationErrorResponse> handleConstraintViolation(
            ConstraintViolationException ex,
            WebRequest request) {
        
        log.warn("Constraint violation: {}", ex.getMessage());
        
        List<FieldError> fieldErrors = ex.getConstraintViolations()
            .stream()
            .map(violation -> new FieldError(
                violation.getPropertyPath().toString(),
                violation.getInvalidValue() != null ? violation.getInvalidValue().toString() : "null",
                violation.getMessage()
            ))
            .toList();
        
        ValidationErrorResponse response = new ValidationErrorResponse(
            LocalDateTime.now(),
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Constraint violation",
            getPath(request),
            fieldErrors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    /**
     * Handle access denied exceptions.
     */
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException ex,
            WebRequest request) {
        
        log.warn("Access denied: {}", ex.getMessage());
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.FORBIDDEN.value(),
            "Forbidden",
            "You don't have permission to access this resource",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }

    /**
     * Handle all other exceptions.
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAllExceptions(
            Exception ex,
            WebRequest request) {
        
        log.error("Unexpected error occurred", ex);
        
        ErrorResponse error = new ErrorResponse(
            LocalDateTime.now(),
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal Server Error",
            "An unexpected error occurred. Please try again later.",
            getPath(request)
        );
        
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }

    /**
     * Extract request path from WebRequest.
     */
    private String getPath(WebRequest request) {
        return ((ServletWebRequest) request).getRequest().getRequestURI();
    }

    /**
     * Standard error response structure.
     */
    public record ErrorResponse(
        LocalDateTime timestamp,
        int status,
        String error,
        String message,
        String path
    ) {}

    /**
     * Validation error response with field details.
     */
    public record ValidationErrorResponse(
        LocalDateTime timestamp,
        int status,
        String error,
        String message,
        String path,
        List<FieldError> fieldErrors
    ) {}

    /**
     * Individual field error.
     */
    public record FieldError(
        String field,
        String rejectedValue,
        String message
    ) {}
}
```

---

## STEP 10: Unit Tests

### Prompt
```
Create comprehensive unit tests for UserService.

Context:
- UserService methods: [paste service class]
- Mocked dependencies: UserRepository, UserMapper, PasswordEncoder

Test cases:
1. register() - successful registration
2. register() - duplicate username throws DuplicateUserException
3. register() - duplicate email throws DuplicateUserException
4. findById() - successful retrieval
5. findById() - user not found throws UserNotFoundException
6. update() - successful update
7. update() - duplicate email throws DuplicateUserException
8. update() - user not found throws UserNotFoundException
9. delete() - successful soft delete
10. delete() - user not found throws UserNotFoundException
11. search() - successful search with results

Use:
- JUnit 5 (@ExtendWith(MockitoExtension.class))
- Mockito (@Mock, @InjectMocks)
- AssertJ for assertions
- Test naming: should[ExpectedBehavior]When[Condition]
```

### Expected Output (abbreviated - showing key tests)
```java
package com.example.userapi.service;

import com.example.userapi.dto.*;
import com.example.userapi.entity.User;
import com.example.userapi.exception.*;
import com.example.userapi.mapper.UserMapper;
import com.example.userapi.repository.UserRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.crypto.password.PasswordEncoder;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private UserMapper userMapper;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    @Test
    void shouldRegisterUserSuccessfullyWhenDataIsValid() {
        // Given
        UserRegistrationDto dto = new UserRegistrationDto(
            "johndoe", "john@example.com", "password123", "John", "Doe"
        );
        
        User user = new User();
        user.setUsername(dto.username());
        user.setEmail(dto.email());
        
        User savedUser = new User();
        savedUser.setId(1L);
        savedUser.setUsername(dto.username());
        
        UserResponseDto responseDto = new UserResponseDto(
            1L, "johndoe", "john@example.com", "John", "Doe", true, null, null
        );

        when(userRepository.existsByUsername(dto.username())).thenReturn(false);
        when(userRepository.existsByEmail(dto.email())).thenReturn(false);
        when(userMapper.toEntity(dto)).thenReturn(user);
        when(passwordEncoder.encode(dto.password())).thenReturn("hashedPassword");
        when(userRepository.save(any(User.class))).thenReturn(savedUser);
        when(userMapper.toResponseDto(savedUser)).thenReturn(responseDto);

        // When
        UserResponseDto result = userService.register(dto);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.username()).isEqualTo("johndoe");
        verify(userRepository).existsByUsername(dto.username());
        verify(userRepository).existsByEmail(dto.email());
        verify(passwordEncoder).encode(dto.password());
        verify(userRepository).save(any(User.class));
    }

    @Test
    void shouldThrowDuplicateUserExceptionWhenUsernameExists() {
        // Given
        UserRegistrationDto dto = new UserRegistrationDto(
            "johndoe", "john@example.com", "password123", "John", "Doe"
        );
        
        when(userRepository.existsByUsername(dto.username())).thenReturn(true);

        // When & Then
        assertThatThrownBy(() -> userService.register(dto))
            .isInstanceOf(DuplicateUserException.class)
            .hasMessageContaining("already exists with username");
        
        verify(userRepository).existsByUsername(dto.username());
        verify(userRepository, never()).save(any());
    }

    @Test
    void shouldThrowDuplicateUserExceptionWhenEmailExists() {
        // Given
        UserRegistrationDto dto = new UserRegistrationDto(
            "johndoe", "john@example.com", "password123", "John", "Doe"
        );
        
        when(userRepository.existsByUsername(dto.username())).thenReturn(false);
        when(userRepository.existsByEmail(dto.email())).thenReturn(true);

        // When & Then
        assertThatThrownBy(() -> userService.register(dto))
            .isInstanceOf(DuplicateUserException.class)
            .hasMessageContaining("already exists with email");
        
        verify(userRepository).existsByEmail(dto.email());
        verify(userRepository, never()).save(any());
    }

    @Test
    void shouldFindUserByIdSuccessfullyWhenUserExists() {
        // Given
        Long userId = 1L;
        User user = new User();
        user.setId(userId);
        user.setUsername("johndoe");
        
        UserResponseDto responseDto = new UserResponseDto(
            userId, "johndoe", "john@example.com", "John", "Doe", true, null, null
        );
        
        when(userRepository.findById(userId)).thenReturn(Optional.of(user));
        when(userMapper.toResponseDto(user)).thenReturn(responseDto);

        // When
        UserResponseDto result = userService.findById(userId);

        // Then
        assertThat(result).isNotNull();
        assertThat(result.id()).isEqualTo(userId);
        verify(userRepository).findById(userId);
    }

    @Test
    void shouldThrowUserNotFoundExceptionWhenUserDoesNotExist() {
        // Given
        Long userId = 999L;
        when(userRepository.findById(userId)).thenReturn(Optional.empty());

        // When & Then
        assertThatThrownBy(() -> userService.findById(userId))
            .isInstanceOf(UserNotFoundException.class)
            .hasMessageContaining("not found with id");
        
        verify(userRepository).findById(userId);
    }

    @Test
    void shouldDeleteUserSuccessfullyWhenUserExists() {
        // Given
        Long userId = 1L;
        User user = new User();
        user.setId(userId);
        user.setEnabled(true);
        
        when(userRepository.findById(userId)).thenReturn(Optional.of(user));
        when(userRepository.save(any(User.class))).thenReturn(user);

        // When
        userService.delete(userId);

        // Then
        assertThat(user.getEnabled()).isFalse();
        verify(userRepository).findById(userId);
        verify(userRepository).save(user);
    }
}
```

---

## STEP 11: Integration Tests

### Prompt
```
Create integration tests for UserController endpoints.

Context:
- UserController endpoints: [paste controller methods]
- Mock UserService

Test scenarios:
1. POST /api/v1/users/register - successful registration (201)
2. POST /api/v1/users/register - validation errors (400)
3. POST /api/v1/users/register - duplicate username (409)
4. GET /api/v1/users/{id} - successful retrieval (200)
5. GET /api/v1/users/{id} - user not found (404)
6. PUT /api/v1/users/{id} - successful update (200)
7. DELETE /api/v1/users/{id} - successful deletion (204)
8. GET /api/v1/users/search - successful search (200)

Use:
- @WebMvcTest(UserController.class)
- MockMvc for HTTP requests
- @MockBean for UserService
- ObjectMapper for JSON serialization
- Test all HTTP status codes and response bodies
```

### Expected Output (abbreviated - showing key tests)
```java
package com.example.userapi.controller;

import com.example.userapi.dto.*;
import com.example.userapi.exception.*;
import com.example.userapi.service.UserService;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.test.web.servlet.MockMvc;

import java.time.LocalDateTime;

import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @MockBean
    private UserService userService;

    @Test
    void shouldRegisterUserSuccessfullyWhenDataIsValid() throws Exception {
        // Given
        UserRegistrationDto request = new UserRegistrationDto(
            "johndoe", "john@example.com", "password123", "John", "Doe"
        );
        
        UserResponseDto response = new UserResponseDto(
            1L, "johndoe", "john@example.com", "John", "Doe", 
            true, LocalDateTime.now(), LocalDateTime.now()
        );
        
        when(userService.register(any(UserRegistrationDto.class))).thenReturn(response);

        // When & Then
        mockMvc.perform(post("/api/v1/users/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.username").value("johndoe"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
        
        verify(userService).register(any(UserRegistrationDto.class));
    }

    @Test
    void shouldReturnBadRequestWhenValidationFails() throws Exception {
        // Given - Invalid data (short username)
        UserRegistrationDto request = new UserRegistrationDto(
            "jo", "john@example.com", "password123", "John", "Doe"
        );

        // When & Then
        mockMvc.perform(post("/api/v1/users/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest());
        
        verify(userService, never()).register(any());
    }

    @Test
    void shouldReturnConflictWhenUsernameExists() throws Exception {
        // Given
        UserRegistrationDto request = new UserRegistrationDto(
            "johndoe", "john@example.com", "password123", "John", "Doe"
        );
        
        when(userService.register(any(UserRegistrationDto.class)))
            .thenThrow(DuplicateUserException.byUsername("johndoe"));

        // When & Then
        mockMvc.perform(post("/api/v1/users/register")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.status").value(409))
            .andExpect(jsonPath("$.message").value(containsString("already exists")));
        
        verify(userService).register(any(UserRegistrationDto.class));
    }

    @Test
    void shouldGetUserByIdSuccessfullyWhenUserExists() throws Exception {
        // Given
        Long userId = 1L;
        UserResponseDto response = new UserResponseDto(
            userId, "johndoe", "john@example.com", "John", "Doe",
            true, LocalDateTime.now(), LocalDateTime.now()
        );
        
        when(userService.findById(userId)).thenReturn(response);

        // When & Then
        mockMvc.perform(get("/api/v1/users/{id}", userId))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(userId))
            .andExpect(jsonPath("$.username").value("johndoe"));
        
        verify(userService).findById(userId);
    }

    @Test
    void shouldReturnNotFoundWhenUserDoesNotExist() throws Exception {
        // Given
        Long userId = 999L;
        when(userService.findById(userId))
            .thenThrow(UserNotFoundException.byId(userId));

        // When & Then
        mockMvc.perform(get("/api/v1/users/{id}", userId))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.status").value(404))
            .andExpect(jsonPath("$.message").value(containsString("not found")));
        
        verify(userService).findById(userId);
    }

    @Test
    void shouldUpdateUserSuccessfullyWhenDataIsValid() throws Exception {
        // Given
        Long userId = 1L;
        UserUpdateDto request = new UserUpdateDto(
            "newemail@example.com", "John", "Doe"
        );
        
        UserResponseDto response = new UserResponseDto(
            userId, "johndoe", "newemail@example.com", "John", "Doe",
            true, LocalDateTime.now(), LocalDateTime.now()
        );
        
        when(userService.update(eq(userId), any(UserUpdateDto.class))).thenReturn(response);

        // When & Then
        mockMvc.perform(put("/api/v1/users/{id}", userId)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.email").value("newemail@example.com"));
        
        verify(userService).update(eq(userId), any(UserUpdateDto.class));
    }

    @Test
    void shouldDeleteUserSuccessfully() throws Exception {
        // Given
        Long userId = 1L;
        doNothing().when(userService).delete(userId);

        // When & Then
        mockMvc.perform(delete("/api/v1/users/{id}", userId))
            .andExpect(status().isNoContent());
        
        verify(userService).delete(userId);
    }
}
```

---

## Summary: What We Built

Through this 11-step prompt chain, we created:

### ✅ Complete Components
1. **User Entity** - JPA entity with validation
2. **UserRepository** - Spring Data JPA repository
3. **DTOs** - 5 different DTOs for various use cases
4. **Custom Exceptions** - 3 domain-specific exceptions
5. **UserMapper** - Entity ↔ DTO conversion
6. **UserService** - Complete business logic (7 methods)
7. **UserController** - REST API (6 endpoints)
8. **Exception Handler** - Global error handling
9. **Unit Tests** - Service layer tests
10. **Integration Tests** - Controller tests

### 🎯 Key Benefits of This Chain

1. **Quality**: Each component focuses on one responsibility
2. **Validation**: Each step's output reviewed before next step
3. **Completeness**: Nothing missed - from entity to tests
4. **Consistency**: All code follows same patterns and standards
5. **Reusability**: Can reuse this chain for other entities (Product, Order, etc.)
6. **Documentation**: Clear record of what was built and why

### 📋 How to Use This Chain

1. **Copy the prompts** from each step
2. **Paste into Copilot Chat** (with `@workspace`)
3. **Review the output** from each step
4. **Pass output to next step** as context
5. **Adjust as needed** based on your specific requirements

### 🔄 Adapting for Other Entities

To create a Product API, Order API, etc., simply:
1. Replace "User" with your entity name in Step 1
2. Adjust fields and relationships for your domain
3. Follow the same 11-step chain structure
4. All patterns and practices remain the same

---

---

## BONUS: Role Entity Chain

The User entity references a Role entity for role-based access control. Here's the complete chain for implementing the Role entity.

### ROLE STEP 1: Role Entity Design

#### Prompt
```
@workspace Use .github/prompts/create-entity.md

Create a Role entity for role-based access control with:
- id: Long (Primary Key, auto-generated)
- name: String (unique, required, e.g., USER, ADMIN, MODERATOR)
- description: String (optional, describes the role)
- users: Set<User> (Many-to-Many relationship, mapped by roles)
- createdAt: LocalDateTime

Use proper JPA annotations and include equals/hashCode based on name.
```

#### Expected Output
```java
package com.example.userapi.entity;

import jakarta.persistence.*;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;

import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

/**
 * Role entity for role-based access control.
 */
@Entity
@Table(name = "roles", uniqueConstraints = {
    @UniqueConstraint(columnNames = "name")
})
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Role name is required")
    @Size(max = 50, message = "Role name cannot exceed 50 characters")
    @Column(nullable = false, unique = true, length = 50)
    private String name;

    @Size(max = 255, message = "Description cannot exceed 255 characters")
    @Column(length = 255)
    private String description;

    @ManyToMany(mappedBy = "roles", fetch = FetchType.LAZY)
    private Set<User> users = new HashSet<>();

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    /**
     * Convenience constructor for creating a role with name.
     */
    public Role(String name) {
        this.name = name;
    }

    /**
     * Convenience constructor for creating a role with name and description.
     */
    public Role(String name, String description) {
        this.name = name;
        this.description = description;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Role role = (Role) o;
        return Objects.equals(name, role.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name);
    }

    @Override
    public String toString() {
        return "Role{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", description='" + description + '\'' +
                ", createdAt=" + createdAt +
                '}';
    }
}
```

---

### ROLE STEP 2: Role Repository

#### Prompt
```
@workspace Use .github/prompts/create-repository.md

For the Role entity, create RoleRepository with:
1. findByName(String name) - find role by name
2. existsByName(String name) - check if role exists
3. findAll() - get all roles (no pagination needed for small set)
```

#### Expected Output
```java
package com.example.userapi.repository;

import com.example.userapi.entity.Role;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

/**
 * Repository interface for Role entity.
 */
@Repository
public interface RoleRepository extends JpaRepository<Role, Long> {

    /**
     * Find role by name.
     *
     * @param name the role name (e.g., "USER", "ADMIN")
     * @return optional containing role if found
     */
    Optional<Role> findByName(String name);

    /**
     * Check if role exists by name.
     *
     * @param name the role name
     * @return true if exists, false otherwise
     */
    boolean existsByName(String name);
}
```

---

### ROLE STEP 3: Role DTOs

#### Prompt
```
@workspace Use .github/prompts/create-dto.md

Create Role DTOs using Java 17 records:

1. RoleDto - for API responses
   Fields: id, name, description

2. RoleCreateDto - for creating new roles (admin only)
   Fields: name, description
   Validation: name required, max 50 chars

3. RoleAssignmentDto - for assigning roles to users
   Fields: roleNames (Set<String>)
   Validation: at least one role required
```

#### Expected Output
```java
package com.example.userapi.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.Size;

import java.util.Set;

/**
 * DTO for role response.
 */
public record RoleDto(
    Long id,
    String name,
    String description
) {}

/**
 * DTO for creating a new role.
 */
public record RoleCreateDto(
    @NotBlank(message = "Role name is required")
    @Size(max = 50, message = "Role name cannot exceed 50 characters")
    String name,

    @Size(max = 255, message = "Description cannot exceed 255 characters")
    String description
) {}

/**
 * DTO for assigning roles to a user.
 */
public record RoleAssignmentDto(
    @NotEmpty(message = "At least one role is required")
    Set<String> roleNames
) {}
```

---

### ROLE STEP 4: Role Mapper

#### Prompt
```
@workspace Use .github/prompts/create-mapper.md

Create RoleMapper with methods:
1. toDto(Role role) -> RoleDto
2. toEntity(RoleCreateDto dto) -> Role
3. toDtoList(Collection<Role> roles) -> List<RoleDto>
4. toRoleSet(Set<String> roleNames, RoleRepository) -> Set<Role>
```

#### Expected Output
```java
package com.example.userapi.mapper;

import com.example.userapi.dto.RoleCreateDto;
import com.example.userapi.dto.RoleDto;
import com.example.userapi.entity.Role;
import com.example.userapi.exception.RoleNotFoundException;
import com.example.userapi.repository.RoleRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.Collection;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;

/**
 * Mapper for converting between Role entity and DTOs.
 */
@Component
@RequiredArgsConstructor
public class RoleMapper {

    private final RoleRepository roleRepository;

    /**
     * Convert Role entity to DTO.
     */
    public RoleDto toDto(Role role) {
        if (role == null) {
            return null;
        }

        return new RoleDto(
            role.getId(),
            role.getName(),
            role.getDescription()
        );
    }

    /**
     * Convert RoleCreateDto to Role entity.
     */
    public Role toEntity(RoleCreateDto dto) {
        if (dto == null) {
            return null;
        }

        return new Role(dto.name(), dto.description());
    }

    /**
     * Convert collection of Role entities to list of DTOs.
     */
    public List<RoleDto> toDtoList(Collection<Role> roles) {
        if (roles == null) {
            return List.of();
        }

        return roles.stream()
            .map(this::toDto)
            .collect(Collectors.toList());
    }

    /**
     * Convert set of role names to set of Role entities.
     *
     * @throws RoleNotFoundException if any role is not found
     */
    public Set<Role> toRoleSet(Set<String> roleNames) {
        if (roleNames == null || roleNames.isEmpty()) {
            return Set.of();
        }

        return roleNames.stream()
            .map(name -> roleRepository.findByName(name)
                .orElseThrow(() -> new RoleNotFoundException("Role not found: " + name)))
            .collect(Collectors.toSet());
    }
}
```

---

### ROLE STEP 5: Role Service

#### Prompt
```
@workspace Use .github/prompts/create-service.md

Create RoleService with:
1. findAll() -> List<RoleDto> - get all roles
2. findByName(String name) -> RoleDto - find role by name
3. create(RoleCreateDto) -> RoleDto - create new role (admin only)
4. assignRolesToUser(Long userId, RoleAssignmentDto) -> UserResponseDto
5. removeRoleFromUser(Long userId, String roleName) -> UserResponseDto

Include validation for:
- Duplicate role names
- Cannot remove last ADMIN role
- Role must exist before assignment
```

#### Expected Output
```java
package com.example.userapi.service;

import com.example.userapi.dto.*;
import com.example.userapi.entity.Role;
import com.example.userapi.entity.User;
import com.example.userapi.exception.*;
import com.example.userapi.mapper.RoleMapper;
import com.example.userapi.mapper.UserMapper;
import com.example.userapi.repository.RoleRepository;
import com.example.userapi.repository.UserRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.Set;

/**
 * Service for managing roles.
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class RoleService {

    private final RoleRepository roleRepository;
    private final UserRepository userRepository;
    private final RoleMapper roleMapper;
    private final UserMapper userMapper;

    /**
     * Get all available roles.
     */
    public List<RoleDto> findAll() {
        log.debug("Finding all roles");
        List<Role> roles = roleRepository.findAll();
        return roleMapper.toDtoList(roles);
    }

    /**
     * Find role by name.
     */
    public RoleDto findByName(String name) {
        log.debug("Finding role by name: {}", name);

        Role role = roleRepository.findByName(name)
            .orElseThrow(() -> new RoleNotFoundException("Role not found: " + name));

        return roleMapper.toDto(role);
    }

    /**
     * Create a new role.
     */
    @Transactional
    public RoleDto create(RoleCreateDto dto) {
        log.info("Creating new role: {}", dto.name());

        if (roleRepository.existsByName(dto.name())) {
            throw new DuplicateRoleException("Role already exists: " + dto.name());
        }

        Role role = roleMapper.toEntity(dto);
        Role savedRole = roleRepository.save(role);

        log.debug("Role created with id: {}", savedRole.getId());
        return roleMapper.toDto(savedRole);
    }

    /**
     * Assign roles to a user.
     */
    @Transactional
    public UserResponseDto assignRolesToUser(Long userId, RoleAssignmentDto dto) {
        log.info("Assigning roles {} to user {}", dto.roleNames(), userId);

        User user = userRepository.findById(userId)
            .orElseThrow(() -> UserNotFoundException.byId(userId));

        Set<Role> roles = roleMapper.toRoleSet(dto.roleNames());
        user.getRoles().addAll(roles);

        User updatedUser = userRepository.save(user);
        log.debug("Roles assigned successfully");

        return userMapper.toResponseDto(updatedUser);
    }

    /**
     * Remove a role from a user.
     */
    @Transactional
    public UserResponseDto removeRoleFromUser(Long userId, String roleName) {
        log.info("Removing role {} from user {}", roleName, userId);

        User user = userRepository.findById(userId)
            .orElseThrow(() -> UserNotFoundException.byId(userId));

        Role role = roleRepository.findByName(roleName)
            .orElseThrow(() -> new RoleNotFoundException("Role not found: " + roleName));

        // Check if this is the last admin
        if ("ADMIN".equals(roleName) && isLastAdmin(user)) {
            throw new BusinessException("Cannot remove the last ADMIN role");
        }

        user.getRoles().remove(role);
        User updatedUser = userRepository.save(user);

        log.debug("Role removed successfully");
        return userMapper.toResponseDto(updatedUser);
    }

    /**
     * Check if user is the last admin.
     */
    private boolean isLastAdmin(User user) {
        if (!user.getRoles().stream().anyMatch(r -> "ADMIN".equals(r.getName()))) {
            return false;
        }

        long adminCount = userRepository.countByRolesName("ADMIN");
        return adminCount <= 1;
    }
}
```

---

### ROLE STEP 6: Role Controller

#### Prompt
```
@workspace Use .github/prompts/create-controller.md

Create RoleController with endpoints:
1. GET /api/v1/roles - List all roles (authenticated)
2. GET /api/v1/roles/{name} - Get role by name (authenticated)
3. POST /api/v1/roles - Create new role (ADMIN only)
4. POST /api/v1/users/{id}/roles - Assign roles to user (ADMIN only)
5. DELETE /api/v1/users/{id}/roles/{roleName} - Remove role from user (ADMIN only)
```

#### Expected Output
```java
package com.example.userapi.controller;

import com.example.userapi.dto.*;
import com.example.userapi.service.RoleService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * REST controller for role management.
 */
@RestController
@RequestMapping("/api/v1")
@RequiredArgsConstructor
@Validated
@Tag(name = "Role Management", description = "APIs for managing roles")
public class RoleController {

    private final RoleService roleService;

    /**
     * Get all available roles.
     */
    @GetMapping("/roles")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Get all roles", description = "Retrieve list of all available roles")
    public ResponseEntity<List<RoleDto>> getAllRoles() {
        List<RoleDto> roles = roleService.findAll();
        return ResponseEntity.ok(roles);
    }

    /**
     * Get role by name.
     */
    @GetMapping("/roles/{name}")
    @PreAuthorize("isAuthenticated()")
    @Operation(summary = "Get role by name", description = "Retrieve a role by its name")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Role found"),
        @ApiResponse(responseCode = "404", description = "Role not found")
    })
    public ResponseEntity<RoleDto> getRoleByName(@PathVariable String name) {
        RoleDto role = roleService.findByName(name);
        return ResponseEntity.ok(role);
    }

    /**
     * Create a new role.
     */
    @PostMapping("/roles")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Create role", description = "Create a new role (Admin only)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Role created successfully"),
        @ApiResponse(responseCode = "400", description = "Invalid input data"),
        @ApiResponse(responseCode = "409", description = "Role already exists")
    })
    public ResponseEntity<RoleDto> createRole(@Valid @RequestBody RoleCreateDto dto) {
        RoleDto role = roleService.create(dto);
        return ResponseEntity.status(HttpStatus.CREATED).body(role);
    }

    /**
     * Assign roles to a user.
     */
    @PostMapping("/users/{userId}/roles")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Assign roles to user", description = "Assign one or more roles to a user (Admin only)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Roles assigned successfully"),
        @ApiResponse(responseCode = "404", description = "User or role not found")
    })
    public ResponseEntity<UserResponseDto> assignRoles(
            @PathVariable Long userId,
            @Valid @RequestBody RoleAssignmentDto dto) {
        UserResponseDto user = roleService.assignRolesToUser(userId, dto);
        return ResponseEntity.ok(user);
    }

    /**
     * Remove a role from a user.
     */
    @DeleteMapping("/users/{userId}/roles/{roleName}")
    @PreAuthorize("hasRole('ADMIN')")
    @Operation(summary = "Remove role from user", description = "Remove a role from a user (Admin only)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Role removed successfully"),
        @ApiResponse(responseCode = "404", description = "User or role not found"),
        @ApiResponse(responseCode = "400", description = "Cannot remove last admin")
    })
    public ResponseEntity<UserResponseDto> removeRole(
            @PathVariable Long userId,
            @PathVariable String roleName) {
        UserResponseDto user = roleService.removeRoleFromUser(userId, roleName);
        return ResponseEntity.ok(user);
    }
}
```

---

### ROLE STEP 7: Role Exceptions

#### Prompt
```
Create custom exceptions for roles:
1. RoleNotFoundException - when role doesn't exist
2. DuplicateRoleException - when creating duplicate role
```

#### Expected Output
```java
package com.example.userapi.exception;

/**
 * Exception thrown when a role is not found.
 */
public class RoleNotFoundException extends RuntimeException {

    public RoleNotFoundException(String message) {
        super(message);
    }

    public RoleNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Exception thrown when attempting to create a duplicate role.
 */
public class DuplicateRoleException extends RuntimeException {

    public DuplicateRoleException(String message) {
        super(message);
    }

    public DuplicateRoleException(String message, Throwable cause) {
        super(message, cause);
    }
}

/**
 * Exception thrown for general business rule violations.
 */
public class BusinessException extends RuntimeException {

    public BusinessException(String message) {
        super(message);
    }

    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

### ROLE STEP 8: Update Exception Handler

#### Prompt
```
Update GlobalExceptionHandler to handle role exceptions:
- RoleNotFoundException → 404
- DuplicateRoleException → 409
- BusinessException → 400
```

#### Expected Output (additions to GlobalExceptionHandler)
```java
/**
 * Handle role not found exceptions.
 */
@ExceptionHandler(RoleNotFoundException.class)
public ResponseEntity<ErrorResponse> handleRoleNotFound(
        RoleNotFoundException ex,
        WebRequest request) {

    log.warn("Role not found: {}", ex.getMessage());

    ErrorResponse error = new ErrorResponse(
        LocalDateTime.now(),
        HttpStatus.NOT_FOUND.value(),
        "Not Found",
        ex.getMessage(),
        getPath(request)
    );

    return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
}

/**
 * Handle duplicate role exceptions.
 */
@ExceptionHandler(DuplicateRoleException.class)
public ResponseEntity<ErrorResponse> handleDuplicateRole(
        DuplicateRoleException ex,
        WebRequest request) {

    log.warn("Duplicate role: {}", ex.getMessage());

    ErrorResponse error = new ErrorResponse(
        LocalDateTime.now(),
        HttpStatus.CONFLICT.value(),
        "Conflict",
        ex.getMessage(),
        getPath(request)
    );

    return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
}

/**
 * Handle business rule exceptions.
 */
@ExceptionHandler(BusinessException.class)
public ResponseEntity<ErrorResponse> handleBusinessException(
        BusinessException ex,
        WebRequest request) {

    log.warn("Business rule violation: {}", ex.getMessage());

    ErrorResponse error = new ErrorResponse(
        LocalDateTime.now(),
        HttpStatus.BAD_REQUEST.value(),
        "Bad Request",
        ex.getMessage(),
        getPath(request)
    );

    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
}
```

---

### ROLE STEP 9: Database Migration for Roles

#### Prompt
```
@workspace Use .github/prompts/create-migration.md

Create Flyway migrations for the Role entity:
1. V2__Create_roles_table.sql - Create roles table
2. V3__Create_user_roles_table.sql - Create junction table
3. V4__Insert_default_roles.sql - Insert USER, ADMIN, MODERATOR roles
```

#### Expected Output
See the create-migration.md prompt template for complete migration scripts.

---

## Next Steps

After completing this chain, you would typically add:
- **Step 12**: JWT Authentication implementation (use create-security-config.md)
- **Step 13**: Database migrations (use create-migration.md)
- **Step 14**: API documentation (OpenAPI/Swagger UI)
- **Step 15**: Performance tests
- **Step 16**: Security tests

Each of these could be their own prompt chain!

---

This is **real-world prompt chaining** in action. Copy these prompts, adapt them to your needs, and build high-quality APIs efficiently!
