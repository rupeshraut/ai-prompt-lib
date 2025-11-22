# Create Database Migration

Create database migration scripts using Flyway or Liquibase following best practices.

## Overview

Database migrations provide:
- Version control for database schema
- Repeatable deployments across environments
- Rollback capabilities
- Team collaboration on schema changes

## Flyway Migrations

### Setup

**Maven Dependency:**
```xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
```

**Application Properties:**
```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
```

### Migration File Naming Convention

```
V{version}__{description}.sql

Examples:
V1__Create_users_table.sql
V2__Create_roles_table.sql
V3__Create_user_roles_table.sql
V4__Add_email_verification_fields.sql
```

### Migration Templates

#### V1__Create_users_table.sql

```sql
-- Create users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    enabled BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_enabled ON users(enabled);

-- Add comments
COMMENT ON TABLE users IS 'Stores user account information';
COMMENT ON COLUMN users.username IS 'Unique username for login';
COMMENT ON COLUMN users.password IS 'BCrypt hashed password';
COMMENT ON COLUMN users.enabled IS 'Soft delete flag - false means deleted';
```

#### V2__Create_roles_table.sql

```sql
-- Create roles table
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(255),
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

-- Insert default roles
INSERT INTO roles (name, description) VALUES
    ('USER', 'Standard user role'),
    ('ADMIN', 'Administrator role with full access'),
    ('MODERATOR', 'Moderator role with limited admin access');

-- Add comment
COMMENT ON TABLE roles IS 'Stores available user roles';
```

#### V3__Create_user_roles_table.sql

```sql
-- Create user_roles junction table
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    assigned_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id, role_id),
    CONSTRAINT fk_user_roles_user
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT fk_user_roles_role
        FOREIGN KEY (role_id) REFERENCES roles(id) ON DELETE CASCADE
);

-- Create indexes for foreign keys
CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);
CREATE INDEX idx_user_roles_role_id ON user_roles(role_id);

-- Add comment
COMMENT ON TABLE user_roles IS 'Many-to-many relationship between users and roles';
```

#### V4__Add_audit_fields.sql

```sql
-- Add audit fields to users table
ALTER TABLE users
    ADD COLUMN created_by BIGINT,
    ADD COLUMN updated_by BIGINT;

-- Add foreign keys for audit fields
ALTER TABLE users
    ADD CONSTRAINT fk_users_created_by
        FOREIGN KEY (created_by) REFERENCES users(id),
    ADD CONSTRAINT fk_users_updated_by
        FOREIGN KEY (updated_by) REFERENCES users(id);
```

#### V5__Create_refresh_tokens_table.sql

```sql
-- Create refresh tokens table for JWT authentication
CREATE TABLE refresh_tokens (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    token VARCHAR(500) NOT NULL UNIQUE,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    revoked BOOLEAN NOT NULL DEFAULT FALSE,
    CONSTRAINT fk_refresh_tokens_user
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Create indexes
CREATE INDEX idx_refresh_tokens_user_id ON refresh_tokens(user_id);
CREATE INDEX idx_refresh_tokens_token ON refresh_tokens(token);
CREATE INDEX idx_refresh_tokens_expires_at ON refresh_tokens(expires_at);

-- Add comment
COMMENT ON TABLE refresh_tokens IS 'Stores JWT refresh tokens for token refresh mechanism';
```

---

## Liquibase Migrations

### Setup

**Maven Dependency:**
```xml
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

**Application Properties:**
```yaml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml
```

### Master Changelog

**db/changelog/db.changelog-master.xml:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <include file="db/changelog/changes/001-create-users-table.xml"/>
    <include file="db/changelog/changes/002-create-roles-table.xml"/>
    <include file="db/changelog/changes/003-create-user-roles-table.xml"/>
    <include file="db/changelog/changes/004-add-audit-fields.xml"/>

</databaseChangeLog>
```

### Migration Templates

#### 001-create-users-table.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="001-create-users-table" author="developer">
        <comment>Create users table</comment>

        <createTable tableName="users" remarks="Stores user account information">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="username" type="VARCHAR(50)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="email" type="VARCHAR(100)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="password" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="first_name" type="VARCHAR(50)"/>
            <column name="last_name" type="VARCHAR(50)"/>
            <column name="enabled" type="BOOLEAN" defaultValueBoolean="true">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <createIndex tableName="users" indexName="idx_users_username">
            <column name="username"/>
        </createIndex>

        <createIndex tableName="users" indexName="idx_users_email">
            <column name="email"/>
        </createIndex>

        <createIndex tableName="users" indexName="idx_users_enabled">
            <column name="enabled"/>
        </createIndex>

        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

#### 002-create-roles-table.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="002-create-roles-table" author="developer">
        <comment>Create roles table</comment>

        <createTable tableName="roles" remarks="Stores available user roles">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="name" type="VARCHAR(50)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="description" type="VARCHAR(255)"/>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <rollback>
            <dropTable tableName="roles"/>
        </rollback>
    </changeSet>

    <changeSet id="002-insert-default-roles" author="developer">
        <comment>Insert default roles</comment>

        <insert tableName="roles">
            <column name="name" value="USER"/>
            <column name="description" value="Standard user role"/>
        </insert>
        <insert tableName="roles">
            <column name="name" value="ADMIN"/>
            <column name="description" value="Administrator role with full access"/>
        </insert>
        <insert tableName="roles">
            <column name="name" value="MODERATOR"/>
            <column name="description" value="Moderator role with limited admin access"/>
        </insert>

        <rollback>
            <delete tableName="roles">
                <where>name IN ('USER', 'ADMIN', 'MODERATOR')</where>
            </delete>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

#### 003-create-user-roles-table.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="003-create-user-roles-table" author="developer">
        <comment>Create user_roles junction table</comment>

        <createTable tableName="user_roles" remarks="Many-to-many relationship between users and roles">
            <column name="user_id" type="BIGINT">
                <constraints nullable="false"/>
            </column>
            <column name="role_id" type="BIGINT">
                <constraints nullable="false"/>
            </column>
            <column name="assigned_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <addPrimaryKey
            tableName="user_roles"
            columnNames="user_id, role_id"
            constraintName="pk_user_roles"/>

        <addForeignKeyConstraint
            baseTableName="user_roles"
            baseColumnNames="user_id"
            referencedTableName="users"
            referencedColumnNames="id"
            constraintName="fk_user_roles_user"
            onDelete="CASCADE"/>

        <addForeignKeyConstraint
            baseTableName="user_roles"
            baseColumnNames="role_id"
            referencedTableName="roles"
            referencedColumnNames="id"
            constraintName="fk_user_roles_role"
            onDelete="CASCADE"/>

        <createIndex tableName="user_roles" indexName="idx_user_roles_user_id">
            <column name="user_id"/>
        </createIndex>

        <createIndex tableName="user_roles" indexName="idx_user_roles_role_id">
            <column name="role_id"/>
        </createIndex>

        <rollback>
            <dropTable tableName="user_roles"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

---

## Best Practices

### General Guidelines

1. **Never modify existing migrations**
   - Create new migrations for changes
   - Old migrations should remain unchanged after deployment

2. **One change per migration**
   - Keep migrations focused and atomic
   - Easier to debug and rollback

3. **Always include rollback**
   - Define how to undo each change
   - Test rollbacks before deployment

4. **Use descriptive names**
   - Name migrations clearly
   - Include what the migration does

5. **Test migrations**
   - Test on a copy of production data
   - Verify rollback works correctly

### Naming Conventions

| Type | Flyway | Liquibase |
|------|--------|-----------|
| Create table | V1__Create_users_table.sql | 001-create-users-table.xml |
| Add column | V5__Add_email_verified_to_users.sql | 005-add-email-verified.xml |
| Create index | V8__Add_index_on_users_email.sql | 008-add-users-email-index.xml |
| Data migration | V10__Migrate_user_roles_data.sql | 010-migrate-user-roles.xml |

### Index Guidelines

Create indexes for:
- Foreign key columns
- Columns used in WHERE clauses
- Columns used in ORDER BY
- Columns used in JOIN conditions

```sql
-- Good: Index on frequently queried columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_enabled ON users(enabled);

-- Good: Composite index for common query patterns
CREATE INDEX idx_users_status_created ON users(enabled, created_at);
```

### Data Type Guidelines

| Java Type | PostgreSQL | MySQL |
|-----------|------------|-------|
| Long | BIGINT/BIGSERIAL | BIGINT AUTO_INCREMENT |
| String | VARCHAR(n) | VARCHAR(n) |
| Boolean | BOOLEAN | BOOLEAN/TINYINT(1) |
| LocalDateTime | TIMESTAMP | DATETIME |
| LocalDate | DATE | DATE |
| BigDecimal | DECIMAL(p,s) | DECIMAL(p,s) |
| byte[] | BYTEA | BLOB |

### Constraint Guidelines

```sql
-- Primary Key
CONSTRAINT pk_table_name PRIMARY KEY (id)

-- Foreign Key with CASCADE
CONSTRAINT fk_table_column
    FOREIGN KEY (column) REFERENCES other_table(id)
    ON DELETE CASCADE

-- Unique constraint
CONSTRAINT uq_table_column UNIQUE (column)

-- Check constraint
CONSTRAINT chk_table_column CHECK (column > 0)

-- Not null with default
column_name TYPE NOT NULL DEFAULT value
```

---

## Rollback Strategies

### Flyway Rollback (Undo migrations - Teams/Enterprise only)

```sql
-- U1__Create_users_table.sql (Undo migration)
DROP TABLE IF EXISTS users;
```

### Liquibase Rollback

```xml
<rollback>
    <dropTable tableName="users"/>
</rollback>
```

### Manual Rollback Script

Always create a companion rollback script:

```sql
-- rollback/V1__Create_users_table_rollback.sql
DROP INDEX IF EXISTS idx_users_email;
DROP INDEX IF EXISTS idx_users_username;
DROP TABLE IF EXISTS users;
```

---

## Migration Testing

```java
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class MigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15")
        .withDatabaseName("testdb");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private DataSource dataSource;

    @Test
    void migrationsShouldRun() {
        // Migrations run automatically on startup
        // Verify tables exist
        assertThat(tableExists("users")).isTrue();
        assertThat(tableExists("roles")).isTrue();
        assertThat(tableExists("user_roles")).isTrue();
    }

    private boolean tableExists(String tableName) {
        try (Connection conn = dataSource.getConnection()) {
            DatabaseMetaData meta = conn.getMetaData();
            ResultSet rs = meta.getTables(null, null, tableName, null);
            return rs.next();
        } catch (SQLException e) {
            return false;
        }
    }
}
```
