# Spring Security + JWT — Part 1: Foundation

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Previous:** Spring Security Concepts (Tutorial 1)
> **Part 1 Topics:** Setup, Database Design, Password Security, JWT Implementation, UserDetailsService

> ⚠️ এই Part শেষে — Database ready হবে, JWT generate করতে পারবে, Spring Security এর সাথে User connect হবে। কিন্তু এখনো Login API থাকবে না — সেটা Part 2 এ।

---

## সূচিপত্র

1. [Phase 0: Setup](#phase-0)
2. [Phase 1: Database Design (User, Role, Permission)](#phase-1)
3. [Phase 2: Password Security (BCrypt)](#phase-2)
4. [Phase 3: JWT Implementation](#phase-3)
5. [Phase 4: UserDetailsService](#phase-4)
6. [Part 1 এর সারসংক্ষেপ](#summary)

---

<a name="phase-0"></a>

# Phase 0 — Setup

## কী কী Dependency দরকার?

`pom.xml` এ এগুলো add করো —

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT Library (jjwt) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
```

## প্রতিটা Dependency কী করে?

### `spring-boot-starter-security`

Spring Security এর সব core class পাওয়া যায় —
- `SecurityFilterChain`, `UserDetailsService`, `PasswordEncoder` ইত্যাদি
- Default behavior: সব API protected হয়ে যায় auto

### `jjwt-api`, `jjwt-impl`, `jjwt-jackson`

JWT তৈরি, parse, validate করার library —
- `jjwt-api` = interfaces
- `jjwt-impl` = actual implementation
- `jjwt-jackson` = JSON serialize/deserialize

আগে যেমন JPA + Hibernate ছিল (interface + implementation), এটাও সেরকম।

---

## Project Structure

```
src/main/java/com/example/bookstore/
├── entity/
│   ├── User.java
│   ├── Role.java
│   └── Permission.java
├── repository/
│   ├── UserRepository.java
│   ├── RoleRepository.java
│   └── PermissionRepository.java
├── security/
│   ├── JwtUtil.java
│   ├── CustomUserDetails.java
│   ├── CustomUserDetailsService.java
│   ├── JwtAuthenticationFilter.java       (Part 2)
│   └── SecurityConfig.java                (Part 2)
├── auth/
│   ├── AuthController.java                (Part 2)
│   ├── AuthService.java                   (Part 2)
│   └── dto/
│       ├── LoginRequest.java              (Part 2)
│       ├── RegisterRequest.java           (Part 2)
│       └── LoginResponse.java             (Part 2)
└── BookstoreApplication.java
```

`security/` folder = সব security related class
`auth/` folder = Login/Register এর জন্য

---

<a name="phase-1"></a>

# Phase 1 — Database Design (User, Role, Permission)

## কেন এই 3 table?

আমরা concept tutorial এ দেখেছিলাম —

```
User      → কে এই person
Role      → User এর "type" (ADMIN, EDITOR, MEMBER)
Permission → Specific কাজের অনুমতি (BOOK_READ, BOOK_DELETE)
```

### Database Structure

```
users:                roles:               permissions:
┌────┬────────┐       ┌────┬────────┐      ┌────┬─────────────┐
│ id │ name   │       │ id │ name   │      │ id │ name        │
├────┼────────┤       ├────┼────────┤      ├────┼─────────────┤
│ 1  │ Hasib  │       │ 1  │ ADMIN  │      │ 1  │ BOOK_READ   │
│ 2  │ Karim  │       │ 2  │ EDITOR │      │ 2  │ BOOK_CREATE │
└────┴────────┘       └────┴────────┘      │ 3  │ BOOK_DELETE │
                                            └────┴─────────────┘

user_roles (M-N):     role_permissions (M-N):
┌─────────┬─────────┐ ┌─────────┬───────────────┐
│ user_id │ role_id │ │ role_id │ permission_id │
├─────────┼─────────┤ ├─────────┼───────────────┤
│ 1       │ 1       │ │ 1       │ 1             │
│ 2       │ 2       │ │ 1       │ 2             │
└─────────┴─────────┘ │ 1       │ 3             │
                      │ 2       │ 1             │
                      │ 2       │ 2             │
                      └─────────┴───────────────┘
```

দুইটা **Many-to-Many** relationship — যা আগে JPA তে শিখেছো।

---

## User Entity

```java
@Entity
@Table(name = "users")    // ⭐ "user" SQL keyword, তাই "users" use করি
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String password;       // ⭐ Hashed password store হবে

    @Column(nullable = false)
    private boolean enabled = true;   // Account active কিনা

    @ManyToMany(fetch = FetchType.EAGER)    // ⭐ EAGER কেন — পরে explain
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}
```

## প্রতিটা Annotation/Keyword Decode

### `@Table(name = "users")`

`user` SQL এ reserved keyword (PostgreSQL, MySQL এ)। তাই table name `users` দিচ্ছি।

### `@Column(unique = true)`

Database level এ duplicate prevent। দুইজন user একই username/email নিতে পারবে না।

### `private String password`

⚠️ **এটা hashed password।** কখনো plain text save করবো না (পরের phase এ explain)।

### `private boolean enabled`

Account suspend করতে চাইলে — `false` করে দাও। Login করতে পারবে না।

### `@ManyToMany(fetch = FetchType.EAGER)`

User load করার সময় Roles ও সাথে load হবে।

**কেন EAGER এখানে?**

প্রতিটা request এ Spring Security user এর role check করবে। প্রতিবার আলাদা query করলে slow হবে। তাই EAGER।

(Normal business code এ আমরা LAZY use করি, কিন্তু Security এর জন্য EAGER ভালো।)

### `Set<Role>` কেন, `List<Role>` না?

User এর একই Role দুইবার থাকতে পারে না। তাই `Set` (duplicate prevent)।

### `@JoinTable`

Many-to-Many এর join table এর structure define করছে —

```
@JoinTable(
    name = "user_roles",                                    // join table এর নাম
    joinColumns = @JoinColumn(name = "user_id"),            // এই entity এর FK
    inverseJoinColumns = @JoinColumn(name = "role_id")      // অন্য entity এর FK
)
```

---

## Role Entity

```java
@Entity
@Table(name = "roles")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Role {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;     // "ADMIN", "EDITOR", "MEMBER"

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "role_permissions",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id")
    )
    private Set<Permission> permissions = new HashSet<>();
}
```

## Permission Entity

```java
@Entity
@Table(name = "permissions")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class Permission {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String name;    // "BOOK_READ", "BOOK_CREATE", etc.
}
```

Permission সবচেয়ে simple — শুধু একটা name।

---

## Repositories

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByUsername(String username);

    Optional<User> findByEmail(String email);

    boolean existsByUsername(String username);

    boolean existsByEmail(String email);
}
```

### `findByUsername(String username)`

Login এর সময় username দিয়ে user খুঁজে। Spring Data JPA automatically query বানিয়ে দেয় —

```sql
SELECT * FROM users WHERE username = ?
```

### `Optional<User>`

Null safety — User পেলে present, না পেলে empty।

### `existsByUsername`

Register এর সময় check — username এর duplicate কিনা।

---

```java
public interface RoleRepository extends JpaRepository<Role, Long> {

    Optional<Role> findByName(String name);
}
```

```java
public interface PermissionRepository extends JpaRepository<Permission, Long> {

    Optional<Permission> findByName(String name);
}
```

---

## Sample Data Seed করা (Testing এর জন্য)

Application start হলে database এ default Role + Permission auto-create করতে চাই।

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DataSeeder implements CommandLineRunner {

    private final RoleRepository roleRepository;
    private final PermissionRepository permissionRepository;

    @Override
    @Transactional
    public void run(String... args) throws Exception {

        // Permissions create করি
        Permission bookRead = createPermissionIfNotExists("BOOK_READ");
        Permission bookCreate = createPermissionIfNotExists("BOOK_CREATE");
        Permission bookUpdate = createPermissionIfNotExists("BOOK_UPDATE");
        Permission bookDelete = createPermissionIfNotExists("BOOK_DELETE");
        Permission userManage = createPermissionIfNotExists("USER_MANAGE");

        // ADMIN role + সব permissions
        if (roleRepository.findByName("ADMIN").isEmpty()) {
            Role admin = new Role();
            admin.setName("ADMIN");
            admin.setPermissions(Set.of(
                bookRead, bookCreate, bookUpdate, bookDelete, userManage
            ));
            roleRepository.save(admin);
            log.info("ADMIN role created");
        }

        // EDITOR role + কিছু permissions
        if (roleRepository.findByName("EDITOR").isEmpty()) {
            Role editor = new Role();
            editor.setName("EDITOR");
            editor.setPermissions(Set.of(
                bookRead, bookCreate, bookUpdate
            ));
            roleRepository.save(editor);
            log.info("EDITOR role created");
        }

        // MEMBER role + শুধু read
        if (roleRepository.findByName("MEMBER").isEmpty()) {
            Role member = new Role();
            member.setName("MEMBER");
            member.setPermissions(Set.of(bookRead));
            roleRepository.save(member);
            log.info("MEMBER role created");
        }
    }

    private Permission createPermissionIfNotExists(String name) {
        return permissionRepository.findByName(name)
            .orElseGet(() -> {
                Permission p = new Permission();
                p.setName(name);
                return permissionRepository.save(p);
            });
    }
}
```

### `CommandLineRunner` কী?

এটা একটা interface। `run()` method app start হওয়ার সময় auto call হয়।

**ফলাফল:** App start করলে database এ default data থাকবে — manually seed করতে হবে না।

---

<a name="phase-2"></a>

# Phase 2 — Password Security (BCrypt)

## Password কেন Hash করতে হয়?

### Plain text এ save করলে কী হয়?

```sql
users table:
┌────┬───────┬──────────────┐
│ id │ name  │ password     │
├────┼───────┼──────────────┤
│ 1  │ Hasib │ MyPass123    │  ← কেউ DB দেখলে পেয়ে যাবে!
└────┴───────┴──────────────┘
```

Database leak হলে — সবার password পেয়ে যাবে hacker। 😱

### Hash করলে কী হয়?

```sql
users table:
┌────┬───────┬─────────────────────────────────────────────────────────┐
│ id │ name  │ password                                                │
├────┼───────┼─────────────────────────────────────────────────────────┤
│ 1  │ Hasib │ $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL│
└────┴───────┴─────────────────────────────────────────────────────────┘
```

DB leak হলেও — original password বের করা practically impossible।

---

## Hashing কীভাবে কাজ করে?

```
Plain Password    → Hash Function → Hashed Password
"MyPass123"       →     BCrypt    → "$2a$10$..."

One-way function:
Hashed Password   → ❌ → Plain Password (impossible)
```

### তাহলে Login এর সময় কীভাবে check করে?

```
User login করলো: "MyPass123"
        ↓
Server: Database থেকে stored hash পেলো → "$2a$10$..."
        ↓
Server: User এর "MyPass123" কে hash করলো
        ↓
Server: দুইটা hash compare করলো
        ↓
Match হলে → ✅ Login successful
```

**Original password কখনো compare করা হয় না — শুধু hash compare।**

---

## BCrypt কী?

> **BCrypt = একটা popular, secure hashing algorithm।**

### বৈশিষ্ট্য

```
✅ Salt automatic generate করে (rainbow table attack prevent)
✅ Slow on purpose (brute force attack prevent)
✅ Strength adjustable (10, 12, 14...)
```

### "Salt" কী?

Random data যা password এর সাথে mix করা হয় hash করার আগে।

```
"MyPass123" + "random_salt_xyz" → hash
```

### একই Password হলে — Hash কি একই হবে?

```
User 1: "MyPass123" → hash: "$2a$10$abc..."
User 2: "MyPass123" → hash: "$2a$10$xyz..."   ← আলাদা!
```

**Salt আলাদা, তাই hash ও আলাদা।** Hacker একটা password decode করতে পারলেও, সব account এ apply করতে পারবে না।

---

## Spring Security তে PasswordEncoder

Spring এ একটা interface আছে — `PasswordEncoder`।

### দুইটা মূল method

```java
public interface PasswordEncoder {

    String encode(CharSequence rawPassword);
    // → Plain password নাও, hash return করো

    boolean matches(CharSequence rawPassword, String encodedPassword);
    // → Plain password আর hashed password compare করো
}
```

### Configuration

`SecurityConfig` class এ Bean হিসেবে define করবো (Part 2 এ পুরো config)। আপাতত শুধু এটুকু —

```java
@Configuration
public class PasswordConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
        // 10 = strength (default 10, higher = slower but more secure)
    }
}
```

### Use করা

```java
@Service
@RequiredArgsConstructor
public class UserService {

    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public User registerUser(String username, String email, String rawPassword) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setPassword(passwordEncoder.encode(rawPassword));    // ⭐ Hash করে save
        return userRepository.save(user);
    }
}
```

```java
public boolean checkPassword(String rawPassword, User user) {
    return passwordEncoder.matches(rawPassword, user.getPassword());
}
```

---

<a name="phase-3"></a>

# Phase 3 — JWT Implementation

এবার সবচেয়ে important part — JWT generate, parse, validate করার class বানাবো।

## Setup — Secret Key

JWT signature বানাতে এবং verify করতে **Secret Key** লাগে। এটা `application.properties` এ রাখি —

```properties
# application.properties

jwt.secret=YourVerySecretKeyMustBeAtLeast256BitsLongForHS256Algorithm12345
jwt.expiration=3600000
# 1 hour in milliseconds
```

⚠️ **Production এ এটা environment variable এ রাখো — code এ commit করো না!**

---

## JwtUtil Class

পুরো JWT এর কাজ এক class এ —

```java
@Component
public class JwtUtil {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.expiration}")
    private long expirationMs;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
    }

    // ১. Token Generate
    public String generateToken(String username, Set<String> roles, Set<String> permissions) {
        Date now = new Date();
        Date expiry = new Date(now.getTime() + expirationMs);

        return Jwts.builder()
            .subject(username)
            .claim("roles", roles)
            .claim("permissions", permissions)
            .issuedAt(now)
            .expiration(expiry)
            .signWith(getSigningKey())
            .compact();
    }

    // ২. Token থেকে Username extract
    public String extractUsername(String token) {
        return parseClaims(token).getSubject();
    }

    // ৩. Token থেকে Roles extract
    @SuppressWarnings("unchecked")
    public Set<String> extractRoles(String token) {
        List<String> roles = parseClaims(token).get("roles", List.class);
        return new HashSet<>(roles);
    }

    // ৪. Token Validate
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    // ৫. Token expire হয়েছে কিনা
    public boolean isTokenExpired(String token) {
        return parseClaims(token).getExpiration().before(new Date());
    }

    // Helper method
    private Claims parseClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

---

## প্রতিটা Method বিস্তারিত

### Method 1: `generateToken()`

#### কী করে?

User এর info নিয়ে একটা JWT তৈরি করে।

#### Code Decode

```java
return Jwts.builder()
    .subject(username)                      // sub claim
    .claim("roles", roles)                  // custom claim
    .claim("permissions", permissions)      // custom claim
    .issuedAt(now)                          // iat claim
    .expiration(expiry)                     // exp claim
    .signWith(getSigningKey())              // signature
    .compact();                             // build the token string
```

প্রতিটা step —

| Step | কী করে | JWT Payload এ যা যোগ হয় |
|---|---|---|
| `.subject(username)` | User identifier | `"sub": "hasib"` |
| `.claim("roles", roles)` | Custom data | `"roles": ["ADMIN"]` |
| `.claim("permissions", permissions)` | Custom data | `"permissions": ["BOOK_READ"]` |
| `.issuedAt(now)` | Token কখন তৈরি | `"iat": 1735596800` |
| `.expiration(expiry)` | কখন expire হবে | `"exp": 1735683200` |
| `.signWith(getSigningKey())` | Signature বানায় | (Header.Payload.SIGNATURE) |
| `.compact()` | Final string return | `"eyJhbGc...."` |

#### Output Example

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJoYXNpYiIsInJvbGVzIjpbIkFETUlOIl0sImV4cCI6MTczNTY4MzIwMH0.kPeRjMq...
```

### Method 2: `extractUsername()`

```java
public String extractUsername(String token) {
    return parseClaims(token).getSubject();
}
```

Token থেকে `sub` field বের করে — যেটা username।

### Method 3: `extractRoles()`

```java
public Set<String> extractRoles(String token) {
    List<String> roles = parseClaims(token).get("roles", List.class);
    return new HashSet<>(roles);
}
```

Token এর `roles` claim থেকে list বের করে।

### Method 4: `validateToken()`

```java
public boolean validateToken(String token) {
    try {
        parseClaims(token);
        return true;
    } catch (JwtException | IllegalArgumentException e) {
        return false;
    }
}
```

`parseClaims()` যদি successfully chalу — token valid। যদি exception throw করে — invalid।

#### কী কী Reason এ Invalid হয়?

- Signature wrong (tampered)
- Token expired
- Malformed token
- Invalid algorithm

### Method 5: `isTokenExpired()`

```java
public boolean isTokenExpired(String token) {
    return parseClaims(token).getExpiration().before(new Date());
}
```

`exp` claim থেকে date নিয়ে current date এর সাথে compare।

---

## Helper Method: `parseClaims()`

```java
private Claims parseClaims(String token) {
    return Jwts.parser()
        .verifyWith(getSigningKey())
        .build()
        .parseSignedClaims(token)
        .getPayload();
}
```

#### কী করে?

1. Token কে parse করে
2. Signature verify করে (secret key দিয়ে)
3. Payload (claims) return করে

**যদি signature mismatch হয় → exception।**

---

## `getSigningKey()` কী?

```java
private SecretKey getSigningKey() {
    return Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
}
```

String secret কে actual `SecretKey` object এ convert করে। JWT library এর জন্য দরকার।

---

<a name="phase-4"></a>

# Phase 4 — UserDetailsService

## কেন এই Class দরকার?

Spring Security একটা generic framework। সে জানে না তোমার User database কেমন structure।

তাই Spring বলে —
> *"আমাকে একটা bridge দাও — যেটা আমার চাইলে user info return করবে।"*

সেই bridge হলো **`UserDetailsService` interface।**

---

## Spring Security এর User Model

Spring এর নিজস্ব User concept আছে — `UserDetails` interface।

```java
public interface UserDetails {

    Collection<? extends GrantedAuthority> getAuthorities();
    String getPassword();
    String getUsername();
    boolean isAccountNonExpired();
    boolean isAccountNonLocked();
    boolean isCredentialsNonExpired();
    boolean isEnabled();
}
```

আমাদের `User` entity Spring এর `UserDetails` থেকে আলাদা। তাই **convert করতে হবে।**

```
আমাদের User (database)  →  Spring এর UserDetails
```

---

## CustomUserDetails Class

`UserDetails` interface implement করে নিজের wrapper বানাই —

```java
public class CustomUserDetails implements UserDetails {

    private final User user;

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // Roles add করি (ROLE_ prefix সহ)
        user.getRoles().forEach(role -> {
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));

            // প্রতিটা role এর permissions ও add করি
            role.getPermissions().forEach(permission ->
                authorities.add(new SimpleGrantedAuthority(permission.getName()))
            );
        });

        return authorities;
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return true;       // আমরা expiry implement করিনি
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;       // Lock implement করিনি
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return true;
    }

    @Override
    public boolean isEnabled() {
        return user.isEnabled();
    }

    public User getUser() {
        return user;
    }
}
```

---

## প্রতিটা Method কী করে?

### `getAuthorities()`

User এর সব permission/role return করে। Spring এই list দেখে decide করে user কোন API access করতে পারবে।

#### কেন `ROLE_` prefix?

Spring Security এর convention — Role এর নাম এ `ROLE_` prefix থাকতে হয়।

```
Database এ:    "ADMIN"
Spring এ:      "ROLE_ADMIN"
```

`@PreAuthorize("hasRole('ADMIN')")` লিখলে Spring automatically `ROLE_` prefix যোগ করে check করে।

#### Permissions এ prefix নেই কেন?

Permissions = `Authority`। Spring এ `hasAuthority('BOOK_READ')` দিয়ে check হয় — prefix ছাড়া।

### `getPassword()`, `getUsername()`

আমাদের `User` entity থেকে data return।

### `isAccountNonExpired()`, `isAccountNonLocked()` ইত্যাদি

Account এর state। `false` return করলে — login deny হবে।

আমরা সব `true` রাখছি (simple রাখার জন্য)। চাইলে database এ flag add করে control করতে পারো।

### `isEnabled()`

`User.enabled = false` হলে — user login করতে পারবে না।

---

## CustomUserDetailsService Class

এবার Spring এর `UserDetailsService` implement করি —

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found with username: " + username
            ));

        return new CustomUserDetails(user);
    }
}
```

## প্রতিটা Line Decode

### `implements UserDetailsService`

Spring এর interface। একটা মাত্র method আছে —

```java
UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
```

### `loadUserByUsername(String username)`

Spring **automatic এই method call করে** যখন authentication করে। তোমার কাজ — username দিয়ে user খুঁজে return করা।

### `@Transactional(readOnly = true)`

Lazy load এর জন্য Session দরকার। আমরা EAGER use করেছি — তাই strict দরকার না, কিন্তু safe practice।

### `findByUsername(username).orElseThrow(...)`

Database থেকে user খুঁজি। না পেলে — `UsernameNotFoundException`।

### `return new CustomUserDetails(user)`

আমাদের `User` কে Spring এর `UserDetails` wrapper এ wrap।

---

## পুরো Flow Visualize

```
Spring Security:
"User login করতে চাইছে"
        ↓
"আমাকে user info দাও"
        ↓
CustomUserDetailsService.loadUserByUsername("hasib")
        ↓
Database query: SELECT * FROM users WHERE username = 'hasib'
                + JOIN user_roles + JOIN role_permissions
        ↓
User entity পেলো
        ↓
new CustomUserDetails(user) → wrapper
        ↓
Spring পেলো:
- username
- password (hashed)
- authorities (roles + permissions)
- enabled status
        ↓
Spring password check করবে (পরবর্তী phase)
```

---

<a name="summary"></a>

# Part 1 এর সারসংক্ষেপ

## যা যা শিখলাম

```
✅ Phase 0: Spring Security + JWT dependency setup
✅ Phase 1: User-Role-Permission database design
   - Many-to-Many relationships
   - Sample data seeding
✅ Phase 2: Password Security
   - কেন hash করতে হয়
   - BCrypt কীভাবে কাজ করে
   - PasswordEncoder
✅ Phase 3: JWT Implementation
   - JwtUtil class (generate, parse, validate)
   - Secret key management
✅ Phase 4: UserDetailsService
   - Spring Security এর সাথে database connect
   - CustomUserDetails wrapper
   - CustomUserDetailsService
```

## এখন তোমার যা আছে

```
✅ Database ready (User, Role, Permission tables)
✅ Default data seeded (ADMIN, EDITOR, MEMBER roles)
✅ Password hash করার system (BCrypt)
✅ JWT generate, parse, validate করার tool
✅ Spring Security database থেকে user load করতে পারে
```

## এখনো যা নেই (Part 2 এ আসছে)

```
❌ SecurityConfig (কোন API protected, কোন না)
❌ JWT Filter (request এ token check)
❌ Login API (POST /auth/login)
❌ Register API (POST /auth/register)
❌ Working authentication flow
```

## Visualization

```
Part 1 শেষে — তোমার system এ এই building blocks আছে:

┌──────────────────────────────────────────┐
│  Database (User, Role, Permission)       │  ✅
└──────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────┐
│  Repositories (CRUD operations)          │  ✅
└──────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────┐
│  PasswordEncoder (BCrypt)                │  ✅
└──────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────┐
│  JwtUtil (generate/validate JWT)         │  ✅
└──────────────────────────────────────────┘
              ↓
┌──────────────────────────────────────────┐
│  CustomUserDetailsService                │  ✅
│  (Spring Security ↔ Database bridge)     │
└──────────────────────────────────────────┘

Part 2 এ যোগ হবে:

┌──────────────────────────────────────────┐
│  SecurityConfig                          │  ❌
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│  JwtAuthenticationFilter                 │  ❌
└──────────────────────────────────────────┘
┌──────────────────────────────────────────┐
│  Auth APIs (Register, Login)             │  ❌
└──────────────────────────────────────────┘
```

## পরের Part এ যা শিখবো

**Part 2: Authentication Flow**

```
✅ SecurityConfig — সব config এক জায়গায়
✅ JWT Authentication Filter — Request এ token check
✅ AuthService — Login/Register logic
✅ AuthController — APIs
✅ DTO design (LoginRequest, RegisterRequest, LoginResponse)
✅ Public vs Protected APIs setup
```

Part 2 শেষে — **তুমি actually login করতে পারবে।** Postman এ Token পাবে।

---

Part 1 পড়ে কোথাও gap থাকলে বলো — clarify করবো। তারপর Part 2 এ যাবো। 🚀

Happy coding!
