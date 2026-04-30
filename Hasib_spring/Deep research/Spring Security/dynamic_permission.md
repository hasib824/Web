# RBAC + Per-User Permission Override Plan

## Context

Current Spring Boot app (`E:\All Web\security-system\security`) has `Users`, `Rules`, `Permissions` entities and join tables, but the RBAC layer is **not wired**: `CustomeUSerDetails.getAuthorities()` returns `List.of()`, JWT carries no authorities, no `@PreAuthorize` exists. Tables are dead weight.

User requirement: each user belongs to one or more **Rules** (roles). Each Rule has a set of **Permissions**. The system must let an admin:
- grant an EXTRA permission to a SPECIFIC user (additive)
- DENY a permission for a SPECIFIC user that comes from their Rule (subtractive)
- without mutating the Rule itself, so other users sharing the same Rule are unaffected

Enterprise pattern (AWS IAM, Azure RBAC, Keycloak, Okta) for this is identical: **`effective = (∪ rule.permissions) ∪ user_grants − user_denies`**, with **DENY taking precedence**. We implement that natively in Spring Security — no Casbin/OPA.

Decisions confirmed by user:
- **Permission source per request:** always DB lookup (with Caffeine cache); enables instant revocation.
- **Override schema:** single `UserPermissionOverride` table with `GRANT|DENY` enum + `unique(user_id, permission_id)`.
- **JWT expiry:** bump to 30 minutes.
- **Admin bootstrap:** manual SQL once, no seeder code.

---

## 1. Data model

### 1.1 Bug fixes on existing entities

`entity/Rules.java` — line 16-17: id has no `@GeneratedValue`. Add `@GeneratedValue(strategy = GenerationType.IDENTITY)`. Add `@Table(name = "rules")` and `unique = true` on `rulename`.

`entity/Permissions.java` — line 19: add `unique = true` on `permission` column.

`entity/Users.java` — keep `Set<Rules> rulesSet` EAGER (resolver runs in `@Transactional`, so EAGER is fine and avoids `LazyInitializationException`).

### 1.2 New entity `entity/UserPermissionOverride.java`

```java
@Entity
@Table(name = "user_permission_override",
       uniqueConstraints = @UniqueConstraint(
           name = "uk_user_permission",
           columnNames = {"user_id", "permission_id"}))
public class UserPermissionOverride {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id") Users user;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "permission_id") Permissions permission;

    @Enumerated(EnumType.STRING)
    @Column(name = "override_type", nullable = false, length = 8)
    OverrideType type;          // GRANT or DENY

    @Column(length = 500) String reason;
    @Column(name = "expires_at") Instant expiresAt;     // null = never
    @Column(name = "created_at", nullable = false, updatable = false) Instant createdAt;
    @Column(name = "created_by") Long createdBy;
}
```

### 1.3 New enum `entity/OverrideType.java`

```java
public enum OverrideType { GRANT, DENY }
```

---

## 2. Repositories

Create under new package `repository/`:

- `RulesRepository extends JpaRepository<Rules, Long>` — `Optional<Rules> findByRulename(String)`
- `PermissionsRepository extends JpaRepository<Permissions, Long>` — `Optional<Permissions> findByPermission(String)`
- `UserPermissionOverrideRepository extends JpaRepository<UserPermissionOverride, Long>`:
  ```java
  @Query("select o from UserPermissionOverride o " +
         "where o.user.id = :userId " +
         "and (o.expiresAt is null or o.expiresAt > :now)")
  List<UserPermissionOverride> findActiveByUserId(@Param("userId") Long userId,
                                                  @Param("now") Instant now);
  Optional<UserPermissionOverride> findByUserIdAndPermissionId(Long userId, Long permId);
  ```

Move existing `UserRepository.java` into this package for consistency (optional cleanup).

---

## 3. Effective-permission resolution

New service `service/PermissionResolverService.java`:

```java
@Service @RequiredArgsConstructor
public class PermissionResolverService {
    private final UserPermissionOverrideRepository overrideRepo;

    @Cacheable(value = "effectivePermissions", key = "#user.id")
    @Transactional(readOnly = true)
    public Set<String> resolveEffectivePermissions(Users user) {
        Set<String> base = user.getRulesSet().stream()
            .flatMap(r -> r.getPermissions().stream())
            .map(Permissions::getPermission)
            .collect(Collectors.toCollection(HashSet::new));

        List<UserPermissionOverride> overrides =
            overrideRepo.findActiveByUserId(user.getId(), Instant.now());

        Set<String> grants = overrides.stream()
            .filter(o -> o.getType() == OverrideType.GRANT)
            .map(o -> o.getPermission().getPermission()).collect(toSet());
        Set<String> denies = overrides.stream()
            .filter(o -> o.getType() == OverrideType.DENY)
            .map(o -> o.getPermission().getPermission()).collect(toSet());

        base.addAll(grants);
        base.removeAll(denies);   // DENY wins
        return base;
    }

    // For admin "list with source" endpoint:
    public List<EffectivePermissionView> resolveWithSource(Users user) { ... }
}
```

**Caching:** `config/CacheConfig.java` registers Caffeine `effectivePermissions` cache with TTL 90s. Every admin write (`assignRule`, `removeRule`, `addOverride`, `removeOverride`, `updateRulePermissions`) calls `@CacheEvict(value="effectivePermissions", key="#userId")` — for rule-permission mutations, evict for all users carrying that rule (use `allEntries=true` since rules are few).

---

## 4. Authorities & JWT — DB-lookup mode (per user choice)

Authorities are resolved on **every** request, not embedded in the JWT. JWT only carries `subject=username` (existing behavior). Cache absorbs the load.

### 4.1 `CustomeUSerDetails.java` changes

Store and return real authorities:

```java
public class CustomeUSerDetails implements UserDetails {
    private final Users users;
    private final Collection<? extends GrantedAuthority> authorities;

    public CustomeUSerDetails(Users users, Collection<? extends GrantedAuthority> authorities) {
        this.users = users;
        this.authorities = authorities;
    }
    @Override public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }
    // rest unchanged
}
```

### 4.2 `CustomeUserDetailsService.java` changes

Inject resolver, build authorities, pass to `CustomeUSerDetails`:

```java
private final UserRepository userRepository;
private final PermissionResolverService permissionResolver;

@Override @Transactional(readOnly = true)
public UserDetails loadUserByUsername(String username) {
    Users u = userRepository.findByUsername(username);
    if (u == null) throw new UsernameNotFoundException("User not exists");
    Set<String> perms = permissionResolver.resolveEffectivePermissions(u);
    var authorities = perms.stream().map(SimpleGrantedAuthority::new).toList();
    return new CustomeUSerDetails(u, authorities);
}
```

### 4.3 `JWTAuthenTicationFilter.java` changes (lines 47-59)

Already calls `customeUserDetailsService.loadUserByUsername(...)` — that now returns authority-laden `UserDetails`. Just propagate them:

```java
UserDetails userDetails = customeUserDetailsService.loadUserByUsername(userName);
UsernamePasswordAuthenticationToken authToken =
    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
SecurityContextHolder.getContext().setAuthentication(authToken);
```

That's the only change in this file.

### 4.4 `JWTUtil.java` — no structural change

Token continues to carry only `subject`. No `permissions` claim needed (DB-lookup mode).

### 4.5 `AuthService.login(...)` — no change to token issuance

`jwtUtil.generateToken(username)` stays as-is.

---

## 5. Authorization enforcement

### 5.1 Convention

Authority strings = bare permission names: `USER_READ`, `USER_MANAGE_ROLES`, `USER_MANAGE_PERMISSIONS`, `RULE_MANAGE`, `PERMISSION_MANAGE`, `VESSEL_READ`, etc. **No `ROLE_` prefix.** Use `hasAuthority(...)` not `hasRole(...)`.

### 5.2 `SecurityConfig.java` changes

- Add `.sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))` — currently missing, important for JWT.
- Add URL-level rule for admin: `.requestMatchers("/api/admin/**").hasAuthority("USER_MANAGE_PERMISSIONS")` (defense in depth alongside method security).

### 5.3 Controllers

Annotate protected handlers, e.g.:
- `ShowVesselController.showVessels()` → `@PreAuthorize("hasAuthority('VESSEL_READ')")`
- Admin handlers each with their permission.

`@EnableMethodSecurity` already on `SecurityConfig` (line 25).

---

## 6. Admin APIs

New package `admin/`:

### 6.1 `admin/RuleAssignmentController.java`

| Method | Path | Authority | Body |
|---|---|---|---|
| POST | `/api/admin/users/{userId}/rules/{ruleId}` | `USER_MANAGE_ROLES` | — |
| DELETE | `/api/admin/users/{userId}/rules/{ruleId}` | `USER_MANAGE_ROLES` | — |
| GET | `/api/admin/users/{userId}/rules` | `USER_READ` | — |

### 6.2 `admin/UserPermissionOverrideController.java`

| Method | Path | Authority | Body |
|---|---|---|---|
| POST | `/api/admin/users/{userId}/permissions/grant` | `USER_MANAGE_PERMISSIONS` | `{permissionId, reason, expiresAt?}` |
| POST | `/api/admin/users/{userId}/permissions/deny` | `USER_MANAGE_PERMISSIONS` | same |
| DELETE | `/api/admin/overrides/{overrideId}` | `USER_MANAGE_PERMISSIONS` | — |
| GET | `/api/admin/users/{userId}/permissions/effective` | `USER_READ` | returns `[{permission, source: RULE\|GRANT\|DENY, ruleName?}]` |
| GET | `/api/admin/users/{userId}/overrides` | `USER_READ` | raw override list |

Each write: `@CacheEvict("effectivePermissions", key="#userId")`. Behavior on conflict: if an override row already exists for `(userId, permissionId)`, switch its `type` (GRANT↔DENY) via UPDATE — uphold the unique constraint.

### 6.3 `admin/RulePermissionController.java`

CRUD for rules + permissions + the `role_permission` link. Guarded by `RULE_MANAGE` / `PERMISSION_MANAGE`. Needed to bootstrap the system from SQL-seeded super-admin.

### 6.4 Self-protection guard

In `UserPermissionOverrideController`, reject any request where `userId == authenticated principal id` AND permission == `USER_MANAGE_PERMISSIONS` — prevents lock-out.

DTOs: `dto/admin/CreateOverrideRequest.java`, `dto/admin/EffectivePermissionView.java` (record).

Errors reuse existing `exception/` package style (extend `DuplicateResourceException` pattern; add `NotFoundException` if needed).

---

## 7. application.yml changes

```yaml
jwt:
  secret: ${JWT_SECRET:YourVerySecretKeyMustBeAtLeast256BitsLongForHS256Algorithm12345}
  expiration: 1800000   # 30 minutes
```

Move secret to env with current value as dev fallback.

---

## 8. Bootstrap (manual SQL — per user choice)

After first run (Hibernate creates tables via `ddl-auto: update`), run once against `security-db`:

```sql
-- permissions
INSERT INTO permissions (permission) VALUES
 ('USER_READ'), ('USER_MANAGE_ROLES'), ('USER_MANAGE_PERMISSIONS'),
 ('RULE_MANAGE'), ('PERMISSION_MANAGE'), ('VESSEL_READ');

-- rules
INSERT INTO rules (rulename) VALUES ('SUPER_ADMIN'), ('USER');

-- super_admin gets all
INSERT INTO role_permission (rule_id, permission_id)
  SELECT (SELECT id FROM rules WHERE rulename='SUPER_ADMIN'), id FROM permissions;

-- USER gets baseline
INSERT INTO role_permission (rule_id, permission_id)
  SELECT (SELECT id FROM rules WHERE rulename='USER'),
         id FROM permissions WHERE permission IN ('USER_READ','VESSEL_READ');

-- bootstrap admin user (BCrypt hash of chosen password — generate offline)
INSERT INTO users (username, email, password)
  VALUES ('admin', 'admin@local', '<<bcrypt-hash>>');

INSERT INTO user_rules (user_id, role_id)
  SELECT (SELECT id FROM users WHERE username='admin'),
         (SELECT id FROM rules WHERE rulename='SUPER_ADMIN');
```

Document this snippet in repo `README.md` (or new `docs/bootstrap.sql`).

---

## 9. Files to create / modify

### Create

Under `E:\All Web\security-system\security\src\main\java\com\security_backend\security\`:

- `entity/UserPermissionOverride.java`
- `entity/OverrideType.java`
- `repository/RulesRepository.java`
- `repository/PermissionsRepository.java`
- `repository/UserPermissionOverrideRepository.java`
- `service/PermissionResolverService.java`
- `dto/admin/CreateOverrideRequest.java`
- `dto/admin/EffectivePermissionView.java`
- `admin/RuleAssignmentController.java`
- `admin/UserPermissionOverrideController.java`
- `admin/RulePermissionController.java`
- `config/CacheConfig.java`
- `exception/NotFoundException.java` (if not already covered)

Plus dependency in `pom.xml`: `com.github.ben-manes.caffeine:caffeine` and `spring-boot-starter-cache`.

Plus `docs/bootstrap.sql` in project root.

### Modify

- `entity/Rules.java` — add `@GeneratedValue` on id; `unique=true` on `rulename`; explicit `@Table(name="rules")`
- `entity/Permissions.java` — `unique=true` on `permission`
- `CustomeUSerDetails.java` — accept and return authorities
- `CustomeUserDetailsService.java` — inject resolver, populate authorities
- `JWTAuthenTicationFilter.java` (lines 47-59) — propagate `userDetails.getAuthorities()` into the auth token
- `SecurityConfig.java` — add `STATELESS`, `/api/admin/**` matcher, `@EnableCaching`
- `SecurityApplication.java` — add `@EnableCaching` if not on `SecurityConfig`
- `application.yml` — JWT expiry to 30 min, secret via env
- `vessels/ShowVesselController.java` — add `@PreAuthorize("hasAuthority('VESSEL_READ')")`

### Delete

- Stray duplicate imports in `SecurityConfig.java` (lines 17-21 are duplicates of 7-14).

---

## 10. Audit & safety

- `UserPermissionOverride` row itself is the audit record (`created_at`, `created_by`, `reason`).
- `expires_at` supports temporary grants/denies; resolver query already filters expired.
- Conflict prevention via `unique(user_id, permission_id)` — admin write switches `type` instead of inserting a second row.
- Self-lockout guard in admin controller.
- Cache TTL 90s + explicit eviction → revocation propagates within ~1 cache window even without eviction; immediate with eviction.

---

## 11. Verification

### Manual flow (Postman / curl)

1. Run app once → tables created via `ddl-auto: update`.
2. Run `docs/bootstrap.sql` against SQL Server — creates permissions, rules, `admin` user.
3. `POST /api/login` as `admin` → capture token.
4. `POST /api/register` `{username:"alice",email,password}` → 200.
5. `POST /api/admin/users/{aliceId}/rules/{userRuleId}` (admin token) → 200.
6. Login as alice → `GET /api/get-vessels` → **200** (USER rule has `VESSEL_READ`).
7. As admin: `POST /api/admin/users/{aliceId}/permissions/deny` `{permissionId: VESSEL_READ.id, reason:"trial"}`.
8. Alice's next call (after ≤90s cache TTL or immediate eviction) `GET /api/get-vessels` → **403** — proves DENY beats rule-granted permission.
9. As admin: `DELETE /api/admin/overrides/{overrideId}` → alice's call → **200** (back to baseline).
10. As admin: `POST /api/admin/users/{aliceId}/permissions/grant` `{permissionId: USER_MANAGE_ROLES.id}` → alice gains a permission her rule lacks.
11. `GET /api/admin/users/{aliceId}/permissions/effective` after each step — verify `source` is `RULE` / `GRANT` / `DENY` correctly.
12. Negative isolation: register `bob`, assign same `USER` rule, no overrides — bob's `GET /api/get-vessels` stays **200** through every alice override → proves overrides are per-user only.

### Optional unit/integration tests (later)

- `PermissionResolverServiceTest` — mocks override repo; covers union, deny precedence, expiry filtering.
- `UserPermissionOverrideControllerIT` — `@SpringBootTest` walking the 12-step flow.
- `JWTAuthenTicationFilterTest` — round-trip with authorities propagating from `UserDetails`.

---

## Critical files (quick index)

- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\CustomeUserDetailsService.java`
- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\CustomeUSerDetails.java`
- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\JWTAuthenTicationFilter.java`
- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\SecurityConfig.java`
- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\entity\Rules.java`
- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\entity\Permissions.java`
- `E:\All Web\security-system\security\src\main\java\com\security_backend\security\entity\Users.java`
- `E:\All Web\security-system\security\src\main\resources\application.yml`
