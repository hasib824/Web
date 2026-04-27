# Spring Security + JWT — Complete Implementation Guide

> **এটা একটা Reference/Checklist** — কোন order এ, কী কী করতে হবে পুরো system বানাতে।
> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Level:** Production-Grade Implementation

---

## সূচিপত্র

1. [Phase 0: Prerequisites Check](#phase-0)
2. [Phase 1: Database Design](#phase-1)
3. [Phase 2: Password Security Setup](#phase-2)
4. [Phase 3: JWT Utility Setup](#phase-3)
5. [Phase 4: Spring Security Integration](#phase-4)
6. [Phase 5: DTOs Design](#phase-5)
7. [Phase 6: Security Configuration](#phase-6)
8. [Phase 7: JWT Authentication Filter](#phase-7)
9. [Phase 8: Authentication Service](#phase-8)
10. [Phase 9: Exception Handling](#phase-9)
11. [Phase 10: Complete Flow Test](#phase-10)
12. [Phase 11: Method Level Security](#phase-11)
13. [Phase 12: Advanced Features](#phase-12)
14. [Phase 13: Production Checklist](#phase-13)
15. [Build Order Summary](#build-order)
16. [Common Mistakes](#common-mistakes)
17. [Debugging Guide](#debugging-guide)
18. [Final Tips](#final-tips)

---

<a name="phase-0"></a>

## Phase 0: Prerequisites Check

```
✅ Spring Boot project setup (Spring Initializr)
✅ Database running (PostgreSQL/MySQL)
✅ Dependencies added:
   - spring-boot-starter-web
   - spring-boot-starter-data-jpa
   - spring-boot-starter-security
   - spring-boot-starter-validation
   - jjwt (JWT library: jjwt-api, jjwt-impl, jjwt-jackson)
   - database driver (postgresql/mysql)
   - lombok (optional but recommended)
```

### Dependencies (pom.xml)

```xml
<!-- Spring Boot Starters -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>

<!-- JWT -->
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

<!-- Database -->
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Lombok (optional) -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

---

<a name="phase-1"></a>

## Phase 1: Database Design (Foundation)

### Step 1.1: Entity Design Order

```
বানানোর order:

1st → Permission entity
2nd → Role entity
3rd → User entity

কেন এই order?
User → Role এর উপর depend করে
Role → Permission এর উপর depend করে
তাই dependency নেই এমন entity আগে বানাও
```

### Step 1.2: Permission Entity

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

### Step 1.3: Role Entity

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

**⚠️ কেন EAGER?** Security data তাই প্রতি request এ লাগবে।

### Step 1.4: User Entity

```java
@Entity
@Table(name = "users")
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
    private String password;       // Hashed password

    @Column(nullable = false)
    private boolean enabled = true;

    @ManyToMany(fetch = FetchType.EAGER)
    @JoinTable(
        name = "user_roles",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<Role> roles = new HashSet<>();
}
```

### Step 1.5: Repositories

```java
public interface PermissionRepository extends JpaRepository<Permission, Long> {
    Optional<Permission> findByName(String name);
}

public interface RoleRepository extends JpaRepository<Role, Long> {
    Optional<Role> findByName(String name);
}

public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

### Step 1.6: Data Seeder

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

        // Permissions
        Permission bookRead = createPermissionIfNotExists("BOOK_READ");
        Permission bookCreate = createPermissionIfNotExists("BOOK_CREATE");
        Permission bookUpdate = createPermissionIfNotExists("BOOK_UPDATE");
        Permission bookDelete = createPermissionIfNotExists("BOOK_DELETE");
        Permission userManage = createPermissionIfNotExists("USER_MANAGE");

        // ADMIN role
        if (roleRepository.findByName("ADMIN").isEmpty()) {
            Role admin = new Role();
            admin.setName("ADMIN");
            admin.setPermissions(Set.of(
                bookRead, bookCreate, bookUpdate, bookDelete, userManage
            ));
            roleRepository.save(admin);
            log.info("ADMIN role created");
        }

        // EDITOR role
        if (roleRepository.findByName("EDITOR").isEmpty()) {
            Role editor = new Role();
            editor.setName("EDITOR");
            editor.setPermissions(Set.of(bookRead, bookCreate, bookUpdate));
            roleRepository.save(editor);
            log.info("EDITOR role created");
        }

        // MEMBER role
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

### Test করো

```bash
mvn spring-boot:run

Console এ দেখবে:
ADMIN role created
EDITOR role created
MEMBER role created

Database check:
SELECT * FROM permissions;  -- 5 rows
SELECT * FROM roles;         -- 3 rows
SELECT * FROM role_permissions;  -- mappings
```

---

<a name="phase-2"></a>

## Phase 2: Password Security Setup

### Step 2.1: PasswordEncoder Bean

```java
@Configuration
public class PasswordConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }
}
```

### Step 2.2: Test করো

```java
@SpringBootTest
class PasswordEncoderTest {

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Test
    void testPasswordEncoding() {
        String raw = "MyPass123";
        String encoded = passwordEncoder.encode(raw);

        System.out.println("Raw: " + raw);
        System.out.println("Encoded: " + encoded);

        assertTrue(passwordEncoder.matches(raw, encoded));
    }
}
```

---

<a name="phase-3"></a>

## Phase 3: JWT Utility Setup

### Step 3.1: application.properties

```properties
# JWT Configuration
jwt.secret=YourVerySecretKeyMustBeAtLeast256BitsLongForHS256Algorithm12345
jwt.expiration=3600000
# 1 hour = 3600000 milliseconds

# Production এ environment variable use করো:
# jwt.secret=${JWT_SECRET}
```

### Step 3.2: JwtUtil Class

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

    // 1. Token Generate
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

    // 2. Username Extract
    public String extractUsername(String token) {
        return parseClaims(token).getSubject();
    }

    // 3. Roles Extract
    @SuppressWarnings("unchecked")
    public Set<String> extractRoles(String token) {
        List<String> roles = parseClaims(token).get("roles", List.class);
        return new HashSet<>(roles);
    }

    // 4. Token Validate
    public boolean validateToken(String token) {
        try {
            parseClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    // 5. Token Expired Check
    public boolean isTokenExpired(String token) {
        return parseClaims(token).getExpiration().before(new Date());
    }

    // Helper
    private Claims parseClaims(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }
}
```

### Step 3.3: Test করো

```java
@SpringBootTest
class JwtUtilTest {

    @Autowired
    private JwtUtil jwtUtil;

    @Test
    void testTokenGeneration() {
        Set<String> roles = Set.of("MEMBER");
        Set<String> permissions = Set.of("BOOK_READ");

        String token = jwtUtil.generateToken("testuser", roles, permissions);
        System.out.println("Token: " + token);

        String username = jwtUtil.extractUsername(token);
        assertEquals("testuser", username);

        assertTrue(jwtUtil.validateToken(token));
    }
}
```

---

<a name="phase-4"></a>

## Phase 4: Spring Security Integration

### Step 4.1: CustomUserDetails Wrapper

```java
public class CustomUserDetails implements UserDetails {

    private final User user;

    public CustomUserDetails(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        Set<GrantedAuthority> authorities = new HashSet<>();

        user.getRoles().forEach(role -> {
            // Role add with "ROLE_" prefix
            authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));

            // Permissions add without prefix
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
        return true;
    }

    @Override
    public boolean isAccountNonLocked() {
        return true;
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

### Step 4.2: CustomUserDetailsService

```java
@Service
@RequiredArgsConstructor
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username)
            throws UsernameNotFoundException {

        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found with username: " + username
            ));

        return new CustomUserDetails(user);
    }
}
```

### Step 4.3: Test করো

```java
@SpringBootTest
class CustomUserDetailsServiceTest {

    @Autowired
    private CustomUserDetailsService userDetailsService;

    @Test
    void testLoadUserByUsername() {
        // আগে একটা user create করে test করো
        UserDetails details = userDetailsService.loadUserByUsername("testuser");

        assertNotNull(details);
        System.out.println("Username: " + details.getUsername());
        System.out.println("Authorities: " + details.getAuthorities());
    }
}
```

---

<a name="phase-5"></a>

## Phase 5: DTOs Design

### Step 5.1: RegisterRequest

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class RegisterRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    private String username;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    @NotBlank(message = "Full name is required")
    private String fullName;
}
```

### Step 5.2: LoginRequest

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class LoginRequest {

    @NotBlank(message = "Username is required")
    private String username;

    @NotBlank(message = "Password is required")
    private String password;
}
```

### Step 5.3: LoginResponse

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class LoginResponse {
    private String token;
    private String tokenType;       // "Bearer"
    private long expiresIn;
    private String username;
    private Set<String> roles;
    private Set<String> permissions;
}
```

### Step 5.4: ErrorResponse (if not exists)

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private String path;
    private String timestamp;
}
```

---

<a name="phase-6"></a>

## Phase 6: Security Configuration

### Step 6.1: SecurityConfig Class

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;
    // JwtAuthFilter পরে inject করবো

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider());
            // .addFilterBefore() পরে যোগ করবো

        return http.build();
    }
}
```

### Step 6.2: Test করো

```bash
mvn spring-boot:run

Console এ Spring Security related logs দেখবে
Error না থাকলে → Next step
```

---

<a name="phase-7"></a>

## Phase 7: JWT Authentication Filter

### Step 7.1: JwtAuthenticationFilter Class

```java
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final CustomUserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        // Step 1: Header extract
        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // Step 2: Token extract
        final String token = authHeader.substring(7);

        try {
            // Step 3: Validate
            if (!jwtUtil.validateToken(token)) {
                filterChain.doFilter(request, response);
                return;
            }

            // Step 4: Extract username
            String username = jwtUtil.extractUsername(token);

            // Step 5: SecurityContext check
            if (username != null
                && SecurityContextHolder.getContext().getAuthentication() == null) {

                // Step 6: Load UserDetails
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // Step 7: Create Authentication
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );

                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );

                // Step 8: Set SecurityContext
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }

        } catch (Exception e) {
            logger.error("Cannot set user authentication: " + e.getMessage());
        }

        // Step 9: Next filter
        filterChain.doFilter(request, response);
    }
}
```

### Step 7.2: SecurityConfig Update

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;  // ← যোগ করো
    private final CustomUserDetailsService userDetailsService;

    // ... বাকি সব same

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);  // ← যোগ করো

        return http.build();
    }
}
```

### Step 7.3: Test করো

```bash
# Dummy protected endpoint বানাও test এর জন্য
@RestController
@RequestMapping("/api/test")
public class TestController {

    @GetMapping("/protected")
    public String protectedEndpoint() {
        return "Protected endpoint accessed!";
    }
}

# Postman এ:
GET /api/test/protected (No token)
Expected: 401 Unauthorized ✅
```

---

<a name="phase-8"></a>

## Phase 8: Authentication Service

### Step 8.1: AuthService

```java
@Service
@RequiredArgsConstructor
@Transactional
public class AuthService {

    private final UserRepository userRepository;
    private final RoleRepository roleRepository;
    private final PasswordEncoder passwordEncoder;
    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;

    @Value("${jwt.expiration}")
    private long jwtExpiration;

    public void register(RegisterRequest request) {

        // Duplicate check
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateResourceException(
                "Username already exists: " + request.getUsername()
            );
        }

        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException(
                "Email already exists: " + request.getEmail()
            );
        }

        // Default role
        Role memberRole = roleRepository.findByName("MEMBER")
            .orElseThrow(() -> new ResourceNotFoundException("MEMBER role not found"));

        // Create user
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .enabled(true)
            .roles(Set.of(memberRole))
            .build();

        userRepository.save(user);
    }

    public LoginResponse login(LoginRequest request) {

        // Authenticate
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        // Extract user info
        CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
        User user = userDetails.getUser();

        // Extract roles
        Set<String> roles = user.getRoles().stream()
            .map(Role::getName)
            .collect(Collectors.toSet());

        // Extract permissions
        Set<String> permissions = user.getRoles().stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(Permission::getName)
            .collect(Collectors.toSet());

        // Generate token
        String token = jwtUtil.generateToken(user.getUsername(), roles, permissions);

        // Build response
        return LoginResponse.builder()
            .token(token)
            .tokenType("Bearer")
            .expiresIn(jwtExpiration)
            .username(user.getUsername())
            .roles(roles)
            .permissions(permissions)
            .build();
    }
}
```

### Step 8.2: AuthController

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<Map<String, String>> register(
            @Valid @RequestBody RegisterRequest request) {

        authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(
            Map.of("message", "User registered successfully")
        );
    }

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @Valid @RequestBody LoginRequest request) {

        LoginResponse response = authService.login(request);
        return ResponseEntity.ok(response);
    }
}
```

---

<a name="phase-9"></a>

## Phase 9: Exception Handling

### Step 9.1: GlobalExceptionHandler Update

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // ... আগের handlers

    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleBadCredentials(
            BadCredentialsException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            "Invalid username or password",
            request.getRequestURI(),
            LocalDateTime.now().toString()
        );
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }

    @ExceptionHandler(UsernameNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UsernameNotFoundException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            "User not found",
            request.getRequestURI(),
            LocalDateTime.now().toString()
        );
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.FORBIDDEN.value(),
            "Forbidden",
            "You don't have permission to access this resource",
            request.getRequestURI(),
            LocalDateTime.now().toString()
        );
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }
}
```

---

<a name="phase-10"></a>

## Phase 10: Complete Flow Test

### Test 1: Register

```http
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{
  "username": "hasib",
  "email": "hasib@mail.com",
  "password": "MyPass123!",
  "fullName": "Hasibuzzaman"
}

Expected: 201 Created
Response: { "message": "User registered successfully" }

Database Check:
SELECT * FROM users WHERE username = 'hasib';
-- password hashed, enabled = true

SELECT * FROM user_roles WHERE user_id = 1;
-- role_id = 3 (MEMBER)
```

### Test 2: Register Duplicate

```http
POST http://localhost:8080/api/auth/register
(same username)

Expected: 409 Conflict
Response: { "message": "Username already exists: hasib" }
```

### Test 3: Login Success

```http
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
  "username": "hasib",
  "password": "MyPass123!"
}

Expected: 200 OK
Response:
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600000,
  "username": "hasib",
  "roles": ["MEMBER"],
  "permissions": ["BOOK_READ"]
}
```

### Test 4: Login Wrong Password

```http
POST http://localhost:8080/api/auth/login

{
  "username": "hasib",
  "password": "WRONG"
}

Expected: 401 Unauthorized
Response: { "message": "Invalid username or password" }
```

### Test 5: Protected API Without Token

```http
GET http://localhost:8080/api/test/protected

Expected: 401 Unauthorized
```

### Test 6: Protected API With Token

```http
GET http://localhost:8080/api/test/protected
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...

Expected: 200 OK
Response: "Protected endpoint accessed!"
```

---

<a name="phase-11"></a>

## Phase 11: Method Level Security (Part 3)

### Step 11.1: Enable Method Security

SecurityConfig এ ইতোমধ্যে `@EnableMethodSecurity` আছে। এখন use করো।

### Step 11.2: @PreAuthorize Apply

```java
@RestController
@RequestMapping("/api/books")
@RequiredArgsConstructor
public class BookController {

    private final BookService bookService;

    @GetMapping
    @PreAuthorize("hasAuthority('BOOK_READ')")
    public List<Book> getAllBooks() {
        return bookService.getAllBooks();
    }

    @PostMapping
    @PreAuthorize("hasAuthority('BOOK_CREATE')")
    public Book createBook(@Valid @RequestBody BookDTO dto) {
        return bookService.createBook(dto);
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOK_UPDATE')")
    public Book updateBook(@PathVariable Long id, @Valid @RequestBody BookDTO dto) {
        return bookService.updateBook(id, dto);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOK_DELETE')")
    public void deleteBook(@PathVariable Long id) {
        bookService.deleteBook(id);
    }
}
```

### Step 11.3: Test Authorization

```http
# MEMBER user login করো → Token নাও

DELETE http://localhost:8080/api/books/1
Authorization: Bearer <MEMBER_TOKEN>

Expected: 403 Forbidden
Reason: MEMBER এর BOOK_DELETE permission নেই

# ADMIN user login করো → Token নাও

DELETE http://localhost:8080/api/books/1
Authorization: Bearer <ADMIN_TOKEN>

Expected: 200 OK
Reason: ADMIN এর BOOK_DELETE permission আছে
```

---

<a name="phase-12"></a>

## Phase 12: Advanced Features (Optional)

### Feature 1: Refresh Token

#### RefreshToken Entity

```java
@Entity
@Table(name = "refresh_tokens")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String token;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(nullable = false)
    private LocalDateTime expiresAt;

    @Column(nullable = false)
    private boolean revoked = false;
}
```

#### RefreshTokenRepository

```java
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    Optional<RefreshToken> findByToken(String token);

    void deleteByUser(User user);

    @Modifying
    @Query("UPDATE RefreshToken rt SET rt.revoked = true WHERE rt.user.id = :userId")
    void revokeAllByUserId(Long userId);
}
```

#### RefreshTokenService

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;

    @Value("${jwt.refresh.expiration:604800000}")  // 7 days
    private long refreshExpirationMs;

    public RefreshToken createRefreshToken(User user) {
        refreshTokenRepository.revokeAllByUserId(user.getId());

        RefreshToken token = RefreshToken.builder()
            .token(UUID.randomUUID().toString())
            .user(user)
            .expiresAt(LocalDateTime.now().plusNanos(refreshExpirationMs * 1_000_000))
            .revoked(false)
            .build();

        return refreshTokenRepository.save(token);
    }

    public RefreshToken validateRefreshToken(String token) {
        RefreshToken refreshToken = refreshTokenRepository.findByToken(token)
            .orElseThrow(() -> new UnauthorizedException("Invalid refresh token"));

        if (refreshToken.isRevoked()) {
            throw new UnauthorizedException("Refresh token revoked");
        }

        if (refreshToken.getExpiresAt().isBefore(LocalDateTime.now())) {
            throw new UnauthorizedException("Refresh token expired");
        }

        return refreshToken;
    }

    public void revokeToken(String token) {
        refreshTokenRepository.findByToken(token).ifPresent(rt -> {
            rt.setRevoked(true);
            refreshTokenRepository.save(rt);
        });
    }
}
```

#### AuthService Update

```java
// login() method এ যোগ করো:
RefreshToken refreshToken = refreshTokenService.createRefreshToken(user);

// LoginResponse এ যোগ করো:
.refreshToken(refreshToken.getToken())
```

#### Refresh Endpoint

```java
@PostMapping("/refresh")
public ResponseEntity<RefreshResponse> refresh(
        @Valid @RequestBody RefreshRequest request) {

    RefreshToken refreshToken = refreshTokenService
        .validateRefreshToken(request.getRefreshToken());

    User user = refreshToken.getUser();

    // Generate new access token
    Set<String> roles = user.getRoles().stream()
        .map(Role::getName)
        .collect(Collectors.toSet());

    Set<String> permissions = user.getRoles().stream()
        .flatMap(role -> role.getPermissions().stream())
        .map(Permission::getName)
        .collect(Collectors.toSet());

    String newAccessToken = jwtUtil.generateToken(
        user.getUsername(), roles, permissions
    );

    return ResponseEntity.ok(RefreshResponse.builder()
        .accessToken(newAccessToken)
        .tokenType("Bearer")
        .expiresIn(jwtExpiration)
        .build());
}
```

### Feature 2: Logout

```java
@PostMapping("/logout")
public ResponseEntity<Map<String, String>> logout(
        @Valid @RequestBody LogoutRequest request) {

    refreshTokenService.revokeToken(request.getRefreshToken());

    return ResponseEntity.ok(Map.of("message", "Logged out successfully"));
}
```

### Feature 3: Get Current User

```java
@GetMapping("/me")
public ResponseEntity<UserInfoDTO> getCurrentUser(
        @AuthenticationPrincipal CustomUserDetails userDetails) {

    User user = userDetails.getUser();

    return ResponseEntity.ok(UserInfoDTO.builder()
        .username(user.getUsername())
        .email(user.getEmail())
        .roles(user.getRoles().stream()
            .map(Role::getName)
            .collect(Collectors.toSet()))
        .build());
}
```

---

<a name="phase-13"></a>

## Phase 13: Production Checklist

```
Security:
[ ] HTTPS enabled
[ ] Strong secret key (64+ chars, environment variable)
[ ] Short access token expiry (≤ 1 hour)
[ ] Refresh token implemented
[ ] Logout works
[ ] Generic error messages (no info leak)
[ ] No sensitive data in JWT payload
[ ] CORS configured properly
[ ] Rate limiting on login endpoint

Testing:
[ ] All flows tested (register, login, protected API)
[ ] Wrong password → 401
[ ] No permission → 403
[ ] Token expired → 401
[ ] Different roles tested (ADMIN, EDITOR, MEMBER)
[ ] Refresh token flow tested
[ ] Logout tested

Code Quality:
[ ] No hardcoded secrets
[ ] Environment variables for sensitive data
[ ] Exception handling complete
[ ] Validation on all DTOs
[ ] Logging configured
[ ] Transaction boundaries correct (@Transactional)

Database:
[ ] Passwords hashed (BCrypt)
[ ] Unique constraints on username/email
[ ] Indexes on frequently queried columns
[ ] Roles and permissions seeded
[ ] Foreign keys configured

Documentation:
[ ] API documentation (Swagger/OpenAPI)
[ ] README with setup instructions
[ ] Environment variables documented
```

---

<a name="build-order"></a>

## Build Order — Final Summary

```
Phase 1: Database Design
  1. Permission entity
  2. Role entity
  3. User entity
  4. Repositories
  5. DataSeeder

Phase 2: Password Security
  6. PasswordEncoder Bean

Phase 3: JWT
  7. application.properties (jwt.secret)
  8. JwtUtil class

Phase 4: Spring Security Bridge
  9. CustomUserDetails
  10. CustomUserDetailsService

Phase 5: DTOs
  11. RegisterRequest
  12. LoginRequest
  13. LoginResponse
  14. ErrorResponse

Phase 6: Security Config
  15. SecurityConfig (beans 1-3)

Phase 7: Filter
  16. JwtAuthenticationFilter
  17. SecurityConfig update (addFilterBefore)

Phase 8: Auth Logic
  18. AuthService
  19. AuthController

Phase 9: Error Handling
  20. GlobalExceptionHandler update

Phase 10: Test
  21. Register flow
  22. Login flow
  23. Protected API flows

Phase 11: Authorization
  24. @PreAuthorize add
  25. Test permissions

Phase 12: Advanced (Optional)
  26. Refresh Token
  27. Logout
  28. Get Current User

Phase 13: Production Ready
  29. Production checklist verify
  30. Deploy
```

---

<a name="common-mistakes"></a>

## Common Mistakes — এড়িয়ে চলো

```
❌ filterChain.doFilter() ভুলে যাওয়া
   → Request আটকে যাবে, Controller reach হবে না

❌ @Valid annotation ভুলে যাওয়া
   → Validation কাজ করবে না

❌ SecurityConfig এ addFilterBefore() না দেওয়া
   → JwtAuthFilter চলবে না

❌ CustomUserDetails এ getAuthorities() ভুল implementation
   → Permission check কাজ করবে না

❌ JWT secret weak রাখা
   → Security breach হতে পারে

❌ Token URL এ পাঠানো
   → Log এ expose হবে, unsafe

❌ Production এ HTTPS ছাড়া deploy করা
   → Token sniff করা যাবে

❌ @ManyToMany এ fetch type ভুল
   → LazyInitializationException

❌ Password plain text save করা
   → Security breach

❌ Generic error message না দেওয়া
   → Info leak হবে hacker এর কাছে
```

---

<a name="debugging-guide"></a>

## Debugging Guide — যদি কাজ না করে

### Problem 1: 401 সব জায়গায়

```
Check:
□ JwtAuthFilter চলছে কিনা (log add করো)
□ SecurityContext এ user set হচ্ছে কিনা
□ Token valid কিনা (JwtUtil test করো)
□ SecurityConfig এ addFilterBefore() আছে কিনা

Debug:
JwtAuthFilter এ প্রতিটা step এ log দাও:
logger.info("Step 1: Header = " + authHeader);
logger.info("Step 8: SecurityContext set");
```

### Problem 2: 403 আসছে

```
Check:
□ @PreAuthorize এর permission database এ আছে কিনা
□ getAuthorities() সঠিক authorities return করছে কিনা
□ Role → Permission mapping ঠিক আছে কিনা

Debug:
CustomUserDetails এ log দাও:
logger.info("Authorities: " + getAuthorities());
```

### Problem 3: Login Fail করছে

```
Check:
□ Password encode হচ্ছে কিনা (register এ)
□ AuthenticationManager configured কিনা
□ CustomUserDetailsService loadUserByUsername() ঠিকমতো কাজ করছে কিনা

Debug:
AuthService.login() এ log:
logger.info("Authenticating: " + request.getUsername());
```

### Problem 4: Token Generate হচ্ছে না

```
Check:
□ jwt.secret set করা আছে কিনা
□ Secret key যথেষ্ট লম্বা কিনা (256 bits minimum)
□ JwtUtil inject হচ্ছে কিনা

Debug:
JwtUtil test লিখো আলাদাভাবে
```

### Problem 5: Database Query Issue

```
Check:
□ @ManyToMany fetch = EAGER দিয়েছো কিনা
□ @Transactional দিয়েছো কিনা
□ Database connection ঠিক আছে কিনা

Debug:
application.properties এ:
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

SQL queries দেখো
```

---

<a name="final-tips"></a>

## Final Tips

```
✅ একবারে সব না বানিয়ে phase by phase করো
✅ প্রতি phase এ test করো
✅ Console log দেখো — error না থাকলে next phase
✅ Postman collection বানাও — সব APIs test এর জন্য
✅ Database check করো — data ঠিকমতো save হচ্ছে কিনা
✅ Part 1, 2, 3 tutorials side by side খোলা রাখো — reference হিসেবে
✅ Git commit করো প্রতি working phase এ
✅ Test coverage maintain করো
✅ Code review করো নিজে — common mistakes check করো
✅ Production deploy এর আগে security checklist verify করো
```

---

## Resources

```
Official Docs:
- Spring Security: https://spring.io/projects/spring-security
- JWT: https://jwt.io/
- BCrypt: https://en.wikipedia.org/wiki/Bcrypt

Tools:
- Postman: API testing
- jwt.io: Token decode/verify
- DB Browser: Database inspection
```

---

## শেষ কথা

এই guide follow করে step by step implementation করো। কোনো step এ আটকে গেলে:

1. Error message carefully পড়ো
2. Debugging guide check করো
3. Specific part এর tutorial খুলে detail দেখো
4. Console logs add করে trace করো
5. Test লিখে individual component verify করো

Remember: **Production-grade system বানাতে time লাগে। তাড়াহুড়া করো না।**

🚀 **Happy Building!**
