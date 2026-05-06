# 🏗️ DataSoft Microservices System — Complete Implementation Guide

**Tech Stack:** Spring Boot 3.x + MSSQL Server + JWT + @PreAuthorize + Microservices Architecture

---

## 📋 Table of Contents

1. [System Architecture Overview](#1-system-architecture-overview)
2. [Why এই Design Choices?](#2-why-এই-design-choices)
3. [MSSQL Database Design](#3-mssql-database-design)
4. [Project Structure](#4-project-structure)
5. [Auth Service Implementation](#5-auth-service-implementation)
6. [Permission Service Implementation](#6-permission-service-implementation)
   - 6.1 application.yml
   - 6.2 Entities
   - 6.3 Repositories
   - 6.4 Permission Calculation Service
   - 6.5 Event DTO
   - 6.6 Permission Controller
   - 6.7 Redis Configuration *(নতুন)*
   - 6.8 Kafka Producer Configuration *(নতুন)*
7. [Order Service Implementation](#7-order-service-implementation)
   - 7.1 application.yml
   - 7.2 Permission Client (Feign)
   - 7.3 Permission Cache (Caffeine)
   - 7.4 Kafka Consumer Configuration *(নতুন)*
   - 7.5 Permission Changed Event DTO *(নতুন)*
   - 7.6 Feign Configuration *(নতুন)*
   - 7.7 Security Config + PermissionEvaluator
   - 7.8 Header-to-SecurityContext Filter
   - 7.9 Order Entity & Repository
   - 7.10 Order Controller (@PreAuthorize)
   - 7.11 Order Security Service
   - 7.12 OrderService Business Logic *(নতুন)*
8. [Gateway Implementation](#8-gateway-implementation)
9. [Common Module (Shared)](#9-common-module-shared)
10. [Dockerfile](#10-dockerfile-সব-service-এর-জন্য) *(নতুন)*
11. [Docker Compose Setup](#11-docker-compose-setup)
12. [End-to-End Flow Examples](#12-end-to-end-flow-examples)
13. [Testing Strategy](#13-testing-strategy)

---

## 1. System Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                     React Frontend (Future)                       │
└────────────────────────────┬─────────────────────────────────────┘
                             │ HTTPS + JWT
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│              Spring Cloud Gateway (Port 8080)                     │
│                                                                   │
│  ✓ JWT Validation (signature + expiry)                           │
│  ✓ Public route exception (login, register)                      │
│  ✓ Header injection: X-User-Id, X-User-Roles                     │
│  ✓ Routing to downstream services                                │
└─────┬─────────────────┬──────────────────────┬──────────────────┘
      │                 │                      │
      ▼                 ▼                      ▼
┌──────────┐    ┌──────────────┐      ┌──────────────────┐
│  Auth    │    │ Permission   │      │  Order           │
│  Service │    │  Service     │◄─────│  Service         │
│  :8081   │    │  :8082       │Feign │  :8083           │
└────┬─────┘    └──────┬───────┘      └────────┬─────────┘
     │                 │                       │
     │           ┌─────┴─────┐                 │
     │           │           │                 │
     ▼           ▼           ▼                 ▼
┌────────┐  ┌────────┐  ┌─────────┐      ┌─────────┐
│ MSSQL  │  │ MSSQL  │  │  Redis  │◄─────│  Kafka  │
│auth_db │  │perm_db │  │ (cache) │      │ (events)│
└────────┘  └────────┘  └─────────┘      └─────────┘
```

---

## 2. Why এই Design Choices?

### Microservice Boundaries — কেন এই 3টা Services?

| Service | Responsibility | Why Separate? |
|---------|---------------|---------------|
| **Auth Service** | User identity, login, register, JWT issue | Independently scale; auth bottleneck হলে শুধু এটা scale |
| **Permission Service** | Role/permission CRUD, override management | Centralized authority; multiple services use করবে |
| **Order Service** | Business logic example | Demonstrates how services use authorization |
| **Gateway** | Single entry point | Edge security, routing |

**Bounded Context Principle** — প্রতিটা service একটা specific domain handle করে।

### Why MSSQL?

DataSoft enterprise context-এ MSSQL natural fit:
- ✅ Strong ACID compliance
- ✅ Excellent Spring Data JPA support
- ✅ Enterprise tooling (SSMS, profiling)
- ✅ Stored procedures, advanced query optimization
- ✅ Familiar to DBAs in corporate environment

### Why Separate Databases per Service?

```
auth_db (Auth Service only)        ← Auth Service-এর private data
permission_db (Permission only)    ← Permission Service-এর private data
```

**Benefits:**
- **Loose coupling** — Auth Service-এর schema change Permission Service-কে break করবে না
- **Independent scaling** — High-traffic service-এর DB আলাদা scale
- **Failure isolation** — এক DB down → অন্য service চলতে পারে
- **Schema autonomy** — Each team independently evolves schema

**Trade-off:** Cross-service queries impossible — events/API calls দিয়ে data sync করতে হয়।

### Why JWT (Stateless Authentication)?

| Aspect | Stateless JWT | Stateful Session |
|--------|---------------|------------------|
| Storage | Client-side | Server-side |
| Scalability | ✅ Excellent | ❌ Sticky session needed |
| Microservices fit | ✅ Perfect | ❌ Session sharing complex |
| Logout | ⚠️ Tricky (blacklist) | ✅ Easy |
| Token revocation | ⚠️ Short-lived + Redis | ✅ Immediate |

JWT microservices-এ winner — কারণ Gateway এবং downstream services-এ session sharing complex।

### Why @PreAuthorize?

Already discussed at length — Spring Security ecosystem-এর full benefit:
- SpEL expressions
- Object-level security
- @PostAuthorize, @PostFilter
- Audit events
- Test support (`@WithMockUser`)

---

## 3. MSSQL Database Design

### 3.1 Auth Service Database (`auth_db`)

```sql
-- Database create
CREATE DATABASE auth_db;
GO

USE auth_db;
GO

-- Users table
CREATE TABLE users (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    email NVARCHAR(255) NOT NULL UNIQUE,
    password_hash NVARCHAR(255) NOT NULL,
    full_name NVARCHAR(255),
    enabled BIT NOT NULL DEFAULT 1,
    account_locked BIT NOT NULL DEFAULT 0,
    failed_login_attempts INT NOT NULL DEFAULT 0,
    last_login_at DATETIME2 NULL,
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    updated_at DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);

CREATE INDEX idx_users_email ON users(email);

-- User roles (many-to-many with role names as strings)
CREATE TABLE user_roles (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    role_name NVARCHAR(50) NOT NULL,
    assigned_at DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    
    CONSTRAINT fk_user_roles_user FOREIGN KEY (user_id) 
        REFERENCES users(id) ON DELETE CASCADE,
    CONSTRAINT uq_user_role UNIQUE (user_id, role_name)
);

CREATE INDEX idx_user_roles_user_id ON user_roles(user_id);

-- Refresh tokens (for token rotation)
CREATE TABLE refresh_tokens (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    user_id BIGINT NOT NULL,
    token NVARCHAR(500) NOT NULL UNIQUE,
    expires_at DATETIME2 NOT NULL,
    revoked BIT NOT NULL DEFAULT 0,
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    
    CONSTRAINT fk_refresh_user FOREIGN KEY (user_id) 
        REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_refresh_token ON refresh_tokens(token);
CREATE INDEX idx_refresh_user ON refresh_tokens(user_id);

-- Sample seed data
INSERT INTO users (email, password_hash, full_name, enabled) 
VALUES 
    ('admin@datasoft.com', '$2a$10$N9qo8uLOickgx2ZMRZoMye7NOmQgvnSLLTJ0dB7qOdOGM7GAVyfC.', 'Admin User', 1),
    ('hasib@datasoft.com', '$2a$10$N9qo8uLOickgx2ZMRZoMye7NOmQgvnSLLTJ0dB7qOdOGM7GAVyfC.', 'Hasibuzzaman', 1),
    ('karim@datasoft.com', '$2a$10$N9qo8uLOickgx2ZMRZoMye7NOmQgvnSLLTJ0dB7qOdOGM7GAVyfC.', 'Karim Rahman', 1);
-- All passwords are "password123" (BCrypt encoded)

INSERT INTO user_roles (user_id, role_name) VALUES
    (1, 'ADMIN'),
    (2, 'MANAGER'),
    (3, 'MANAGER');
```

**MSSQL-specific things to note:**
- `IDENTITY(1,1)` — auto-increment (PostgreSQL-এ `SERIAL`, MySQL-এ `AUTO_INCREMENT`)
- `NVARCHAR` — Unicode support (Bengali names!)
- `BIT` — boolean (0/1)
- `DATETIME2` — modern date/time type (better than old `DATETIME`)
- `SYSDATETIME()` — current timestamp
- `GO` — batch separator (SSMS/sqlcmd specific)

### 3.2 Permission Service Database (`permission_db`)

```sql
CREATE DATABASE permission_db;
GO

USE permission_db;
GO

-- Roles definition
CREATE TABLE roles (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    name NVARCHAR(50) NOT NULL UNIQUE,
    description NVARCHAR(255),
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);

-- All available permissions in system
CREATE TABLE permissions (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    name NVARCHAR(100) NOT NULL UNIQUE,         -- ORDER_DELETE, REPORT_EXPORT
    description NVARCHAR(255),
    service_name NVARCHAR(50) NOT NULL,         -- order-service, report-service
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME()
);

CREATE INDEX idx_permissions_service ON permissions(service_name);

-- Role-Permission mapping (many-to-many)
CREATE TABLE role_permissions (
    role_id BIGINT NOT NULL,
    permission_id BIGINT NOT NULL,
    
    CONSTRAINT pk_role_permissions PRIMARY KEY (role_id, permission_id),
    CONSTRAINT fk_rp_role FOREIGN KEY (role_id) 
        REFERENCES roles(id) ON DELETE CASCADE,
    CONSTRAINT fk_rp_permission FOREIGN KEY (permission_id) 
        REFERENCES permissions(id) ON DELETE CASCADE
);

CREATE INDEX idx_rp_role ON role_permissions(role_id);
CREATE INDEX idx_rp_permission ON role_permissions(permission_id);

-- 🔥 KEY TABLE: User-level permission overrides
CREATE TABLE user_permission_overrides (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    user_id BIGINT NOT NULL,                    -- Auth Service থেকে আসা userId
    permission_id BIGINT NOT NULL,
    override_type NVARCHAR(10) NOT NULL,        -- GRANT or REVOKE
    granted_by BIGINT NOT NULL,                 -- কোন admin করলো (auth user_id)
    reason NVARCHAR(500),
    expires_at DATETIME2 NULL,                  -- temporary override
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    
    CONSTRAINT fk_override_permission FOREIGN KEY (permission_id) 
        REFERENCES permissions(id),
    CONSTRAINT chk_override_type CHECK (override_type IN ('GRANT', 'REVOKE')),
    CONSTRAINT uq_user_permission UNIQUE (user_id, permission_id)
);

CREATE INDEX idx_overrides_user ON user_permission_overrides(user_id);
CREATE INDEX idx_overrides_expiry ON user_permission_overrides(expires_at);

-- Seed data: Roles
INSERT INTO roles (name, description) VALUES
    ('ADMIN', 'System Administrator'),
    ('MANAGER', 'Manager Role'),
    ('USER', 'Regular User');

-- Seed data: Permissions for Order Service
INSERT INTO permissions (name, description, service_name) VALUES
    ('ORDER_VIEW', 'View orders', 'order-service'),
    ('ORDER_CREATE', 'Create new orders', 'order-service'),
    ('ORDER_UPDATE', 'Update existing orders', 'order-service'),
    ('ORDER_DELETE', 'Delete orders', 'order-service'),
    ('ORDER_BULK_IMPORT', 'Bulk import orders', 'order-service'),
    ('REPORT_VIEW', 'View reports', 'order-service'),
    ('REPORT_EXPORT', 'Export reports', 'order-service'),
    ('USER_MANAGE', 'Manage users', 'auth-service'),
    ('PERMISSION_MANAGE', 'Manage permissions', 'permission-service');

-- ADMIN gets ALL permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 1, id FROM permissions;

-- MANAGER gets order management permissions
INSERT INTO role_permissions (role_id, permission_id)
SELECT 2, id FROM permissions 
WHERE name IN ('ORDER_VIEW', 'ORDER_CREATE', 'ORDER_UPDATE', 
               'REPORT_VIEW', 'REPORT_EXPORT');

-- USER gets read-only
INSERT INTO role_permissions (role_id, permission_id)
SELECT 3, id FROM permissions 
WHERE name IN ('ORDER_VIEW', 'REPORT_VIEW');

-- Example override: Hasib (user_id=2, MANAGER) এর REPORT_EXPORT revoked
INSERT INTO user_permission_overrides 
    (user_id, permission_id, override_type, granted_by, reason)
VALUES (
    2, 
    (SELECT id FROM permissions WHERE name = 'REPORT_EXPORT'),
    'REVOKE',
    1,  -- Admin user_id
    'Client request: temporary export restriction'
);
```

### 3.3 Order Service Database (`order_db`)

```sql
CREATE DATABASE order_db;
GO

USE order_db;
GO

CREATE TABLE orders (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    order_number NVARCHAR(50) NOT NULL UNIQUE,
    customer_name NVARCHAR(255) NOT NULL,
    total_amount DECIMAL(18, 2) NOT NULL,
    status NVARCHAR(20) NOT NULL DEFAULT 'PENDING',
    owner_id BIGINT NOT NULL,                   -- কোন user create করেছে
    created_at DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    updated_at DATETIME2 NOT NULL DEFAULT SYSDATETIME(),
    
    CONSTRAINT chk_status CHECK (status IN ('PENDING', 'CONFIRMED', 'SHIPPED', 'DELIVERED', 'CANCELLED'))
);

CREATE INDEX idx_orders_owner ON orders(owner_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_number ON orders(order_number);

-- Sample data
INSERT INTO orders (order_number, customer_name, total_amount, status, owner_id) VALUES
    ('ORD-001', 'ABC Corp', 15000.00, 'PENDING', 2),
    ('ORD-002', 'XYZ Ltd', 25000.00, 'CONFIRMED', 2),
    ('ORD-003', 'PQR Inc', 8500.00, 'PENDING', 3);
```

---

## 4. Project Structure

```
datasoft-microservices/
│
├── docker-compose.yml              # All infrastructure
├── pom.xml                         # Parent POM (multi-module)
│
├── common/                         # Shared module
│   ├── pom.xml
│   └── src/main/java/com/datasoft/common/
│       ├── dto/
│       ├── exception/
│       └── security/               # Common security classes
│
├── gateway/
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/main/java/com/datasoft/gateway/
│       ├── GatewayApplication.java
│       ├── config/
│       └── filter/
│
├── auth-service/
│   ├── pom.xml
│   ├── Dockerfile
│   └── src/main/java/com/datasoft/auth/
│       ├── AuthApplication.java
│       ├── config/
│       ├── controller/
│       ├── service/
│       ├── repository/
│       ├── entity/
│       ├── dto/
│       └── security/
│
├── permission-service/
│   └── (similar structure)
│
└── order-service/
    └── (similar structure)
```

### Parent POM (Multi-Module Maven)

```xml
<!-- pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>
    
    <groupId>com.datasoft</groupId>
    <artifactId>microservices-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <modules>
        <module>common</module>
        <module>gateway</module>
        <module>auth-service</module>
        <module>permission-service</module>
        <module>order-service</module>
    </modules>
    
    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.0</spring-cloud.version>
        <jjwt.version>0.12.3</jjwt.version>
    </properties>
    
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

---

## 5. Auth Service Implementation

### 5.1 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>com.datasoft</groupId>
        <artifactId>microservices-parent</artifactId>
        <version>1.0.0</version>
    </parent>
    
    <artifactId>auth-service</artifactId>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        
        <!-- 🔥 MSSQL JDBC Driver -->
        <dependency>
            <groupId>com.microsoft.sqlserver</groupId>
            <artifactId>mssql-jdbc</artifactId>
        </dependency>
        
        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>${jjwt.version}</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>${jjwt.version}</version>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Common module -->
        <dependency>
            <groupId>com.datasoft</groupId>
            <artifactId>common</artifactId>
            <version>1.0.0</version>
        </dependency>
        
        <!-- Lombok (optional but useful) -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Test -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

### 5.2 application.yml (MSSQL Configuration)

```yaml
server:
  port: 8081

spring:
  application:
    name: auth-service
  
  datasource:
    # 🔥 MSSQL Connection String
    url: jdbc:sqlserver://${DB_HOST:localhost}:${DB_PORT:1433};databaseName=auth_db;encrypt=true;trustServerCertificate=true
    username: ${DB_USERNAME:sa}
    password: ${DB_PASSWORD:YourStrong@Pass123}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
  
  jpa:
    database-platform: org.hibernate.dialect.SQLServerDialect
    hibernate:
      ddl-auto: validate    # Production-এ validate, dev-এ update
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        jdbc:
          time_zone: Asia/Dhaka

# JWT Configuration
jwt:
  secret: ${JWT_SECRET:VGhpc0lzQVZlcnlMb25nU2VjcmV0S2V5Rm9ySldUVG9rZW5HZW5lcmF0aW9uMjAyNQ==}
  access-token-expiration: 900000      # 15 minutes
  refresh-token-expiration: 604800000   # 7 days
  issuer: auth.datasoft.com

logging:
  level:
    com.datasoft: DEBUG
    org.springframework.security: DEBUG
```

**MSSQL Connection String Breakdown:**

```
jdbc:sqlserver://localhost:1433;databaseName=auth_db;encrypt=true;trustServerCertificate=true
       │            │      │              │              │              │
       │            │      │              │              │              └─ Self-signed cert allow (dev only!)
       │            │      │              │              └─ TLS encryption
       │            │      │              └─ Database name
       │            │      └─ Default MSSQL port
       │            └─ Server hostname
       └─ JDBC protocol for SQL Server
```

### 5.3 Entity Classes

```java
// User.java
package com.datasoft.auth.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 🔥 MSSQL IDENTITY
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String email;
    
    @Column(name = "password_hash", nullable = false)
    private String passwordHash;
    
    @Column(name = "full_name")
    private String fullName;
    
    @Column(nullable = false)
    private Boolean enabled = true;
    
    @Column(name = "account_locked", nullable = false)
    private Boolean accountLocked = false;
    
    @Column(name = "failed_login_attempts", nullable = false)
    private Integer failedLoginAttempts = 0;
    
    @Column(name = "last_login_at")
    private LocalDateTime lastLoginAt;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    @Builder.Default
    private Set<UserRole> roles = new HashSet<>();
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    // Helper method
    public Set<String> getRoleNames() {
        Set<String> names = new HashSet<>();
        for (UserRole r : roles) {
            names.add(r.getRoleName());
        }
        return names;
    }
}
```

```java
// UserRole.java
package com.datasoft.auth.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "user_roles")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UserRole {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(name = "role_name", nullable = false, length = 50)
    private String roleName;
    
    @Column(name = "assigned_at", nullable = false, updatable = false)
    private LocalDateTime assignedAt;
    
    @PrePersist
    protected void onCreate() {
        assignedAt = LocalDateTime.now();
    }
}
```

```java
// RefreshToken.java
package com.datasoft.auth.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "refresh_tokens")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class RefreshToken {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @Column(nullable = false, unique = true, length = 500)
    private String token;
    
    @Column(name = "expires_at", nullable = false)
    private LocalDateTime expiresAt;
    
    @Column(nullable = false)
    private Boolean revoked = false;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}
```

### 5.4 Repository Layer

```java
// UserRepository.java
package com.datasoft.auth.repository;

import com.datasoft.auth.entity.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

```java
// RefreshTokenRepository.java
package com.datasoft.auth.repository;

import com.datasoft.auth.entity.RefreshToken;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByToken(String token);
    
    @Modifying
    @Query("UPDATE RefreshToken r SET r.revoked = true WHERE r.userId = :userId")
    void revokeAllByUserId(Long userId);
}
```

### 5.5 JWT Utility

```java
// JwtUtil.java
package com.datasoft.auth.security;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;

@Component
public class JwtUtil {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.access-token-expiration}")
    private Long accessTokenExpiration;
    
    @Value("${jwt.refresh-token-expiration}")
    private Long refreshTokenExpiration;
    
    @Value("${jwt.issuer}")
    private String issuer;
    
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
    
    /**
     * Access Token generate করে — short-lived (15 min)
     * Roles embed করে কিন্তু permissions না (size optimization)
     */
    public String generateAccessToken(Long userId, String email, Set<String> roles) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("email", email);
        claims.put("roles", roles);
        claims.put("type", "ACCESS");
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userId.toString())
                .setIssuer(issuer)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + accessTokenExpiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }
    
    /**
     * Refresh Token generate — long-lived (7 days)
     * Minimal claims, only userId
     */
    public String generateRefreshToken(Long userId) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("type", "REFRESH");
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userId.toString())
                .setIssuer(issuer)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + refreshTokenExpiration))
                .signWith(getSigningKey(), SignatureAlgorithm.HS256)
                .compact();
    }
    
    public Claims validateAndParseClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .requireIssuer(issuer)
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
    
    public Long getUserIdFromToken(String token) {
        return Long.parseLong(validateAndParseClaims(token).getSubject());
    }
    
    public boolean isTokenValid(String token) {
        try {
            validateAndParseClaims(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

### 5.6 Security Configuration

```java
// SecurityConfig.java
package com.datasoft.auth.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // strength 12 — production grade
    }
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()  // public endpoints
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            );
        
        return http.build();
    }
}
```

### 5.7 DTOs

```java
// LoginRequest.java
package com.datasoft.auth.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;

@Data
public class LoginRequest {
    @NotBlank @Email
    private String email;
    
    @NotBlank
    private String password;
}

// RegisterRequest.java
@Data
public class RegisterRequest {
    @NotBlank @Email
    private String email;
    
    @NotBlank 
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
    
    @NotBlank
    private String fullName;
}

// AuthResponse.java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    private String tokenType = "Bearer";
    private Long expiresIn;
    private UserInfo user;
}

// UserInfo.java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Builder
public class UserInfo {
    private Long id;
    private String email;
    private String fullName;
    private Set<String> roles;
}
```

### 5.8 AuthService

```java
// AuthService.java
package com.datasoft.auth.service;

import com.datasoft.auth.dto.*;
import com.datasoft.auth.entity.*;
import com.datasoft.auth.repository.*;
import com.datasoft.auth.security.JwtUtil;
import com.datasoft.common.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Service
@RequiredArgsConstructor
@Transactional
public class AuthService {
    
    private final UserRepository userRepository;
    private final RefreshTokenRepository refreshTokenRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtUtil jwtUtil;
    
    public AuthResponse register(RegisterRequest request) {
        // Email uniqueness check
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("EMAIL_EXISTS", "এই email already registered");
        }
        
        // Create user
        User user = User.builder()
                .email(request.getEmail())
                .passwordHash(passwordEncoder.encode(request.getPassword()))
                .fullName(request.getFullName())
                .enabled(true)
                .accountLocked(false)
                .failedLoginAttempts(0)
                .build();
        
        // Default role: USER
        UserRole defaultRole = UserRole.builder()
                .user(user)
                .roleName("USER")
                .build();
        user.getRoles().add(defaultRole);
        
        user = userRepository.save(user);
        
        // Generate tokens
        return generateAuthResponse(user);
    }
    
    public AuthResponse login(LoginRequest request) {
        User user = userRepository.findByEmail(request.getEmail())
                .orElseThrow(() -> new BusinessException("INVALID_CREDENTIALS", "Invalid email or password"));
        
        // Account checks
        if (!user.getEnabled()) {
            throw new BusinessException("ACCOUNT_DISABLED", "Account disabled");
        }
        if (user.getAccountLocked()) {
            throw new BusinessException("ACCOUNT_LOCKED", "Account locked due to multiple failed attempts");
        }
        
        // Password verify
        if (!passwordEncoder.matches(request.getPassword(), user.getPasswordHash())) {
            // Failed attempts increment
            user.setFailedLoginAttempts(user.getFailedLoginAttempts() + 1);
            if (user.getFailedLoginAttempts() >= 5) {
                user.setAccountLocked(true);
            }
            userRepository.save(user);
            throw new BusinessException("INVALID_CREDENTIALS", "Invalid email or password");
        }
        
        // Reset failed attempts on success
        user.setFailedLoginAttempts(0);
        user.setLastLoginAt(LocalDateTime.now());
        userRepository.save(user);
        
        return generateAuthResponse(user);
    }
    
    public AuthResponse refreshToken(String refreshToken) {
        // Validate refresh token signature
        if (!jwtUtil.isTokenValid(refreshToken)) {
            throw new BusinessException("INVALID_TOKEN", "Invalid refresh token");
        }
        
        // Check DB for revocation
        RefreshToken stored = refreshTokenRepository.findByToken(refreshToken)
                .orElseThrow(() -> new BusinessException("TOKEN_NOT_FOUND", "Refresh token not found"));
        
        if (stored.getRevoked()) {
            throw new BusinessException("TOKEN_REVOKED", "Token has been revoked");
        }
        
        if (stored.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new BusinessException("TOKEN_EXPIRED", "Refresh token expired");
        }
        
        // Get user
        Long userId = jwtUtil.getUserIdFromToken(refreshToken);
        User user = userRepository.findById(userId)
                .orElseThrow(() -> new BusinessException("USER_NOT_FOUND", "User not found"));
        
        // Token rotation: revoke old, issue new
        stored.setRevoked(true);
        refreshTokenRepository.save(stored);
        
        return generateAuthResponse(user);
    }
    
    public void logout(Long userId) {
        // Revoke all refresh tokens
        refreshTokenRepository.revokeAllByUserId(userId);
        // Note: Access token expire হওয়া পর্যন্ত valid থাকবে
        // True immediate logout হলে Redis blacklist use করতে হবে
    }
    
    private AuthResponse generateAuthResponse(User user) {
        Set<String> roles = user.getRoleNames();
        
        String accessToken = jwtUtil.generateAccessToken(user.getId(), user.getEmail(), roles);
        String refreshTokenStr = jwtUtil.generateRefreshToken(user.getId());
        
        // Store refresh token in DB
        RefreshToken refreshToken = RefreshToken.builder()
                .userId(user.getId())
                .token(refreshTokenStr)
                .expiresAt(LocalDateTime.now().plusDays(7))
                .revoked(false)
                .build();
        refreshTokenRepository.save(refreshToken);
        
        UserInfo userInfo = UserInfo.builder()
                .id(user.getId())
                .email(user.getEmail())
                .fullName(user.getFullName())
                .roles(roles)
                .build();
        
        return AuthResponse.builder()
                .accessToken(accessToken)
                .refreshToken(refreshTokenStr)
                .tokenType("Bearer")
                .expiresIn(900L)  // 15 min in seconds
                .user(userInfo)
                .build();
    }
}
```

### 5.9 AuthController

```java
// AuthController.java
package com.datasoft.auth.controller;

import com.datasoft.auth.dto.*;
import com.datasoft.auth.service.AuthService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    
    private final AuthService authService;
    
    @PostMapping("/register")
    public ResponseEntity<AuthResponse> register(@Valid @RequestBody RegisterRequest request) {
        return ResponseEntity.ok(authService.register(request));
    }
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@Valid @RequestBody LoginRequest request) {
        return ResponseEntity.ok(authService.login(request));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(@RequestBody RefreshRequest request) {
        return ResponseEntity.ok(authService.refreshToken(request.getRefreshToken()));
    }
    
    @PostMapping("/logout")
    public ResponseEntity<Void> logout(@RequestHeader("X-User-Id") Long userId) {
        authService.logout(userId);
        return ResponseEntity.noContent().build();
    }
    
    @GetMapping("/me")
    public ResponseEntity<UserInfo> getCurrentUser(@RequestHeader("X-User-Id") Long userId) {
        // Implementation: fetch user details
        // ...
    }
}
```

---

## 6. Permission Service Implementation

### 6.1 application.yml

```yaml
server:
  port: 8082

spring:
  application:
    name: permission-service
  
  datasource:
    url: jdbc:sqlserver://${DB_HOST:localhost}:${DB_PORT:1433};databaseName=permission_db;encrypt=true;trustServerCertificate=true
    username: ${DB_USERNAME:sa}
    password: ${DB_PASSWORD:YourStrong@Pass123}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
  
  jpa:
    database-platform: org.hibernate.dialect.SQLServerDialect
    hibernate:
      ddl-auto: validate
    show-sql: true
  
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKER:localhost:9092}
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer

permission:
  cache:
    ttl-minutes: 5
```

### 6.2 Entities

```java
// Role.java
package com.datasoft.permission.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "roles")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Role {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 50)
    private String name;
    
    @Column(length = 255)
    private String description;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    @Builder.Default
    private Set<Permission> permissions = new HashSet<>();
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}
```

```java
// Permission.java
@Entity
@Table(name = "permissions")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Permission {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true, length = 100)
    private String name;
    
    @Column(length = 255)
    private String description;
    
    @Column(name = "service_name", nullable = false, length = 50)
    private String serviceName;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}
```

```java
// UserPermissionOverride.java
@Entity
@Table(name = "user_permission_overrides")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class UserPermissionOverride {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "user_id", nullable = false)
    private Long userId;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "permission_id", nullable = false)
    private Permission permission;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "override_type", nullable = false, length = 10)
    private OverrideType overrideType;
    
    @Column(name = "granted_by", nullable = false)
    private Long grantedBy;
    
    @Column(length = 500)
    private String reason;
    
    @Column(name = "expires_at")
    private LocalDateTime expiresAt;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
    
    public enum OverrideType {
        GRANT, REVOKE
    }
}
```

### 6.3 Repositories

```java
// PermissionRepository.java
@Repository
public interface PermissionRepository extends JpaRepository<Permission, Long> {
    Optional<Permission> findByName(String name);
}

// RoleRepository.java
@Repository
public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(String name);
}

// UserPermissionOverrideRepository.java
@Repository
public interface UserPermissionOverrideRepository 
        extends JpaRepository<UserPermissionOverride, Long> {
    
    List<UserPermissionOverride> findByUserId(Long userId);
    
    @Query("""
        SELECT o FROM UserPermissionOverride o 
        WHERE o.userId = :userId 
        AND (o.expiresAt IS NULL OR o.expiresAt > :now)
    """)
    List<UserPermissionOverride> findActiveByUserId(
        @Param("userId") Long userId, 
        @Param("now") LocalDateTime now);
    
    Optional<UserPermissionOverride> findByUserIdAndPermissionId(Long userId, Long permissionId);
}
```

### 6.4 Permission Calculation Service

```java
// PermissionService.java
package com.datasoft.permission.service;

import com.datasoft.permission.entity.*;
import com.datasoft.permission.repository.*;
import com.datasoft.permission.event.PermissionChangedEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Duration;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class PermissionService {
    
    private final PermissionRepository permissionRepository;
    private final RoleRepository roleRepository;
    private final UserPermissionOverrideRepository overrideRepository;
    private final RedisTemplate<String, Set<String>> redisTemplate;
    private final KafkaTemplate<String, PermissionChangedEvent> kafkaTemplate;
    
    @Value("${permission.cache.ttl-minutes}")
    private int cacheTtlMinutes;
    
    private static final String CACHE_KEY_PREFIX = "user_permissions:";
    private static final String KAFKA_TOPIC = "permission.changed";
    
    /**
     * 🔥 CORE METHOD: User এর effective permissions calculate করে
     * 
     * Formula: (Role Permissions - Revoked Overrides) + Granted Overrides
     */
    @Transactional(readOnly = true)
    public Set<String> getEffectivePermissions(Long userId, Set<String> userRoles) {
        String cacheKey = CACHE_KEY_PREFIX + userId;
        
        // 1. Redis cache check
        Set<String> cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            log.debug("Cache HIT for user {}", userId);
            return cached;
        }
        
        log.debug("Cache MISS for user {}, calculating from DB", userId);
        
        // 2. Role-based permissions fetch
        Set<String> rolePermissions = new HashSet<>();
        for (String roleName : userRoles) {
            Optional<Role> role = roleRepository.findByName(roleName);
            role.ifPresent(r -> {
                Set<String> permNames = r.getPermissions().stream()
                    .map(Permission::getName)
                    .collect(Collectors.toSet());
                rolePermissions.addAll(permNames);
            });
        }
        
        // 3. User overrides fetch
        List<UserPermissionOverride> overrides = 
            overrideRepository.findActiveByUserId(userId, LocalDateTime.now());
        
        Set<String> grants = overrides.stream()
            .filter(o -> o.getOverrideType() == UserPermissionOverride.OverrideType.GRANT)
            .map(o -> o.getPermission().getName())
            .collect(Collectors.toSet());
        
        Set<String> revokes = overrides.stream()
            .filter(o -> o.getOverrideType() == UserPermissionOverride.OverrideType.REVOKE)
            .map(o -> o.getPermission().getName())
            .collect(Collectors.toSet());
        
        // 4. 🎯 Effective permissions calculation
        Set<String> effective = new HashSet<>(rolePermissions);
        effective.removeAll(revokes);     // Revoke first
        effective.addAll(grants);         // Then add grants
        
        // 5. Cache the result
        redisTemplate.opsForValue().set(cacheKey, effective, Duration.ofMinutes(cacheTtlMinutes));
        
        return effective;
    }
    
    /**
     * Permission check — fast path
     */
    public boolean hasPermission(Long userId, Set<String> userRoles, String permission) {
        return getEffectivePermissions(userId, userRoles).contains(permission);
    }
    
    /**
     * Admin override add করে
     */
    @Transactional
    public void addOverride(Long userId, String permissionName, 
                           UserPermissionOverride.OverrideType type,
                           Long grantedBy, String reason, LocalDateTime expiresAt) {
        
        Permission permission = permissionRepository.findByName(permissionName)
            .orElseThrow(() -> new RuntimeException("Permission not found: " + permissionName));
        
        // Existing override check
        Optional<UserPermissionOverride> existing = 
            overrideRepository.findByUserIdAndPermissionId(userId, permission.getId());
        
        UserPermissionOverride override;
        if (existing.isPresent()) {
            // Update
            override = existing.get();
            override.setOverrideType(type);
            override.setReason(reason);
            override.setExpiresAt(expiresAt);
            override.setGrantedBy(grantedBy);
        } else {
            // Create new
            override = UserPermissionOverride.builder()
                .userId(userId)
                .permission(permission)
                .overrideType(type)
                .grantedBy(grantedBy)
                .reason(reason)
                .expiresAt(expiresAt)
                .build();
        }
        
        overrideRepository.save(override);
        
        // 🔥 Cache invalidate + event publish
        invalidateCache(userId);
        publishPermissionChange(userId, permissionName, type);
        
        log.info("Override {} for user {} on permission {} by admin {}", 
            type, userId, permissionName, grantedBy);
    }
    
    @Transactional
    public void removeOverride(Long userId, String permissionName) {
        Permission permission = permissionRepository.findByName(permissionName)
            .orElseThrow(() -> new RuntimeException("Permission not found"));
        
        overrideRepository.findByUserIdAndPermissionId(userId, permission.getId())
            .ifPresent(o -> {
                overrideRepository.delete(o);
                invalidateCache(userId);
                publishPermissionChange(userId, permissionName, null);
            });
    }
    
    private void invalidateCache(Long userId) {
        redisTemplate.delete(CACHE_KEY_PREFIX + userId);
        log.debug("Cache invalidated for user {}", userId);
    }
    
    private void publishPermissionChange(Long userId, String permission, 
                                          UserPermissionOverride.OverrideType type) {
        PermissionChangedEvent event = PermissionChangedEvent.builder()
            .userId(userId)
            .permission(permission)
            .changeType(type != null ? type.name() : "REMOVED")
            .timestamp(LocalDateTime.now())
            .build();
        
        kafkaTemplate.send(KAFKA_TOPIC, userId.toString(), event);
        log.info("Published permission change event for user {}", userId);
    }
}
```

### 6.5 Event DTO

```java
// PermissionChangedEvent.java
package com.datasoft.permission.event;

import lombok.*;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PermissionChangedEvent {
    private Long userId;
    private String permission;
    private String changeType;  // GRANT, REVOKE, REMOVED
    private LocalDateTime timestamp;
}
```

### 6.6 Permission Controller

```java
// PermissionController.java
package com.datasoft.permission.controller;

import com.datasoft.permission.dto.*;
import com.datasoft.permission.entity.UserPermissionOverride;
import com.datasoft.permission.service.PermissionService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

@RestController
@RequestMapping("/api/permissions")
@RequiredArgsConstructor
public class PermissionController {
    
    private final PermissionService permissionService;
    
    /**
     * Effective permissions fetch (downstream services use)
     */
    @GetMapping("/effective")
    public ResponseEntity<EffectivePermissionsResponse> getEffective(
            @RequestParam Long userId,
            @RequestParam String roles) {
        
        Set<String> userRoles = new HashSet<>(Arrays.asList(roles.split(",")));
        Set<String> permissions = permissionService.getEffectivePermissions(userId, userRoles);
        
        return ResponseEntity.ok(new EffectivePermissionsResponse(userId, permissions));
    }
    
    /**
     * Single permission check (faster)
     */
    @GetMapping("/check")
    public ResponseEntity<PermissionCheckResponse> check(
            @RequestParam Long userId,
            @RequestParam String roles,
            @RequestParam String permission) {
        
        Set<String> userRoles = new HashSet<>(Arrays.asList(roles.split(",")));
        boolean hasPermission = permissionService.hasPermission(userId, userRoles, permission);
        
        return ResponseEntity.ok(new PermissionCheckResponse(userId, permission, hasPermission));
    }
    
    /**
     * Admin: Override add
     */
    @PostMapping("/overrides")
    public ResponseEntity<Void> addOverride(
            @RequestBody OverrideRequest request,
            @RequestHeader("X-User-Id") Long adminId) {
        
        permissionService.addOverride(
            request.getUserId(),
            request.getPermission(),
            UserPermissionOverride.OverrideType.valueOf(request.getOverrideType()),
            adminId,
            request.getReason(),
            request.getExpiresAt()
        );
        
        return ResponseEntity.ok().build();
    }
    
    /**
     * Admin: Override remove
     */
    @DeleteMapping("/overrides")
    public ResponseEntity<Void> removeOverride(
            @RequestParam Long userId,
            @RequestParam String permission) {
        
        permissionService.removeOverride(userId, permission);
        return ResponseEntity.noContent().build();
    }
}
```

---

### 6.7 Redis Configuration (Permission Service)

```java
// RedisConfig.java
package com.datasoft.permission.config;

import com.fasterxml.jackson.annotation.JsonAutoDetect;
import com.fasterxml.jackson.annotation.PropertyAccessor;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.util.Set;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Set<String>> redisTemplate(RedisConnectionFactory factory) {
        RedisTemplate<String, Set<String>> template = new RedisTemplate<>();
        template.setConnectionFactory(factory);

        // Key: plain String serialize
        template.setKeySerializer(new StringRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());

        // Value: JSON serialize — Set<String> handle করতে পারবে
        ObjectMapper mapper = new ObjectMapper();
        mapper.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        mapper.activateDefaultTyping(
            mapper.getPolymorphicTypeValidator(),
            ObjectMapper.DefaultTyping.NON_FINAL
        );
        GenericJackson2JsonRedisSerializer jsonSerializer =
            new GenericJackson2JsonRedisSerializer(mapper);

        template.setValueSerializer(jsonSerializer);
        template.setHashValueSerializer(jsonSerializer);
        template.afterPropertiesSet();
        return template;
    }
}
```

### 6.8 Kafka Producer Configuration (Permission Service)

```java
// KafkaProducerConfig.java
package com.datasoft.permission.config;

import com.datasoft.permission.event.PermissionChangedEvent;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.JsonSerializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaProducerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ProducerFactory<String, PermissionChangedEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);

        // Reliability settings — production grade
        config.put(ProducerConfig.ACKS_CONFIG, "all");           // সব replica acknowledge করলে তবেই success
        config.put(ProducerConfig.RETRIES_CONFIG, 3);            // failure হলে 3 বার retry
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // duplicate message পাঠাবে না
        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, PermissionChangedEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }
}
```

---

## 7. Order Service Implementation

এটাই সবচেয়ে important — কারণ এখানে `@PreAuthorize` actually use হবে।

### 7.1 application.yml

```yaml
server:
  port: 8083

spring:
  application:
    name: order-service
  
  datasource:
    url: jdbc:sqlserver://${DB_HOST:localhost}:${DB_PORT:1433};databaseName=order_db;encrypt=true;trustServerCertificate=true
    username: ${DB_USERNAME:sa}
    password: ${DB_PASSWORD:YourStrong@Pass123}
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver
  
  jpa:
    database-platform: org.hibernate.dialect.SQLServerDialect
    hibernate:
      ddl-auto: validate
  
  kafka:
    bootstrap-servers: ${KAFKA_BROKER:localhost:9092}
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      properties:
        spring.json.trusted.packages: "*"

services:
  permission:
    url: ${PERMISSION_SERVICE_URL:http://localhost:8082}

permission:
  cache:
    max-size: 10000
    expire-minutes: 5
```

### 7.2 Permission Client (Feign)

```java
// PermissionClient.java
package com.datasoft.order.client;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

import java.util.Set;

@FeignClient(name = "permission-service", url = "${services.permission.url}")
public interface PermissionClient {
    
    @GetMapping("/api/permissions/effective")
    EffectivePermissionsResponse getEffectivePermissions(
        @RequestParam Long userId,
        @RequestParam String roles
    );
}
```

### 7.3 Permission Cache (Caffeine)

```java
// PermissionCache.java
package com.datasoft.order.security;

import com.datasoft.order.client.PermissionClient;
import com.github.benmanes.caffeine.cache.Cache;
import com.github.benmanes.caffeine.cache.Caffeine;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Component;

import jakarta.annotation.PostConstruct;
import java.time.Duration;
import java.util.Set;

@Component
@Slf4j
public class PermissionCache {
    
    private final PermissionClient permissionClient;
    
    @Value("${permission.cache.max-size}")
    private int maxSize;
    
    @Value("${permission.cache.expire-minutes}")
    private int expireMinutes;
    
    private Cache<String, Set<String>> cache;
    
    public PermissionCache(PermissionClient permissionClient) {
        this.permissionClient = permissionClient;
    }
    
    @PostConstruct
    public void init() {
        cache = Caffeine.newBuilder()
            .maximumSize(maxSize)
            .expireAfterWrite(Duration.ofMinutes(expireMinutes))
            .recordStats()  // Metrics for monitoring
            .build();
        log.info("Permission cache initialized: maxSize={}, ttl={}min", maxSize, expireMinutes);
    }
    
    /**
     * Get permissions — cache first, fallback to Permission Service
     */
    public Set<String> getPermissions(Long userId, Set<String> roles) {
        String key = buildKey(userId, roles);
        
        return cache.get(key, k -> {
            log.debug("Cache miss for {}, calling Permission Service", key);
            String rolesStr = String.join(",", roles);
            return permissionClient.getEffectivePermissions(userId, rolesStr).getPermissions();
        });
    }
    
    public boolean hasPermission(Long userId, Set<String> roles, String permission) {
        return getPermissions(userId, roles).contains(permission);
    }
    
    /**
     * 🔥 Kafka listener: Permission change event ashle cache invalidate
     */
    @KafkaListener(topics = "permission.changed", groupId = "order-service")
    public void onPermissionChanged(PermissionChangedEvent event) {
        log.info("Received permission change event for user {}", event.getUserId());
        // User এর সব cached entry invalidate করো
        invalidateUser(event.getUserId());
    }
    
    public void invalidateUser(Long userId) {
        // Caffeine doesn't support prefix delete natively
        // So we iterate (in production, consider a different cache structure)
        cache.asMap().keySet().removeIf(key -> key.startsWith(userId + ":"));
        log.debug("Invalidated cache for user {}", userId);
    }
    
    private String buildKey(Long userId, Set<String> roles) {
        return userId + ":" + String.join(",", roles);
    }
}
```

### 7.4 Kafka Consumer Configuration (Order Service)

```java
// KafkaConsumerConfig.java
package com.datasoft.order.config;

import com.datasoft.order.event.PermissionChangedEvent;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.core.*;
import org.springframework.kafka.support.serializer.ErrorHandlingDeserializer;
import org.springframework.kafka.support.serializer.JsonDeserializer;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaConsumerConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Bean
    public ConsumerFactory<String, PermissionChangedEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-service");
        config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

        // ErrorHandlingDeserializer wrap — deserialization error হলে crash করবে না
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, ErrorHandlingDeserializer.class);
        config.put(ErrorHandlingDeserializer.KEY_DESERIALIZER_CLASS, StringDeserializer.class);
        config.put(ErrorHandlingDeserializer.VALUE_DESERIALIZER_CLASS, JsonDeserializer.class);

        // JSON config
        config.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        config.put(JsonDeserializer.VALUE_DEFAULT_TYPE, PermissionChangedEvent.class.getName());
        config.put(JsonDeserializer.USE_TYPE_INFO_HEADERS, false);

        return new DefaultKafkaConsumerFactory<>(config);
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, PermissionChangedEvent>
            kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, PermissionChangedEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }
}
```

### 7.5 Permission Changed Event DTO (Order Service Consumer Side)

```java
// PermissionChangedEvent.java
// Note: এটা Order Service এর নিজস্ব copy — Permission Service এর Event DTO-র mirror
package com.datasoft.order.event;

import lombok.*;
import java.time.LocalDateTime;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PermissionChangedEvent {
    private Long userId;
    private String permission;
    private String changeType;  // GRANT, REVOKE, REMOVED
    private LocalDateTime timestamp;
}
```

### 7.6 Feign Configuration (Order Service)

```java
// FeignConfig.java
package com.datasoft.order.config;

import feign.Logger;
import feign.RequestInterceptor;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.context.request.RequestContextHolder;
import org.springframework.web.context.request.ServletRequestAttributes;

@Configuration
@EnableFeignClients(basePackages = "com.datasoft.order.client")
public class FeignConfig {

    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC;
    }

    /**
     * 🔥 Downstream Feign call এর সাথে Gateway-injected headers forward করে
     * নাহলে Permission Service এ user context পাবে না
     */
    @Bean
    public RequestInterceptor requestInterceptor() {
        return template -> {
            ServletRequestAttributes attrs = (ServletRequestAttributes)
                RequestContextHolder.getRequestAttributes();

            if (attrs != null) {
                String userId = attrs.getRequest().getHeader("X-User-Id");
                String roles  = attrs.getRequest().getHeader("X-User-Roles");

                if (userId != null) template.header("X-User-Id", userId);
                if (roles  != null) template.header("X-User-Roles", roles);
            }
        };
    }
}
```

### 7.7 Security Configuration with Custom PermissionEvaluator

```java
// CustomPermissionEvaluator.java
package com.datasoft.order.security;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.stereotype.Component;

import java.io.Serializable;
import java.util.Set;
import java.util.stream.Collectors;

@Component
@RequiredArgsConstructor
@Slf4j
public class CustomPermissionEvaluator implements PermissionEvaluator {
    
    private final PermissionCache permissionCache;
    
    /**
     * @PreAuthorize("hasPermission(null, 'ORDER_DELETE')") এই method call করবে
     */
    @Override
    public boolean hasPermission(Authentication auth, Object targetDomainObject, Object permission) {
        if (auth == null || !auth.isAuthenticated()) {
            return false;
        }
        
        Long userId = Long.parseLong(auth.getName());
        Set<String> roles = extractRoles(auth);
        String permName = permission.toString();
        
        boolean hasPermission = permissionCache.hasPermission(userId, roles, permName);
        log.debug("Permission check: user={}, permission={}, result={}", userId, permName, hasPermission);
        
        return hasPermission;
    }
    
    /**
     * Object-level: @PreAuthorize("hasPermission(#orderId, 'Order', 'DELETE')")
     */
    @Override
    public boolean hasPermission(Authentication auth, Serializable targetId, 
                                  String targetType, Object permission) {
        // Future: Object-specific check (e.g., ownership)
        return hasPermission(auth, null, permission);
    }
    
    private Set<String> extractRoles(Authentication auth) {
        return auth.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .filter(a -> a.startsWith("ROLE_"))
            .map(a -> a.substring(5))  // Remove "ROLE_" prefix
            .collect(Collectors.toSet());
    }
}
```

```java
// SecurityConfig.java
package com.datasoft.order.config;

import com.datasoft.order.security.*;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.access.PermissionEvaluator;
import org.springframework.security.access.expression.method.DefaultMethodSecurityExpressionHandler;
import org.springframework.security.access.expression.method.MethodSecurityExpressionHandler;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableMethodSecurity(prePostEnabled = true)  // 🔥 @PreAuthorize enable
public class SecurityConfig {
    
    @Autowired
    private CustomPermissionEvaluator permissionEvaluator;
    
    @Autowired
    private HeaderToSecurityContextFilter headerFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )
            // 🔥 Header থেকে SecurityContext populate
            .addFilterBefore(headerFilter, 
                org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter.class);
        
        return http.build();
    }
    
    @Bean
    public MethodSecurityExpressionHandler methodSecurityExpressionHandler() {
        DefaultMethodSecurityExpressionHandler handler = new DefaultMethodSecurityExpressionHandler();
        handler.setPermissionEvaluator(permissionEvaluator);  // 🔥 Custom evaluator inject
        return handler;
    }
}
```

### 7.8 Header-to-SecurityContext Filter

```java
// HeaderToSecurityContextFilter.java
package com.datasoft.order.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

@Component
@Slf4j
public class HeaderToSecurityContextFilter extends OncePerRequestFilter {
    
    private static final String USER_ID_HEADER = "X-User-Id";
    private static final String ROLES_HEADER = "X-User-Roles";
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        
        String userId = request.getHeader(USER_ID_HEADER);
        String rolesHeader = request.getHeader(ROLES_HEADER);
        
        if (userId != null && rolesHeader != null) {
            // 🔥 ROLE_ prefix add করো (Spring Security convention)
            List<SimpleGrantedAuthority> authorities = Arrays.stream(rolesHeader.split(","))
                .map(String::trim)
                .filter(r -> !r.isEmpty())
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
            
            UsernamePasswordAuthenticationToken auth = new UsernamePasswordAuthenticationToken(
                userId,           // principal = userId string
                null,             // credentials
                authorities       // ROLE_ADMIN, ROLE_MANAGER ইত্যাদি
            );
            
            SecurityContextHolder.getContext().setAuthentication(auth);
            log.debug("SecurityContext populated for user {}", userId);
        }
        
        chain.doFilter(request, response);
    }
}
```

### 7.9 Order Entity & Repository

```java
// Order.java
@Entity
@Table(name = "orders")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", nullable = false, unique = true, length = 50)
    private String orderNumber;
    
    @Column(name = "customer_name", nullable = false, length = 255)
    private String customerName;
    
    @Column(name = "total_amount", nullable = false, precision = 18, scale = 2)
    private BigDecimal totalAmount;
    
    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private OrderStatus status = OrderStatus.PENDING;
    
    @Column(name = "owner_id", nullable = false)
    private Long ownerId;
    
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;
    
    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;
    
    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        updatedAt = LocalDateTime.now();
    }
    
    @PreUpdate
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
    
    public enum OrderStatus {
        PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
    }
}

// OrderRepository.java
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByOwnerId(Long ownerId);
    Optional<Order> findByOrderNumber(String orderNumber);
}
```

### 7.10 Order Controller — `@PreAuthorize` in Action

```java
// OrderController.java
package com.datasoft.order.controller;

import com.datasoft.order.dto.*;
import com.datasoft.order.entity.Order;
import com.datasoft.order.service.OrderService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PostAuthorize;
import org.springframework.security.access.prepost.PostFilter;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    
    private final OrderService orderService;
    
    /**
     * Public to all authenticated users with ORDER_VIEW permission
     */
    @GetMapping
    @PreAuthorize("hasPermission(null, 'ORDER_VIEW')")
    public ResponseEntity<List<Order>> getAllOrders() {
        return ResponseEntity.ok(orderService.findAll());
    }
    
    /**
     * 🔥 PostAuthorize: returns order, then checks ownership
     * Owner দেখতে পারবে, অথবা ADMIN
     */
    @GetMapping("/{id}")
    @PreAuthorize("hasPermission(null, 'ORDER_VIEW')")
    @PostAuthorize("returnObject.body.ownerId == authentication.principal or hasRole('ADMIN')")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.findById(id));
    }
    
    /**
     * 🔥 PostFilter: শুধু সেই orders দেখাও যেগুলো user view করতে পারবে
     */
    @GetMapping("/my-orders")
    @PreAuthorize("isAuthenticated()")
    @PostFilter("filterObject.ownerId == authentication.principal")
    public ResponseEntity<List<Order>> getMyOrders() {
        return ResponseEntity.ok(orderService.findAll());
    }
    
    /**
     * Create — ORDER_CREATE permission লাগবে
     */
    @PostMapping
    @PreAuthorize("hasPermission(null, 'ORDER_CREATE')")
    public ResponseEntity<Order> createOrder(@Valid @RequestBody CreateOrderRequest request,
                                              @AuthenticationPrincipal String userId) {
        Order order = orderService.create(request, Long.parseLong(userId));
        return ResponseEntity.ok(order);
    }
    
    /**
     * Update — ORDER_UPDATE permission + ownership check
     */
    @PutMapping("/{id}")
    @PreAuthorize("hasPermission(null, 'ORDER_UPDATE') and " +
                  "(@orderSecurityService.isOwner(#id, authentication.principal) or hasRole('ADMIN'))")
    public ResponseEntity<Order> updateOrder(@PathVariable Long id,
                                              @Valid @RequestBody UpdateOrderRequest request) {
        return ResponseEntity.ok(orderService.update(id, request));
    }
    
    /**
     * Delete — ORDER_DELETE permission + ownership বা ADMIN role
     */
    @DeleteMapping("/{id}")
    @PreAuthorize("hasPermission(null, 'ORDER_DELETE') and " +
                  "(@orderSecurityService.isOwner(#id, authentication.principal) or hasRole('ADMIN'))")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.delete(id);
        return ResponseEntity.noContent().build();
    }
    
    /**
     * Bulk import — শুধু ADMIN বা specific permission সহ MANAGER
     */
    @PostMapping("/bulk-import")
    @PreAuthorize("hasRole('ADMIN') or " +
                  "(hasRole('MANAGER') and hasPermission(null, 'ORDER_BULK_IMPORT'))")
    public ResponseEntity<Integer> bulkImport(@RequestBody List<CreateOrderRequest> requests,
                                                @AuthenticationPrincipal String userId) {
        int imported = orderService.bulkImport(requests, Long.parseLong(userId));
        return ResponseEntity.ok(imported);
    }
    
    /**
     * Reports export — REPORT_EXPORT permission
     * Hasib (revoked) এই endpoint hit করলে 403 পাবে!
     */
    @GetMapping("/reports/export")
    @PreAuthorize("hasPermission(null, 'REPORT_EXPORT')")
    public ResponseEntity<byte[]> exportReports() {
        byte[] csv = orderService.exportReports();
        return ResponseEntity.ok()
            .header("Content-Type", "text/csv")
            .header("Content-Disposition", "attachment; filename=orders.csv")
            .body(csv);
    }
}
```

### 7.11 Order Security Service (used in SpEL)

```java
// OrderSecurityService.java
package com.datasoft.order.security;

import com.datasoft.order.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;

@Service("orderSecurityService")  // 🔥 Bean name SpEL এ use হবে
@RequiredArgsConstructor
public class OrderSecurityService {
    
    private final OrderRepository orderRepository;
    
    /**
     * @PreAuthorize("@orderSecurityService.isOwner(#id, authentication.principal)")
     */
    public boolean isOwner(Long orderId, String userIdStr) {
        Long userId = Long.parseLong(userIdStr);
        return orderRepository.findById(orderId)
            .map(order -> order.getOwnerId().equals(userId))
            .orElse(false);
    }
    
    /**
     * Check if order is in modifiable state
     */
    public boolean isModifiable(Long orderId) {
        return orderRepository.findById(orderId)
            .map(order -> order.getStatus() != Order.OrderStatus.SHIPPED 
                       && order.getStatus() != Order.OrderStatus.DELIVERED)
            .orElse(false);
    }
}
```

### 7.12 OrderService (Business Logic Layer)

```java
// OrderService.java
package com.datasoft.order.service;

import com.datasoft.order.dto.*;
import com.datasoft.order.entity.Order;
import com.datasoft.order.repository.OrderRepository;
import com.datasoft.common.exception.BusinessException;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.UUID;

@Service
@RequiredArgsConstructor
@Slf4j
@Transactional
public class OrderService {

    private final OrderRepository orderRepository;

    public List<Order> findAll() {
        return orderRepository.findAll();
    }

    public Order findById(Long id) {
        return orderRepository.findById(id)
            .orElseThrow(() -> new BusinessException("ORDER_NOT_FOUND",
                "Order not found: " + id));
    }

    public Order create(CreateOrderRequest request, Long ownerId) {
        Order order = Order.builder()
            .orderNumber("ORD-" + UUID.randomUUID().toString().substring(0, 8).toUpperCase())
            .customerName(request.getCustomerName())
            .totalAmount(request.getTotalAmount())
            .status(Order.OrderStatus.PENDING)
            .ownerId(ownerId)
            .build();

        Order saved = orderRepository.save(order);
        log.info("Order {} created by user {}", saved.getOrderNumber(), ownerId);
        return saved;
    }

    public Order update(Long id, UpdateOrderRequest request) {
        Order order = findById(id);
        if (request.getCustomerName() != null) order.setCustomerName(request.getCustomerName());
        if (request.getTotalAmount() != null) order.setTotalAmount(request.getTotalAmount());
        if (request.getStatus() != null) order.setStatus(request.getStatus());
        return orderRepository.save(order);
    }

    public void delete(Long id) {
        Order order = findById(id);
        if (order.getStatus() == Order.OrderStatus.SHIPPED ||
            order.getStatus() == Order.OrderStatus.DELIVERED) {
            throw new BusinessException("ORDER_NOT_DELETABLE",
                "Shipped/Delivered orders delete করা যাবে না");
        }
        orderRepository.deleteById(id);
        log.info("Order {} deleted", id);
    }

    public int bulkImport(List<CreateOrderRequest> requests, Long ownerId) {
        int count = 0;
        for (CreateOrderRequest req : requests) {
            create(req, ownerId);
            count++;
        }
        return count;
    }

    public byte[] exportReports() {
        StringBuilder csv = new StringBuilder("OrderNumber,Customer,Amount,Status\n");
        for (Order order : orderRepository.findAll()) {
            csv.append(String.format("%s,%s,%s,%s\n",
                order.getOrderNumber(),
                order.getCustomerName(),
                order.getTotalAmount(),
                order.getStatus()));
        }
        return csv.toString().getBytes();
    }
}
```

---

## 8. Gateway Implementation

### 8.1 application.yml

```yaml
server:
  port: 8080

spring:
  application:
    name: gateway
  
  cloud:
    gateway:
      routes:
        - id: auth-service
          uri: http://${AUTH_SERVICE_HOST:localhost}:8081
          predicates:
            - Path=/api/auth/**
        
        - id: permission-service
          uri: http://${PERMISSION_SERVICE_HOST:localhost}:8082
          predicates:
            - Path=/api/permissions/**
        
        - id: order-service
          uri: http://${ORDER_SERVICE_HOST:localhost}:8083
          predicates:
            - Path=/api/orders/**

jwt:
  secret: ${JWT_SECRET:VGhpc0lzQVZlcnlMb25nU2VjcmV0S2V5Rm9ySldUVG9rZW5HZW5lcmF0aW9uMjAyNQ==}
  issuer: auth.datasoft.com
```

### 8.2 JWT Filter (Reactive)

```java
// JwtAuthenticationFilter.java
package com.datasoft.gateway.filter;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.gateway.filter.GatewayFilterChain;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.core.Ordered;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.stereotype.Component;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.List;

@Component
@Slf4j
public class JwtAuthenticationFilter implements GlobalFilter, Ordered {
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.issuer}")
    private String issuer;
    
    private static final List<String> PUBLIC_PATHS = List.of(
        "/api/auth/login",
        "/api/auth/register",
        "/api/auth/refresh",
        "/actuator/health"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getURI().getPath();
        
        // Public paths skip
        if (isPublicPath(path)) {
            return chain.filter(exchange);
        }
        
        // Token extract
        String authHeader = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }
        
        String token = authHeader.substring(7);
        
        try {
            // 🔥 JWT validate
            Claims claims = Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .requireIssuer(issuer)
                .build()
                .parseClaimsJws(token)
                .getBody();
            
            String userId = claims.getSubject();
            @SuppressWarnings("unchecked")
            List<String> roles = claims.get("roles", List.class);
            
            // 🔥 Header inject downstream এর জন্য
            ServerHttpRequest mutated = exchange.getRequest().mutate()
                .header("X-User-Id", userId)
                .header("X-User-Roles", String.join(",", roles))
                .header("X-User-Email", claims.get("email", String.class))
                .build();
            
            log.debug("JWT validated for user {}, roles: {}", userId, roles);
            
            return chain.filter(exchange.mutate().request(mutated).build());
            
        } catch (Exception e) {
            log.warn("JWT validation failed: {}", e.getMessage());
            return unauthorized(exchange, "Invalid or expired token");
        }
    }
    
    private boolean isPublicPath(String path) {
        return PUBLIC_PATHS.stream().anyMatch(path::startsWith);
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().add("X-Error-Message", message);
        return exchange.getResponse().setComplete();
    }
    
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }
    
    @Override
    public int getOrder() {
        return -100;  // High priority
    }
}
```

---

## 9. Common Module (Shared)

```java
// BusinessException.java
package com.datasoft.common.exception;

import lombok.Getter;

@Getter
public class BusinessException extends RuntimeException {
    private final String errorCode;
    
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}

// GlobalExceptionHandler.java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        return ResponseEntity.badRequest().body(
            new ErrorResponse(ex.getErrorCode(), ex.getMessage(), Instant.now())
        );
    }
    
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        log.warn("Access denied: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(
            new ErrorResponse("ACCESS_DENIED", "এই কাজের অনুমতি নেই", Instant.now())
        );
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            errors.put(fieldName, error.getDefaultMessage());
        });
        return ResponseEntity.badRequest().body(errors);
    }
}

// ErrorResponse.java
@Data
@AllArgsConstructor
public class ErrorResponse {
    private String errorCode;
    private String message;
    private Instant timestamp;
}
```

---

## 10. Dockerfile (সব Service-এর জন্য)

প্রতিটা microservice-এর root-এ একটা `Dockerfile` রাখো। Multi-stage build use করছি — final image ছোট রাখতে:

```dockerfile
# Dockerfile
# Stage 1: Build
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
RUN apk add --no-cache maven
COPY pom.xml .
# Dependency layer cache আলাদা — code change হলে dependency re-download হবে না
RUN mvn dependency:go-offline -B
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run (JDK না, শুধু JRE — image size কমে)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=builder /app/target/*.jar app.jar

# Actuator health endpoint check করে
HEALTHCHECK --interval=30s --timeout=5s --start-period=60s \
    CMD wget --quiet --tries=1 --spider http://localhost:${SERVER_PORT:-8081}/actuator/health || exit 1

EXPOSE 8081
ENTRYPOINT ["java", \
    "-Djava.security.egd=file:/dev/./urandom", \
    "-XX:+UseContainerSupport", \
    "-XX:MaxRAMPercentage=75.0", \
    "-jar", "/app/app.jar"]
```

**JVM flags explanation:**
- `-Djava.security.egd=file:/dev/./urandom` → container-এ random number generation faster
- `-XX:+UseContainerSupport` → Docker memory limit কে JVM heap limit হিসেবে চেনে
- `-XX:MaxRAMPercentage=75.0` → Container RAM এর 75% JVM heap (বাকি 25% OS + non-heap)

**প্রতিটা service-এ port আলাদা:**

| Service | EXPOSE port |
|---------|-------------|
| Gateway | 8080 |
| Auth Service | 8081 |
| Permission Service | 8082 |
| Order Service | 8083 |

---

## 11. Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  
  # 🔥 MSSQL Server
  mssql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    container_name: datasoft-mssql
    environment:
      ACCEPT_EULA: "Y"
      MSSQL_SA_PASSWORD: "YourStrong@Pass123"
      MSSQL_PID: "Developer"
    ports:
      - "1433:1433"
    volumes:
      - mssql_data:/var/opt/mssql
    healthcheck:
      test: /opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P "YourStrong@Pass123" -Q "SELECT 1" || exit 1
      interval: 10s
      timeout: 5s
      retries: 10
  
  # MSSQL initialization (creates databases)
  mssql-init:
    image: mcr.microsoft.com/mssql-tools
    depends_on:
      mssql:
        condition: service_healthy
    volumes:
      - ./db-init:/scripts
    command: >
      /bin/bash -c "
        for i in {1..50}; do
          /opt/mssql-tools/bin/sqlcmd -S mssql -U sa -P 'YourStrong@Pass123' -d master -i /scripts/init-databases.sql && break
          sleep 2
        done
      "
  
  redis:
    image: redis:7-alpine
    container_name: datasoft-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
  
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: datasoft-zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
  
  kafka:
    image: confluentinc/cp-kafka:latest
    container_name: datasoft-kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
  
  # Application Services
  auth-service:
    build: ./auth-service
    container_name: datasoft-auth
    depends_on:
      mssql-init:
        condition: service_completed_successfully
    ports:
      - "8081:8081"
    environment:
      DB_HOST: mssql
      DB_PORT: 1433
      DB_USERNAME: sa
      DB_PASSWORD: YourStrong@Pass123
      JWT_SECRET: VGhpc0lzQVZlcnlMb25nU2VjcmV0S2V5Rm9ySldUVG9rZW5HZW5lcmF0aW9uMjAyNQ==
  
  permission-service:
    build: ./permission-service
    container_name: datasoft-permission
    depends_on:
      mssql-init:
        condition: service_completed_successfully
      redis:
        condition: service_started
      kafka:
        condition: service_started
    ports:
      - "8082:8082"
    environment:
      DB_HOST: mssql
      DB_USERNAME: sa
      DB_PASSWORD: YourStrong@Pass123
      REDIS_HOST: redis
      KAFKA_BROKER: kafka:29092
  
  order-service:
    build: ./order-service
    container_name: datasoft-order
    depends_on:
      - mssql-init
      - kafka
      - permission-service
    ports:
      - "8083:8083"
    environment:
      DB_HOST: mssql
      DB_USERNAME: sa
      DB_PASSWORD: YourStrong@Pass123
      KAFKA_BROKER: kafka:29092
      PERMISSION_SERVICE_URL: http://permission-service:8082
  
  gateway:
    build: ./gateway
    container_name: datasoft-gateway
    depends_on:
      - auth-service
      - permission-service
      - order-service
    ports:
      - "8080:8080"
    environment:
      AUTH_SERVICE_HOST: auth-service
      PERMISSION_SERVICE_HOST: permission-service
      ORDER_SERVICE_HOST: order-service
      JWT_SECRET: VGhpc0lzQVZlcnlMb25nU2VjcmV0S2V5Rm9ySldUVG9rZW5HZW5lcmF0aW9uMjAyNQ==

volumes:
  mssql_data:
  redis_data:
```

### Database Init Script

```sql
-- db-init/init-databases.sql
-- This file creates all required databases and runs schema

CREATE DATABASE auth_db;
GO
CREATE DATABASE permission_db;
GO
CREATE DATABASE order_db;
GO

-- Then run individual schema scripts
-- (Include the SQL from sections 3.1, 3.2, 3.3)
```

---

## 12. End-to-End Flow Examples

### Example 1: Hasib Login করে

```
1. POST http://localhost:8080/api/auth/login
   Body: { "email": "hasib@datasoft.com", "password": "password123" }
   
2. Gateway:
   - Path /api/auth/login → public path → skip JWT validation
   - Forward to Auth Service (port 8081)

3. Auth Service:
   - Email lookup in MSSQL: users table
   - BCrypt password verify
   - Roles fetch: user_roles table → ["MANAGER"]
   - Generate JWT:
     {
       "sub": "2",
       "email": "hasib@datasoft.com",
       "roles": ["MANAGER"],
       "iss": "auth.datasoft.com",
       "exp": 1735690500
     }
   - Store refresh token in DB
   - Return AuthResponse

4. Response:
   {
     "accessToken": "eyJhbG...",
     "refreshToken": "eyJhbG...",
     "user": { "id": 2, "email": "hasib@datasoft.com", "roles": ["MANAGER"] }
   }
```

### Example 2: Hasib Order Create করে (Success)

```
1. POST http://localhost:8080/api/orders
   Authorization: Bearer eyJhbG...
   Body: { "customerName": "ABC Corp", "totalAmount": 5000 }

2. Gateway:
   - JWT validate ✅
   - Extract userId=2, roles=["MANAGER"]
   - Inject headers: X-User-Id: 2, X-User-Roles: MANAGER
   - Forward to Order Service

3. Order Service - HeaderToSecurityContextFilter:
   - Read X-User-Id: 2
   - Read X-User-Roles: MANAGER
   - Set SecurityContext: principal=2, authorities=[ROLE_MANAGER]

4. OrderController.createOrder():
   - @PreAuthorize("hasPermission(null, 'ORDER_CREATE')")
   - CustomPermissionEvaluator.hasPermission(auth, null, "ORDER_CREATE"):
     - userId=2, roles=[MANAGER]
     - PermissionCache.hasPermission(2, [MANAGER], "ORDER_CREATE")
     - Caffeine cache check → MISS
     - Feign call: PermissionClient.getEffectivePermissions(2, "MANAGER")
   
5. Permission Service:
   - getEffectivePermissions(2, [MANAGER])
   - Redis check: "user_permissions:2" → MISS
   - DB query:
     - MANAGER role permissions: [ORDER_VIEW, ORDER_CREATE, ORDER_UPDATE, REPORT_VIEW, REPORT_EXPORT]
     - Hasib's overrides: REVOKE REPORT_EXPORT
   - Calculate effective: 
     - Start: [ORDER_VIEW, ORDER_CREATE, ORDER_UPDATE, REPORT_VIEW, REPORT_EXPORT]
     - Remove REVOKE: [ORDER_VIEW, ORDER_CREATE, ORDER_UPDATE, REPORT_VIEW]
   - Cache in Redis (5 min TTL)
   - Return permissions

6. Order Service:
   - Caffeine cache populated
   - "ORDER_CREATE" in [ORDER_VIEW, ORDER_CREATE, ORDER_UPDATE, REPORT_VIEW] → ✅
   - Method executes
   - Order created in MSSQL

7. Response: 201 Created with Order data
```

### Example 3: Hasib Report Export চেষ্টা (Failure!)

```
1. GET http://localhost:8080/api/orders/reports/export
   Authorization: Bearer eyJhbG...

2-3. Gateway + Filter same as before

4. OrderController.exportReports():
   - @PreAuthorize("hasPermission(null, 'REPORT_EXPORT')")
   - CustomPermissionEvaluator.hasPermission(auth, null, "REPORT_EXPORT"):
     - PermissionCache: [ORDER_VIEW, ORDER_CREATE, ORDER_UPDATE, REPORT_VIEW]
     - "REPORT_EXPORT" NOT in set → ❌

5. AccessDeniedException thrown

6. GlobalExceptionHandler:
   - 403 Forbidden
   - Body: { "errorCode": "ACCESS_DENIED", "message": "এই কাজের অনুমতি নেই" }
```

### Example 4: Karim (অন্য MANAGER) Report Export — Success!

```
Same MANAGER role, but no override
Permission Service:
   - MANAGER permissions: [..., REPORT_EXPORT]
   - Karim's overrides: NONE
   - Effective: [..., REPORT_EXPORT]
   
Result: ✅ 200 OK with CSV data
```

🎯 **এটাই তোমার requirement এর core demonstration:**
- Same role (MANAGER), same role permissions
- কিন্তু user-level override দিয়ে individual user এর permission control
- অন্য MANAGER users affected হচ্ছে না

### Example 5: Admin Override Removes — Real-time Effect

```
1. Admin: DELETE /api/permissions/overrides?userId=2&permission=REPORT_EXPORT

2. Permission Service:
   - Override delete from DB
   - Redis: del "user_permissions:2"
   - Kafka publish: PermissionChangedEvent { userId: 2, permission: "REPORT_EXPORT" }

3. Order Service Kafka Listener:
   - Receive event
   - PermissionCache.invalidateUser(2)
   - Caffeine: keys with "2:" prefix removed

4. Hasib next request:
   - Caffeine MISS → Permission Service call
   - Redis MISS → DB calculate
   - New effective: [ORDER_VIEW, ORDER_CREATE, ORDER_UPDATE, REPORT_VIEW, REPORT_EXPORT]
   - Now REPORT_EXPORT included!

5. Hasib retries export → ✅ Success
```

---

## 13. Testing Strategy

### Unit Tests (Auth Service)

```java
@SpringBootTest
class AuthServiceTest {
    
    @MockBean
    private UserRepository userRepository;
    
    @MockBean
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private AuthService authService;
    
    @Test
    void login_validCredentials_returnsTokens() {
        // Given
        User user = User.builder()
            .id(1L)
            .email("test@datasoft.com")
            .passwordHash("hashed")
            .enabled(true)
            .accountLocked(false)
            .roles(Set.of(UserRole.builder().roleName("USER").build()))
            .build();
        
        when(userRepository.findByEmail("test@datasoft.com"))
            .thenReturn(Optional.of(user));
        when(passwordEncoder.matches("password", "hashed"))
            .thenReturn(true);
        
        // When
        AuthResponse response = authService.login(
            new LoginRequest("test@datasoft.com", "password")
        );
        
        // Then
        assertNotNull(response.getAccessToken());
        assertNotNull(response.getRefreshToken());
    }
}
```

### Controller Tests (Order Service)

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private OrderService orderService;
    
    @Test
    @WithMockUser(username = "1", authorities = {"ROLE_MANAGER"})
    void createOrder_withManagerRole_returns200() throws Exception {
        // Permission cache mock
        when(permissionCache.hasPermission(1L, Set.of("MANAGER"), "ORDER_CREATE"))
            .thenReturn(true);
        
        mockMvc.perform(post("/api/orders")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"customerName\":\"Test\",\"totalAmount\":1000}"))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(username = "2", authorities = {"ROLE_USER"})
    void deleteOrder_withoutPermission_returns403() throws Exception {
        when(permissionCache.hasPermission(2L, Set.of("USER"), "ORDER_DELETE"))
            .thenReturn(false);
        
        mockMvc.perform(delete("/api/orders/1"))
            .andExpect(status().isForbidden());
    }
}
```

---

## 🎯 Summary: এই Architecture-এ যা শিখবে

| Component | Concept Learned |
|-----------|-----------------|
| **Gateway** | Reactive Spring (WebFlux), JWT validation, header injection |
| **Auth Service** | JWT generation, BCrypt, refresh tokens, MSSQL integration |
| **Permission Service** | RBAC + user overrides, Redis caching, Kafka events |
| **Order Service** | @PreAuthorize ecosystem, Caffeine cache, Feign client |
| **MSSQL** | JPA with SQL Server dialect, IDENTITY columns, NVARCHAR |
| **Common** | Shared DTOs, exception handling, error responses |

## 🚀 Running the System

```bash
# 1. Start infrastructure
docker-compose up -d mssql redis kafka zookeeper

# 2. Wait for MSSQL ready, run schema
docker exec -it datasoft-mssql /opt/mssql-tools/bin/sqlcmd \
    -S localhost -U sa -P 'YourStrong@Pass123' \
    -i /scripts/init-databases.sql

# 3. Start services
mvn clean install
docker-compose up -d auth-service permission-service order-service gateway

# 4. Test
curl -X POST http://localhost:8080/api/auth/login \
    -H "Content-Type: application/json" \
    -d '{"email":"hasib@datasoft.com","password":"password123"}'
```

## 📚 পরবর্তী Steps

এই foundation solid হলে explore করো:
1. **Distributed Tracing** — Zipkin + Sleuth
2. **Circuit Breaker** — Resilience4j on Feign calls
3. **Rate Limiting** — Redis-based at Gateway
4. **Audit Logging** — Spring Data Envers
5. **Multi-Tenancy** — tenant_id discriminator
6. **GraphQL Gateway** — REST microservices behind GraphQL
7. **Service Mesh** — Istio for inter-service security

প্রতিটা step incrementally — একটা একটা concept master করো। 🚀