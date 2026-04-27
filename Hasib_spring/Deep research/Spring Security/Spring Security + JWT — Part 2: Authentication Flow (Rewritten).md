# Spring Security + JWT — Part 2: Authentication Flow (Rewritten)

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Previous:** Part 1 (Foundation)
> **Part 2 Topics:** SecurityConfig, JWT Filter, Login/Register APIs, Complete Auth Flow

> ⚠️ এই Part শেষে — তুমি actually login করতে পারবে, JWT Token পাবে, Postman দিয়ে test করতে পারবে।

---

## সূচিপত্র

1. [Big Picture — পুরো System একনজরে](#big-picture)
2. [কোন Piece কেন দরকার](#pieces)
3. [Build Order — কোনটা আগে বানাবো](#build-order)
4. [Segment 1: DTOs — Request/Response Format](#segment-1)
5. [Segment 2: SecurityConfig — Building এর নিয়ম](#segment-2)
6. [Segment 3: JwtAuthenticationFilter — Security Guard](#segment-3)
7. [Segment 4: AuthService — HR Department](#segment-4)
8. [Segment 5: AuthController — Reception Desk](#segment-5)
9. [Segment 6: Exception Handler Update](#segment-6)
10. [Segment 7: Complete Flow Trace — Register থেকে Protected API](#segment-7)
11. [Segment 8: Postman Test](#segment-8)
12. [সারসংক্ষেপ](#summary)

---

<a name="big-picture"></a>

# Big Picture — পুরো System একনজরে

## Office Building Analogy

তোমার Spring Boot App একটা Office Building —

```
🚪 Main Gate           = SecurityConfig
                          "কোন door public, কোন door এ ID Card লাগবে"

👮 Security Guard      = JwtAuthenticationFilter
                          "প্রতি request এ ID Card (Token) check করে"

🪪 ID Card Machine     = JwtUtil (Part 1 এ বানিয়েছি)
                          "Token বানায়, verify করে, পড়ে"

📋 Employee Register   = UserDetailsService (Part 1 এ বানিয়েছি)
                          "Database থেকে user info এনে দেয়"

🔐 Password Vault      = PasswordEncoder (Part 1 এ বানিয়েছি)
                          "Password hash করে, match করে"

📝 Visitor Form        = DTOs
                          "Client কী data পাঠাবে, কী পাবে"

🏢 Reception Desk      = AuthController
                          "API endpoint — register, login"

👨‍💼 HR Department       = AuthService
                          "Register/Login এর actual logic"
```

---

## Request এর Journey — একটা Map

যখন কোনো request আসে, সে এই path follow করে —

```
Client Request আসলো
    │
    ▼
┌──────────────────────────────┐
│  Step 1: SecurityConfig      │  "এই URL কি public?"
│  (নিয়ম চেক)                 │  
│                              │  /api/auth/** → public → Step 4 এ সরাসরি
│                              │  /api/books   → protected → Step 2 এ
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Step 2: JwtAuthFilter       │  "Token আছে? Valid?"
│  (Security Guard)            │  
│                              │  Token valid → user authenticate → Step 3
│                              │  Token নেই/invalid → 401 Unauthorized
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Step 3: Authorization       │  "এই user এর permission আছে?"
│  (@PreAuthorize check)       │  
│                              │  আছে → Step 4
│                              │  নেই → 403 Forbidden
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Step 4: Controller/Service  │  actual কাজ হয়
│  (Office Room)               │  
└──────────────────────────────┘
```

---

<a name="pieces"></a>

# কোন Piece কেন দরকার — একটা একটা করে

## Part 1 এ যা বানিয়েছি (ইতোমধ্যে আছে)

```
✅ User, Role, Permission entities   → Database structure
✅ Repositories                       → Database query
✅ PasswordEncoder config             → Password hash/match
✅ JwtUtil                            → Token generate/validate/parse
✅ CustomUserDetails                  → Spring এর format এ User wrap
✅ CustomUserDetailsService           → Database থেকে User load
```

## Part 2 এ যা বানাবো (নতুন)

### Piece 1: DTOs

```
কী?    → Client ↔ Server data format
কেন?   → Entity সরাসরি expose করলে security risk
        → Validation apply করার জায়গা
কোথায়? → Controller এর parameter এবং return type

RegisterRequest  → client register এ কী পাঠাবে
LoginRequest     → client login এ কী পাঠাবে
LoginResponse    → login success এ server কী ফেরত দিবে
```

### Piece 2: SecurityConfig

```
কী?    → পুরো Security এর central configuration
কেন?   → Spring কে বলতে হবে কোন API public, কোন protected
        → JWT Filter কোথায় বসবে
        → Session রাখবো না (stateless)
কোথায়? → App start এ Spring এই config পড়ে নিয়ম setup করে

এটা না থাকলে?
→ সব API default protected হয়ে যায়
→ Spring এর default login page আসে
→ JWT system কাজ করে না
```

### Piece 3: JwtAuthenticationFilter

```
কী?    → প্রতিটা request এ Token verify করার filter
কেন?   → Client Token পাঠায় Header এ
        → কেউ তো সেটা পড়বে, verify করবে, Spring কে জানাবে
        → এই "কেউ" হলো এই Filter
কোথায়? → Spring Security এর Filter Chain এ বসে
        → Controller এর আগেই কাজ শেষ

এটা না থাকলে?
→ Client Token পাঠালেও Spring বুঝতো না কে পাঠাচ্ছে
→ সব protected API তে 401 আসতো
→ Login করেও কিছু করতে পারতো না!
```

### Piece 4: AuthService

```
কী?    → Register এবং Login এর business logic
কেন?   → Controller এ logic রাখা messy
        → Duplicate check, password hash, token generate সব এখানে
কোথায়? → Service layer — Controller call করে

এটা না থাকলে?
→ Controller এ সব code mixed
→ Testing কঠিন
→ Reuse করা যায় না
```

### Piece 5: AuthController

```
কী?    → REST API endpoints
কেন?   → Client কে URL দিতে হবে
        → POST /auth/register, POST /auth/login
কোথায়? → HTTP request receive করে → AuthService call করে → response দেয়

এটা না থাকলে?
→ Client register/login করার কোনো URL পেতো না
```

### Piece 6: Exception Handler Update

```
কী?    → Authentication/Authorization error সুন্দরভাবে handle
কেন?   → Wrong password → 401 + "Invalid credentials"
        → No permission → 403 + "Access denied"
        → Handle না করলে raw error যায়
কোথায়? → GlobalExceptionHandler এ নতুন handler যোগ
```

---

<a name="build-order"></a>

# Build Order — কোনটা আগে বানাবো

```
Step 1: DTOs                → format ঠিক করি আগে
Step 2: SecurityConfig      → নিয়ম ঠিক করি
Step 3: JwtAuthFilter       → Token check system
Step 4: AuthService         → business logic
Step 5: AuthController      → API endpoints
Step 6: Exception Handler   → error handling
Step 7: Test                → Postman দিয়ে verify
```

চলো এক এক করে build করি —

---

<a name="segment-1"></a>

# Segment 1: DTOs — Request/Response Format

## কী এবং কেন?

DTO = Data Transfer Object। Client ↔ Server data চলাচলের format।

### কেন Entity সরাসরি দেবো না?

```java
// ❌ Entity সরাসরি
@PostMapping("/register")
public User register(@RequestBody User user) {
    return userRepository.save(user);
}
```

**সমস্যা:**
- Client password field দেখতে পাবে response এ
- Client id, roles manipulate করতে পারবে
- Internal structure expose
- Validation mixed with entity

**DTO দিলে:**
- Client শুধু যা দরকার তাই দেখে
- Server শুধু যা দরকার তাই নেয়
- Clean separation

---

## RegisterRequest — "Register এ Client কী পাঠাবে?"

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

### কোন Annotation কী করে?

তুমি আগেই Validation tutorial এ সব শিখেছো —
- `@NotBlank` → খালি হতে পারবে না
- `@Size` → length limit
- `@Email` → email format check

### কেন `role` field নেই?

User নিজে role set করতে পারবে না! Registration এ সবাই default `MEMBER` পায়। ADMIN পরে assign করবে।

---

## LoginRequest — "Login এ Client কী পাঠাবে?"

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

শুধু username + password। Simple।

---

## LoginResponse — "Login success এ Server কী ফেরত দিবে?"

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class LoginResponse {

    private String token;           // JWT Token
    private String tokenType;      // "Bearer"
    private long expiresIn;         // কতক্ষণ valid (milliseconds)
    private String username;        // কে login করেছে
    private Set<String> roles;      // roles list
    private Set<String> permissions; // permissions list
}
```

### কেন roles/permissions response এ দিচ্ছি?

Frontend এর জন্য — UI তে button show/hide করতে পারবে।

```
ADMIN → "Delete Book" button দেখাবে
MEMBER → "Delete Book" button দেখাবে না
```

Frontend Token decode না করেও এই info পায়।

---

<a name="segment-2"></a>

# Segment 2: SecurityConfig — Building এর নিয়ম

## কী করে এই Class?

একটা বাড়ির মালিক যেমন ঠিক করে —
- কোন দরজা সবার জন্য খোলা
- কোন দরজায় চাবি লাগবে
- Security guard কোথায় দাঁড়াবে

SecurityConfig ঠিক তাই করে —
- কোন API public
- কোন API তে Token লাগবে
- JWT Filter কোথায় বসবে

---

## কিছু Concept আগে বুঝি

### Concept 1: Filter Chain কী?

Spring Security একটা chain এর মতো কাজ করে। Request আসলে কতগুলো filter একে একে চলে —

```
Request → Filter 1 → Filter 2 → Filter 3 → ... → Controller
```

প্রতিটা filter request কে examine করে। Problem থাকলে block করে, না থাকলে পরেরটাতে পাঠায়।

আমাদের JWT Filter এই chain এ যোগ করতে হবে।

### Concept 2: Stateless Session কী?

**Session-based (পুরনো):**
```
Server memory তে: "ABC123 → Hasib"
প্রতি request এ session ID দেখে user চিনে
```

**Stateless (আমাদের):**
```
Server memory তে কিছু নেই
প্রতি request এ JWT Token দেখে user চিনে
```

JWT-based system এ session দরকার নেই — তাই disable করি।

### Concept 3: AuthenticationManager কী?

Spring Security এর core component — actual authentication কাজটা করে।

```
"hasib" + "MyPass123" দিলাম
        ↓
AuthenticationManager:
  1. UserDetailsService দিয়ে "hasib" খুঁজলো
  2. Database থেকে User পেলো
  3. PasswordEncoder দিয়ে password match করলো
  4. Match → "Authenticated!"
     No match → "BadCredentialsException!"
```

Login API তে আমরা এটা use করবো।

### Concept 4: AuthenticationProvider কী?

AuthenticationManager নিজে authenticate করে না — সে **Provider** কে দিয়ে করায়।

```
AuthenticationManager → "কে authenticate করতে পারে?"
        ↓
AuthenticationProvider → "আমি! Database থেকে করবো"
        ↓
DaoAuthenticationProvider:
  - UserDetailsService use করে user load
  - PasswordEncoder use করে password check
```

**Dao** = Data Access Object → Database based authentication।

---

## SecurityConfig Class

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final CustomUserDetailsService userDetailsService;

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
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }
}
```

---

## প্রতিটা Part কী করে — ভেঙে বুঝি

### Annotations

```java
@Configuration        // "এই class এ Spring Bean definitions আছে"
@EnableWebSecurity    // "Spring Security চালু করো"
@EnableMethodSecurity // "@PreAuthorize use করার permission দাও"
```

### Bean 1: PasswordEncoder

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(10);
}
```

Part 1 এ explain করেছি — password hash করার tool।

### Bean 2: AuthenticationProvider

```java
@Bean
public AuthenticationProvider authenticationProvider() {
    DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
    provider.setUserDetailsService(userDetailsService);   // user খুঁজবে কোথায়
    provider.setPasswordEncoder(passwordEncoder());        // password match কীভাবে
    return provider;
}
```

**দুইটা জিনিস set করছি:**
- `userDetailsService` → "user database থেকে খুঁজো" (Part 1 এ বানিয়েছি)
- `passwordEncoder` → "BCrypt দিয়ে password match করো"

### Bean 3: AuthenticationManager

```java
@Bean
public AuthenticationManager authenticationManager(
        AuthenticationConfiguration config) throws Exception {
    return config.getAuthenticationManager();
}
```

Spring এর default AuthenticationManager নিচ্ছি। Login API তে use করবো।

### Bean 4: SecurityFilterChain (সবচেয়ে Important!)

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        // ① CSRF disable
        .csrf(csrf -> csrf.disable())

        // ② Session disable (stateless)
        .sessionManagement(session ->
            session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )

        // ③ URL rules
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/auth/**").permitAll()
            .requestMatchers("/api/public/**").permitAll()
            .anyRequest().authenticated()
        )

        // ④ আমাদের provider
        .authenticationProvider(authenticationProvider())

        // ⑤ JWT Filter যোগ করা
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

    return http.build();
}
```

#### ① `.csrf(csrf -> csrf.disable())`

> "CSRF protection বন্ধ করো।"

**কেন বন্ধ?**

CSRF attack cookie-based system এ হয়। আমাদের JWT system এ Token **manually** পাঠাতে হয় (Header এ)। Browser automatically পাঠায় না। তাই CSRF risk নেই।

```
Cookie-based: Browser নিজে cookie পাঠায় → CSRF risk আছে
JWT-based:    Developer manually token দেয় → CSRF risk নেই
```

#### ② `.sessionManagement(... STATELESS)`

> "Server এ কোনো session save করো না।"

JWT-based system এ session দরকার নেই — Token ই সব info বহন করে।

#### ③ `.authorizeHttpRequests(...)`

> "কোন URL এ কে access করতে পারবে।"

```java
.requestMatchers("/api/auth/**").permitAll()
// → Register, Login → সবার জন্য open
//   (Token ছাড়াই access করা যাবে)

.requestMatchers("/api/public/**").permitAll()
// → Public data → সবার জন্য

.anyRequest().authenticated()
// → বাকি সব → Token দরকার
```

⚠️ **Order matters!** উপর থেকে নিচে check হয়। `anyRequest()` সবসময় শেষে।

#### ④ `.authenticationProvider(authenticationProvider())`

> "আমাদের custom provider register করো।"

#### ⑤ `.addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)`

> "আমার JWT Filter কে Spring এর default filter এর আগে বসাও।"

**কেন আগে?**

```
আগে: ... → UsernamePasswordAuthFilter → ...
পরে: ... → JwtAuthFilter → UsernamePasswordAuthFilter → ...
                ↑
        আমাদের filter আগে চলে
        Token check করে
        Valid হলে user authenticate করে
        পরের filter আর কিছু করতে হয় না
```

---

<a name="segment-3"></a>

# Segment 3: JwtAuthenticationFilter — Security Guard

## এই Filter কী?

ধরো Office Building এর gate এ একজন Security Guard দাঁড়িয়ে আছে।

**প্রতিজন যে office এ আসে, তাকে Guard check করে:**

```
লোক এলো → Guard:
  1. "ID Card দেখান" (Token check)
  2. "আসল ID কিনা verify করি" (Signature check)
  3. "Expire হয়নি তো?" (Expiry check)
  4. "OK, ঢুকুন" (SecurityContext এ set)

ID Card না থাকলে?
  → Public room হলে → ঢুকতে পারবে
  → Private room হলে → "Sorry, ID Card লাগবে" (401)
```

**JwtAuthenticationFilter ঠিক এই কাজই করে — প্রতিটা request এ।**

---

## কেন দরকার?

Part 1 এ JwtUtil বানিয়েছি যেটা Token verify করতে পারে। কিন্তু **কে call করবে JwtUtil কে?**

```
Client Token পাঠালো Header এ
        ↓
কে Token extract করবে? → JwtAuthenticationFilter
কে JwtUtil.validate() call করবে? → JwtAuthenticationFilter
কে Spring কে বলবে "এটা Hasib"? → JwtAuthenticationFilter
```

**JwtUtil = Tool (চাবি verify করার machine)**
**JwtAuthenticationFilter = Guard (যে machine use করে)**

---

## OncePerRequestFilter কী?

এটা Spring এর একটা base class। Guarantee করে — **প্রতি request এ filter একবারই চলবে।**

কেন দরকার? কখনো কখনো Spring internally একই request multiple times process করে (forward, redirect ইত্যাদি)। তখন filter দুইবার চললে ভুল হবে।

---

## Filter Class

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

        // Step 1: Authorization Header থেকে Token খোঁজো
        final String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);
            return;
        }

        // Step 2: "Bearer " prefix সরিয়ে Token নাও
        final String token = authHeader.substring(7);

        try {
            // Step 3: Token validate
            if (!jwtUtil.validateToken(token)) {
                filterChain.doFilter(request, response);
                return;
            }

            // Step 4: Token থেকে username extract
            String username = jwtUtil.extractUsername(token);

            // Step 5: SecurityContext এ user আগে থেকে নেই কিনা check
            if (username != null
                && SecurityContextHolder.getContext().getAuthentication() == null) {

                // Step 6: Database থেকে UserDetails load
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // Step 7: Authentication object তৈরি
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                        userDetails,
                        null,
                        userDetails.getAuthorities()
                    );

                authToken.setDetails(
                    new WebAuthenticationDetailsSource().buildDetails(request)
                );

                // Step 8: SecurityContext এ set — "এই request Hasib এর"
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }

        } catch (Exception e) {
            logger.error("Cannot set user authentication: " + e.getMessage());
        }

        // Step 9: Next filter এ পাঠাও
        filterChain.doFilter(request, response);
    }
}
```

---

## প্রতিটা Step — "Guard কী করছে?"

### Step 1: "ID Card আছে কিনা দেখি"

```java
final String authHeader = request.getHeader("Authorization");

if (authHeader == null || !authHeader.startsWith("Bearer ")) {
    filterChain.doFilter(request, response);
    return;
}
```

Request এর Header এ `Authorization: Bearer eyJhbGc...` আছে কিনা দেখি।

**না থাকলে →** skip করে পরের filter এ পাঠাই। Error throw করি না!

**কেন error না?** কারণ `/api/auth/login` এ Token থাকবে না — সেটা public API। সেখানে error দিলে login ই করতে পারবে না!

### Step 2: "ID Card টা বের করি"

```java
final String token = authHeader.substring(7);
```

`"Bearer "` = 7 character (space সহ)। তাই index 7 থেকে actual token নিই।

```
"Bearer eyJhbGciOiJIUzI1NiJ9..."
 0123456
        ↑ index 7 থেকে token শুরু
```

### Step 3: "ID Card আসল কিনা verify"

```java
if (!jwtUtil.validateToken(token)) {
    filterChain.doFilter(request, response);
    return;
}
```

Part 1 এ JwtUtil এর `validateToken()` বানিয়েছি। এটা —
- Signature check করে (tampered কিনা)
- Expiry check করে (মেয়াদ শেষ কিনা)

Invalid হলে → skip। Error throw করি না — পরে SecurityConfig এর `.anyRequest().authenticated()` 401 দিবে।

### Step 4: "ID Card এ নাম কী?"

```java
String username = jwtUtil.extractUsername(token);
```

Token এর payload থেকে `sub` claim → username বের।

### Step 5: "এই লোক কি আগেই check হয়েছে?"

```java
if (username != null
    && SecurityContextHolder.getContext().getAuthentication() == null) {
```

#### SecurityContextHolder কী?

Spring Security একটা "register book" রাখে — **SecurityContext।** প্রতিটা request এর জন্য আলাদা।

```
Request 1 এর SecurityContext: "এটা Hasib এর request"
Request 2 এর SecurityContext: "এটা Karim এর request"
```

`getAuthentication() == null` মানে — এই request এ এখনো কেউ authenticate হয়নি। তাহলে আমরা করবো।

### Step 6: "Database থেকে পুরো তথ্য আনি"

```java
UserDetails userDetails = userDetailsService.loadUserByUsername(username);
```

Part 1 এ বানানো `CustomUserDetailsService` — database থেকে User + Roles + Permissions সব আনে।

**কেন Token থেকে না নিয়ে database থেকে?**

Token এ roles থাকলেও — সেটা পুরনো হতে পারে। মাঝখানে admin role change করে থাকতে পারে। Fresh data database থেকে নিলে safe।

### Step 7: "Authentication object তৈরি"

```java
UsernamePasswordAuthenticationToken authToken =
    new UsernamePasswordAuthenticationToken(
        userDetails,                    // কে? → User info
        null,                           // password? → দরকার নেই (Token দিয়ে verified)
        userDetails.getAuthorities()    // কী পারে? → Roles + Permissions
    );
```

| Parameter | কী | মান |
|---|---|---|
| 1st | Principal (user) | CustomUserDetails object |
| 2nd | Credentials (password) | null (Token দিয়ে already verified) |
| 3rd | Authorities | roles + permissions list |

```java
authToken.setDetails(
    new WebAuthenticationDetailsSource().buildDetails(request)
);
```

Request এর extra info (IP address, session ID etc.) যোগ করি। Spring কখনো কখনো audit এর জন্য দরকার করে।

### Step 8: "Register book এ entry দাও"

```java
SecurityContextHolder.getContext().setAuthentication(authToken);
```

**এটা সবচেয়ে important line!**

এই moment এ Spring জানে — "এই request Hasib এর, তার MEMBER role আছে, BOOK_READ permission আছে।"

এই তথ্য পরে কাজে লাগবে —
- `@PreAuthorize("hasRole('ADMIN')")` → check করবে
- `@AuthenticationPrincipal` → current user পাবে
- Controller এ `Authentication` inject করে user জানা যাবে

### Step 9: "পরের guard এর কাছে পাঠাও"

```java
filterChain.doFilter(request, response);
```

Filter chain এ পরের filter কে চালু করো।

⚠️ **এই line না দিলে request একদম আটকে যাবে!** Filter chain ভেঙে যাবে। সবসময় শেষে এই line দিতে হবে।

---

## Flow Visualization — Token সহ Request

```
GET /api/books
Authorization: Bearer eyJhbGc...
        ↓
JwtAuthenticationFilter.doFilterInternal():
        ↓
  Step 1: Header পেলাম → "Bearer eyJhbGc..."
        ↓
  Step 2: Token extract → "eyJhbGc..."
        ↓
  Step 3: validateToken() → true ✅
        ↓
  Step 4: extractUsername() → "hasib"
        ↓
  Step 5: SecurityContext empty → হ্যাঁ
        ↓
  Step 6: loadUserByUsername("hasib") → UserDetails
        ↓
  Step 7: Authentication object তৈরি
        ↓
  Step 8: SecurityContext.setAuthentication(authToken) ✅
        ↓
  Step 9: filterChain.doFilter() → next filter → ... → Controller
        ↓
  BookController.getAllBooks() চলবে ✅
```

## Flow Visualization — Token ছাড়া Request

```
GET /api/books
(Authorization header নেই)
        ↓
JwtAuthenticationFilter.doFilterInternal():
        ↓
  Step 1: Header নেই → null
        ↓
  Skip! filterChain.doFilter() → next filter
        ↓
  SecurityContext empty
        ↓
  SecurityConfig: .anyRequest().authenticated() → "authenticated না!"
        ↓
  401 Unauthorized ❌
```

---

<a name="segment-4"></a>

# Segment 4: AuthService — HR Department

## কী করে?

Register এবং Login এর **actual business logic।**

Controller শুধু request নেয় আর response দেয়। Real কাজটা AuthService করে।

```
Controller = Reception Desk → "Form নিন, এখানে বসুন"
AuthService = HR Department → "Form verify করি, process করি, result দিই"
```

---

## AuthService Class

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

    // ── Register ──────────────────────────────────
    public void register(RegisterRequest request) {

        // ক) Username আগে কেউ নিয়েছে কিনা
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateResourceException(
                "Username already exists: " + request.getUsername()
            );
        }

        // খ) Email আগে কেউ নিয়েছে কিনা
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new DuplicateResourceException(
                "Email already exists: " + request.getEmail()
            );
        }

        // গ) Default role (MEMBER) assign
        Role memberRole = roleRepository.findByName("MEMBER")
            .orElseThrow(() -> new ResourceNotFoundException("MEMBER role not found"));

        // ঘ) User create + password hash + save
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .enabled(true)
            .roles(Set.of(memberRole))
            .build();

        userRepository.save(user);
    }

    // ── Login ─────────────────────────────────────
    public LoginResponse login(LoginRequest request) {

        // ক) AuthenticationManager দিয়ে authenticate
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(),
                request.getPassword()
            )
        );

        // খ) Authenticated user info
        CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
        User user = userDetails.getUser();

        // গ) Roles extract
        Set<String> roles = user.getRoles().stream()
            .map(Role::getName)
            .collect(Collectors.toSet());

        // ঘ) Permissions extract
        Set<String> permissions = user.getRoles().stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(Permission::getName)
            .collect(Collectors.toSet());

        // ঙ) JWT Token generate
        String token = jwtUtil.generateToken(user.getUsername(), roles, permissions);

        // চ) Response build
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

## Register Method — Step by Step

### "কেন এই Order?"

```
1st: Duplicate check  → ভুল data ঢোকানো prevent
2nd: Role assign       → default permission setup
3rd: Password hash     → security
4th: Save              → database এ persist
```

### ক) Duplicate Check

```java
if (userRepository.existsByUsername(request.getUsername())) {
    throw new DuplicateResourceException(...);
}
```

আগে Exception Handling tutorial এ বানিয়েছিলাম `DuplicateResourceException` — সেটাই use করছি। GlobalExceptionHandler 409 Conflict return করবে।

### খ) Default Role Assign

```java
Role memberRole = roleRepository.findByName("MEMBER")
    .orElseThrow(...);
```

Part 1 এর DataSeeder এ MEMBER role তৈরি করেছি। সেটা খুঁজে নিচ্ছি।

### গ) Password Hash + Save

```java
.password(passwordEncoder.encode(request.getPassword()))
```

"MyPass123" → "$2a$10$abc..." (hashed) → database এ save।

---

## Login Method — Step by Step (সবচেয়ে Important!)

### ক) AuthenticationManager.authenticate()

```java
Authentication authentication = authenticationManager.authenticate(
    new UsernamePasswordAuthenticationToken(
        request.getUsername(),
        request.getPassword()
    )
);
```

### এই এক line এ পর্দার পেছনে কী হয়?

```
authenticationManager.authenticate() call হলো
        ↓
Spring DaoAuthenticationProvider activate হলো
        ↓
Step 1: CustomUserDetailsService.loadUserByUsername("hasib")
        → Database query: SELECT * FROM users WHERE username = 'hasib'
        → User পেলো
        → CustomUserDetails wrapper তৈরি
        ↓
Step 2: PasswordEncoder.matches("MyPass123", "$2a$10$abc...")
        → BCrypt internally:
          → stored hash থেকে salt extract
          → "MyPass123" + salt দিয়ে hash
          → compare
          → Match ✅
        ↓
Step 3: Authentication object return
        → { principal: CustomUserDetails, authenticated: true }

যদি password wrong হতো?
        → BadCredentialsException throw
        → GlobalExceptionHandler ধরে → 401 Unauthorized
```

**এই সব একটা method call এ!** তুমি manually কিছু করো না — Spring করে।

### খ) User Info Extract

```java
CustomUserDetails userDetails = (CustomUserDetails) authentication.getPrincipal();
User user = userDetails.getUser();
```

`getPrincipal()` → authenticated user return করে। Type cast করে আমাদের wrapper থেকে actual `User` entity নিই।

### গ-ঘ) Roles ও Permissions Extract

```java
// Roles: User → Roles → প্রতিটা Role এর name
Set<String> roles = user.getRoles().stream()
    .map(Role::getName)
    .collect(Collectors.toSet());
// Result: {"MEMBER"}

// Permissions: User → Roles → প্রতিটা Role → Permissions → প্রতিটার name
Set<String> permissions = user.getRoles().stream()
    .flatMap(role -> role.getPermissions().stream())
    .map(Permission::getName)
    .collect(Collectors.toSet());
// Result: {"BOOK_READ"}
```

#### `flatMap` কেন?

User এর অনেক Role আছে, প্রতি Role এ অনেক Permission। `flatMap` nested lists কে flat করে।

```
User → [ADMIN, EDITOR]
ADMIN → [BOOK_READ, BOOK_CREATE, BOOK_DELETE]
EDITOR → [BOOK_READ, BOOK_CREATE]

flatMap result → [BOOK_READ, BOOK_CREATE, BOOK_DELETE, BOOK_READ, BOOK_CREATE]
collect(toSet) → {BOOK_READ, BOOK_CREATE, BOOK_DELETE}  (duplicate removed)
```

### ঙ) JWT Token Generate

```java
String token = jwtUtil.generateToken(user.getUsername(), roles, permissions);
```

Part 1 এর JwtUtil — username, roles, permissions নিয়ে JWT তৈরি করে।

### চ) Response Build

```java
return LoginResponse.builder()
    .token(token)                    // "eyJhbGc..."
    .tokenType("Bearer")            // Token type
    .expiresIn(jwtExpiration)        // কত ms valid
    .username(user.getUsername())     // "hasib"
    .roles(roles)                    // {"MEMBER"}
    .permissions(permissions)        // {"BOOK_READ"}
    .build();
```

---

<a name="segment-5"></a>

# Segment 5: AuthController — Reception Desk

## কী করে?

API endpoint provide করে — Client এর request receive করে, AuthService কে call করে, response ফেরত দেয়।

```
Client → AuthController → AuthService → Response
```

---

## AuthController Class

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

### প্রতিটা Part Decode

#### `@RequestMapping("/api/auth")`

Base URL = `/api/auth`। SecurityConfig এ `/api/auth/**` permitAll দিয়েছি — তাই এই URLs public।

#### `@Valid @RequestBody RegisterRequest`

- `@Valid` → DTO এর validation rules check করো (Validation tutorial এ শিখেছি)
- `@RequestBody` → JSON body কে Java object এ convert করো

#### Register Return: 201 Created

Registration successful = নতুন resource তৈরি = HTTP 201।

#### Login Return: 200 OK + LoginResponse

Login successful = Token সহ response।

---

<a name="segment-6"></a>

# Segment 6: Exception Handler Update

## কেন Update দরকার?

আগের GlobalExceptionHandler এ authentication/authorization error handle করিনি।

```
Wrong password → BadCredentialsException → handle না করলে raw error
No permission → AccessDeniedException → handle না করলে raw error
```

---

## নতুন Handlers যোগ করি

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // ← আগের সব handlers এখানে আছে →

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

    // ২. User not found (login এ)
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

    // ৩. Permission নেই
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

### 401 vs 403 — কোনটা কখন?

```
401 Unauthorized → "তুমি কে?" (Authentication fail)
                   → Token নেই, invalid, expired
                   → Wrong password

403 Forbidden    → "তোমার অনুমতি নেই" (Authorization fail)
                   → Login করেছো, কিন্তু এই API তে access নেই
                   → MEMBER trying DELETE /books
```

### কেন Generic Message?

```java
"Invalid username or password"    // ✅ Generic

// নিচেরগুলো ❌ — hacker info পেয়ে যায়
"Username not found"              // hacker বুঝবে username exist না
"Wrong password"                  // hacker বুঝবে username exist করে!
```

---

<a name="segment-7"></a>

# Segment 7: Complete Flow Trace

## Flow 1: Register

```
POST /api/auth/register
{ "username": "hasib", "email": "hasib@mail.com", "password": "MyPass123" }

Step 1: SecurityConfig →  "/api/auth/**" permitAll → public ✅
Step 2: JwtAuthFilter  →  Authorization header নেই → skip
Step 3: AuthController.register() call হলো
Step 4: @Valid         →  DTO validation pass ✅
Step 5: AuthService.register():
        → existsByUsername("hasib") → false ✅
        → existsByEmail("hasib@mail.com") → false ✅
        → findRole("MEMBER") → পেলো ✅
        → passwordEncoder.encode("MyPass123") → "$2a$10$..."
        → userRepository.save(user)
Step 6: Response → 201 Created
        { "message": "User registered successfully" }
```

## Flow 2: Login

```
POST /api/auth/login
{ "username": "hasib", "password": "MyPass123" }

Step 1:  SecurityConfig → "/api/auth/**" permitAll → public ✅
Step 2:  JwtAuthFilter → Authorization header নেই → skip
Step 3:  AuthController.login() call হলো
Step 4:  @Valid → DTO validation pass ✅
Step 5:  AuthService.login():
         → authenticationManager.authenticate()
            → CustomUserDetailsService.loadUserByUsername("hasib")
               → Database query → User পেলো
            → PasswordEncoder.matches("MyPass123", "$2a$10$...")
               → Match ✅
            → Authentication object return
         → roles extract → {"MEMBER"}
         → permissions extract → {"BOOK_READ"}
         → jwtUtil.generateToken("hasib", roles, permissions)
            → "eyJhbGc..."
Step 6:  Response → 200 OK
         {
           "token": "eyJhbGc...",
           "tokenType": "Bearer",
           "username": "hasib",
           "roles": ["MEMBER"],
           "permissions": ["BOOK_READ"]
         }
```

## Flow 3: Protected API (Token সহ)

```
GET /api/books
Authorization: Bearer eyJhbGc...

Step 1:  SecurityConfig → "/api/books" → anyRequest().authenticated()
Step 2:  JwtAuthFilter:
         → Header extract → "Bearer eyJhbGc..."
         → Token extract → "eyJhbGc..."
         → validateToken() → true ✅
         → extractUsername() → "hasib"
         → SecurityContext empty → হ্যাঁ
         → loadUserByUsername("hasib") → UserDetails
         → Authentication object তৈরি
         → SecurityContext.setAuthentication() ✅
Step 3:  @PreAuthorize check (if any) → pass ✅
Step 4:  BookController.getAllBooks() → books return
Step 5:  Response → 200 OK → books list
```

## Flow 4: Protected API (Token ছাড়া)

```
GET /api/books
(Authorization header নেই)

Step 1:  SecurityConfig → "/api/books" → anyRequest().authenticated()
Step 2:  JwtAuthFilter → Header নেই → skip
Step 3:  SecurityContext empty → authenticated না
Step 4:  Response → 401 Unauthorized
```

## Flow 5: Wrong Password

```
POST /api/auth/login
{ "username": "hasib", "password": "WRONG" }

Step 1-4: Same as Login Flow
Step 5:   authenticationManager.authenticate()
          → PasswordEncoder.matches("WRONG", "$2a$10$...") → No match ❌
          → BadCredentialsException throw
Step 6:   GlobalExceptionHandler.handleBadCredentials()
Step 7:   Response → 401 Unauthorized
          { "message": "Invalid username or password" }
```

---

<a name="segment-8"></a>

# Segment 8: Postman Test

## Step 1: App Start

```bash
mvn spring-boot:run
```

Console —

```
ADMIN role created
EDITOR role created
MEMBER role created
```

## Step 2: Register

```
POST http://localhost:8080/api/auth/register
Content-Type: application/json

{
  "username": "hasib",
  "email": "hasib@mail.com",
  "password": "MyPass123!",
  "fullName": "Hasibuzzaman"
}
```

Response: `201 Created` ✅

## Step 3: Login

```
POST http://localhost:8080/api/auth/login
Content-Type: application/json

{
  "username": "hasib",
  "password": "MyPass123!"
}
```

Response:

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "tokenType": "Bearer",
  "expiresIn": 3600000,
  "username": "hasib",
  "roles": ["MEMBER"],
  "permissions": ["BOOK_READ"]
}
```

## Step 4: Token Copy করো

Postman এ Environment Variable এ রাখো —

```
Variable: TOKEN
Value: eyJhbGciOiJIUzI1NiJ9...
```

## Step 5: Protected API — Token ছাড়া

```
GET http://localhost:8080/api/books
(কোনো Authorization header নেই)
```

Response: `401 Unauthorized` ❌

## Step 6: Protected API — Token সহ

```
GET http://localhost:8080/api/books
Authorization: Bearer {{TOKEN}}
```

Response: `200 OK` ✅

```json
[
  { "id": 1, "title": "Pother Pachali" }
]
```

---

<a name="summary"></a>

# Part 2 এর সারসংক্ষেপ

## কোন Piece কী করে — Final

```
DTOs                    → Client ↔ Server data format
SecurityConfig          → "কোন API public, কোন protected"
JwtAuthenticationFilter → "প্রতি request এ Token verify করে user চিনে"
AuthService             → "Register/Login এর business logic"
AuthController          → "API endpoint provide করে"
ExceptionHandler        → "Auth error সুন্দরভাবে handle করে"
```

## সব Piece কীভাবে Connect

```
Part 1 এ বানিয়েছি (Foundation):
Database ← Repository ← UserDetailsService ← CustomUserDetails
PasswordEncoder
JwtUtil

Part 2 এ বানিয়েছি (Flow):
DTOs ← AuthController ← AuthService ← (Part 1 এর সব)
SecurityConfig (সব connect করে)
JwtAuthFilter (প্রতি request এ চলে)
```

## Build Order

```
1. DTOs              → format ঠিক
2. SecurityConfig    → নিয়ম ঠিক
3. JwtAuthFilter     → token verify system
4. AuthService       → logic
5. AuthController    → API
6. ExceptionHandler  → error handling
7. Test              → verify
```

---

Part 2 এখন clear? কোনো specific segment নিয়ে আরো প্রশ্ন থাকলে বলো। 🚀

Happy coding!
