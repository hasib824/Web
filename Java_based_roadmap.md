# Android → Full-Stack Web: A Leverage-First Roadmap

**For:** Senior Android engineer (5+ yr Java/Kotlin, Compose, Clean Arch, Hilt, Coroutines, Room, Retrofit, Gradle, KMP-comfortable)
**Target stack:** Java 21 + Spring Boot 3.x • React + TypeScript • Oracle Database
**Pace:** Aggressive — 3–4 months at 15–20 hr/week (~240–320 hr total budget)
**Philosophy:** You are not a beginner. This roadmap is built to exploit what you already know, warn you about concepts that *look* similar but aren't, and skip the 40% of every standard tutorial that wastes your time.

---

## Table of Contents

1. [TL;DR & Calendar-at-a-Glance](#tldr--calendar-at-a-glance)
2. [How to Read This Document](#how-to-read-this-document)
3. [Leverage Audit](#leverage-audit)
4. [Phase 0 — Java 21 Catch-Up for Java Veterans](#phase-0--java-21-catch-up-for-java-veterans)
5. [Phase 1 — Spring Boot Core](#phase-1--spring-boot-core)
6. [Phase 2 — Oracle DB + JPA/Hibernate](#phase-2--oracle-db--jpahibernate)
7. [Phase 3 — TypeScript + React](#phase-3--typescript--react)
8. [Phase 4 — Integration, Docker, CI/CD, Observability](#phase-4--integration-docker-cicd-observability)
9. [Capstone Project — HabitForge](#capstone-project--habitforge)
10. [Anti-Patterns: Android Devs Moving to Web](#anti-patterns-android-devs-moving-to-web)
11. [Learning Techniques](#learning-techniques)
12. [Appendix A — Cheat Sheet (Android ↔ Spring ↔ React)](#appendix-a--cheat-sheet)
13. [Appendix B — Resource Index](#appendix-b--resource-index)
14. [Appendix C — Interview-Readiness Checklist](#appendix-c--interview-readiness-checklist)

---

## TL;DR & Calendar-at-a-Glance

You will reach three milestones:

| Milestone | When | What it means |
|---|---|---|
| **Junior-ready** | End of Week 6 | Can ship a CRUD REST API with Spring Boot + Oracle, pass a code-screen |
| **Mid-level productive** | End of Week 12 | Can build an end-to-end feature (API + UI + DB) unsupervised |
| **Production-confident** | End of Week 16 | Can own a service: security, observability, deploy, debug prod issues |

### 16-Week Calendar

| Week | Primary Focus | Parallel Track | Capstone Milestone |
|---|---|---|---|
| 1 | Phase 0 — Java 21 refresh | — | — |
| 2 | Phase 1 — Spring Boot (HTTP, controllers) | — | Capstone scaffold repo |
| 3 | Phase 1 — DI, layering, validation | Capstone: user CRUD API | Users + auth skeleton |
| 4 | Phase 1 — Spring Data JPA basics | Capstone: habits CRUD | Habits domain |
| 5 | Phase 1 — Security (JWT) | **Phase 2 starts** — Oracle fundamentals | JWT auth working |
| 6 | Phase 1 — Testing, Actuator, profiles | Phase 2 — PL/SQL read-level, indexes | Integration tests green |
| 7 | Phase 2 — Oracle data types, sequences, execution plans | — | Oracle migration from H2 |
| 8 | Phase 2 — HikariCP tuning, N+1 diagnosis | **Phase 3 starts** — TS fundamentals | Reporting endpoint with native query |
| 9 | Phase 3 — React core, hooks | — | Login + habit list screen |
| 10 | Phase 3 — Routing, TanStack Query | — | Full habit CRUD UI |
| 11 | Phase 3 — Forms, Zustand, Tailwind | — | Check-in + streak view |
| 12 | Phase 3 — Testing (Vitest + RTL + MSW) | — | Frontend tests green |
| 13 | Phase 4 — Docker Compose (Oracle XE + app + nginx) | — | Runs on `docker compose up` |
| 14 | Phase 4 — CORS, auth flow end-to-end, env config | — | Deployed to a VPS or Fly.io |
| 15 | Phase 4 — CI (GitHub Actions), observability | — | Pipeline green, metrics visible |
| 16 | Polish, docs, README, interview prep | — | Capstone shipped + blog post |

**Weekly rhythm:** 3 weekday evenings × 2 hr + one 6–9 hr weekend block = 12–15 hr core + 3–5 hr spillover.

---

## How to Read This Document

- **Do not linear-read.** Skim the leverage audit, then jump into Phase 0. Come back to appendices when stuck.
- **Every phase has a `Skip this` section.** Trust it. The biggest time-waste for experienced devs is re-learning things because a tutorial author assumed you didn't.
- **Trap warnings are non-negotiable.** When you see a `⚠️ Trap` block, stop and re-read. These are the bugs that will cost you days if you miss them.
- **Exit criteria are binary.** If you can't do what the exit criteria say, don't move on. The roadmap compounds — skipping ahead with gaps breaks Phase 4.
- **The capstone runs in parallel** with phases. Don't wait until the end to start building; build the relevant slice *during* each phase.

---

## Leverage Audit

Your Android background gives you an asymmetric advantage. Here's the honest breakdown:

### What Transfers Directly (≈ 40% of the new stack)

| Concept | Why it transfers |
|---|---|
| **Layered architecture** | Controller→Service→Repository is Clean Architecture with different names |
| **Dependency injection** | Spring IoC and Hilt solve the same problem; only mechanics differ |
| **Build tools** | Gradle Kotlin DSL works fine for Spring; Maven is just different XML |
| **REST contract thinking** | You've consumed hundreds of APIs via Retrofit — serving them is the flip side |
| **Reactive streams mental model** | Flow → Reactor Flux is a syntax change, not a paradigm change |
| **Type systems** | Kotlin's null-safety and generics translate to TypeScript faster than JS devs can learn it |
| **Immutability** | `data class` muscle memory → Java 21 `record`, TS readonly |
| **Sealed hierarchies** | Kotlin `sealed class` → TS discriminated unions (less ergonomic, same idea) |
| **Coroutines discipline** | Scope/cancellation thinking maps to effect cleanup in React and virtual thread lifecycle in Java 21 |
| **Testing mindset** | JUnit 5 is JUnit 4 evolved; you already know this |
| **Version control, tooling ergonomics** | Unchanged |

### What Looks Similar But Isn't (the traps — flagged in every phase)

| Looks like | But differs because |
|---|---|
| `@Autowired` vs `@Inject` | Spring scans classpath at runtime; Hilt generates code at compile. Debugging is different |
| Room vs JPA | JPA lazy-loads by default; Room eagerly fetches. **N+1 queries are a new failure mode** |
| Compose recomposition vs React re-render | Compose is smart-skippable by default; React re-renders the whole function subtree unless you memoize |
| `LaunchedEffect` vs `useEffect` | useEffect's deps array has stale-closure footguns Compose doesn't |
| Android lifecycle vs HTTP request lifecycle | Android: long-lived. HTTP: stateless per request. State must go somewhere explicit |
| OkHttp interceptor vs Spring filter | Filter chain order is load-bearing; filters see the raw servlet request |
| Kotlin null-safety vs TS strict null checks | Similar ergonomics; TS `any` is a trapdoor that `Any?` is not |
| Retrofit interface vs Spring `@RestController` | Retrofit is a client DSL; controller is a dispatcher. Don't treat the annotations symmetrically |

### What's Net-New (≈ 20% — spend real time here)

- **HTTP semantics as a primary concern** (status codes, verbs, headers, CORS, caching, idempotency)
- **Server-side security thinking** (auth/authz, CSRF, session vs token, injection vectors)
- **Browser runtime constraints** (JS event loop, DOM, hydration, bundle size)
- **SQL execution planning** (you never optimized a join in Room)
- **Database connection pooling**
- **Deployment and ops basics** (Docker, reverse proxies, env config, observability)

---

## Phase 0 — Java 21 Catch-Up for Java Veterans

**Goal:** Catch up on 13+ years of Java evolution (Java 8 → 21 LTS). Write idiomatic modern Java using records, pattern matching, virtual threads — not Java 8 in new syntax.

**Hours:** 10–12 hr • **Calendar:** Week 1

**Your context:** Long-time Java dev (pre-Kotlin, likely Java 6/7/8 era for Android). Kotlin only in the last year. You already know OOP, generics, collections, lambdas, try-with-resources, `CompletableFuture`. What you're missing is **what landed in Java 9–21** — that's the focus. No beginner recap. No "Kotlin → Java translation" angle.

### Core Concepts — What's New Since Java 8

Java shipped 13 releases between 8 and 21. These are the ones that matter:

**Syntax & language:**
- **`var`** (Java 10) — local type inference. Only locals, only when initialized
- **Text blocks** `"""..."""` (Java 15) — multiline strings, preserves relative indentation
- **Switch expressions** (Java 14) — `switch` as an expression with `->` arms, no fallthrough, exhaustiveness for enums/sealed
- **Pattern matching for `instanceof`** (Java 16) — `if (o instanceof String s) { s.length(); }` — binding variable
- **Pattern matching for `switch`** (Java 21) — patterns in case labels, including record and type patterns
- **Records** (Java 16) — nominal tuples; final, equals/hashCode/toString generated. Can have custom methods, compact constructors
- **Sealed classes/interfaces** (Java 17) — `sealed ... permits A, B, C`. Pair with pattern matching for exhaustive `switch`
- **Enhanced `instanceof` + sealed + switch patterns** = data-oriented programming in Java. This is the paradigm shift since you last used Java

**Concurrency:**
- **Virtual threads** (Java 21) — Project Loom. `Thread.ofVirtual().start(...)` or `Executors.newVirtualThreadPerTaskExecutor()`. Cheap — millions possible. Spring Boot 3.2+ can run the whole request pool on virtual threads with one property
- **Structured concurrency** (Java 21, still preview in some releases) — `StructuredTaskScope`. Scoped cancellation, fork/join ergonomics
- **`CompletableFuture` updates** (since Java 9) — `completeOnTimeout`, `orTimeout`, `delayedExecutor`

**APIs & platform:**
- **`Optional<T>`** (Java 8) — you've seen it, but idiom has settled: return-type only, never field/param
- **Stream additions** — `takeWhile`, `dropWhile` (Java 9), `toList()` (Java 16), `mapMulti` (Java 16), `Collectors.teeing` (Java 12)
- **Factory methods** on `List`, `Set`, `Map` (Java 9) — `List.of(...)`, immutable
- **`HttpClient`** (Java 11) — stdlib replacement for OkHttp/Apache HttpClient
- **Java module system / JPMS** (Java 9) — **read-level only**. Spring Boot ignores it; you won't write `module-info.java`

**Build tooling (what changed in the ecosystem):**
- **Gradle with Kotlin DSL** — you know this from Android
- **Maven** — still dominates Spring world. Learn the lifecycle: `validate → compile → test → package → verify → install → deploy`
- **JDK distributions** — you probably used Oracle JDK 8 last. Now: **Eclipse Temurin** (default), Amazon Corretto, Azul Zulu. All free. Pick Temurin
- **jshell** (Java 9) — REPL. Useful for quick experiments

### Java 8 (what you knew) → Java 21 (what's new)

| Used to write (Java 8 / older) | Modern Java 21 |
|---|---|
| `public class User { private String name; /* getters, equals, hashCode, toString */ }` | `record User(String name) {}` |
| `if (o instanceof String) { String s = (String) o; ... }` | `if (o instanceof String s) { ... }` |
| Big `if/else if` tree on type | `switch` expression with type patterns |
| `String s = "line1\n" + "line2\n";` | Text block `"""line1\nline2\n"""` |
| `Map<String, List<User>> m = new HashMap<>();` | `var m = new HashMap<String, List<User>>();` |
| Executor with 200 platform threads for IO | Virtual thread executor — unbounded, cheap |
| Abstract class + known subclass hierarchy comment | `sealed interface Shape permits Circle, Square` |
| Nested factory + constructor chains | Record + compact constructor |
| `Collectors.toUnmodifiableList()` or `.collect(toList())` | `.toList()` (Java 16) — returns unmodifiable |
| `Optional.ofNullable(x).orElse(default)` | Same, but never put `Optional` in fields |

### ⚠️ Traps

> **Trap 1 — Using Java 8 idioms in Java 21 code.** You'll reach for verbose POJOs, anonymous inner classes, `StringBuilder` concatenation, etc. Spring 3 codebases use records/pattern matching heavily — if you write Java 8 style, code reviews will ping you. Deliberately use the new features.

> **Trap 2 — `var` overuse.** `var` is locals-only AND only when the initializer makes the type obvious. `var result = service.process();` hides the type unnecessarily. Use `var` for long generics (`Map<String, List<Order>>`) and builders; prefer explicit types for opaque returns.

> **Trap 3 — Pattern matching exhaustiveness is only for sealed + enum.** Normal class hierarchy → compiler can't prove exhaustiveness → you still need a `default`. This trips up devs expecting Kotlin `when`-like behavior.

> **Trap 4 — Records are final and immutable.** No inheritance. No setters. If you need mutation, records aren't the answer — use a class. Hibernate has workarounds but prefer DTO records + mutable entity classes.

> **Trap 5 — Virtual threads ≠ coroutines.** No colored functions (no `suspend`), no structured cancellation by default (unless using `StructuredTaskScope`), pinning issues with `synchronized` blocks (use `ReentrantLock`). Read the pinning docs before heavy use.

> **Trap 6 — `Optional` is not Kotlin `?`.** It's heavier (allocates), opt-in, return-type idiom only. Don't wrap every nullable field.

> **Trap 7 — Streams single-use.** Consume once. `Stream.of(...).filter(...)` — after terminal op, stream is closed. Unlike Kotlin `Sequence` which can be re-iterated.

> **Trap 8 — `equals`/`hashCode` on records covers all components.** Including mutable ones if you stuff a `List` in. Be aware if you mix records with mutable fields.

### Hands-On (4–5 hr, most value here)

Pick **one**. Don't spend more than a weekend on this:

1. **Refactor old Java 8 code to Java 21.** Grab any Java 8 project from your history (or an open-source one like `google/guava` examples). Modernize: POJOs → records, `if/else` chains → switch patterns, `instanceof` casts → pattern variables, streams → `.toList()`. Feel the before/after.

2. **Java 21 HTTP mini-client.** Using `java.net.http.HttpClient` + virtual threads + records (for request/response), build a tool that hits 100 URLs concurrently and prints status/latency. Exercises: records, virtual threads, switch expressions on status-code ranges.

3. **JEP Cafe exercises.** Jose Paumard's "JEP Cafe" channel has practical problems solved with modern features. Code along to 3–4 episodes.

### Best Resources

**Priority 1 — Free, official, highest signal (6–8 hr):**

| Resource | Type | Time | Why |
|---|---|---|---|
| **[Inside Java](https://www.youtube.com/@java) YouTube** | Video | 3–4 hr | Nicolai Parlog + Jose Paumard. Official Oracle. Short 10–20 min videos per feature. Playlists: "Java Language Updates", "Virtual Threads" |
| **[dev.java](https://dev.java)** official tutorials | Docs | 2–3 hr | Oracle's modern learning site. Read: "Records", "Sealed Classes", "Pattern Matching", "Virtual Threads" sections |
| **[Baeldung — Java 9 to 21 feature guides](https://www.baeldung.com/java-9-features)** | Articles | 1–2 hr | Quick reference with runnable samples per release |
| **[nipafx.dev](https://nipafx.dev)** (Nicolai Parlog's blog) | Articles | 1 hr | Deep, opinionated, current. Search "Java 21", "pattern matching" |

**Priority 2 — Topic deep dives (watch on demand):**

- **Virtual Threads:** Ron Pressler's ["Virtual Threads: New Foundations for High-Scale Java"](https://www.youtube.com/results?search_query=Ron+Pressler+Virtual+Threads) (Devoxx talk, ~45 min) — the canonical explainer
- **Pattern Matching + Data-Oriented Programming:** Brian Goetz's ["Data-Oriented Programming in Java"](https://www.youtube.com/results?search_query=Brian+Goetz+data-oriented+programming) — paradigm shift from classic OOP
- **Records:** [JEP 395](https://openjdk.org/jeps/395) + Nicolai's "Java Records" 20-min video
- **Sealed classes:** [JEP 409](https://openjdk.org/jeps/409) + pattern matching for switch [JEP 441](https://openjdk.org/jeps/441) — read the motivation sections, skip the spec
- **Streams catch-up (Java 9–21 additions):** Jose Paumard's "JEP Cafe" episodes on `takeWhile`, `mapMulti`, `teeing`, `toList`

**Priority 3 — Books (optional, selective):**

- **"Modern Java in Action"** (Urma/Fusco/Mycroft) — read selectively:
  - **Skip:** Ch 1–3 (OOP/collections basics you know), Ch 17–18 (Reactive — Spring handles this differently)
  - **Read:** Ch 4–6 (Streams — catches up post-Java 8 additions), Ch 10 (Optional idioms), Ch 11 (CompletableFuture deep), Ch 15 (CompletableFuture patterns)
- **"Java By Comparison"** (Harrer/Lenhard/Dietrich) — short, pattern-per-page style. Good for shaking off Java 8 habits
- **Skip:** "Effective Java" 3rd ed *for this phase* — excellent book, but read it after you're productive in Spring, not before. It's a polish book, not a catch-up book
- **Skip:** any "Java 8 Functional Programming" book — you already have Kotlin-level fluency with lambdas/streams

**Priority 4 — Interactive practice (3–4 hr):**

- **[Exercism Java track](https://exercism.org/tracks/java)** — free, mentor-reviewed. Pick problems tagged "records", "streams", "Optional". 5–8 problems enough
- **[JetBrains Academy Java track](https://www.jetbrains.com/academy/)** — skip the intro modules; jump to "Java 17 features" project if you want graded checkpoints

### Skip This

- Any course/book titled "Java for Beginners" / "Java Fundamentals" / "OOP in Java" — you have decades on this
- Generics variance tutorials (`? extends T`, `? super T`) — you already know PECS
- Collections framework walkthroughs — unchanged fundamentals
- Lambda/Stream **introduction** tutorials — you got this in Kotlin and Java 8. Only catch up on what's **new since Java 8**
- JVM internals (classloaders, bytecode, GC tuning) — not needed until Phase 4 oncall territory
- JPMS (modules) — skim a one-pager. Spring Boot doesn't use it
- Swing / AWT / JavaFX — irrelevant
- JDBC raw API — JPA covers it
- Servlet API deep-dive — Spring abstracts
- Old `java.util.Date` / `Calendar` — you likely used `java.time` already since Java 8; don't re-learn legacy
- "Effective Java" (save for later — post-Spring)

### Exit Criteria

- [ ] Can read a Spring `@Service` using records, `var`, pattern-matching switch, sealed interfaces without reaching for docs
- [ ] Can refactor a Java 8 POJO class to a record + compact constructor + custom method
- [ ] Can explain virtual threads vs platform threads in 3 sentences, including the pinning gotcha
- [ ] Can write a `switch` expression with type patterns over a sealed hierarchy
- [ ] Can set up a Java 21 project with Gradle Kotlin DSL AND read/modify a Maven `pom.xml` without googling
- [ ] Know which JDK distribution to install (Temurin) and have it on PATH

---

## Phase 1 — Spring Boot Core

**Goal:** Ship production-quality Spring Boot REST APIs: DI, validation, persistence, security, testing, observability.

**Hours:** 70–80 hr • **Calendar:** Weeks 2–6

This is the biggest phase. Do not rush it. A weak Spring foundation makes everything after harder.

### Core Concepts (ordered)

**Week 2 — Foundations**
- Spring Boot project anatomy: `@SpringBootApplication`, auto-configuration, starters, the fat JAR
- IoC container, bean lifecycle, `@Component`/`@Service`/`@Repository`/`@Configuration`
- Constructor injection (always prefer), `@Qualifier`, `@Primary`
- `application.yml` + `@ConfigurationProperties`, profiles (`dev`/`test`/`prod`)
- Spring MVC: `@RestController`, `@RequestMapping` family, `@PathVariable`, `@RequestParam`, `@RequestBody`
- Jackson: serialization, `@JsonProperty`, `@JsonIgnore`, date/time handling (use `JavaTimeModule`)

**Week 3 — Request handling, validation, layering**
- Layering: Controller → Service → Repository (your Clean Arch translates directly)
- DTOs vs entities — **never expose JPA entities in API responses**
- Bean Validation (`@Valid`, `@NotBlank`, `@Size`, custom validators)
- `@ControllerAdvice` + `@ExceptionHandler` for global error handling
- ResponseEntity, problem+json (RFC 7807), proper HTTP status discipline
- Request/response logging: filters vs interceptors vs AOP

**Week 4 — Persistence (Spring Data JPA intro, deep dive in Phase 2)**
- `@Entity`, `@Id`, `@GeneratedValue`, relationships (`@OneToMany`, `@ManyToOne`, `@ManyToMany`)
- JpaRepository interface hierarchy, derived queries, `@Query` (JPQL + native)
- Transactions: `@Transactional`, propagation, isolation, rollback rules
- Flyway migrations — start here, not Liquibase
- H2 for local dev → swap to Oracle in Phase 2

**Week 5 — Security**
- Spring Security 6 filter chain mental model
- Password hashing (BCrypt), `UserDetailsService`
- JWT resource server: issue + validate flow, `SecurityFilterChain` bean config
- Authorities vs roles vs scopes
- Method security (`@PreAuthorize`), CORS, CSRF (and when to disable for REST)
- OAuth2 concepts (authorization code flow, PKCE) — understand, don't implement from scratch

**Week 6 — Testing, observability, packaging**
- JUnit 5 + Mockito basics (you know this, but patterns differ on server)
- `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest` — slice tests
- MockMvc for controllers, `TestRestTemplate` for full HTTP
- **Testcontainers** (Oracle XE or PostgreSQL) — integration tests against real DB
- Actuator endpoints (health, metrics, info), hiding sensitive endpoints in prod
- Logging with SLF4J + Logback, MDC for request correlation IDs
- Building executable JARs, Dockerfile basics (full Docker in Phase 4)

### Android/Kotlin Analog

| Android concept | Spring Boot | Notes |
|---|---|---|
| Hilt `@Module` + `@Provides` | `@Configuration` + `@Bean` | Identical mental model |
| `@Inject` constructor | Constructor injection (annotation-less since Spring 4.3) | Prefer over field injection, always |
| Retrofit interface | `@RestController` class | Opposite side of the wire |
| OkHttp interceptor | `SecurityFilterChain` / `HandlerInterceptor` / `@Aspect` | Three different layers, pick by concern |
| ViewModel state | `@SessionScope` bean or external cache (Redis) | Usually you don't — HTTP is stateless, store in DB or token |
| Clean Arch Repository interface | Spring Data `JpaRepository` | Spring generates the impl |
| Android lifecycle | Request lifecycle + bean scopes | `@RequestScope` exists but use sparingly |
| Gradle `implementation()` | Maven `<dependency>` or Gradle (same syntax) | Use BOM (`spring-boot-dependencies`) to align versions |
| `Result<T>` sealed type | `ResponseEntity<T>` + exception handlers | Different contract; errors flow through exceptions |
| Moshi/Gson | Jackson | Spring's default; Gson/Moshi not idiomatic here |

### ⚠️ Traps

> **Trap 1 — Field injection tempts you.** `@Autowired` on a field looks clean. It's untestable and hides dependencies. Always constructor-inject. Lombok `@RequiredArgsConstructor` makes it zero-boilerplate.

> **Trap 2 — Leaking JPA entities to the API.** An entity with lazy associations + Jackson serialization = `LazyInitializationException` or accidental N+1 on every request. Always map to a DTO in the service layer.

> **Trap 3 — `@Transactional` on controller methods.** Don't. Put it on service methods. Transactions should bracket *business operations*, not HTTP handlers.

> **Trap 4 — Thinking a bean is per-request.** Default scope is singleton. Storing user-specific state in a field of an `@Service` bean is a cross-user data leak waiting to happen.

> **Trap 5 — `@Component` scan missing your class.** Beans in packages outside `@SpringBootApplication`'s base package won't be picked up. Symptom: `NoSuchBeanDefinitionException`.

> **Trap 6 — Security misconfiguration.** `csrf().disable()` without understanding why. `permitAll()` on the wrong matcher. Read the filter chain docs twice.

### Hands-On Project

**"QuoteVault"** — a quote-keeper API.

- `POST /users` (signup), `POST /login` (issue JWT)
- `GET /quotes`, `POST /quotes`, `PUT /quotes/{id}`, `DELETE /quotes/{id}` — user-scoped
- `GET /quotes/random?tag=motivation`
- Bean validation on all inputs
- Global exception handler returning problem+json
- Flyway migrations for schema
- 80%+ test coverage: slice tests + one full integration test with Testcontainers
- Actuator with a custom health indicator checking DB connectivity

Scope: ~30–40 hr. Build it alongside reading, not after.

### Best Resources

- **Book (primary): "Spring Start Here" — Laurențiu Spilcă.** Best modern intro. Read all of it; it's concise.
  - Skip: chapter on generic Java recap if any
  - Prioritize: Spring Context, AOP (skim), Spring Data, Spring Security
- **Book (secondary): "Spring Security in Action" — Laurențiu Spilcă.** Read chapters 1–9 and 11. Skip OAuth2 server implementation chapters; you're a *client/resource server*, not an auth server.
- **Official: Spring Reference Documentation** at `docs.spring.io/spring-boot/reference/`. Use as a lookup — not linear reading.
- **Baeldung** for targeted articles (e.g., "Spring @Transactional propagation"). Quality varies; cross-check with official docs.
- **YouTube: Dan Vega**, **Laur Spilca**, **Java Brains (Koushik)** — Dan Vega for current, Java Brains for explanations. Avoid tutorials older than 2023 (pre-Spring Boot 3 — breaking changes).
- **GitHub reading:** `spring-projects/spring-petclinic` — the canonical reference app. Read it cover to cover once.
- **Skip:** "Spring in Action" (dated); any Udemy course titled "Spring Boot from scratch" longer than 20 hr (bloat).

### Skip This

- Spring XML configuration (you'll never touch it)
- Spring Boot 2.x specifics (use 3.x — breaking changes around jakarta.* namespace)
- JSP, Thymeleaf, Mustache templating (you're serving JSON, not HTML)
- Manual servlet programming
- Spring Cloud, microservices patterns (premature — learn the monolith first)
- WebFlux / reactive stack (default to blocking Spring MVC — simpler and virtual threads close the throughput gap). Revisit only if a job demands it
- Deep AOP authoring (read-level is enough)

### Exit Criteria

- [ ] Can scaffold a Spring Boot 3.x project with Gradle Kotlin DSL in 5 min
- [ ] Can explain the filter chain order and where JWT validation fits
- [ ] Can write a slice test (`@WebMvcTest`) and an integration test (`@SpringBootTest` + Testcontainers)
- [ ] Can diagnose "why is my bean not being injected" in under 10 min
- [ ] Can design a REST API for a new feature with correct status codes and error responses without asking
- [ ] Have shipped QuoteVault with tests green in CI (local, for now)

---

## Phase 2 — Oracle DB + JPA/Hibernate

**Goal:** Treat Oracle as a first-class system, not a generic SQL dialect. Diagnose slow queries. Write Oracle-native SQL when JPA can't.

**Hours:** 40–50 hr • **Calendar:** Weeks 5–8 (overlapping Phase 1)

### Core Concepts

- Oracle architecture at read level: instance vs database, tablespaces, data files, redo logs (enough to talk to a DBA without embarrassment)
- Oracle editions: XE (free, 2 CPU / 2 GB) for dev; SE/EE for prod. Know the limits
- Oracle-specific data types: `VARCHAR2` (not `VARCHAR`), `NUMBER(p,s)`, `DATE` vs `TIMESTAMP`, `CLOB`, `BLOB`, `RAW`
- **Sequences** for IDs (no auto-increment pre-12c; `IDENTITY` columns since 12c — prefer sequences + triggers for portability)
- Identifier case: unquoted identifiers fold to uppercase. `users` table is actually `USERS`. Quoted identifiers preserve case and become a nightmare. **Don't quote.**
- Indexes: B-tree, bitmap (read-mostly), function-based, composite. Selectivity matters
- Execution plans: `EXPLAIN PLAN`, `DBMS_XPLAN`, reading cost/cardinality/operations
- Statistics and the cost-based optimizer — why plans change over time
- Hints (`/*+ INDEX(t idx_name) */`) — last resort, not first
- Bind variables vs literals — Oracle's shared pool and the parse-cost penalty of literal SQL
- PL/SQL read-level: procedures, functions, packages, cursors. Know enough to read a DBA's stored proc
- Views vs materialized views
- Connection pooling with **HikariCP** (Spring Boot default): pool size tuning, leak detection

### JPA / Hibernate Deep Dive (the real skill)

- Entity lifecycle: transient → managed → detached → removed
- First-level cache (persistence context) vs second-level cache (Ehcache, Caffeine)
- Fetch strategies: `LAZY` (default for `@OneToMany`, `@ManyToMany`) vs `EAGER` (default for `@ManyToOne`, `@OneToOne`). **Understand both before writing any relationship.**
- The N+1 problem — diagnose with Hibernate statistics + P6Spy
- JPQL vs native queries vs Criteria API — when to pick which
- Projections and DTO mapping (`interface`-based or constructor expressions)
- Entity graphs (`@EntityGraph`) to control fetching per query
- Pagination: `Pageable`, `Slice`, be aware of `countQuery` overhead
- `@Version` optimistic locking
- `flush()` timing, dirty checking
- Open Session In View anti-pattern — **disable it** (`spring.jpa.open-in-view=false`)

### Android/Kotlin Analog

| Room concept | JPA | Notes |
|---|---|---|
| `@Entity` | `@Entity` | Same annotation name; JPA is richer |
| `@Dao` with `@Query` | `JpaRepository` with `@Query` / derived methods | Spring generates the impl |
| `@Relation` | `@OneToMany` / `@ManyToOne` | JPA lazy-loads by default — Room doesn't |
| Type converters | `@Converter(autoApply=true)` | Same mechanism |
| Migrations (Room auto-migrate) | Flyway / Liquibase | Manual SQL files; Flyway is simpler |
| Single-threaded SQLite | Pooled connections to remote Oracle | Connection pool tuning matters |
| No joins needed on small data | Joins are *the* performance concern | Learn `EXPLAIN PLAN` |

### ⚠️ Traps

> **Trap 1 — Default EAGER on `@ManyToOne`.** JPA fetches the parent on every load. If parent has a `@ManyToOne` to grandparent, it cascades. Override to `LAZY` unless you have a reason.

> **Trap 2 — N+1 queries.** Iterating a `@OneToMany` collection issues one query per parent. Fix with `JOIN FETCH` in JPQL or `@EntityGraph`. Diagnose with `hibernate.generate_statistics=true` + a log check.

> **Trap 3 — `spring.jpa.open-in-view=true` (the default).** Transactions stay open across the whole HTTP request so lazy fields "just work" in JSON serialization. Performance disaster. Turn it off and map to DTOs in the service.

> **Trap 4 — Oracle DATE has no sub-second precision.** Use `TIMESTAMP`. `DATE` is *actually* a datetime but truncated to seconds.

> **Trap 5 — `NUMBER` without precision.** `NUMBER` alone is a floating-point abomination. Always `NUMBER(p,s)`.

> **Trap 6 — Empty string is NULL in Oracle.** `''` and `NULL` are equivalent. Kotlin habits of checking `s.isEmpty()` after a round-trip to Oracle will fail — it'll be null.

> **Trap 7 — Sequence caching and gaps.** Sequences cache values per session. Restart = gap. Don't rely on IDs being contiguous.

> **Trap 8 — Bind variable sniffing.** Oracle caches a plan for the first bind values it sees. If those are skewed, later queries use a bad plan. Symptom: "same query, wildly different performance". Fix: `CURSOR_SHARING`, adaptive cursor sharing, or rewrite.

> **Trap 9 — Default isolation.** Oracle is READ COMMITTED (Spring's default too). Don't assume SERIALIZABLE without setting it explicitly.

### Hands-On Project

Extend QuoteVault (from Phase 1):

1. Migrate from H2 to Oracle XE (run via Docker: `gvenzl/oracle-xe`)
2. Add a `quote_views` table tracking every GET on a quote (user_id, quote_id, viewed_at)
3. Add endpoint `GET /quotes/{id}/stats` returning total views, unique viewers, views-last-7-days — **written as a single native Oracle query using window functions**
4. Add pagination, sorting, filter-by-tag to `GET /quotes`
5. Deliberately introduce an N+1 on a list endpoint, then fix it with `@EntityGraph`. Document before/after query counts
6. Configure HikariCP pool size and log leak detection
7. Use Flyway for all schema changes

### Best Resources

- **Book: "Expert Oracle Database Architecture" — Tom Kyte (3rd ed).** The bible. Read selectively:
  - **Prioritize:** Ch 1 (Architecture overview), Ch 6 (Locking), Ch 7 (Concurrency and Multiversioning), Ch 9 (Data Types), Ch 11 (Indexes)
  - **Skim:** Ch 5 (Oracle Processes), Ch 13 (Partitioning)
  - **Skip for now:** Parallel execution, distributed transactions, RAC
- **Book: "Oracle SQL Developer" guide** from Oracle — enough to use SQL Developer as your IDE
- **Oracle Live SQL** (livesql.oracle.com) — practice queries without installing anything
- **Vlad Mihalcea's blog** (vladmihalcea.com) — **the** authority on Hibernate performance. Bookmark and read widely. His book *High-Performance Java Persistence* is worth buying if you want depth
- **Thoughts-on-Java / Thorben Janssen** — JPA tutorials with realistic examples
- **Oracle Docs:** "SQL Language Reference" for syntax; skim, don't read linearly
- **PL/SQL:** Oracle's "PL/SQL Language Reference" — chapters on block structure, procedures, exceptions. Stop there unless you're writing packages
- **Skip:** Any "Learn SQL from zero" course; Oracle certifications (OCA/OCP) — overkill and exam-focused

### Skip This

- Oracle RAC (clustering) — ops concern, not dev
- Oracle Data Guard, Exadata, GoldenGate — enterprise ops
- Deep PL/SQL authoring (triggers, complex packages) — read-level only for Spring work
- Oracle Forms/Reports — legacy, not in your target
- Materialized view refresh strategies (beyond "ON COMMIT" / "ON DEMAND" awareness)
- OracleJDBC driver internals
- Enterprise Manager (Cloud Control) — DBA tool

### Exit Criteria

- [ ] Can read an `EXPLAIN PLAN` and identify a full table scan vs index range scan
- [ ] Can diagnose an N+1 from application logs in under 5 min
- [ ] Can write a window function query (RANK, LAG, SUM OVER) in native SQL
- [ ] Can explain LAZY vs EAGER and justify your choice for a relationship
- [ ] Understand HikariCP pool sizing rule of thumb: `connections = ((core_count × 2) + effective_spindle_count)`
- [ ] Know when to drop to native query vs JPQL

---

## Phase 3 — TypeScript + React

**Goal:** Build production React UIs with TypeScript. Understand what makes React re-render, fetch data sanely, and ship accessible, tested components.

**Hours:** 60–70 hr • **Calendar:** Weeks 8–12

### Core Concepts

**Week 8 — TypeScript fundamentals (fast, thanks to Kotlin)**
- Type system: primitives, unions, intersections, literal types, `never`, `unknown` vs `any`
- Interfaces vs type aliases (use `type`; prefer intersection over extension)
- Generics + constraints (mirror of Kotlin)
- Discriminated unions = your `sealed class`
- Utility types: `Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, `Parameters`, `Awaited`
- `satisfies` operator (2022) — better than type annotations for config objects
- Narrowing and control flow analysis
- `tsconfig.json` — `strict: true`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`
- Module resolution (CommonJS vs ESM — you'll encounter both)

**Week 9 — React core + hooks**
- The mental model: UI is a function of state; re-render is not re-mount
- JSX is not HTML: `className`, `htmlFor`, `{expressions}`, closing tags
- Function components (class components — read-level only, you will see them in old code)
- Hooks: `useState`, `useEffect`, `useMemo`, `useCallback`, `useRef`, `useContext`, `useReducer`, `useId`, `useDeferredValue`, `useTransition`
- Rules of hooks (top-level only, function components only) — lint rule catches this
- Keys in lists (stable IDs, not array index)
- Children, render props, composition over inheritance
- Controlled vs uncontrolled inputs

**Week 10 — Routing + data fetching**
- **React Router v6+** — declarative routes, loaders/actions if using data routers, navigation hooks
- **TanStack Query** for server state — queries, mutations, cache invalidation, optimistic updates, pagination, infinite queries
- Why **not** useEffect for fetching — stale closures, race conditions, double-fetch in dev StrictMode
- HTTP client: `fetch` + a small wrapper, or Axios (either is fine; `fetch` has fewer deps)
- Error boundaries

**Week 11 — Forms, client state, styling**
- **React Hook Form + Zod** — schema-first validation, minimal re-renders
- **Zustand** for client-only state (modals, sidebar, theme). One store or multiple slices
- **Tailwind CSS** — utility-first, JIT compiler, `@apply` sparingly, component extraction via composition
- `clsx` or `tailwind-merge` for conditional classes
- Accessibility basics: semantic HTML, labels, focus management, ARIA only when necessary

**Week 12 — Testing + tooling**
- **Vite** — the build tool. Familiar territory (Gradle-ish in principle)
- **Vitest** — Jest-compatible, faster, works natively with TS and Vite
- **React Testing Library** — test behavior via the DOM, not implementation details
- **MSW (Mock Service Worker)** — mock HTTP at the network layer for tests + dev
- ESLint (flat config) + Prettier
- Bundle analysis (`vite-plugin-visualizer`), code splitting with `React.lazy` + `Suspense`

### Android/Kotlin Analog

| Compose concept | React | Notes |
|---|---|---|
| `@Composable` function | Function component | Both are "UI as function" |
| `remember { mutableStateOf() }` | `useState()` | React re-runs whole function; Compose is smart-skippable |
| `LaunchedEffect(key)` | `useEffect(() => {}, [deps])` | **Deps array is a trap** — read trap 1 below |
| `DisposableEffect` | `useEffect` with cleanup return | Same pattern |
| `derivedStateOf` | `useMemo` | Both memoize derived values |
| `rememberCoroutineScope` | `useRef` + ref to controller | No direct equivalent; use TanStack Query for async |
| `CompositionLocal` | `Context` + `useContext` | Very close mental model |
| `Modifier` chains | Tailwind utility classes | Both composable; Tailwind is strings, Modifier is type-safe |
| Compose Preview | Storybook (optional) | Not core React, add if needed |
| `collectAsState()` on Flow | `useQuery` from TanStack | State management for server data |
| Navigation-Compose | React Router | Similar declarative model |
| ViewModel | Custom hook + (TanStack Query or Zustand) | State lives closer to components in React |
| Hilt `@HiltViewModel` | Nothing — just a hook | Dependency injection is not a React concern |

### ⚠️ Traps

> **Trap 1 — `useEffect` dependency array & stale closures.** Omitting deps → stale values. Including objects/arrays created inline → runs every render. Rule: include everything you reference; memoize objects/arrays with `useMemo`; memoize callbacks with `useCallback`. Or better: **don't reach for useEffect by default** — most "fetching on mount" belongs in TanStack Query.

> **Trap 2 — Compose muscle memory: "it won't re-render if nothing changed".** Wrong in React. React re-runs the entire component function on every state change of the component or any ancestor, unless you wrap in `React.memo` *and* props are referentially stable. New inline `{}` or `() => {}` in props = new ref = re-render.

> **Trap 3 — Fetching in `useEffect`.** Race conditions, no caching, double-fires in StrictMode. Use TanStack Query for anything that touches the network. You will be tempted. Resist.

> **Trap 4 — Treating `any` like Kotlin `Any?`.** `any` disables type checking entirely. Kotlin `Any?` still enforces nullability. Use `unknown` when you genuinely don't know the type.

> **Trap 5 — Mutating state.** `state.push(x); setState(state)` does nothing — referential equality hasn't changed. Always produce new objects/arrays (`[...state, x]`).

> **Trap 6 — Key as array index.** Breaks React's reconciliation for reorderable lists. Use a stable ID from your data.

> **Trap 7 — Tailwind class-name dynamic construction.** `bg-${color}-500` won't be picked up by the JIT compiler. Use complete class names or `clsx` maps.

> **Trap 8 — Redux by default.** You'll see tutorials recommend it. React's docs no longer do. Server state → TanStack Query; client state → Zustand or Context. Redux Toolkit is fine if a job requires it, not as a default.

> **Trap 9 — StrictMode double-rendering in dev.** Confuses newcomers. It's intentional — helps you catch effects with non-idempotent logic. Keep it on.

### Hands-On Project

Continue the capstone (HabitForge) frontend, described below. This phase, you build:

- Auth: login form (RHF + Zod), JWT stored in memory + httpOnly cookie option, axios interceptor / fetch wrapper that attaches the token
- Routing: protected routes, redirect on 401
- Habit list + detail pages with TanStack Query
- Optimistic UI for check-ins
- Form for creating/editing habits with schema validation
- Zustand store for UI state (sidebar, theme)
- Full test suite: RTL for components, MSW for network mocks, Vitest

### Best Resources

- **Official: react.dev (the new docs).** Start here. The "Learn" section is the best React tutorial in existence. Read it cover to cover — it's maybe 4 hours
- **Official: tanstack.com/query/latest** — their docs are the tutorial. Read the "Guides" section
- **Course (if budget): "Epic React" — Kent C. Dodds.** Exhaustive, hands-on. Workshops format. Intermediate-to-advanced patterns
- **Course (alternative): "The Joy of React" — Josh Comeau.** More beginner-focused but the *mental model* chapters are gold regardless of level
- **TypeScript: "Total TypeScript" — Matt Pocock.** Beginner + Intermediate tracks (~20 hr total). You'll fly through beginner because of Kotlin
- **YouTube: Theo (t3.gg)** for opinionated takes and ecosystem scans; **Jack Herrington** for patterns and deep dives; **Web Dev Simplified** for focused explanations
- **Tailwind docs:** tailwindcss.com/docs — scan the full "Core Concepts" and "Customization" sections
- **Book: "Effective TypeScript" — Dan Vanderkam.** Read selectively (80 items, each 2–3 pages). Items on `any`/`unknown`, generics, declaration files
- **GitHub reading:** `TanStack/router` examples, `shadcn-ui/ui` component library source (Tailwind + accessibility done right)
- **Skip:** class components tutorials, Redux courses (unless explicitly needed), Next.js courses (different beast), anything CRA-based (deprecated), "React with JavaScript" courses (you're going straight to TS)

### Skip This

- Class components beyond read-level
- Redux, MobX, Recoil, Jotai deep dives (pick TanStack Query + Zustand and move on)
- CSS-in-JS libraries (styled-components, emotion) — Tailwind is the current default
- CSS modules (work fine, but one less paradigm)
- Create React App (dead, replaced by Vite)
- Next.js / Remix — powerful but distracts from core React; revisit Phase 5 if needed
- GraphQL clients (Apollo, urql) — unless the job needs them; REST is plenty
- Webpack configuration (Vite handles it)
- Enzyme (dead; use RTL)
- PropTypes (dead; use TypeScript)

### Exit Criteria

- [ ] Can scaffold a Vite + React + TS + Tailwind project in 10 min
- [ ] Can explain when/why to use `useMemo` vs `useCallback` vs `React.memo`, and when *not* to
- [ ] Can write a component tree that doesn't re-render unnecessarily and prove it with the React DevTools Profiler
- [ ] Can debug a failing TanStack Query (stale cache, race, dependent query) without guessing
- [ ] Can write a Zod schema and derive the TS type from it (`z.infer`)
- [ ] Can test a form submission with RTL + MSW end-to-end

---

## Phase 4 — Integration, Docker, CI/CD, Observability

**Goal:** Make all three stacks talk, run with one command locally, and ship to a real URL with metrics and logs.

**Hours:** 25 hr • **Calendar:** Weeks 13–16

### Core Concepts

**Wiring**
- CORS: what it is, where it's enforced (browser), how Spring configures it (`CorsConfigurationSource` bean, not `@CrossOrigin` sprinkled on controllers)
- End-to-end auth flow: login → JWT → stored where → sent how → validated in Spring Security filter chain → principal available in controllers
- API contract hygiene: OpenAPI spec with springdoc-openapi, consumed by frontend via `openapi-typescript` for auto-generated types
- Error-response convention: problem+json on the server, typed in TypeScript

**Docker**
- `Dockerfile` for Spring Boot: multi-stage build, distroless or JRE-slim base, layered JARs for faster rebuilds
- `Dockerfile` for frontend: build stage (Node) + serve stage (nginx serving static files)
- `docker-compose.yml`: oracle-xe + spring-app + nginx/react + healthchecks + volumes for DB persistence
- Networks and service discovery via compose
- Env var injection (not baked into images)

**CI/CD (GitHub Actions)**
- Workflow triggers (`push`, `pull_request`)
- Job matrix for parallel backend + frontend tests
- Docker build + push to GHCR (GitHub Container Registry) — free for public repos
- Deployment: Fly.io (free-ish tier supports Oracle XE via Docker) or Render or a cheap VPS (Hetzner, DigitalOcean)
- Secrets management (GitHub Actions secrets, `.env` on server)

**Observability**
- Spring Boot Actuator — `/actuator/health`, `/actuator/metrics`, `/actuator/info`
- Micrometer + Prometheus registry → scrape with Prometheus → dashboard in Grafana (or use Grafana Cloud free tier)
- Structured logging (JSON) with Logstash encoder, ship to Loki or just stdout for now
- Distributed tracing basics — OpenTelemetry + Tempo/Jaeger; just aware, not required

### Android/Kotlin Analog

| Android | Integration phase | Notes |
|---|---|---|
| Firebase Crashlytics | Logs + Sentry or Loki | Application-side error capture |
| Firebase Analytics | Micrometer + Prometheus | Metrics instead of events |
| Play Console rollout | Fly.io / Docker Hub / GHCR | Deploy pipelines |
| ProGuard / R8 | Docker multi-stage | Slimming output artifact |
| Flavors (`buildTypes`) | Spring profiles + Docker env vars | Config per env |

### ⚠️ Traps

> **Trap 1 — CORS only on `@CrossOrigin`.** It works in dev, fails in prod. Configure via `CorsConfigurationSource` bean in Spring Security. Also: CORS is browser-enforced. Curl from a CI script doesn't hit it.

> **Trap 2 — Storing JWT in localStorage.** XSS-accessible. Use an httpOnly cookie OR store in memory + refresh flow. There's no free lunch; pick one and understand the tradeoffs.

> **Trap 3 — Forgetting to rebuild React on deploy.** Frontend bundles are immutable artifacts; you must rebuild and redeploy to change the UI. Dev-only habits don't apply.

> **Trap 4 — Docker container timezone.** Default UTC; application code that uses `LocalDateTime.now()` without a zone is a bug waiting to surface in reports.

> **Trap 5 — Oracle XE on Docker needs special care.** `gvenzl/oracle-free` or `gvenzl/oracle-xe` images are community-maintained. Persist `/opt/oracle/oradata` in a volume or you lose data every restart.

> **Trap 6 — `application.yml` secrets committed to git.** Use env vars always: `${DB_PASSWORD}` placeholders. Set a pre-commit hook to scan for secrets (gitleaks).

### Hands-On

Ship the capstone:

1. Dockerfile for backend (multi-stage, layered)
2. Dockerfile for frontend (build + nginx)
3. `docker-compose.yml` — `docker compose up` boots the whole app on localhost
4. GitHub Actions workflow: lint, test, build, push images on merge to main
5. Deploy to Fly.io (or similar). Public URL, HTTPS via provider
6. Prometheus scraping `/actuator/prometheus`; one Grafana dashboard with request rate, p95 latency, DB pool usage
7. Structured JSON logs with request correlation IDs

### Best Resources

- **Docker: "Docker Deep Dive" — Nigel Poulton.** Concise, current. Read chapters on images, containers, networks, compose
- **"Spring Boot in Practice" — Somnath Musib.** Chapters on Docker and CI/CD patterns for Spring — very practical
- **GitHub Actions docs** at `docs.github.com/en/actions`. Start with "Workflow syntax", then "Docker examples"
- **Fly.io docs** for deployment — best free-ish option for containerized apps with a DB
- **"Observability Engineering" — Majors/Fong-Jones/Miranda.** Read chapters 1–4 and 8 for mental model; skip the rest until you're oncall
- **Micrometer docs** + Baeldung Spring Boot Actuator articles
- **Skip:** Kubernetes anything (massive overkill for this roadmap), AWS certifications, Terraform deep dive

### Skip This

- Kubernetes, Helm, Kustomize — not needed for a single-service app
- Service mesh (Istio, Linkerd)
- Blue/green and canary deploys (aware only)
- Advanced tracing, eBPF — oncall territory
- Infrastructure-as-code (Terraform) — use Fly's simpler abstractions

### Exit Criteria

- [ ] Capstone runs locally with `docker compose up`
- [ ] Capstone is deployed to a public URL with HTTPS
- [ ] CI pipeline runs on every PR, green before merge
- [ ] You can show a Grafana dashboard with real traffic metrics
- [ ] You can explain the full request path: browser → CORS preflight → nginx → Spring filter chain → controller → service → HikariCP → Oracle → back

---

## Capstone Project — HabitForge

**Elevator pitch:** A habit tracker with team accountability. Users create habits, check in daily, join teams, compete on streak leaderboards. Portfolio-grade: every major concept from every phase is exercised.

### Why this project

- **CRUD + relationships:** users, teams, memberships, habits, check-ins — enough to exercise every JPA feature
- **Time-based logic:** streaks, calendars — exercise Oracle date/time + window functions
- **Auth + authz:** users own habits; team admins can kick members — exercise Spring Security properly
- **Real-time-ish UX:** optimistic check-ins, leaderboard updates — exercise TanStack Query mutations + cache invalidation
- **Deployable:** small enough to fit a single Docker Compose + one cheap VPS or Fly.io

### Domain Model

```
users (id, email, password_hash, display_name, created_at)
teams (id, name, invite_code, created_by, created_at)
team_memberships (user_id, team_id, role, joined_at)  -- composite PK
habits (id, user_id, name, cadence, target_per_week, created_at, archived_at)
check_ins (id, habit_id, checked_at DATE, note)
-- leaderboard is a view
```

### Milestones Aligned to Phases

| Phase | HabitForge milestone |
|---|---|
| Phase 1 (Spring) | Backend CRUD: users, teams, habits, check-ins; JWT auth; validation; global error handling; 80% test coverage |
| Phase 2 (Oracle) | Migrate from H2 to Oracle XE; add leaderboard endpoint as a native window-function query; fix an intentionally-introduced N+1; add DB indexes with a plan-based justification |
| Phase 3 (React) | Full UI: login, dashboard, habit list, check-in, team view, leaderboard; TanStack Query everywhere; RHF+Zod forms; Zustand for UI state; Vitest+RTL+MSW tests |
| Phase 4 (Integration) | Docker Compose stack; deployed; CI/CD green; metrics dashboard; blog post about the build |

### "Done" Criteria

- [ ] Public URL, HTTPS, domain optional
- [ ] README with architecture diagram, local setup, deploy instructions
- [ ] A blog post (dev.to / Medium / personal) titled something like *"Android engineer ships a full-stack Spring Boot + React + Oracle app in 16 weeks — here's what I learned"* — this is your strongest portfolio signal
- [ ] Test coverage ≥ 75% backend, ≥ 60% frontend (components that matter)
- [ ] Seed script to populate demo data
- [ ] Open-source it on GitHub. Recruiters look at commit history — keep it clean

---

## Anti-Patterns: Android Devs Moving to Web

These are specific mistakes Android engineers make in this transition. Each has cost someone days.

1. **Treating singletons as state holders.** `@Service` beans are singletons. Putting `private User currentUser` as a field is a concurrent-user bug. State lives in the token, the DB, or a request-scoped bean — never a singleton field.

2. **Field injection by habit.** Kotlin doesn't really have field injection; Android Hilt uses it for framework classes. In Spring, field injection is an anti-pattern for your own code — constructor-inject always.

3. **Ignoring the N+1.** Room doesn't lazy-load, so you've never encountered this. JPA does by default. You will write your first N+1 on day 1. Learn to spot it immediately.

4. **JPA entities as API responses.** Convenient, catastrophic. Causes `LazyInitializationException`, leaks DB schema to clients, bloats payloads. DTO boundary is mandatory.

5. **Compose skip-semantics assumption in React.** "It won't re-render, the value is the same." Wrong. React re-runs components on every state change of self or parent unless memoized and props are referentially stable.

6. **Treating `useEffect` as the async escape hatch.** It's not for fetching. It's for synchronizing with external systems. Use TanStack Query for data, event listeners for subscriptions.

7. **TS `any` as Kotlin `Any?`.** `any` turns off the type system. `unknown` is closer to `Any?` — requires a cast/narrow before use.

8. **Redux because "that's what state management is".** React has moved on. TanStack Query + Zustand (or even just `useState` + Context) covers 95% of apps.

9. **Oracle like SQLite.** You never worried about connection pools, execution plans, or sequences. You will now. Treat Oracle as a new skill, not a slightly different SQLite.

10. **Unquoted identifier trap.** `CREATE TABLE "users"` is not `CREATE TABLE users`. The former forces case-sensitive quoted identifiers everywhere. Don't.

11. **Maven hatred.** You'll hate it at first. Accept: most enterprise Spring is Maven. Learn the lifecycle (`validate → compile → test → package → verify → install → deploy`) and move on. Use Gradle Kotlin DSL for your own projects.

12. **Gradle Kotlin DSL assumptions in Spring.** Many Spring plugins' docs show Groovy. Translate mentally. Occasionally a plugin has a quirk in Kotlin DSL — search first.

13. **Shipping without CORS understanding.** "It works on localhost." It won't in prod from a different domain. Configure Spring Security's CORS *once*, properly.

14. **Treating HTTP like coroutines.** Request lifecycle ≠ coroutine lifecycle. Work that continues after the response is sent must go to an async queue (Spring `@Async`, a message broker, or a scheduled job), not a fire-and-forget on the request thread.

15. **Mobile-first styling habits on desktop.** Compose on phones trains a narrow-viewport instinct. Web has breakpoints; Tailwind makes them easy; think through responsive from the start.

16. **Treating frontend tests like Android instrumentation tests.** RTL is unit-ish, fast, and network-mocked. Don't spin up a browser-automation harness for every component test.

17. **Ignoring bundle size.** On Android, you don't care about 200 KB. On the web, first-byte to paint is measured in KB over slow networks. Learn code splitting (`React.lazy`, `Suspense`) early.

18. **Overbuilding the backend before the frontend exists.** Build both sides of each feature together. Don't spend 4 weeks on a "perfect" API, then discover the frontend needs a different shape.

---

## Learning Techniques

### Daily Rhythm

- **Weekday evenings (2–3 hr):** 60% building, 30% reading, 10% video. Rotate. If the code isn't flowing one night, switch to reading the docs for what you're stuck on.
- **Weekend block (6–9 hr):** one long push on the capstone. Momentum compounds — two hours of setup followed by six of flow is how features ship.
- **Every morning:** 10-min review of yesterday's notes. Recall before reference.
- **Track hours.** A simple spreadsheet. You'll discover you're at 10 hr/wk when you *feel* like 20. Be honest.

### Docs vs Build vs Video — When Each Wins

- **Docs = truth.** When you need to know what a function *actually* does, read the source of truth. Video explanations date fast; docs track the current version.
- **Build = retention.** You forget 80% of what you read within a week unless you apply it. The capstone is not optional — it's how the reading sticks.
- **Video = ergonomics.** Good for "what does it *feel* like to use this tool", "what's the happy path", "what's the community's current default". Bad as a primary learning medium for anything nontrivial.
- **Rule:** if you watch more than 30 min of video in a day, you're procrastinating.

### Using AI Assistants Effectively During This Transition

You are experienced. AI is an accelerator, not a tutor.

**Good uses:**
- "Explain this stacktrace"
- "What's the idiomatic Spring way to do X?"
- "Translate this Kotlin snippet to idiomatic Java 21"
- "Why is my TanStack Query re-fetching on every render?"
- Rubber-ducking a design before committing to it
- Generating boilerplate you understand (entity boilerplate, test scaffolding)

**Bad uses:**
- Having it write the feature while you watch. You lose the rep count that builds intuition.
- Copy-pasting without reading the imports — most hallucination happens there
- Asking about current library versions — AI knowledge cuts off; always cross-reference docs
- Asking "what's the right way" for opinionated decisions — get one senior's opinion (this doc), ship, reflect

**Meta-rule:** if you can't write it without AI a week later, you haven't learned it. Schedule "unplugged" rebuilds once a week — re-implement a previous feature from memory.

### Validating Learning (vs Memorization)

The Feynman test, adapted for this roadmap:

At each phase exit, pick three exit-criteria items and explain them **out loud to an imagined colleague**, no IDE, no docs, no AI. Record yourself if you have to. If you stumble or hand-wave, that's a gap — go back.

Ask yourself per week: *what did I learn this week that a bootcamp grad wouldn't have?* If the answer is "nothing", you're not leveraging your background. Redesign the week.

### How to Know You're Ready for Interviews

- You can draw the Spring filter chain for JWT auth on a whiteboard
- You can explain why a specific query uses an index (or doesn't) in your own app
- You can write a React component from scratch that fetches, memoizes, and renders without consulting a cheat sheet
- You can set up a fresh project in each stack in under 15 min
- You've shipped the capstone to a public URL and someone (friend, former colleague) has used it

---

## Appendix A — Cheat Sheet

### Android ↔ Spring ↔ React — Concept Equivalence

| Android / Kotlin | Spring Boot (Java) | React (TypeScript) |
|---|---|---|
| `data class` | `record` | `interface` / `type` |
| `sealed class` | `sealed interface` | Discriminated union (`type X = A \| B`) |
| Hilt `@Inject` constructor | Constructor injection | Import the hook/module |
| Hilt `@Module` + `@Provides` | `@Configuration` + `@Bean` | Context provider |
| Hilt `@Singleton` | Default bean scope (singleton) | Module-scoped constant |
| `viewModelScope` | Request lifecycle | `useEffect` cleanup |
| `lifecycleScope` | N/A (stateless) | Component lifecycle |
| `Flow<T>` | `Flux<T>` (WebFlux) | RxJS / TanStack Query subscription |
| `StateFlow<T>` | `BehaviorSubject` (Reactor) | Zustand store |
| `suspend fun` | Virtual thread method | `async () => {}` |
| Retrofit `@GET` interface | `@GetMapping` on controller | `useQuery` hook |
| OkHttp interceptor | `SecurityFilterChain` / `HandlerInterceptor` | Axios interceptor / `fetch` wrapper |
| Room `@Entity` | JPA `@Entity` | Zod schema |
| Room DAO | `JpaRepository` | Zustand slice |
| `@Query("SELECT ...")` | JPQL or `@Query` native | N/A |
| `@Serializable` (kotlinx) | Jackson default | `JSON.parse` / Zod `.parse` |
| `Result.success / failure` | Exception + `@ControllerAdvice` | `Result<T, E>` pattern (neverthrow) or error boundary |
| Compose `@Composable` | — | Function component |
| `remember { mutableStateOf(x) }` | — | `const [x, setX] = useState(...)` |
| `LaunchedEffect(key)` | — | `useEffect(() => {}, [deps])` |
| `CompositionLocal` | — | `Context` + `useContext` |
| `NavHost` / Navigation-Compose | — | React Router `<Routes>` |
| `ViewModel` | `@Service` + `@RestController` (server-side state) | Custom hook + TanStack Query |
| Koin `module { }` | `@Configuration` class | N/A (no DI container) |
| Gradle `build.gradle.kts` | `pom.xml` or Gradle | `package.json` + `vite.config.ts` |
| `proguard-rules.pro` | Docker multi-stage | Vite build, tree-shaking |
| Android `R` class | N/A | Import assets / CSS modules |
| Drawable / vector | SVG component / `<img>` | Same |
| `dp` / `sp` units | N/A | `rem` / `em` / `px` + Tailwind spacing scale |
| `ViewModelFactory` | `@Configuration @Bean` | Factory function |
| `ActivityResultContract` | N/A | Form submission flow |
| Fragment/Activity lifecycle | Request lifecycle | Component lifecycle (mount/unmount) |

### Reverse lookup — "How do I do X in the new stack?"

| "In Android I would..." | In Spring | In React |
|---|---|---|
| Use Retrofit to call an API | Call a `RestClient` / `WebClient` bean from a service | `useQuery(['key'], () => fetch(...))` with TanStack Query |
| Store a token | N/A (client concern) | Memory + httpOnly cookie refresh, or secure cookie |
| Cache DB results | Spring Cache (`@Cacheable`) on service methods | TanStack Query's built-in cache |
| Log in debug only | Profile-gated logger config | `import.meta.env.DEV` check |
| Show a loading spinner | N/A | `isLoading` from `useQuery` |
| Pass data to a parent | Callback prop / shared ViewModel | Callback prop / lifted state / Zustand |
| Handle config change | N/A | React state survives re-render by design |
| Unit test a ViewModel | Mock `@Service` dependencies with Mockito | Test hook with `@testing-library/react-hooks` (now built into RTL) |

---

## Appendix B — Resource Index

### Books (ordered by priority per phase)

**Phase 0 — Java 21**
- *Modern Java in Action* (Urma/Fusco/Mycroft) — selective chapters on streams, Optional, CompletableFuture

**Phase 1 — Spring Boot**
- *Spring Start Here* — Laurențiu Spilcă **[PRIMARY]**
- *Spring Security in Action* — Laurențiu Spilcă
- *Spring Boot in Practice* — Somnath Musib (for Docker/CI patterns)

**Phase 2 — Oracle + JPA**
- *Expert Oracle Database Architecture* — Tom Kyte (selective)
- *High-Performance Java Persistence* — Vlad Mihalcea (highly recommended)

**Phase 3 — TypeScript + React**
- *Effective TypeScript* — Dan Vanderkam
- *React Cookbook* (O'Reilly) — for specific patterns as reference

**Phase 4 — Integration**
- *Docker Deep Dive* — Nigel Poulton
- *Observability Engineering* — Majors/Fong-Jones/Miranda (mental model chapters)

### Courses & Instructors

- Dan Vega (YouTube / Udemy) — Spring
- Laur Spilca (YouTube / books) — Spring, Security
- Java Brains / Koushik (YouTube) — Spring clarifications
- Nicolai Parlog / Inside Java (YouTube) — Java 21 features
- Vlad Mihalcea (blog) — Hibernate and JPA performance
- Thorben Janssen (blog) — JPA tutorials
- Matt Pocock — Total TypeScript
- Kent C. Dodds — Epic React
- Josh Comeau — The Joy of React
- Theo (t3.gg) — React/TS opinionated scans
- Jack Herrington (YouTube) — React patterns

### Official Docs Worth Reading Linearly

- react.dev "Learn" section (4 hr, mandatory)
- TanStack Query Guides (2 hr)
- Spring Boot Reference — sections on "Using Spring Boot", "Spring Boot Features", "Testing"
- Tailwind CSS "Core Concepts" section

### GitHub Repos to Study

- `spring-projects/spring-petclinic` — canonical Spring reference
- `shadcn-ui/ui` — Tailwind + accessibility done right
- `TanStack/query` examples — idiomatic patterns
- `vladmihalcea/high-performance-java-persistence` — companion code to his book
- Your own capstone — study your own decisions in retro

### Tools

- IntelliJ IDEA Ultimate (Spring support is worth it; Community works)
- VS Code + extensions: ESLint, Prettier, Tailwind IntelliSense, Error Lens
- Oracle SQL Developer (free)
- DBeaver (free alternative, multi-DB)
- Postman or Insomnia or Bruno (pick one; Bruno is open-source)
- Docker Desktop
- GitHub CLI (`gh`) for PRs from terminal

---

## Appendix C — Interview-Readiness Checklist

### Can you whiteboard this in 10 min?

- [ ] Spring filter chain with JWT, CORS, auth
- [ ] Controller → Service → Repository flow with DTO mapping
- [ ] JPA entity relationship diagram for a small domain
- [ ] React component tree showing state ownership and data flow
- [ ] TanStack Query cache lifecycle (fetch, cache, invalidate, refetch)
- [ ] Docker Compose topology for your capstone

### Can you explain these in ≤3 sentences?

- [ ] Difference between `@Component`, `@Service`, `@Repository`, `@Controller`
- [ ] What `@Transactional` propagation levels mean and when you'd use each
- [ ] Why LAZY loading can cause `LazyInitializationException`
- [ ] How CORS preflight works
- [ ] Why we prefer httpOnly cookies for JWTs (and the tradeoff)
- [ ] React's reconciliation and how `key` affects it
- [ ] Why TanStack Query is superior to `useEffect` for data fetching
- [ ] Difference between `useMemo`, `useCallback`, `React.memo`
- [ ] What HikariCP pool exhaustion looks like in logs
- [ ] Oracle sequence caching behavior

### Can you live-code these in 30 min?

- [ ] Spring Boot REST controller with validation and exception handling
- [ ] JPA entity with a one-to-many relationship + query
- [ ] React component that fetches paginated data with TanStack Query
- [ ] RHF + Zod form with field-level errors and a submit mutation
- [ ] A Spring Security config that only allows authenticated users to access `/api/**`
- [ ] A Dockerfile for a Spring Boot app (multi-stage)

### Can you answer the "why" questions?

- [ ] Why Gradle over Maven (or vice versa) for *this* project?
- [ ] Why TanStack Query over Redux?
- [ ] Why Tailwind over CSS-in-JS?
- [ ] Why Oracle over PostgreSQL in your current target role?
- [ ] Why Spring MVC over WebFlux as default?
- [ ] Why constructor injection over field injection?
- [ ] Why DTO mapping over returning entities?

If you can check every box, you are production-confident. Ship the capstone, write the blog post, update LinkedIn.

---

## Closing Note

You are not starting from zero — you are starting from *five years of senior engineering* and adding three new tools. The tools have their own vocabulary, their own defaults, and their own footguns, but the thinking underneath is the same: layered design, explicit state, contract-driven integration, tested behavior.

The hardest part of this transition is not the material. It's the humility to re-learn what you already half-know — the instinct that says "I've seen this before" when the behavior is subtly different. That's what the trap callouts are for. Read them twice.

Build the capstone. Ship it. Write about it. The second you have a public URL someone can visit, you are no longer "transitioning" — you are a full-stack engineer who also happens to know Android deeply.

Good luck. See you on the other side of Week 16.
