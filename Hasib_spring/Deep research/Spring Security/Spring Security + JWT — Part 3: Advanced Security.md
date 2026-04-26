# Spring Security + JWT — Part 3: Advanced Security

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Previous:** Part 1 (Foundation) + Part 2 (Auth Flow)
> **Part 3 Topics:** Role/Permission based Access, Get Current User, Refresh Token, Logout, Security Mistakes

> ⚠️ এই Part শেষে — তোমার system production-ready হবে। কোন user কোন API access করবে controlled, refresh token কাজ করবে, logout system থাকবে।

---

## সূচিপত্র

1. [Phase 1: Method Level Security](#phase-1)
2. [Phase 2: URL Level Security](#phase-2)
3. [Phase 3: Real Library Scenario](#phase-3)
4. [Phase 4: Get Current User](#phase-4)
5. [Phase 5: Refresh Token](#phase-5)
6. [Phase 6: Logout Implementation](#phase-6)
7. [Phase 7: Common Security Mistakes](#phase-7)
8. [Phase 8: Production Checklist](#phase-8)
9. [Part 3 এর সারসংক্ষেপ](#summary)

---

<a name="phase-1"></a>

# Phase 1 — Method Level Security

এতক্ষণ পর্যন্ত — যেকোনো authenticated user যেকোনো protected API access করতে পারে। কিন্তু আমরা চাই —

```
DELETE /books/1   → শুধু ADMIN
POST /books       → ADMIN, EDITOR
GET /books        → সবাই (authenticated)
```

এটাই **Authorization** — কে কী করতে পারবে।

---

## `@EnableMethodSecurity` Recap

Part 2 এর SecurityConfig এ আমরা এটা enable করেছিলাম —

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity        // ⭐ এই annotation
public class SecurityConfig { }
```

এটা enable করলে — `@PreAuthorize`, `@PostAuthorize`, `@Secured` ব্যবহার করতে পারবো।

আমরা focus করবো **`@PreAuthorize`** এ — সবচেয়ে popular এবং powerful।

---

## `@PreAuthorize` কী?

> *"Method চালানোর আগে এই condition check করো। False হলে — Access Denied।"*

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteBook(Long id) {
    bookRepository.deleteById(id);
}
```

User ADMIN না হলে — method **চলবেই না।** `AccessDeniedException` throw হবে।

---

## কীভাবে কাজ করে?

```
User API call করলো: DELETE /books/1
        ↓
JwtFilter user authenticate করলো
        ↓
SecurityContext এ user info: { roles: ["MEMBER"] }
        ↓
Controller method এ আসলো
        ↓
@PreAuthorize দেখলো → "hasRole('ADMIN')"
        ↓
SecurityContext থেকে user roles check
        ↓
"MEMBER" != "ADMIN" → AccessDeniedException ❌
        ↓
GlobalExceptionHandler ধরলো → 403 Forbidden
```

---

## Role-based Examples

### Single Role

```java
@PreAuthorize("hasRole('ADMIN')")
public void adminOnlyMethod() { }
```

### Multiple Roles (যেকোনো একটা)

```java
@PreAuthorize("hasAnyRole('ADMIN', 'EDITOR')")
public void adminOrEditor() { }
```

### Multiple Roles (সবগুলোই থাকতে হবে)

```java
@PreAuthorize("hasRole('ADMIN') and hasRole('SUPER_USER')")
public void bothRolesNeeded() { }
```

---

## Permission-based Examples

Role এর চেয়ে Permission **আরো granular।**

### Single Permission

```java
@PreAuthorize("hasAuthority('BOOK_DELETE')")
public void deleteBook(Long id) { }
```

### Multiple Permissions (যেকোনো একটা)

```java
@PreAuthorize("hasAnyAuthority('BOOK_CREATE', 'BOOK_UPDATE')")
public void modifyBook() { }
```

---

## `hasRole` vs `hasAuthority` — পার্থক্য

```
hasRole('ADMIN')          → check করে: "ROLE_ADMIN" আছে কিনা
hasAuthority('ADMIN')     → check করে: "ADMIN" আছে কিনা (literal)
```

মনে আছে Part 1 এ আমরা `CustomUserDetails` এ কী করেছিলাম?

```java
authorities.add(new SimpleGrantedAuthority("ROLE_" + role.getName()));
authorities.add(new SimpleGrantedAuthority(permission.getName()));
```

```
Roles    → "ROLE_" prefix সহ store    → hasRole দিয়ে check
Permissions → prefix ছাড়া              → hasAuthority দিয়ে check
```

**Rule of thumb:**
- Roles check করতে — `hasRole`
- Permissions check করতে — `hasAuthority`

---

## Method Parameter Access

কখনো কখনো method এর parameter এর উপর ভিত্তি করে authorize করতে হয় —

```java
@PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
public User getUserDetails(Long userId) { }
```

**মানে:**
- ADMIN হলে — সব user দেখতে পারবে
- নাহলে — শুধু নিজের details দেখতে পারবে

`#userId` = method parameter
`authentication.principal` = current user

---

## Practical Example

```java
@RestController
@RequestMapping("/api/books")
@RequiredArgsConstructor
public class BookController {

    private final BookService bookService;

    // সবাই দেখতে পারবে
    @GetMapping
    @PreAuthorize("hasAuthority('BOOK_READ')")
    public List<Book> getAllBooks() {
        return bookService.getAllBooks();
    }

    // ADMIN, EDITOR create করতে পারবে
    @PostMapping
    @PreAuthorize("hasAuthority('BOOK_CREATE')")
    public Book createBook(@Valid @RequestBody BookDTO dto) {
        return bookService.createBook(dto);
    }

    // ADMIN, EDITOR update করতে পারবে
    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOK_UPDATE')")
    public Book updateBook(@PathVariable Long id, @Valid @RequestBody BookDTO dto) {
        return bookService.updateBook(id, dto);
    }

    // শুধু ADMIN
    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOK_DELETE')")
    public void deleteBook(@PathVariable Long id) {
        bookService.deleteBook(id);
    }
}
```

---

## কোথায় বসাবো — Controller এ নাকি Service এ?

দুই জায়গায়ই দেওয়া যায়, কিন্তু **Service এ বসানো ভালো।**

### কেন?

```java
// Controller এ
@DeleteMapping("/{id}")
@PreAuthorize("hasAuthority('BOOK_DELETE')")
public void deleteBook(@PathVariable Long id) {
    bookService.deleteBook(id);
}

// Service ও থেকে কেউ direct call করলে?
@Service
public class AdminPanelService {
    public void cleanupOldBooks() {
        bookService.deleteBook(oldBookId);    // ❌ কোনো check নেই!
    }
}
```

Controller এ check রাখলে — Service থেকে call করলে authorization bypass হয়।

### সঠিক উপায়

```java
// Service এ
@Service
public class BookService {

    @PreAuthorize("hasAuthority('BOOK_DELETE')")
    public void deleteBook(Long id) {
        bookRepository.deleteById(id);
    }
}
```

এখন **যেখান থেকেই call করো — check হবে।**

---

<a name="phase-2"></a>

# Phase 2 — URL Level Security

`@PreAuthorize` দিয়ে method level security হলো। কিন্তু কখনো **URL pattern based** rule দরকার হয়। সেটা SecurityConfig এ করি।

---

## SecurityConfig Update

Part 2 এর SecurityFilterChain বেশ basic ছিল। এখন granular control —

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth
            // Public APIs
            .requestMatchers("/api/auth/**").permitAll()
            .requestMatchers("/api/public/**").permitAll()

            // Swagger (development এ)
            .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

            // HTTP Method specific
            .requestMatchers(HttpMethod.GET, "/api/books/**").hasAuthority("BOOK_READ")
            .requestMatchers(HttpMethod.POST, "/api/books/**").hasAuthority("BOOK_CREATE")
            .requestMatchers(HttpMethod.PUT, "/api/books/**").hasAuthority("BOOK_UPDATE")
            .requestMatchers(HttpMethod.DELETE, "/api/books/**").hasAuthority("BOOK_DELETE")

            // Admin only URLs
            .requestMatchers("/api/admin/**").hasRole("ADMIN")

            // বাকি সব
            .anyRequest().authenticated()
        )
        .authenticationProvider(authenticationProvider())
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

---

## প্রতিটা Line Decode

### `requestMatchers(HttpMethod.GET, "/api/books/**")`

> *"GET method এ /api/books/ এর সব sub-path।"*

`**` = কোনো level deep পর্যন্ত। যেমন —
- `/api/books`
- `/api/books/1`
- `/api/books/1/reviews`

### `.hasAuthority("BOOK_READ")`

> *"User এর `BOOK_READ` permission থাকতে হবে।"*

### Order Matters!

```java
.requestMatchers("/api/auth/**").permitAll()      // ১ম
.requestMatchers("/api/admin/**").hasRole("ADMIN") // ২য়
.anyRequest().authenticated()                      // শেষে
```

**উপর থেকে নিচে check হয়।** First match জেতে।

`anyRequest()` সবসময় শেষে দিতে হয়।

---

## Method Level vs URL Level — কোনটা কখন?

| | Method Level (`@PreAuthorize`) | URL Level (SecurityConfig) |
|---|---|---|
| কোথায় | Controller/Service method এ | Central config এ |
| Granularity | High (method এর parameter ও check করতে পারে) | Low (URL pattern only) |
| Visibility | Method এর সাথে দেখা যায় | এক জায়গায় central |
| Best For | Business logic security | Public/Private URL distinction |

### Best Practice

```
URL Level    → Public/Private distinction (auth required vs not)
Method Level → Specific permission check (granular)
```

দুইটাই use করো — একসাথে।

---

<a name="phase-3"></a>

# Phase 3 — Real Library Scenario

আমাদের Library Management System এ practical implementation।

## Permission Matrix

| API | ADMIN | EDITOR | MEMBER |
|---|---|---|---|
| GET /api/books | ✅ | ✅ | ✅ |
| GET /api/books/{id} | ✅ | ✅ | ✅ |
| POST /api/books | ✅ | ✅ | ❌ |
| PUT /api/books/{id} | ✅ | ✅ | ❌ |
| DELETE /api/books/{id} | ✅ | ❌ | ❌ |
| GET /api/users | ✅ | ❌ | ❌ |
| POST /api/users | ✅ | ❌ | ❌ |
| DELETE /api/users/{id} | ✅ | ❌ | ❌ |
| GET /api/auth/me | ✅ | ✅ | ✅ |

## Permission Setup (Part 1 এর DataSeeder Update)

```java
// ADMIN — সব permission
admin.setPermissions(Set.of(
    bookRead, bookCreate, bookUpdate, bookDelete,
    userManage, userRead, userCreate, userDelete
));

// EDITOR — book related, delete বাদে
editor.setPermissions(Set.of(
    bookRead, bookCreate, bookUpdate
));

// MEMBER — শুধু read
member.setPermissions(Set.of(bookRead));
```

## BookController

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

    @GetMapping("/{id}")
    @PreAuthorize("hasAuthority('BOOK_READ')")
    public Book getBook(@PathVariable Long id) {
        return bookService.getBook(id);
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

## UserController (Admin Only)

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
@PreAuthorize("hasRole('ADMIN')")    // ⭐ Class level — সব method এ apply
public class UserController {

    private final UserService userService;

    @GetMapping
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }

    @PostMapping
    public User createUser(@Valid @RequestBody UserCreateDTO dto) {
        return userService.createUser(dto);
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

⭐ Class level এ `@PreAuthorize` দিলে — সব method এ apply হয়।

---

## Test Scenarios

### Scenario 1: MEMBER User

```
MEMBER login → Token

GET /api/books          → ✅ 200 OK
GET /api/books/1        → ✅ 200 OK
POST /api/books         → ❌ 403 Forbidden
DELETE /api/books/1     → ❌ 403 Forbidden
GET /api/users          → ❌ 403 Forbidden
```

### Scenario 2: EDITOR User

```
EDITOR login → Token

GET /api/books          → ✅ 200 OK
POST /api/books         → ✅ 201 Created
PUT /api/books/1        → ✅ 200 OK
DELETE /api/books/1     → ❌ 403 Forbidden  (no BOOK_DELETE)
GET /api/users          → ❌ 403 Forbidden  (no ADMIN role)
```

### Scenario 3: ADMIN User

```
ADMIN login → Token

GET /api/books          → ✅ 200 OK
POST /api/books         → ✅ 201 Created
DELETE /api/books/1     → ✅ 200 OK
GET /api/users          → ✅ 200 OK
DELETE /api/users/2     → ✅ 200 OK
```

---

<a name="phase-4"></a>

# Phase 4 — Get Current User in Controller/Service

API এর ভেতরে — "এই request কে করছে?" — জানতে হলে কী করবো?

দুইটা উপায় —

---

## Way 1: `@AuthenticationPrincipal` (Controller এ Best)

```java
@GetMapping("/me")
public UserInfoDTO getCurrentUser(@AuthenticationPrincipal CustomUserDetails userDetails) {

    User user = userDetails.getUser();

    return UserInfoDTO.builder()
        .username(user.getUsername())
        .email(user.getEmail())
        .roles(user.getRoles().stream().map(Role::getName).collect(Collectors.toSet()))
        .build();
}
```

### `@AuthenticationPrincipal` কী?

> *"Spring, current authenticated user inject করো।"*

Spring SecurityContext থেকে user নিয়ে method parameter এ pass করে।

### Type Casting

```java
@AuthenticationPrincipal CustomUserDetails userDetails
```

আমাদের `CustomUserDetails` (Part 1 এ বানিয়েছি) সরাসরি inject হবে। তারপর `userDetails.getUser()` দিয়ে actual User entity।

---

## Way 2: `SecurityContextHolder` (Service/যেকোনো জায়গায়)

```java
@Service
public class AuditService {

    public void logAction(String action) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth != null && auth.isAuthenticated()) {
            String username = auth.getName();
            // log to database
        }
    }
}
```

### কখন এটা use করি?

```
Service/Util এ — যেখানে Controller parameter inject করার সুযোগ নেই
যেমন: AuditLog, Notification, ScheduledTask
```

---

## কোনটা কখন?

```
Controller এ → @AuthenticationPrincipal (clean, declarative)
Service এ    → SecurityContextHolder (যদি Controller থেকে user pass না করো)
Best Practice → Service এ user explicitly pass করো (better testing)
```

---

## Helper Method বানানো

প্রতি জায়গায় boilerplate code এড়াতে —

```java
@Component
public class SecurityUtils {

    public static User getCurrentUser() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth == null || !auth.isAuthenticated() || auth.getPrincipal().equals("anonymousUser")) {
            throw new UnauthorizedException("No authenticated user");
        }

        return ((CustomUserDetails) auth.getPrincipal()).getUser();
    }

    public static String getCurrentUsername() {
        return getCurrentUser().getUsername();
    }

    public static boolean hasRole(String role) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth.getAuthorities().stream()
            .anyMatch(a -> a.getAuthority().equals("ROLE_" + role));
    }
}
```

### Use

```java
User currentUser = SecurityUtils.getCurrentUser();
String username = SecurityUtils.getCurrentUsername();
boolean isAdmin = SecurityUtils.hasRole("ADMIN");
```

---

<a name="phase-5"></a>

# Phase 5 — Refresh Token Mechanism

## কেন দরকার?

### সমস্যা

```
Token expiry: 1 hour

User: 09:00 AM login করলো → Token পেলো (10:00 AM expire)
User: 11:00 AM API call করলো → 401 Unauthorized
User: আবার login করতে হবে! 😫
```

User experience খারাপ — প্রতি ঘণ্টায় re-login।

### সমাধান 1: Long Expiry?

```
Token expiry: 1 month
```

⚠️ **Security risk!** Token চুরি হলে hacker ১ মাস access পাবে।

### সমাধান 2: Refresh Token

```
Access Token  → Short expiry (15 min - 1 hour)
Refresh Token → Long expiry (7 days - 30 days)
```

```
Access Token expire হলে
        ↓
Refresh Token দিয়ে নতুন Access Token পাও
        ↓
User কে আবার login করতে হয় না
```

**Best of both worlds —** UX ভালো, security ও থাকে।

---

## Access Token vs Refresh Token

| | Access Token | Refresh Token |
|---|---|---|
| Expiry | Short (15-60 min) | Long (7-30 days) |
| Use | প্রতি API call এ | শুধু নতুন Access Token নিতে |
| Storage | Memory (frontend) | Secure storage (cookie/DB) |
| If stolen | কম damage (auto-expire) | বেশি damage (long valid) |
| Database | Stateless (no DB) | Stateful (DB তে save) |

---

## Database Design

Refresh Token database এ save করি — revoke করার জন্য।

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

```java
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {

    Optional<RefreshToken> findByToken(String token);

    void deleteByUser(User user);

    @Modifying
    @Query("UPDATE RefreshToken rt SET rt.revoked = true WHERE rt.user.id = :userId")
    void revokeAllByUserId(Long userId);
}
```

---

## RefreshTokenService

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;

    @Value("${jwt.refresh.expiration}")
    private long refreshExpirationMs;     // 7 days = 604800000

    public RefreshToken createRefreshToken(User user) {
        // আগের refresh tokens revoke করি (optional, depends on policy)
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

### কেন `UUID` use করছি JWT না?

Refresh Token JWT হতে হবে না — random unique string ই যথেষ্ট।

```
Access Token  → JWT (claims, expiration internal)
Refresh Token → UUID/random (DB থেকে check)
```

JWT এ refresh করার দরকার নেই — কারণ DB থেকে check করছি।

---

## AuthService Update

### Login

```java
public LoginResponse login(LoginRequest request) {

    Authentication authentication = authenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken(
            request.getUsername(),
            request.getPassword()
        )
    );

    CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
    User user = userDetails.getUser();

    Set<String> roles = user.getRoles().stream()
        .map(Role::getName)
        .collect(Collectors.toSet());

    Set<String> permissions = user.getRoles().stream()
        .flatMap(role -> role.getPermissions().stream())
        .map(Permission::getName)
        .collect(Collectors.toSet());

    String accessToken = jwtUtil.generateToken(user.getUsername(), roles, permissions);

    // ⭐ Refresh Token ও তৈরি
    RefreshToken refreshToken = refreshTokenService.createRefreshToken(user);

    return LoginResponse.builder()
        .accessToken(accessToken)
        .refreshToken(refreshToken.getToken())
        .tokenType("Bearer")
        .expiresIn(jwtExpiration)
        .username(user.getUsername())
        .roles(roles)
        .permissions(permissions)
        .build();
}
```

### Refresh

```java
public RefreshResponse refresh(String refreshTokenStr) {

    // ১. Refresh Token validate
    RefreshToken refreshToken = refreshTokenService.validateRefreshToken(refreshTokenStr);

    // ২. User load
    User user = refreshToken.getUser();

    // ৩. Roles + Permissions
    Set<String> roles = user.getRoles().stream()
        .map(Role::getName)
        .collect(Collectors.toSet());

    Set<String> permissions = user.getRoles().stream()
        .flatMap(role -> role.getPermissions().stream())
        .map(Permission::getName)
        .collect(Collectors.toSet());

    // ৪. নতুন Access Token
    String newAccessToken = jwtUtil.generateToken(user.getUsername(), roles, permissions);

    return RefreshResponse.builder()
        .accessToken(newAccessToken)
        .tokenType("Bearer")
        .expiresIn(jwtExpiration)
        .build();
}
```

---

## RefreshController

```java
@PostMapping("/refresh")
public ResponseEntity<RefreshResponse> refresh(@Valid @RequestBody RefreshRequest request) {
    RefreshResponse response = authService.refresh(request.getRefreshToken());
    return ResponseEntity.ok(response);
}
```

---

## পুরো Flow

```
1. User login → { accessToken, refreshToken } পেলো
                accessToken expires in 1 hour
                refreshToken expires in 7 days

2. User API call করছে accessToken দিয়ে — সব OK

3. ১ ঘণ্টা পর accessToken expired
   API call → 401 Unauthorized

4. Frontend automatically:
   POST /api/auth/refresh
   { refreshToken: "..." }
        ↓
   Server: refreshToken validate → নতুন accessToken
        ↓
   Frontend নতুন token দিয়ে আবার original API call

5. ৭ দিন পর refreshToken ও expire
   → User কে আবার login করতে হবে
```

---

<a name="phase-6"></a>

# Phase 6 — Logout Implementation

## Stateless System এ Logout এর সমস্যা

JWT এর সমস্যা — Token client এর কাছে। Server logout করলেও client এর Token valid থাকে!

```
User logout করলো
        ↓
Server: ... (কী করবে? Token এ তো user info আছে)
        ↓
User Token দিয়ে আবার API call করতে পারবে! 😱
```

---

## সমাধান — তিনটা Approach

### Approach 1: Token Blacklist

Logout হওয়া token গুলো **database/cache এ save** করো। প্রতি request এ check —

```
Token এসেছে
        ↓
Blacklist এ আছে? 
        ↓
আছে → 401 Unauthorized
নেই → Continue
```

**সমস্যা:** প্রতি request এ DB check। Stateless concept ভেঙে যায়।

### Approach 2: Refresh Token Revoke (Best!)

```
Logout এ:
  - Refresh Token revoked = true করো
  - Access Token expire হওয়া পর্যন্ত wait করো (max 1 hour)

User এর কাছে access token থাকলেও — refresh করতে পারবে না।
1 hour পরে complete logout।
```

**সুবিধা:** Stateless থাকে, শুধু refresh time এ DB check।

### Approach 3: Short Access Token + Refresh Revoke

```
Access Token expiry → 5 minutes
Refresh Token → revoke logout এ
```

5 minutes এর বেশি হ্যাকার access পাবে না।

---

## Implementation (Approach 2)

### Logout Endpoint

```java
@PostMapping("/logout")
public ResponseEntity<Map<String, String>> logout(
        @Valid @RequestBody LogoutRequest request) {

    refreshTokenService.revokeToken(request.getRefreshToken());

    return ResponseEntity.ok(Map.of("message", "Logged out successfully"));
}
```

### LogoutRequest DTO

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class LogoutRequest {

    @NotBlank(message = "Refresh token is required")
    private String refreshToken;
}
```

---

## Logout Flow

```
1. User: POST /api/auth/logout
         { refreshToken: "..." }

2. Server:
   - Refresh Token → revoked = true
   - Database update

3. User access token use করলে:
   - Token যতক্ষণ valid (max 1 hour) — চলবে
   - Expire হলে → refresh attempt → revoked! → fail

4. User কে re-login করতে হবে
```

---

<a name="phase-7"></a>

# Phase 7 — Common Security Mistakes

Production এ যেগুলো **never** করো —

---

## Mistake 1: Token URL এ পাঠানো

```
GET /api/books?token=eyJhbGc...    ❌
```

### সমস্যা
- URL log হয় (server, browser history, proxy)
- Referer header এ leak
- Bookmark এ save হয়

### সমাধান
**সবসময় Header এ পাঠাও:**

```
Authorization: Bearer eyJhbGc...    ✅
```

---

## Mistake 2: HTTPS না Use করা

### সমস্যা
HTTP plain text। কেউ network sniff করলে Token পেয়ে যাবে।

```
HTTP request:
  Authorization: Bearer eyJhbGc...    ← visible to anyone on network
```

### সমাধান
**Production এ সবসময় HTTPS।** Let's Encrypt এ free certificate পাও।

---

## Mistake 3: Weak Secret Key

```properties
jwt.secret=secret123    ❌
```

### সমস্যা
Brute force attack এ break হয়ে যাবে।

### সমাধান
**Strong secret — কমপক্ষে ৬৪ characters, random:**

```properties
jwt.secret=3f7Hk9Dx2!@vNm5Lp$wQz8Cn1Bs6Rt#oUe4Gj7Yh0Ai3Eb9Kc2Po5Lz8Sn1Vm4Df...
```

Generate করতে —
```bash
openssl rand -base64 64
```

---

## Mistake 4: Long Expiry

```properties
jwt.expiration=2592000000    # ❌ 30 days!
```

### সমস্যা
Token চুরি হলে hacker ৩০ দিন access পাবে।

### সমাধান
```properties
jwt.expiration=3600000        # ✅ 1 hour for access token
jwt.refresh.expiration=604800000  # 7 days for refresh
```

---

## Mistake 5: Sensitive Data in JWT

```java
.claim("password", user.getPassword())    ❌
.claim("creditCard", user.getCard())       ❌
```

### সমস্যা
JWT payload **encrypted না, encoded।** কেউ decode করে পড়তে পারে।

### সমাধান
শুধু **non-sensitive data:**
- User ID
- Username
- Roles
- Permissions

---

## Mistake 6: Brute Force Protection নেই

### সমস্যা
Attacker login API তে ১ লাখ password try করতে পারে।

### সমাধান
**Rate limiting:**
- ৫ ব্যর্থ attempt → ১৫ মিনিট account lock
- IP-based throttling

Library: **Bucket4j** বা **Resilience4j**

---

## Mistake 7: Error Message থেকে Info Leak

```java
// ❌ Bad
if (user == null) {
    throw new Exception("Username not found");
}
if (!password.matches()) {
    throw new Exception("Wrong password");
}
```

### সমস্যা
Hacker বুঝে যাবে — username exists কিনা।

### সমাধান
**Generic message:**

```java
throw new BadCredentialsException("Invalid username or password");
```

দুইটা case এই same message — hacker বুঝবে না।

---

## Mistake 8: CORS Misconfiguration

```java
.cors(cors -> cors.configurationSource(request -> {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("*"));    // ❌ সব allow!
    return config;
}))
```

### সমাধান
**Specific origins:**

```java
config.setAllowedOrigins(List.of(
    "https://yourapp.com",
    "https://www.yourapp.com"
));
```

---

## Mistake 9: Token Refresh Reuse

```
User refresh token দিল → নতুন access token পেল
পরে আবার same refresh token দিল → আবার access token পেল
```

### সমস্যা
যদি hacker refresh token পায় — অনেকবার use করতে পারে।

### সমাধান
**Refresh Token Rotation:**

```
Refresh দিলে:
1. পুরনো refresh token revoke
2. নতুন refresh token issue
3. দুইটা একসাথে return
```

পুরনো token আর কাজ করবে না।

---

## Mistake 10: No Token Validation in Filter

```java
// ❌ Bad
public boolean validateToken(String token) {
    return token != null && !token.isEmpty();
}
```

### সমাধান
**Always:**
- Signature verify
- Expiration check
- Issuer check
- Audience check

---

<a name="phase-8"></a>

# Phase 8 — Production Checklist

Deploy এর আগে চেক করো —

```
Authentication:
[ ] HTTPS enabled
[ ] Strong secret key (64+ chars, random)
[ ] Access Token short expiry (≤ 1 hour)
[ ] Refresh Token implemented
[ ] Logout works (refresh token revoked)
[ ] Generic error messages (no info leak)

Authorization:
[ ] @PreAuthorize on Service methods (not just Controllers)
[ ] Role + Permission system in place
[ ] URL level + Method level both
[ ] Default deny (whitelist approach)

Security Headers:
[ ] HTTPS only
[ ] Secure cookies (if used)
[ ] CORS specific origins (not *)
[ ] CSRF disabled (for stateless API)

Rate Limiting:
[ ] Login API rate limited
[ ] Brute force protection
[ ] IP-based throttling

Token Management:
[ ] Token in Authorization header (not URL)
[ ] No sensitive data in JWT payload
[ ] Refresh token rotation
[ ] Token blacklist for compromised users

Logging:
[ ] Failed login attempts logged
[ ] Audit log for sensitive operations
[ ] No tokens in logs (sanitize)

Database:
[ ] Passwords hashed (BCrypt strength ≥ 10)
[ ] Refresh tokens in DB
[ ] Old tokens cleanup (scheduled job)

Code Quality:
[ ] No hardcoded secrets in code
[ ] Environment variables for secrets
[ ] Application.properties not in git
[ ] Production profile separated
```

---

<a name="summary"></a>

# Part 3 এর সারসংক্ষেপ

## যা যা শিখলাম

```
✅ Phase 1: Method Level Security
   - @PreAuthorize
   - hasRole vs hasAuthority
   - Multiple roles/permissions
   - Method parameter access

✅ Phase 2: URL Level Security
   - HttpMethod specific rules
   - Order matters
   - Method vs URL level comparison

✅ Phase 3: Real Library Scenario
   - Permission matrix
   - Test scenarios for each role

✅ Phase 4: Get Current User
   - @AuthenticationPrincipal
   - SecurityContextHolder
   - Helper utility class

✅ Phase 5: Refresh Token
   - Why needed
   - Access vs Refresh
   - Database design
   - Implementation
   - Auto-refresh flow

✅ Phase 6: Logout
   - Stateless logout problem
   - 3 approaches
   - Refresh Token revoke (best)

✅ Phase 7: Security Mistakes
   - 10 common mistakes
   - Real attacks + solutions

✅ Phase 8: Production Checklist
   - Final verification
```

## Complete Security System

এখন তোমার full system —

```
┌─────────────────────────────────────────┐
│  Authentication                          │
│  - Register (BCrypt password)            │
│  - Login (JWT generation)                │
│  - Refresh (new access token)            │
│  - Logout (revoke refresh)               │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Authorization                           │
│  - Role-based (@PreAuthorize hasRole)    │
│  - Permission-based (hasAuthority)       │
│  - URL level (SecurityConfig)            │
│  - Method level (controller/service)     │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│  Security Hardening                      │
│  - HTTPS                                 │
│  - Strong secret key                     │
│  - Short token expiry                    │
│  - Refresh token rotation                │
│  - Generic error messages                │
└─────────────────────────────────────────┘
```

## পুরো 3 Part Recap

```
Part 1: Foundation
  - Database design
  - Password security (BCrypt)
  - JWT utility
  - UserDetailsService

Part 2: Authentication Flow
  - SecurityConfig
  - JWT Filter
  - Login/Register APIs
  - Token verification

Part 3: Advanced Security
  - Method/URL level security
  - Get current user
  - Refresh token
  - Logout
  - Security best practices
```

## পরের Steps (Optional Advanced)

```
- OAuth 2.0 (Google/Facebook login)
- Multi-factor authentication (MFA/OTP)
- Email verification
- Password reset flow
- Account lockout (after failed attempts)
- Session management UI (active sessions)
- Role hierarchy (ADMIN > EDITOR > MEMBER)
- Custom permission expressions
- Audit logging
- Scheduled cleanup jobs
```

---

## শেষ কথা

তিন part এর পর — তোমার Spring Security + JWT এর strong foundation আছে। 🎯

এই knowledge দিয়ে তুমি বানাতে পারবে —
- Multi-user web applications
- Role-based admin dashboards
- Secure REST APIs
- Mobile app backends
- Microservices authentication

Code লিখতে গিয়ে কোনো গোলমাল হলে — concepts revisit করো। প্রতিটা piece কীভাবে fit করে সেটা মনে রাখো।

পড়ে কোথাও gap থাকলে বলো — clarify করবো।

🚀 Happy Securing!
