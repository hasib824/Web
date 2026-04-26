# Spring Security + JWT — Part 2: Authentication Flow

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Previous:** Part 1 (Foundation)
> **Part 2 Topics:** SecurityConfig, JWT Filter, Login/Register APIs, Complete Auth Flow

> ⚠️ এই Part শেষে — তুমি actually login করতে পারবে, JWT Token পাবে, Postman দিয়ে test করতে পারবে।

---

## সূচিপত্র

1. [Phase 1: SecurityConfig](#phase-1)
2. [Phase 2: JWT Authentication Filter](#phase-2)
3. [Phase 3: DTO Design](#phase-3)
4. [Phase 4: AuthService](#phase-4)
5. [Phase 5: AuthController](#phase-5)
6. [Phase 6: Test Flow with Postman](#phase-6)
7. [Part 2 এর সারসংক্ষেপ](#summary)

---

<a name="phase-1"></a>

# Phase 1 — SecurityConfig

এই class টা **পুরো Security এর central config।** এখানে সব decision হবে — কোন API public, কোন protected, password কীভাবে check হবে, JWT filter কোথায় বসবে।

---

## প্রথমে কিছু Concept বুঝি

### `SecurityFilterChain` কী?

Spring Security একটা **Filter Chain** ব্যবহার করে। প্রতিটা request এ কতগুলো filter চলে — একে অন্যের পরে।

```
Client Request
    ↓
Filter 1 (Authentication)
    ↓
Filter 2 (Authorization)
    ↓
Filter 3 (CORS)
    ↓
Filter 4 (CSRF)
    ↓
... আরো অনেক filter
    ↓
Controller (যদি সব filter pass করে)
```

আমাদের নিজের **JWT Filter** এই chain এ যোগ করতে হবে।

### Stateless Session

Session-based auth এ Server user info মনে রাখে। কিন্তু JWT-based এ —

> *"Server কিছুই মনে রাখবে না। প্রতিটা request এ Token দেখে বুঝবে user কে।"*

এটাই **Stateless।**

### `AuthenticationManager` কী?

Spring Security এর core component যেটা actual authentication কাজটা করে।

```
User: "Username = hasib, password = MyPass123"
        ↓
AuthenticationManager:
  - UserDetailsService দিয়ে user load করো
  - PasswordEncoder দিয়ে password match করো
  - Match হলে → Authenticated
        ↓
ফলাফল return করে
```

আমরা এটা সরাসরি use করবো — Login এর সময়।

---

## SecurityConfig Class

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity                          // @PreAuthorize use করার জন্য
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final CustomUserDetailsService userDetailsService;

    // ১. PasswordEncoder Bean
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(10);
    }

    // ২. AuthenticationProvider Bean
    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    // ৩. AuthenticationManager Bean
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    // ৪. SecurityFilterChain Bean (সবচেয়ে important!)
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
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## প্রতিটা Annotation / Method Decode

### `@Configuration`

> *"এই class এ Spring Bean definitions আছে।"*

Spring এই class scan করবে এবং `@Bean` methods গুলো execute করবে।

### `@EnableWebSecurity`

> *"Spring Security activate করো।"*

এটা না দিলে security কাজ করবে না।

### `@EnableMethodSecurity`

> *"`@PreAuthorize`, `@PostAuthorize` annotations use করতে allow করো।"*

পরে method level security এর জন্য।

---

### Bean 1: `PasswordEncoder`

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(10);
}
```

Part 1 এ explain করেছি। Strength = 10 (default)।

### Bean 2: `AuthenticationProvider`

```java
@Bean
public AuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);
    provider.setPasswordEncoder(passwordEncoder());
    return provider;
}
```

#### কী করে?

`AuthenticationProvider` actual authentication logic চালায় —

```
User credentials এলো
        ↓
DaoAuthenticationProvider:
  1. UserDetailsService.loadUserByUsername(username) call
  2. Database থেকে User পেলো
  3. PasswordEncoder.matches(rawPwd, hashedPwd) call
  4. Match হলে → Authenticated
```

#### `DaoAuthenticationProvider` কেন?

DAO = Data Access Object. মানে — **Database থেকে user load করে authenticate করে।**

Spring এ অনেক provider আছে (LDAP, OAuth, etc.)। আমরা database use করছি, তাই DAO।

### Bean 3: `AuthenticationManager`

```java
@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

এটা Login API তে use করবো — manually authenticate করার জন্য।

---

### Bean 4: `SecurityFilterChain` (সবচেয়ে Important!)

```java
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
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

প্রতিটা line ভেঙে বুঝি —

#### `.csrf(csrf -> csrf.disable())`

CSRF (Cross-Site Request Forgery) protection disable করছি।

**কেন?**

CSRF attack browser-based applications এ হয় (cookie based)। JWT-based stateless API তে CSRF risk নেই — কারণ Token Header এ পাঠানো হয়, automatically যায় না।

```
Cookie-based: Browser auto cookie পাঠায় → CSRF risk
JWT-based:    Manual Token পাঠাতে হয় → CSRF risk নেই
```

#### `.sessionManagement(... STATELESS)`

> *"Session create করো না।"*

JWT-based এ Server কোনো session রাখে না। প্রতিটা request stateless।

#### `.authorizeHttpRequests(...)`

> *"কোন URL এ কে access করতে পারবে — এটা decide করো।"*

```java
.requestMatchers("/api/auth/**").permitAll()
// → /api/auth/* সব URL public, কেউ access করতে পারে
//   (login, register এর জন্য)

.requestMatchers("/api/public/**").permitAll()
// → /api/public/* public

.anyRequest().authenticated()
// → বাকি সব URL এ JWT Token দরকার
```

**Order matters!** উপর থেকে নিচে check হয়। `anyRequest()` সবসময় শেষে।

#### `.authenticationProvider(authenticationProvider())`

আমাদের custom provider register করি।

#### `.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)`

> *"আমার JWT Filter কে `UsernamePasswordAuthenticationFilter` এর আগে যোগ করো।"*

#### কেন আগে?

```
Default Filter Chain:
  ... → UsernamePasswordAuthenticationFilter → ...
                                                    
                ⬇ আমাদের JWT Filter এখানে বসবে

New Filter Chain:
  ... → JwtAuthenticationFilter → UsernamePasswordAuthenticationFilter → ...
```

JWT Token check হবে আগে। যদি valid হয় — user authenticated হয়ে যায়, পরের filter কিছু করতে হয় না।

---

<a name="phase-2"></a>

# Phase 2 — JWT Authentication Filter (সবচেয়ে Important!)

এই Filter টা প্রতিটা incoming request এ চলে — Token check করে user কে authenticate করে।

## Filter কী?

আগে বুঝেছি — Filter Chain এ অনেক filter থাকে। প্রতিটা request প্রতিটা filter এর মধ্য দিয়ে যায়।

```
Request → Filter 1 → Filter 2 → Filter 3 → ... → Controller
```

প্রতিটা Filter পারে —
- Request modify করতে
- Block করতে
- Information add করতে

আমাদের JWT Filter এর কাজ —

```
Request আসলো
    ↓
Authorization Header থেকে Token extract করো
    ↓
Token valid?
    ↓
Yes → User authenticate করো (SecurityContext এ set)
No  → Skip (পরের filter এ)
    ↓
Next Filter
```

---

## OncePerRequestFilter

Spring Security এর একটা base class। এটা guarantee করে — **প্রতি request এ filter একবারই চলবে।**

(কখনো কখনো একই request multiple time process হয় — তখন ভুল হবে। `OncePerRequestFilter` এই সমস্যা solve করে।)

---

## JwtAuthenticationFilter Class

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

        // ১. Authorization Header থেকে Token extract
        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);   // Token নেই, skip
            return;
        }

        // ২. "Bearer " prefix সরিয়ে Token নাও
        final String token = authHeader.substring(7);

        try {
            // ৩. Token validate
            if (!jwtUtil.validateToken(token)) {
                filterChain.doFilter(request, response);
                return;
            }

            // ৪. Token থেকে username extract
            String username = jwtUtil.extractUsername(token);

            // ৫. SecurityContext এ user আগে থেকে নেই কিনা check
            if (username != null
                && SecurityContextHolder.getContext().getAuthentication() == null) {

                // ৬. Database থেকে UserDetails load
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // ৭. Authentication object তৈরি
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );

                // Request এর details যোগ করি
                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );

                // ৮. SecurityContext এ set করি
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }

        } catch (Exception e) {
            // Token invalid হলে log করো, কিন্তু request চালু রাখো
            logger.error("Cannot set user authentication: " + e.getMessage());
        }

        // ৯. Next filter কে চালু করো
        filterChain.doFilter(request, response);
    }
}
```

---

## প্রতিটা Step বিস্তারিত

### Step 1 — Authorization Header Extract

```java
final String authHeader = request.getHeader("Authorization");
```

Client request এ এটা থাকা উচিত —

```
Authorization: Bearer eyJhbGc...
```

```java
if (authHeader == null || !authHeader.startsWith("Bearer ")) {
    filterChain.doFilter(request, response);
    return;
}
```

Header না থাকলে বা `Bearer ` দিয়ে শুরু না হলে — skip করে next filter এ যাই।

⚠️ **এখানে error throw করি না** — কারণ public API এ Token দরকার নেই।

### Step 2 — Token Extract

```java
final String token = authHeader.substring(7);
```

`"Bearer "` 7 character (space সহ)। তাই `substring(7)` দিয়ে actual token নিই।

```
"Bearer eyJhbGc..."
       ↑
   substring(7) → "eyJhbGc..."
```

### Step 3 — Token Validate

```java
if (!jwtUtil.validateToken(token)) {
    filterChain.doFilter(request, response);
    return;
}
```

Part 1 এ JwtUtil এর `validateToken()` দেখেছি। Signature wrong বা expired হলে false।

### Step 4 — Username Extract

```java
String username = jwtUtil.extractUsername(token);
```

Token এর `sub` claim থেকে username বের।

### Step 5 — SecurityContext Check

```java
if (username != null
    && SecurityContextHolder.getContext().getAuthentication() == null) {
```

#### `SecurityContextHolder` কী?

Spring Security এর একটা **container** যেটা current request এর authenticated user store করে।

```
Thread 1 (User A's request):
  SecurityContext → Authentication(user: "Hasib")

Thread 2 (User B's request):
  SecurityContext → Authentication(user: "Karim")
```

প্রতিটা request এর জন্য আলাদা context।

#### কেন check করি `== null`?

যদি আগের কোনো filter user কে already authenticate করে থাকে — আবার করার দরকার নেই।

### Step 6 — UserDetails Load

```java
UserDetails userDetails = userDetailsService.loadUserByUsername(username);
```

Part 1 এ আমাদের `CustomUserDetailsService` বানিয়েছি। এটা database থেকে user load করে।

### Step 7 — Authentication Object Create

```java
UsernamePasswordAuthenticationToken authToken =
    new UsernamePasswordAuthenticationToken(
        userDetails,        // Principal (user info)
        null,               // Credentials (password — already verified)
        userDetails.getAuthorities()    // Roles + Permissions
    );
```

#### Constructor parameters

| Parameter | কী |
|---|---|
| `userDetails` | User info |
| `null` (credentials) | Password — Token এ already verified, এখানে দরকার নেই |
| `getAuthorities()` | Roles + Permissions list |

### Step 8 — SecurityContext এ Set

```java
SecurityContextHolder.getContext().setAuthentication(authToken);
```

এটা সবচেয়ে important line!

এই moment এ — **Spring বুঝে যায় user authenticated।** পরের filter, controller সবাই জানতে পারে কে user।

```
Controller এ:
@GetMapping("/api/me")
public String me(Authentication auth) {
    return auth.getName();   // → "hasib"
}
```

### Step 9 — Next Filter

```java
filterChain.doFilter(request, response);
```

Filter chain এ পরের filter এ পাঠাও।

⚠️ **এই line না দিলে request আটকে যাবে! সবসময় শেষে দাও।**

---

## পুরো Flow Visualize

```
Request: GET /api/books
Headers: Authorization: Bearer eyJhbGc...
        ↓
JwtAuthenticationFilter:
        ↓
  1. Header extract: "Bearer eyJhbGc..."
        ↓
  2. Token extract: "eyJhbGc..."
        ↓
  3. validateToken("eyJhbGc...") → true
        ↓
  4. extractUsername() → "hasib"
        ↓
  5. SecurityContext empty? → yes
        ↓
  6. loadUserByUsername("hasib") → UserDetails
        ↓
  7. Authentication object create
        ↓
  8. SecurityContext এ set
        ↓
  9. Next filter
        ↓
        ...
        ↓
Controller chalu হলো
```

---

<a name="phase-3"></a>

# Phase 3 — DTO Design

API তে data নেওয়া এবং দেওয়ার জন্য DTO বানাই।

## RegisterRequest

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

Validation আগের tutorial এ শিখেছি — সব apply করা।

## LoginRequest

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

## LoginResponse

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class LoginResponse {
    private String token;
    private String tokenType;       // "Bearer"
    private long expiresIn;          // milliseconds
    private String username;
    private Set<String> roles;
    private Set<String> permissions;
}
```

Login successful হলে এটা return হবে।

---

<a name="phase-4"></a>

# Phase 4 — AuthService

Business logic এর central place — register এবং login।

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

    // ১. Register
    public void register(RegisterRequest request) {

        // Username/email duplicate check
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

        // Default role assign করি (MEMBER)
        Role memberRole = roleRepository.findByName("MEMBER")
            .orElseThrow(() -> new ResourceNotFoundException("MEMBER role not found"));

        // User create
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))   // ⭐ Hash
            .enabled(true)
            .roles(Set.of(memberRole))
            .build();

        userRepository.save(user);
    }

    // ২. Login
    public LoginResponse login(LoginRequest request) {

        // ক) AuthenticationManager দিয়ে authenticate
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        // খ) Authentication successful — user info নাও
        CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
        User user = userDetails.getUser();

        // গ) Roles ও permissions extract
        Set<String> roles = user.getRoles().stream()
            .map(Role::getName)
            .collect(Collectors.toSet());

        Set<String> permissions = user.getRoles().stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(Permission::getName)
            .collect(Collectors.toSet());

        // ঘ) JWT Token generate
        String token = jwtUtil.generateToken(user.getUsername(), roles, permissions);

        // ঙ) Response build
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

---

## প্রতিটা Step বিস্তারিত

### Register Method

#### Duplicate Check

```java
if (userRepository.existsByUsername(request.getUsername())) {
    throw new DuplicateResourceException(...);
}
```

আগে Exception Handling tutorial এ যে custom exception বানিয়েছি — সেটা use করছি।

#### Default Role Assign

```java
Role memberRole = roleRepository.findByName("MEMBER")
    .orElseThrow(...);
```

নতুন registered user automatic `MEMBER` role পায়। ADMIN/EDITOR পরে assign করতে হবে।

#### Password Hash + Save

```java
.password(passwordEncoder.encode(request.getPassword()))
```

Plain text password কখনো save হবে না — সবসময় hash।

---

### Login Method (সবচেয়ে Important!)

#### `authenticationManager.authenticate(...)`

```java
Authentication authentication = authenticationManager.authenticate(
    new UsernamePasswordAuthenticationToken(
        request.getUsername(),
        request.getPassword()
    )
);
```

#### পর্দার পেছনে কী হয়?

```
authenticationManager.authenticate() call হলো
        ↓
Spring DaoAuthenticationProvider কে call করে
        ↓
Provider:
  1. CustomUserDetailsService.loadUserByUsername(username)
     → Database থেকে user নেয়
  2. PasswordEncoder.matches(rawPassword, hashedPassword)
     → Password match করে
  3. Match হলে → Authentication object return
     Match না হলে → BadCredentialsException throw
```

এই এক line এ — পুরো authentication হয়ে গেল!

#### `authentication.getPrincipal()`

```java
CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
```

`Principal` = authenticated user। আমরা type cast করে আমাদের `CustomUserDetails` এ নিচ্ছি।

#### Roles + Permissions Extract

```java
Set<String> roles = user.getRoles().stream()
    .map(Role::getName)
    .collect(Collectors.toSet());

Set<String> permissions = user.getRoles().stream()
    .flatMap(role -> role.getPermissions().stream())
    .map(Permission::getName)
    .collect(Collectors.toSet());
```

User এর Roles → Names extract।
প্রতি Role এর Permissions → Names extract।

#### JWT Generate + Response

```java
String token = jwtUtil.generateToken(user.getUsername(), roles, permissions);
```

Part 1 এর JwtUtil use করছি।

---

<a name="phase-5"></a>

# Phase 5 — AuthController

API endpoints —

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

## Exception Handler Update

আগের `GlobalExceptionHandler` এ Authentication exceptions যোগ করি —

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // আগের সব handlers...

    // ১. Wrong username/password
    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleBadCredentials(
            BadCredentialsException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            "Invalid username or password",
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }

    // ২. User not found
    @ExceptionHandler(UsernameNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleUserNotFound(
            UsernameNotFoundException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.UNAUTHORIZED.value(),
            "Unauthorized",
            "User not found",
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED).body(error);
    }

    // ৩. Access Denied (Authorization fail)
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.FORBIDDEN.value(),
            "Forbidden",
            "You don't have permission to access this resource",
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }
}
```

### 401 vs 403 — পার্থক্য

```
401 Unauthorized → Authentication fail (login করো)
403 Forbidden    → Authorization fail (login করেছো, কিন্তু permission নেই)
```

---

<a name="phase-6"></a>

# Phase 6 — Test Flow with Postman

এবার পুরো system test করি।

## Step 1: App Start

```bash
mvn spring-boot:run
```

Console এ দেখবে —

```
ADMIN role created
EDITOR role created
MEMBER role created
```

DataSeeder default roles তৈরি করেছে।

---

## Step 2: Register

### Request

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{
  "username": "hasib",
  "email": "hasib@mail.com",
  "password": "MyPass123",
  "fullName": "Hasibuzzaman"
}
```

### Response (Success)

```
Status: 201 Created

{
  "message": "User registered successfully"
}
```

### Database Check

```sql
SELECT * FROM users;
-- id=1, username=hasib, password=$2a$10$... (hashed)

SELECT * FROM user_roles;
-- user_id=1, role_id=3 (MEMBER)
```

### Response (Duplicate)

```
Status: 409 Conflict

{
  "status": 409,
  "error": "Conflict",
  "message": "Username already exists: hasib",
  "path": "/api/auth/register",
  "timestamp": "..."
}
```

---

## Step 3: Login

### Request

```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
  "username": "hasib",
  "password": "MyPass123"
}
```

### Response (Success)

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJoYXNpYiIsInJvbGVzIjpbIk1FTUJFUiJdLCJwZXJtaXNzaW9ucyI6WyJCT09LX1JFQUQiXSwiaWF0IjoxNzM1NTk2ODAwLCJleHAiOjE3MzU2ODMyMDB9.kPeRjMq...",
  "tokenType": "Bearer",
  "expiresIn": 3600000,
  "username": "hasib",
  "roles": ["MEMBER"],
  "permissions": ["BOOK_READ"]
}
```

### Response (Wrong Password)

```
Status: 401 Unauthorized

{
  "status": 401,
  "error": "Unauthorized",
  "message": "Invalid username or password",
  "path": "/api/auth/login",
  "timestamp": "..."
}
```

---

## Step 4: Token Save করো

Postman এ Token copy করে রাখো — পরবর্তী API calls এ লাগবে।

```
eyJhbGc...
```

---

## Step 5: Protected API Test

ধরো একটা protected API আছে —

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping
    public List<Book> getAllBooks() {
        return bookService.getAllBooks();
    }
}
```

### Without Token

```
GET http://localhost:8080/api/books
(no Authorization header)
```

```
Status: 401 Unauthorized   ← Token নেই
```

### With Valid Token

```
GET http://localhost:8080/api/books
Authorization: Bearer eyJhbGc...
```

```
Status: 200 OK
[
  { "id": 1, "title": "Pother Pachali" }
]
```

---

## Step 6: Pure Flow Trace

```
Time   Action                          Result
─────────────────────────────────────────────
T1     POST /api/auth/register          User created in DB
T2     POST /api/auth/login             JWT Token returned
T3     Save Token (e.g., postman var)   Token in memory
T4     GET /api/books                   401 Unauthorized
       (no Auth header)
T5     GET /api/books                   200 OK
       (Auth: Bearer ...)               Books returned
```

---

<a name="summary"></a>

# Part 2 এর সারসংক্ষেপ

## যা যা শিখলাম

```
✅ Phase 1: SecurityConfig
   - SecurityFilterChain
   - Stateless session
   - Public vs Protected URLs
   - AuthenticationProvider, AuthenticationManager

✅ Phase 2: JWT Authentication Filter
   - OncePerRequestFilter
   - Token extract from header
   - Token validate
   - SecurityContext এ user set
   - প্রতিটা step বিস্তারিত

✅ Phase 3: DTO Design
   - RegisterRequest, LoginRequest, LoginResponse
   - Validation apply

✅ Phase 4: AuthService
   - Register logic (duplicate check, hash, save)
   - Login logic (authenticate, generate JWT)

✅ Phase 5: AuthController
   - REST APIs
   - Exception Handler updates (401, 403)

✅ Phase 6: Test Flow
   - Postman দিয়ে full flow
```

## এখন তোমার যা আছে

```
✅ Working Authentication System
✅ Register API
✅ Login API → JWT Token
✅ JWT Filter → প্রতি request এ verify
✅ Public/Protected URL setup
✅ Proper error handling (401, 403)
```

## Visualization — পুরো System

```
┌──────────────┐
│   Postman    │
└──────┬───────┘
       │
       │ POST /api/auth/login
       │ { username, password }
       ▼
┌──────────────────────┐
│  AuthController      │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  AuthService         │
│  ↓                   │
│  AuthManager         │
│  ↓                   │
│  CustomUserDetails   │
│  Service             │
│  ↓                   │
│  Database query      │
│  ↓                   │
│  PasswordEncoder     │
│  ↓                   │
│  JwtUtil generate    │
└──────┬───────────────┘
       │
       │ { token: "eyJ..." }
       ▼
┌──────────────┐
│   Postman    │
└──────────────┘

পরের request:

┌──────────────┐
│   Postman    │
└──────┬───────┘
       │
       │ GET /api/books
       │ Authorization: Bearer eyJ...
       ▼
┌──────────────────────┐
│  JwtAuthFilter       │ ← Token verify
│  ↓                   │
│  SecurityContext set │
└──────┬───────────────┘
       │
       ▼
┌──────────────────────┐
│  BookController      │ ← Authorized
└──────┬───────────────┘
       │
       ▼
┌──────────────┐
│   Postman    │ ← books list
└──────────────┘
```

## এখনো যা নেই (Part 3 এ আসছে)

```
❌ Role-based protected APIs (@PreAuthorize)
❌ Permission-based protected APIs
❌ Refresh Token
❌ Token expiry handling (auto refresh)
❌ Logout (Token blacklist)
❌ Password reset
❌ Common security mistakes
```

## পরের Part এ যা শিখবো

**Part 3: Advanced Security**

```
✅ @PreAuthorize দিয়ে method level security
✅ Role-based access (hasRole)
✅ Permission-based access (hasAuthority)
✅ Refresh Token mechanism
✅ Logout implementation
✅ Common security vulnerabilities ও তাদের fix
```

---

Part 2 পড়ে কোথাও gap থাকলে বলো — clarify করবো। তারপর Part 3 এ যাবো। 🚀

Happy coding!
