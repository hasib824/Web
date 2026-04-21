# Phase 4 — Security & JWT Authentication

**Duration:** ১ দিন | ~৭–৮ ঘণ্টা

**Phase শেষে আপনার হাতে থাকবে:**
- Spring Security 6 filter chain properly configured
- `User` entity with roles (OPERATOR, BILLING_CLERK, ADMIN, SUPERVISOR)
- Password hashing with BCrypt
- `POST /auth/login` endpoint যা JWT access token ইস্যু করে
- JWT validation filter — প্রত্যেক request-এ token verify
- Role-based endpoint protection (`@PreAuthorize("hasRole('ADMIN')")`)
- CORS configured for React frontend
- Authenticated user info access (who did this gate-in?)
- Proper 401/403 error responses in ProblemDetail format

---

## Phase 4 Overview

এই phase-এ আমরা এমন একটা system বানাব যেখানে:

1. User `POST /auth/login` এ username + password পাঠাবে
2. Server verify করে একটা JWT return করবে
3. React app সেই JWT localStorage-এ save করে রাখবে
4. প্রত্যেক subsequent request-এ `Authorization: Bearer <token>` header পাঠাবে
5. Server token verify করবে, user identity retrieve করবে, endpoint access allow/deny করবে

### Android mental model

আপনার eKYC app-এ সম্ভবত এরকম pattern ছিল:

```kotlin
// OkHttp interceptor — প্রত্যেক request-এ token add করে
class AuthInterceptor @Inject constructor(
    private val tokenStorage: TokenStorage
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val token = tokenStorage.getAccessToken()
        val authedRequest = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer $token")
            .build()
        return chain.proceed(authedRequest)
    }
}
```

Spring-এ উল্টো side — আপনি **server**, তাই আপনি token **validate** করবেন:

| Android (client) | Spring Boot (server) |
|---|---|
| `AuthInterceptor` token attach করে | `JwtAuthenticationFilter` token parse করে |
| `TokenStorage` (SharedPreferences/DataStore) | No storage — stateless, token নিজেই সব info carry করে |
| 401 হলে login screen-এ navigate | 401 return — React login screen-এ navigate করবে |
| Retrofit `@Header("Authorization")` | `@AuthenticationPrincipal UserDetails user` method parameter |

**সবচেয়ে বড় mental shift:** Spring Security একটা **Servlet Filter chain** — আপনার `@RestController` method call হওয়ার আগে কয়েকটা filter request-টা process করে। সেই filter chain-এই authentication/authorization decide হয়। Controller যখন পৌঁছায়, তখন identity already established।

---

## Step 1 — Dependency যোগ করুন

`pom.xml`-এ:

```xml
<!-- Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT library (jjwt — বাংলাদেশি enterprise projects-এ most common) -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.5</version>
    <scope>runtime</scope>
</dependency>
```

Maven reload।

### ⚠️ App restart করলে কী দেখবেন

Spring Security classpath-এ পেয়ে **by default সব endpoint block করে দেবে**। App চালানোর সময় console-এ এরকম একটা warning দেখবেন:

```
Using generated security password: 8a3f1c92-...
This generated password is for development use only.
```

আর Postman-এ যেকোনো endpoint hit করলে 401 Unauthorized আসবে। এটাই Spring Security-র default — "deny all"। আমরা এখন configure করে selective access দেব।

---

## Step 2 — User Entity, Role Enum, Repository

`user/domain/` package বানান। এখানে user-related সব আসবে।

### Role enum

`user/domain/Role.java`:

```java
package com.yourcompany.portops.user.domain;

public enum Role {
    YARD_OPERATOR,      // gate-in, gate-out, yard moves
    BILLING_CLERK,      // invoice generation, tariff lookup
    SUPERVISOR,         // read access সব module-এ
    ADMIN               // user management, system config
}
```

> **Note:** Spring Security-এর convention হচ্ছে role prefix `ROLE_` — যেমন `ROLE_ADMIN`। কিন্তু `@PreAuthorize("hasRole('ADMIN')")` লিখলে Spring automatically prefix add করে। তাই enum-এ prefix ছাড়া রাখছি।

### User Entity

`user/domain/User.java`:

```java
package com.yourcompany.portops.user.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.time.Instant;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "USERS",
       indexes = {
           @Index(name = "IDX_USERS_USERNAME", columnList = "USERNAME", unique = true),
           @Index(name = "IDX_USERS_EMAIL", columnList = "EMAIL", unique = true)
       })
@Getter
@Setter
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ID")
    private Long id;

    @Column(name = "USERNAME", nullable = false, unique = true, length = 50)
    private String username;

    @Column(name = "EMAIL", nullable = false, unique = true, length = 150)
    private String email;

    /**
     * BCrypt hash, plaintext never stored।
     * Length 60 because BCrypt always produces 60-char hash।
     */
    @Column(name = "PASSWORD_HASH", nullable = false, length = 60)
    private String passwordHash;

    @Column(name = "FULL_NAME", nullable = false, length = 150)
    private String fullName;

    @Column(name = "EMPLOYEE_ID", length = 20)
    private String employeeId;    // company-specific ID

    @Column(name = "ENABLED", nullable = false)
    private boolean enabled = true;

    @Column(name = "ACCOUNT_NON_LOCKED", nullable = false)
    private boolean accountNonLocked = true;

    /**
     * এক user-এর একাধিক role থাকতে পারে।
     * @ElementCollection — simple value collection (not entity), separate join table-এ stored।
     * এটা ManyToMany-র চেয়ে clean যখন role একটা simple enum/string।
     */
    @ElementCollection(fetch = FetchType.EAGER, targetClass = Role.class)
    @CollectionTable(
        name = "USER_ROLES",
        joinColumns = @JoinColumn(name = "USER_ID"),
        indexes = @Index(name = "IDX_USER_ROLES_USER", columnList = "USER_ID")
    )
    @Enumerated(EnumType.STRING)
    @Column(name = "ROLE", length = 30, nullable = false)
    private Set<Role> roles = new HashSet<>();

    @Column(name = "LAST_LOGIN_AT")
    private Instant lastLoginAt;

    @Column(name = "CREATED_AT", nullable = false, updatable = false)
    private Instant createdAt;

    @Column(name = "UPDATED_AT")
    private Instant updatedAt;

    @PrePersist
    void onCreate() {
        this.createdAt = Instant.now();
        this.updatedAt = this.createdAt;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = Instant.now();
    }
}
```

### `@ElementCollection` কেন, `@ManyToMany Role` না?

Roles যেহেতু enum — independent entity না — আলাদা table-এ `Role` entity বানানো overkill। `@ElementCollection` একটা join table বানায় শুধু values store করার জন্য।

**DB-তে যা হবে:**
```
USERS           USER_ROLES
-----           ----------
ID  USERNAME    USER_ID  ROLE
1   alice       1        YARD_OPERATOR
2   bob         1        SUPERVISOR
                2        BILLING_CLERK
                2        ADMIN
```

### EAGER fetch কেন এখানে OK?

Usually আমি strongly recommend করি `LAZY`। কিন্তু roles প্রত্যেক authenticated request-এ লাগে (authorization check-এর জন্য), তাই EAGER logical। Small set, static data — performance impact negligible।

### Repository

`user/persistence/UserRepository.java`:

```java
package com.yourcompany.portops.user.persistence;

import com.yourcompany.portops.user.domain.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

---

## Step 3 — `UserDetailsService` Implementation

Spring Security-র কাছে "user" concept হচ্ছে `UserDetails` interface। আপনার `User` entity-কে সেই interface-এ map করতে হবে।

দুটো approach:
1. `User` entity নিজেই `UserDetails` implement করবে — simple, কিন্তু persistence class-এ security concern ঢুকে যাচ্ছে
2. আলাদা `UserPrincipal` adapter class — cleaner separation

**আমরা option 2 নেব** — enterprise pattern।

### `UserPrincipal` adapter

`user/domain/UserPrincipal.java`:

```java
package com.yourcompany.portops.user.domain;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.stream.Collectors;

/**
 * Security-এর জন্য adapter। User entity-কে Spring-এর UserDetails-এ wrap করে।
 * Authentication-এর পর SecurityContext-এ এটাই থাকবে।
 */
public class UserPrincipal implements UserDetails {

    private final Long userId;
    private final String username;
    private final String passwordHash;
    private final String fullName;
    private final Collection<? extends GrantedAuthority> authorities;
    private final boolean enabled;
    private final boolean accountNonLocked;

    private UserPrincipal(Long userId, String username, String passwordHash, String fullName,
                          Collection<? extends GrantedAuthority> authorities,
                          boolean enabled, boolean accountNonLocked) {
        this.userId = userId;
        this.username = username;
        this.passwordHash = passwordHash;
        this.fullName = fullName;
        this.authorities = authorities;
        this.enabled = enabled;
        this.accountNonLocked = accountNonLocked;
    }

    public static UserPrincipal from(User user) {
        var authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))  // ⚠️ ROLE_ prefix
            .collect(Collectors.toSet());

        return new UserPrincipal(
            user.getId(),
            user.getUsername(),
            user.getPasswordHash(),
            user.getFullName(),
            authorities,
            user.isEnabled(),
            user.isAccountNonLocked()
        );
    }

    // Custom accessors — Controller-এ @AuthenticationPrincipal ব্যবহার করলে এগুলো পাবো
    public Long getUserId() { return userId; }
    public String getFullName() { return fullName; }

    // --- UserDetails interface ---

    @Override public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }
    @Override public String getPassword() { return passwordHash; }
    @Override public String getUsername() { return username; }
    @Override public boolean isEnabled() { return enabled; }
    @Override public boolean isAccountNonLocked() { return accountNonLocked; }
    @Override public boolean isAccountNonExpired() { return true; }
    @Override public boolean isCredentialsNonExpired() { return true; }
}
```

### `ROLE_` prefix-এর গল্প

Spring Security-র convention: authority string-এ `ROLE_` prefix থাকলে সেটাকে "role" হিসেবে treat করে। তাই:

- Entity-তে store: `YARD_OPERATOR`
- UserPrincipal authority: `ROLE_YARD_OPERATOR`
- `@PreAuthorize("hasRole('YARD_OPERATOR')")` — Spring automatically `ROLE_` prefix match করে
- `@PreAuthorize("hasAuthority('ROLE_YARD_OPERATOR')")` — exact string match

দুটোই কাজ করে। `hasRole()` cleaner, এটাই recommended।

### `UserDetailsService`

`user/domain/AppUserDetailsService.java`:

```java
package com.yourcompany.portops.user.domain;

import com.yourcompany.portops.user.persistence.UserRepository;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AppUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public AppUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        return UserPrincipal.from(user);
    }
}
```

### ⚠️ ক্লাস নাম conflict এড়ান

Spring ইতিমধ্যে একটা `UserDetailsService` interface দেয়, আর আমাদের entity class-এর নাম `User`। নাম confusion এড়াতে:

- আমাদের service: `AppUserDetailsService` (প্রিফিক্স App)
- Spring-এর বিল্ট-ইন interface: `UserDetailsService`

এই ধরনের naming clash enterprise codebase-এ common। সবসময় application-specific prefix দিন।

---

## Step 4 — JWT Service

JWT token issue + validate করার logic আলাদা service-এ রাখব।

### JWT config properties

`application.yml`-এ যোগ করুন:

```yaml
portops:
  security:
    jwt:
      # Base64-encoded signing key — MINIMUM 256 bits (32 bytes) for HS256
      # Production-এ এটা env variable থেকে আসবে
      secret: ${JWT_SECRET:dGhpcy1pcy1hLXZlcnktbG9uZy1zZWNyZXQta2V5LWZvci1kZXYtb25seS1kby1ub3QtdXNlLWluLXByb2Q=}
      access-token-expiry-minutes: 60
      issuer: portops-api
```

**SECRET value note:** উপরের base64 string `dev only` key। Production-এ এটা 32+ byte random key, env variable থেকে, গিট-এ কখনো commit না।

Key generate করার quick way:
```bash
# Linux/Mac
openssl rand -base64 48

# Java-তে
# Keys.secretKeyFor(SignatureAlgorithm.HS256); তারপর Encoders.BASE64.encode(key.getEncoded());
```

### Config properties class

`common/config/JwtProperties.java`:

```java
package com.yourcompany.portops.common.config;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "portops.security.jwt")
public record JwtProperties(
    String secret,
    int accessTokenExpiryMinutes,
    String issuer
) {}
```

### Main application class-এ enable

`PortopsApplication.java`:

```java
@SpringBootApplication
@ConfigurationPropertiesScan          // ⬅ এটা add করুন
public class PortopsApplication {
    public static void main(String[] args) {
        SpringApplication.run(PortopsApplication.class, args);
    }
}
```

`@ConfigurationPropertiesScan` নিজে নিজে `@ConfigurationProperties`-marked records খুঁজে bean-এ register করবে।

### JwtService

`common/security/JwtService.java`:

```java
package com.yourcompany.portops.common.security;

import com.yourcompany.portops.common.config.JwtProperties;
import com.yourcompany.portops.user.domain.UserPrincipal;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Date;
import java.util.List;

@Service
public class JwtService {

    private final JwtProperties props;
    private final SecretKey signingKey;

    public JwtService(JwtProperties props) {
        this.props = props;
        byte[] keyBytes = Decoders.BASE64.decode(props.secret());
        this.signingKey = Keys.hmacShaKeyFor(keyBytes);
    }

    /**
     * User-এর জন্য JWT issue করে। Standard claims + custom claims include করে।
     */
    public String generateToken(UserPrincipal principal) {
        Instant now = Instant.now();
        Instant expiry = now.plus(props.accessTokenExpiryMinutes(), ChronoUnit.MINUTES);

        List<String> roles = principal.getAuthorities().stream()
            .map(Object::toString)
            .toList();

        return Jwts.builder()
            .issuer(props.issuer())
            .subject(principal.getUsername())
            .claim("userId", principal.getUserId())
            .claim("fullName", principal.getFullName())
            .claim("roles", roles)
            .issuedAt(Date.from(now))
            .expiration(Date.from(expiry))
            .signWith(signingKey)
            .compact();
    }

    /**
     * Token-এর signature + expiry validate করে claims return করে।
     * Invalid হলে io.jsonwebtoken.JwtException throw।
     */
    public Claims parseAndValidate(String token) {
        return Jwts.parser()
            .verifyWith(signingKey)
            .requireIssuer(props.issuer())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public String extractUsername(String token) {
        return parseAndValidate(token).getSubject();
    }
}
```

### JWT structure recap

একটা JWT তিনটা dot-separated base64 part:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGljZSIsInJvbGVzIjpbIlJPTEVfQURNSU4iXX0.SIGNATURE
│                  │ │                                                  │ │        │
└─── HEADER ───────┘ └────── PAYLOAD (claims) ────────────────────────┘ └────────┘
    { algo, type }     { sub, iss, exp, custom... }                      HMAC-SHA256
```

**Security guarantee:** Payload plaintext (base64 decode করলেই পড়া যায়), কিন্তু signature modify করতে গেলে secret key লাগে। তাই payload-এ sensitive data রাখবেন না — শুধু identity + claims।

---

## Step 5 — JWT Authentication Filter

এই filter প্রত্যেক request-এ একবার চলে — token পড়ে, validate করে, SecurityContext populate করে।

`common/security/JwtAuthenticationFilter.java`:

```java
package com.yourcompany.portops.common.security;

import com.yourcompany.portops.user.domain.AppUserDetailsService;
import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private static final Logger log = LoggerFactory.getLogger(JwtAuthenticationFilter.class);
    private static final String HEADER = "Authorization";
    private static final String PREFIX = "Bearer ";

    private final JwtService jwtService;
    private final AppUserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtService jwtService,
                                   AppUserDetailsService userDetailsService) {
        this.jwtService = jwtService;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(@NonNull HttpServletRequest request,
                                    @NonNull HttpServletResponse response,
                                    @NonNull FilterChain chain) throws ServletException, IOException {

        String token = extractToken(request);

        if (token != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            try {
                String username = jwtService.extractUsername(token);

                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // Authentication object বানানো — credentials null কারণ JWT ইতিমধ্যে verified
                var authToken = new UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.getAuthorities()
                );
                authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authToken);

            } catch (JwtException ex) {
                // Invalid/expired token — log করি, কিন্তু 401 throw করি না
                // Spring Security-র downstream (AuthorizationFilter) এটা handle করবে
                log.debug("JWT validation failed for URI [{}]: {}",
                          request.getRequestURI(), ex.getMessage());
                SecurityContextHolder.clearContext();
            } catch (Exception ex) {
                log.warn("Unexpected error during JWT authentication", ex);
                SecurityContextHolder.clearContext();
            }
        }

        chain.doFilter(request, response);
    }

    private String extractToken(HttpServletRequest request) {
        String header = request.getHeader(HEADER);
        if (header != null && header.startsWith(PREFIX)) {
            return header.substring(PREFIX.length());
        }
        return null;
    }
}
```

### কয়েকটা subtlety

**1. `OncePerRequestFilter` extend কেন?**
- Ensures filter একই request-এ multiple times চলবে না (forwards/includes-এ issue avoid)
- Spring-এর recommended base class for custom filters

**2. Invalid token হলে filter 401 return করছে না**
- Filter শুধু "try to authenticate" করে। Success হলে context set করে, fail হলে context empty থাকে
- পরের filter (Spring's AuthorizationFilter) দেখবে context empty → endpoint-এ authentication required কিনা → 401 return করবে
- এই separation of concerns — filter chain-এর beauty

**3. `SecurityContextHolder.getContext().getAuthentication() == null` check**
- যদি আগে থেকেই কোনো authentication set হয়ে থাকে (e.g., Basic Auth filter-এ), আমরা override করব না
- Multiple auth methods simultaneously support-এর জন্য

---

## Step 6 — Security Configuration

এখন সব piece together — SecurityFilterChain configure করি।

`common/security/SecurityConfig.java`:

```java
package com.yourcompany.portops.common.security;

import com.yourcompany.portops.user.domain.AppUserDetailsService;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.ProviderManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableMethodSecurity        // ⬅ @PreAuthorize enable
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final JwtAuthenticationEntryPoint authEntryPoint;
    private final JwtAccessDeniedHandler accessDeniedHandler;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter,
                          JwtAuthenticationEntryPoint authEntryPoint,
                          JwtAccessDeniedHandler accessDeniedHandler) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.authEntryPoint = authEntryPoint;
        this.accessDeniedHandler = accessDeniedHandler;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(AbstractHttpConfigurer::disable)   // Stateless REST — CSRF not applicable
            .cors(Customizer.withDefaults())          // corsConfigurationSource bean use হবে
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()

                // Role-based
                .requestMatchers(HttpMethod.POST, "/api/v1/containers/*/gate-in")
                    .hasAnyRole("YARD_OPERATOR", "ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/v1/containers")
                    .hasAnyRole("YARD_OPERATOR", "ADMIN")
                .requestMatchers(HttpMethod.DELETE, "/api/v1/**")
                    .hasRole("ADMIN")
                .requestMatchers("/api/v1/billing/**")
                    .hasAnyRole("BILLING_CLERK", "ADMIN")
                .requestMatchers("/api/v1/users/**")
                    .hasRole("ADMIN")

                // বাকিসব authenticated user-এর জন্য open
                .anyRequest().authenticated()
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)   // 401 handler
                .accessDeniedHandler(accessDeniedHandler)   // 403 handler
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);   // strength 12 — reasonable balance of security vs perf
    }

    /**
     * AuthenticationManager — login endpoint-এ username/password verify করার জন্য।
     */
    @Bean
    public AuthenticationManager authenticationManager(AppUserDetailsService userDetailsService,
                                                       PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return new ProviderManager(provider);
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration cfg = new CorsConfiguration();
        cfg.setAllowedOrigins(List.of(
            "http://localhost:3000",           // React dev
            "http://localhost:5173",           // Vite dev
            "https://portops.example.com"      // Production
        ));
        cfg.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        cfg.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        cfg.setExposedHeaders(List.of("Location"));   // Created resource-এর Location header React-এর access দরকার
        cfg.setAllowCredentials(true);
        cfg.setMaxAge(3600L);   // preflight cache duration in seconds

        UrlBasedCorsConfigurationSource src = new UrlBasedCorsConfigurationSource();
        src.registerCorsConfiguration("/**", cfg);
        return src;
    }
}
```

### Filter chain order বোঝা

Request আসলে এই order-এ filter-দের মধ্য দিয়ে যায়:

```
HTTP Request
    ↓
[CORS Filter]             ← browser preflight handle
    ↓
[JwtAuthenticationFilter] ← আমাদের custom (Bearer token parse)
    ↓
[UsernamePasswordAuth..]  ← Spring-এর default (আমরা before-এ আমাদেরটা add করেছি)
    ↓
[AuthorizationFilter]     ← role check, @PreAuthorize
    ↓
Controller method invoke
```

### `@EnableMethodSecurity` কেন?

URL-level security (`requestMatchers(...)`) যথেষ্ট না সব সময়। Method-level: "user শুধু নিজের invoice দেখতে পারবে, অন্যেরটা না" — এই logic-এ `@PreAuthorize("#invoice.userId == authentication.principal.userId")` লাগে।

---

## Step 7 — Custom 401/403 Handlers (ProblemDetail JSON)

Spring Security default-এ 401/403 HTML page return করে। আমরা consistent ProblemDetail JSON চাই।

`common/security/JwtAuthenticationEntryPoint.java`:

```java
package com.yourcompany.portops.common.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ProblemDetail;
import org.springframework.security.core.AuthenticationException;
import org.springframework.security.web.AuthenticationEntryPoint;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Instant;

/**
 * 401 — Authentication missing/invalid।
 */
@Component
public class JwtAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    public JwtAuthenticationEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(HttpServletRequest request,
                         HttpServletResponse response,
                         AuthenticationException authException) throws IOException {

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.UNAUTHORIZED,
            "Authentication is required to access this resource"
        );
        problem.setTitle("Unauthorized");
        problem.setProperty("errorCode", "UNAUTHORIZED");
        problem.setProperty("timestamp", Instant.now().toString());
        problem.setProperty("path", request.getRequestURI());

        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(problem));
    }
}
```

`common/security/JwtAccessDeniedHandler.java`:

```java
package com.yourcompany.portops.common.security;

import com.fasterxml.jackson.databind.ObjectMapper;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.http.MediaType;
import org.springframework.http.ProblemDetail;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.security.web.access.AccessDeniedHandler;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Instant;

/**
 * 403 — Authenticated but lacks required role।
 */
@Component
public class JwtAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    public JwtAccessDeniedHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void handle(HttpServletRequest request,
                       HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException {

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.FORBIDDEN,
            "You do not have permission to access this resource"
        );
        problem.setTitle("Access Denied");
        problem.setProperty("errorCode", "ACCESS_DENIED");
        problem.setProperty("timestamp", Instant.now().toString());
        problem.setProperty("path", request.getRequestURI());

        response.setStatus(HttpStatus.FORBIDDEN.value());
        response.setContentType(MediaType.APPLICATION_PROBLEM_JSON_VALUE);
        response.getWriter().write(objectMapper.writeValueAsString(problem));
    }
}
```

**401 vs 403 clarification:**

| Status | মানে |
|---|---|
| **401 Unauthorized** | "Who are you?" — token missing/invalid। Login করো। |
| **403 Forbidden** | "I know who you are, but you can't do this" — authenticated, but wrong role। |

---

## Step 8 — Auth Endpoint (Login + User Registration)

### DTOs

`user/api/dto/` package-এ:

`LoginRequest.java`:
```java
package com.yourcompany.portops.user.api.dto;

import jakarta.validation.constraints.NotBlank;

public record LoginRequest(
    @NotBlank String username,
    @NotBlank String password
) {}
```

`LoginResponse.java`:
```java
package com.yourcompany.portops.user.api.dto;

import java.util.List;

public record LoginResponse(
    String accessToken,
    String tokenType,       // "Bearer"
    long expiresInSeconds,
    UserInfo user
) {
    public record UserInfo(
        Long id,
        String username,
        String fullName,
        List<String> roles
    ) {}
}
```

`CreateUserRequest.java`:
```java
package com.yourcompany.portops.user.api.dto;

import com.yourcompany.portops.user.domain.Role;
import jakarta.validation.constraints.*;

import java.util.Set;

public record CreateUserRequest(
    @NotBlank @Size(min = 3, max = 50)
    @Pattern(regexp = "[a-z0-9._-]+", message = "Username: lowercase letters, digits, . _ - only")
    String username,

    @NotBlank @Email
    String email,

    @NotBlank @Size(min = 8, max = 100)
    String password,

    @NotBlank @Size(max = 150)
    String fullName,

    String employeeId,

    @NotEmpty
    Set<Role> roles
) {}
```

### Exceptions

`user/domain/exception/DuplicateUsernameException.java`:
```java
package com.yourcompany.portops.user.domain.exception;

import com.yourcompany.portops.common.exception.DomainException;
import org.springframework.http.HttpStatus;

public class DuplicateUsernameException extends DomainException {
    public DuplicateUsernameException(String username) {
        super("Username already taken: " + username);
    }
    @Override public HttpStatus httpStatus() { return HttpStatus.CONFLICT; }
    @Override public String errorCode() { return "DUPLICATE_USERNAME"; }
}
```

`user/domain/exception/InvalidCredentialsException.java`:
```java
package com.yourcompany.portops.user.domain.exception;

import com.yourcompany.portops.common.exception.DomainException;
import org.springframework.http.HttpStatus;

public class InvalidCredentialsException extends DomainException {
    public InvalidCredentialsException() {
        super("Invalid username or password");
    }
    @Override public HttpStatus httpStatus() { return HttpStatus.UNAUTHORIZED; }
    @Override public String errorCode() { return "INVALID_CREDENTIALS"; }
}
```

### AuthService

`user/domain/AuthService.java`:

```java
package com.yourcompany.portops.user.domain;

import com.yourcompany.portops.common.config.JwtProperties;
import com.yourcompany.portops.common.security.JwtService;
import com.yourcompany.portops.user.api.dto.*;
import com.yourcompany.portops.user.domain.exception.InvalidCredentialsException;
import com.yourcompany.portops.user.persistence.UserRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.DisabledException;
import org.springframework.security.authentication.LockedException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Service
@Transactional
public class AuthService {

    private static final Logger log = LoggerFactory.getLogger(AuthService.class);

    private final AuthenticationManager authenticationManager;
    private final JwtService jwtService;
    private final JwtProperties jwtProperties;
    private final UserRepository userRepository;

    public AuthService(AuthenticationManager authenticationManager,
                       JwtService jwtService,
                       JwtProperties jwtProperties,
                       UserRepository userRepository) {
        this.authenticationManager = authenticationManager;
        this.jwtService = jwtService;
        this.jwtProperties = jwtProperties;
        this.userRepository = userRepository;
    }

    public LoginResponse login(LoginRequest request) {
        Authentication auth;
        try {
            auth = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(request.username(), request.password())
            );
        } catch (BadCredentialsException | DisabledException | LockedException ex) {
            // Specific exceptions-কে generic "invalid credentials"-এ convert করি
            // Security best practice: "user exists but password wrong" leak করা যাবে না
            log.warn("Login failed for username={}: {}", request.username(), ex.getClass().getSimpleName());
            throw new InvalidCredentialsException();
        }

        UserPrincipal principal = (UserPrincipal) auth.getPrincipal();
        String token = jwtService.generateToken(principal);

        // lastLoginAt update
        userRepository.findById(principal.getUserId()).ifPresent(u -> {
            u.setLastLoginAt(Instant.now());
            // dirty checking — transaction commit-এ auto update
        });

        log.info("User logged in: username={} userId={}", principal.getUsername(), principal.getUserId());

        return new LoginResponse(
            token,
            "Bearer",
            jwtProperties.accessTokenExpiryMinutes() * 60L,
            new LoginResponse.UserInfo(
                principal.getUserId(),
                principal.getUsername(),
                principal.getFullName(),
                principal.getAuthorities().stream().map(Object::toString).toList()
            )
        );
    }
}
```

### UserService (registration)

`user/domain/UserService.java`:

```java
package com.yourcompany.portops.user.domain;

import com.yourcompany.portops.user.api.dto.CreateUserRequest;
import com.yourcompany.portops.user.domain.exception.DuplicateUsernameException;
import com.yourcompany.portops.user.persistence.UserRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class UserService {

    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    public Long createUser(CreateUserRequest request) {
        if (userRepository.existsByUsername(request.username())) {
            throw new DuplicateUsernameException(request.username());
        }

        User user = new User();
        user.setUsername(request.username());
        user.setEmail(request.email());
        user.setPasswordHash(passwordEncoder.encode(request.password()));   // ⬅ BCrypt hash
        user.setFullName(request.fullName());
        user.setEmployeeId(request.employeeId());
        user.setRoles(request.roles());

        User saved = userRepository.save(user);
        log.info("Created user: username={} roles={}", saved.getUsername(), saved.getRoles());
        return saved.getId();
    }
}
```

### AuthController

`user/api/AuthController.java`:

```java
package com.yourcompany.portops.user.api;

import com.yourcompany.portops.user.api.dto.*;
import com.yourcompany.portops.user.domain.AuthService;
import com.yourcompany.portops.user.domain.UserService;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.Map;

@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final AuthService authService;
    private final UserService userService;

    public AuthController(AuthService authService, UserService userService) {
        this.authService = authService;
        this.userService = userService;
    }

    @PostMapping("/login")
    public LoginResponse login(@Valid @RequestBody LoginRequest request) {
        return authService.login(request);
    }

    /**
     * User creation — only ADMIN পারে।
     * অথবা, একটা "bootstrap admin" endpoint বানাতে পারেন first-time setup-এর জন্য,
     * যা শুধু DB খালি থাকলে কাজ করে।
     */
    @PostMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Map<String, Long>> createUser(@Valid @RequestBody CreateUserRequest request) {
        Long id = userService.createUser(request);
        return ResponseEntity
            .created(URI.create("/api/v1/auth/users/" + id))
            .body(Map.of("id", id));
    }
}
```

---

## Step 9 — `@AuthenticationPrincipal` দিয়ে User Info Access

আপনার `ContainerMovementService.gateIn()` method-এ "who performed this gate-in?" record করা লাগবে। Controller থেকে user identity পাঠাতে হবে।

### YardActivity-এ operator field

Phase 2-এর `YardActivity.java` entity-তে operator link add করুন:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "OPERATOR_USER_ID")
private User operator;
```

`User` import করুন (`com.yourcompany.portops.user.domain.User`)।

### Controller update

`ContainerController.gateIn`:

```java
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import com.yourcompany.portops.user.domain.UserPrincipal;

@PostMapping("/{id}/gate-in")
@PreAuthorize("hasAnyRole('YARD_OPERATOR', 'ADMIN')")
public YardActivityResponse gateIn(@PathVariable Long id,
                                   @Valid @RequestBody GateInRequest request,
                                   @AuthenticationPrincipal UserPrincipal principal) {
    return movementService.gateIn(id, request, principal.getUserId());
}
```

### Service update

`ContainerMovementService.gateIn(...)`:

```java
public YardActivityResponse gateIn(Long containerId, GateInRequest request, Long operatorUserId) {
    Container container = containerRepository.findById(containerId)
        .orElseThrow(() -> new ContainerNotFoundException(containerId));

    // ... existing logic ...

    User operator = userRepository.getReferenceById(operatorUserId);  // ⬅ lazy reference, DB hit এড়ানো
    gateActivity.setOperator(operator);
    moveActivity.setOperator(operator);

    // ... rest
}
```

### `getReferenceById` vs `findById`

| Method | Behavior |
|---|---|
| `findById(id)` | DB query করে full entity load করে |
| `getReferenceById(id)` | একটা proxy return করে, DB-তে query করে না। Shadow object যেটার শুধু ID আছে। |

আমরা operator-এর actual data লাগাচ্ছি না — শুধু foreign key set করছি। তাই `getReferenceById` ব্যবহার করলাম — **এক ডাটাবেস roundtrip save হলো**। এটা enterprise systems-এ হাজার hit/sec হলে matter করে।

**Caveat:** Reference return করা entity lazy — outside transaction access করলে `LazyInitializationException`।

### UserRepository inject করতে ভুলবেন না

`ContainerMovementService` constructor-এ UserRepository যোগ করুন।

---

## Step 10 — Bootstrap First Admin User

Chicken-and-egg problem: `POST /api/v1/auth/users` requires ADMIN role, কিন্তু শুরুতে কোনো user-ই নেই।

**Option: CommandLineRunner** — startup-এ একবার চালান, যদি user count 0 হয়।

`user/domain/UserBootstrap.java`:

```java
package com.yourcompany.portops.user.domain;

import com.yourcompany.portops.user.persistence.UserRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Profile;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Component;

import java.util.Set;

@Component
@Profile("dev")   // শুধু dev profile-এ চলবে
public class UserBootstrap implements CommandLineRunner {

    private static final Logger log = LoggerFactory.getLogger(UserBootstrap.class);

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserBootstrap(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    @Override
    public void run(String... args) {
        if (userRepository.count() > 0) return;

        log.info("No users found. Creating bootstrap users...");

        User admin = new User();
        admin.setUsername("admin");
        admin.setEmail("admin@portops.local");
        admin.setPasswordHash(passwordEncoder.encode("Admin@123"));
        admin.setFullName("System Administrator");
        admin.setRoles(Set.of(Role.ADMIN));

        User operator = new User();
        operator.setUsername("yard_op_1");
        operator.setEmail("op1@portops.local");
        operator.setPasswordHash(passwordEncoder.encode("Op@123"));
        operator.setFullName("Yard Operator One");
        operator.setRoles(Set.of(Role.YARD_OPERATOR));

        User billing = new User();
        billing.setUsername("billing_clerk");
        billing.setEmail("billing@portops.local");
        billing.setPasswordHash(passwordEncoder.encode("Bill@123"));
        billing.setFullName("Billing Clerk");
        billing.setRoles(Set.of(Role.BILLING_CLERK));

        userRepository.saveAll(java.util.List.of(admin, operator, billing));
        log.info("Bootstrap users created: admin, yard_op_1, billing_clerk");
    }
}
```

> **Never do this in prod.** Production-এ user creation manual/migration script-এ হবে। `@Profile("dev")` এটা guarantee করছে।

---

## Step 11 — Postman-এ End-to-End Test

App restart করুন। Console-এ দেখবেন bootstrap log:
```
Bootstrap users created: admin, yard_op_1, billing_clerk
```

### Test 1: Unauthenticated access → 401

```
GET http://localhost:8080/api/v1/containers
```

**Expected:** 401 Unauthorized
```json
{
  "status": 401,
  "title": "Unauthorized",
  "detail": "Authentication is required to access this resource",
  "errorCode": "UNAUTHORIZED"
}
```

### Test 2: Login

```
POST http://localhost:8080/api/v1/auth/login
Content-Type: application/json

{
  "username": "yard_op_1",
  "password": "Op@123"
}
```

**Expected:** 200 OK
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiJ9.eyJ...",
  "tokenType": "Bearer",
  "expiresInSeconds": 3600,
  "user": {
    "id": 2,
    "username": "yard_op_1",
    "fullName": "Yard Operator One",
    "roles": ["ROLE_YARD_OPERATOR"]
  }
}
```

Copy accessToken। Postman-এ easier way — **Postman environment variable**-এ save করে রাখুন:

Tests tab-এ script:
```js
pm.environment.set("token", pm.response.json().accessToken);
```

### Test 3: Authenticated request

Postman Header tab-এ:
```
Authorization: Bearer {{token}}
```

```
GET http://localhost:8080/api/v1/containers
Authorization: Bearer <token>
```

**Expected:** 200 OK, container list।

### Test 4: Invalid credentials

```
POST http://localhost:8080/api/v1/auth/login
{
  "username": "yard_op_1",
  "password": "wrongPassword"
}
```

**Expected:** 401 Unauthorized
```json
{
  "errorCode": "INVALID_CREDENTIALS"
}
```

### Test 5: Role-based access — forbidden

yard_op_1 token দিয়ে admin-only endpoint hit করুন:

```
POST http://localhost:8080/api/v1/auth/users
Authorization: Bearer <yard_op_1 token>
Content-Type: application/json

{
  "username": "new_user",
  "email": "new@portops.local",
  "password": "Pass@123",
  "fullName": "New User",
  "roles": ["YARD_OPERATOR"]
}
```

**Expected:** 403 Forbidden
```json
{
  "status": 403,
  "errorCode": "ACCESS_DENIED"
}
```

### Test 6: Admin role — success

Login as admin, same request:

```
POST http://localhost:8080/api/v1/auth/login
{
  "username": "admin",
  "password": "Admin@123"
}
```

Token update। আগের POST /auth/users retry — **201 Created**।

### Test 7: Gate-in with operator identity

yard_op_1 token দিয়ে:

```
POST http://localhost:8080/api/v1/containers/1/gate-in
Authorization: Bearer <yard_op_1 token>
Content-Type: application/json

{
  "yardSlotCode": "A-07-03-2"
}
```

**Expected:** 200 OK + DB-তে `YARD_ACTIVITIES.OPERATOR_USER_ID` = 2।

### Test 8: Expired token

`application.yml`-এ temporarily:
```yaml
portops:
  security:
    jwt:
      access-token-expiry-minutes: 0   # immediate expiry
```

Restart, login, token copy, retry যেকোনো authenticated endpoint।

**Expected:** 401 Unauthorized (token expired)।

Expiry আবার 60-এ revert করুন।

### Test 9: Malformed token

```
GET http://localhost:8080/api/v1/containers
Authorization: Bearer thisIsNotAJwt
```

**Expected:** 401 Unauthorized, console-এ DEBUG log — "JWT validation failed"।

---

## Phase 4 Milestone Checklist

- [ ] `User` entity + `Role` enum + `USER_ROLES` join table auto-generated DB-তে
- [ ] Bootstrap users startup-এ create হয় (dev profile)
- [ ] `POST /api/v1/auth/login` valid credentials-এ JWT return করে
- [ ] Invalid credentials → 401 Unauthorized (ProblemDetail format)
- [ ] Protected endpoint no-token → 401
- [ ] Protected endpoint wrong role → 403
- [ ] `@PreAuthorize` method-level protection কাজ করে
- [ ] `@AuthenticationPrincipal UserPrincipal` inject হয় controller method-এ
- [ ] Gate-in করলে `YARD_ACTIVITIES.OPERATOR_USER_ID` populate হয়
- [ ] Expired token handle হয় (401)
- [ ] BCrypt hash DB-তে stored (60 chars starting with `$2a$` or `$2b$`)
- [ ] CORS headers React origin-এর জন্য return হয় (OPTIONS preflight test)

---

## Self-Check Questions

1. `UserDetails` interface কি — আমরা কেন `User` entity-র বদলে আলাদা `UserPrincipal` adapter বানালাম?
2. `ROLE_` prefix-এর গল্প — `hasRole("ADMIN")` আর `hasAuthority("ROLE_ADMIN")`-এ পার্থক্য কী?
3. JWT-র তিনটা part কী? Signature কেন tamper-proof?
4. JWT payload plaintext — তাহলে কেন এটা secure? কোন জিনিস payload-এ রাখা যাবে না?
5. Filter chain-এ JwtAuthenticationFilter কেন UsernamePasswordAuthenticationFilter-এর before বসানো হয়েছে?
6. 401 আর 403 status-এর conceptual পার্থক্য?
7. `CsrfConfigurer::disable` — কেন stateless REST API-তে CSRF protection দরকার নেই?
8. `SessionCreationPolicy.STATELESS` না set করলে কী হবে?
9. `getReferenceById` আর `findById`-এর পার্থক্য — কখন কোনটা ব্যবহার করবেন?
10. `DaoAuthenticationProvider` কীভাবে username/password verify করে?
11. `@PreAuthorize` vs `requestMatchers(...).hasRole(...)` — কখন কোনটা ভালো?
12. BCrypt strength parameter 12 এর মানে — আরো বাড়ালে/কমালে কী হয়?

---

## Troubleshooting

### Problem: সব endpoint-এ 401, login-ও
**কারণ:** `/api/v1/auth/**` permitAll হয়নি, অথবা filter chain ordering ভুল।
**সমাধান:** SecurityConfig-এ `requestMatchers("/api/v1/auth/**").permitAll()` verify করুন। Order matters — specific rules আগে, general rules পরে।

### Problem: Login success কিন্তু subsequent request 401
**Checklist:**
1. Postman Header-এ `Authorization: Bearer <token>` format ঠিক? `Bearer` আর token-এর মধ্যে space?
2. Token expire হয়নি তো?
3. Console log-এ "JWT validation failed" আসছে?

### Problem: `BadCredentialsException` throw হচ্ছে কিন্তু ProblemDetail JSON আসছে না
**কারণ:** Exception filter chain-এর মধ্যে, `@RestControllerAdvice` reach করে না।
**সমাধান:** `AuthService.login()`-এ BadCredentialsException catch করে `InvalidCredentialsException` throw করছেন তা verify করুন। Controller-level exception handler filter chain-এর বাইরে।

### Problem: Bootstrap user create হচ্ছে না
**Checklist:**
1. Profile `dev` active? `application.yml`-এ `spring.profiles.active: dev`?
2. `@Component` annotated?
3. Startup log-এ "Bootstrap users" message আসছে?

### Problem: `io.jsonwebtoken.security.WeakKeyException`
**কারণ:** JWT secret 256 bits (32 bytes)-এর কম।
**সমাধান:** Base64-decoded secret minimum 32 bytes হতে হবে। `openssl rand -base64 48` দিয়ে generate করুন।

### Problem: `ClassCastException: principal is String, not UserPrincipal`
**কারণ:** কোনো endpoint-এ authentication set হয়নি, কিন্তু controller `@AuthenticationPrincipal UserPrincipal` expect করছে।
**সমাধান:** সেই endpoint authenticated কিনা verify করুন, অথবা parameter `Object` type দিয়ে instanceof check করুন।

### Problem: React থেকে CORS error
**Checklist:**
1. React-এর exact origin (e.g., `http://localhost:3000`) CORS allowedOrigins list-এ আছে?
2. `allowCredentials(true)` এবং React-এ `credentials: 'include'`?
3. Browser DevTools → Network → preflight OPTIONS request কি 200 return করছে?

### Problem: `@PreAuthorize` ignore হচ্ছে
**কারণ:** `@EnableMethodSecurity` নেই SecurityConfig-এ।
**সমাধান:** Add `@EnableMethodSecurity` annotation।

### Problem: Password log-এ print হয়ে যাচ্ছে
**কারণ:** `LoginRequest.toString()` Lombok/record auto-generated, password include করে।
**সমাধান:** Log-এ কখনো `loginRequest` পুরোটা print করবেন না — শুধু `loginRequest.username()`।

---

## Security Best Practices (Phase 4-এর বাইরে, কিন্তু জানা জরুরি)

### ১. Refresh token pattern
Access token short-lived (15-60 min)। Refresh token long-lived (days)। Separate endpoint `POST /auth/refresh`। Phase 4-এ skip করলাম complexity-র জন্য, Week 2-এ add করবেন।

### ২. Token storage in React
- **httpOnly cookie:** Secure, XSS-safe, কিন্তু CSRF protection লাগবে
- **localStorage:** Simple, কিন্তু XSS-এ vulnerable

Enterprise default: httpOnly cookie + CSRF token। Banking app-এ mandatory।

### ৩. Rate limiting on login endpoint
Brute-force attack prevent করতে। Phase 7-এ Bucket4j / Resilience4j দিয়ে।

### ৪. Account lockout
N failed attempts → account lock, admin reset। User entity-তে ইতিমধ্যে `accountNonLocked` আছে — logic implement করা বাকি।

### ৫. Password complexity + breach check
HaveIBeenPwned API integration, regex beyond minimum length।

### ৬. JWT secret rotation
Production-এ periodically key rotate করতে হয়। Two active keys — old + new, grace period-এ দুটোই accept।

### ৭. HTTPS only
Production-এ JWT শুধু HTTPS-এ যাবে। Spring Security: `.requiresChannel(c -> c.anyRequest().requiresSecure())`।

---

## Phase 4 Wrap-up

এখন আপনার API **production-grade authenticated**। React team এই spec এ consume করতে পারবে:

1. `POST /api/v1/auth/login` → accessToken
2. All subsequent requests: `Authorization: Bearer <token>` header
3. Role-specific endpoints — 403 if wrong role
4. Expired token → 401, React login screen-এ redirect

### Phase 5-এ যাব

- `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest` — তিনটা test slice
- Mockito দিয়ে service layer unit test
- `MockMvc` দিয়ে controller test (security সহ — `@WithMockUser`)
- OpenAPI/Swagger integration — React team self-documenting spec পাবে
- `@Scheduled` — nightly billing job skeleton
- `@Async` — notification fire-and-forget
- Spring Boot Actuator — health/metrics endpoints
- Final code review — সব phase-এর build-up check

### আমাকে যা পাঠাবেন

1. **"Phase 4 done"** confirmation
2. **Postman screenshot** — Test 5 (403 Forbidden wrong role) + Test 7 (gate-in with operator identity, DB-তে OPERATOR_USER_ID populated)
3. **Decoded JWT** — jwt.io-তে পেস্ট করে payload screenshot (sensitive data leak check)
4. **Console log** — একটা successful login + একটা failed login-এর log lines
5. **Your thought** — আপনার banking eKYC app-এ যে auth pattern ছিল (OAuth, JWT, session?), Spring-এর approach-এর সাথে মূল পার্থক্য কী মনে হচ্ছে?

Phase 4-এর পর আপনার app-এ আর কোনো fundamental piece missing নেই। Phase 5-এ আমরা polishing করব — tests, documentation, production concerns।

শুরু করুন। আটকালে জানান।
