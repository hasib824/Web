# Phase 1 — Foundation & First Endpoint

**Stack চূড়ান্ত:** Java + Oracle/MSSQL
**সময়:** ৪–৬ ঘণ্টা
**Phase শেষে আপনার হাতে থাকবে:** চলমান Spring Boot app, proper feature-first package structure, এবং `/api/v1/containers/ping` endpoint যেটা Postman থেকে hit করে response পাবেন।

---

## এই Phase-এ আপনি যা শিখবেন

1. Spring Boot project generate করা এবং dependencies বোঝা
2. Feature-first package structure (আপনার Android feature module pattern-এর মতো)
3. `@SpringBootApplication` আসলে কী করে — IoC container startup
4. Stereotype annotations (`@RestController`, `@Service`, `@Component`) কোনটা কোথায়
5. Constructor injection (Hilt `@Inject` constructor-এর Spring equivalent)
6. `application.yml` ও profiles (Android build variants-এর সমতুল্য)
7. আপনার প্রথম REST endpoint

---

## Step 1 — Project Generate করুন

`https://start.spring.io` এ যান। এই settings ঠিক করুন:

| Field | Value |
|---|---|
| Project | **Maven** |
| Language | **Java** |
| Spring Boot | **3.3.x** (latest stable 3.x) |
| Group | `com.yourcompany` |
| Artifact | `portops` |
| Name | `portops` |
| Package name | `com.yourcompany.portops` |
| Packaging | **Jar** |
| Java | **21** (আপনার machine-এ যেটা আছে — 17 বা 21) |

### Dependencies যোগ করুন (ডান দিকে "ADD DEPENDENCIES")

- **Spring Web** — REST API বানানোর জন্য
- **Spring Data JPA** — DB layer (Phase 2-এ ব্যবহার হবে)
- **Validation** — `@Valid`, `@NotNull` — Bean Validation
- **Lombok** — boilerplate কমানোর জন্য
- **Spring Boot DevTools** — code change-এ auto-restart

### যা এখন add করবেন না

- **Spring Security** → Phase 4-এ যোগ করব
- **Oracle/MSSQL Driver** → নিচের Step 2-এ manual add করব

**কারণ:** একসাথে সব dependency নিলে startup log অনেক noisy হয়, আর আপনি auto-configuration-এ কী হচ্ছে সেটা trace করতে পারবেন না। Layer-by-layer add করলে প্রত্যেকটা dependency কী নিয়ে আসছে সেটা টের পাবেন।

**"GENERATE"** চাপুন → ZIP download হবে → unzip → IntelliJ IDEA-তে **Open as Project** করুন।

প্রথমবার open করলে IntelliJ Maven dependencies resolve করবে। ২–৫ মিনিট লাগতে পারে। নিচে status bar-এ "Indexing" শেষ হওয়া পর্যন্ত অপেক্ষা করুন।

---

## Step 2 — DB Driver Manual Add করুন

`pom.xml` খুলুন। `<dependencies>` section-এ এই দুটো যোগ করুন:

```xml
<!-- Oracle JDBC driver -->
<dependency>
    <groupId>com.oracle.database.jdbc</groupId>
    <artifactId>ojdbc11</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- MSSQL JDBC driver -->
<dependency>
    <groupId>com.microsoft.sqlserver</groupId>
    <artifactId>mssql-jdbc</artifactId>
    <scope>runtime</scope>
</dependency>
```

`scope=runtime` মানে: compile time-এ এই JARs আপনার code-এ access হবে না (আপনি direct `oracle.jdbc.*` import করবেন না), শুধু runtime-এ JVM এগুলো খুঁজবে DB connection বানানোর সময়।

**Maven reload:** IntelliJ-এ ডান পাশে Maven tab → refresh icon। অথবা `pom.xml`-এর উপরে floating "Load Maven Changes" button চাপুন।

> ⚠️ **এই Phase-এ DB connect করব না।** driver এখন রাখলাম যাতে Phase 2-এ `application.yml` লেখার সময় dependency issue না আসে। Phase 1-এ app boot হবে DB ছাড়াই।

---

## Step 3 — Feature-First Package Structure বানান

Spring Initializr থেকে default structure পাবেন:

```
src/main/java/com/yourcompany/portops/
    └── PortopsApplication.java
```

এটাকে রূপান্তর করুন এই structure-এ:

```
src/main/java/com/yourcompany/portops/
    ├── PortopsApplication.java       ← entry point
    ├── container/
    │     ├── api/                    ← controllers + DTOs
    │     ├── domain/                 ← entities + enums
    │     └── persistence/            ← JPA repositories (Phase 2)
    ├── shipment/
    │     ├── api/
    │     ├── domain/
    │     └── persistence/
    ├── yard/
    │     ├── api/
    │     ├── domain/
    │     └── persistence/
    ├── billing/
    │     ├── api/
    │     ├── domain/
    │     └── persistence/
    └── common/
          ├── exception/              ← domain exceptions (Phase 3)
          └── config/                 ← cross-cutting config (Phase 4)
```

IntelliJ-এ `portops` package-এ **right-click → New → Package** → নাম দিন। এখন প্রত্যেকটা leaf package (যেমন `container.api`) খালি থাকবে — সেটাই ঠিক।

### কেন এই structure? (Android background থেকে mapping)

Android-এ আপনি সম্ভবত এরকম feature module pattern ব্যবহার করেন:

```
:feature_kyc
    ├── :feature_kyc:data
    ├── :feature_kyc:domain
    └── :feature_kyc:presentation
```

Spring Boot-এ **Gradle module-এর বদলে package** দিয়ে একই separation করছি। পার্থক্য:

| Android feature module | Spring Boot feature package |
|---|---|
| `:feature_kyc:presentation` (UI) | `container.api` (HTTP endpoints) |
| `:feature_kyc:domain` (use cases, models) | `container.domain` (entities, services) |
| `:feature_kyc:data` (repository impl) | `container.persistence` (JPA repos) |

**ভুল যেটা করবেন না:** root-এ `controllers/`, `services/`, `repositories/` বানাবেন না। এটা scales বাজেভাবে — ৫টা module হলেই `controllers/` package-এ ৩০টা class জমে যায়, আর আপনি container কোডের জন্য তিন জায়গায় ঘুরতে হবে।

---

## Step 4 — `@SpringBootApplication` বুঝুন

`PortopsApplication.java` খুলুন:

```java
package com.yourcompany.portops;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PortopsApplication {
    public static void main(String[] args) {
        SpringApplication.run(PortopsApplication.class, args);
    }
}
```

এই ৯ লাইন কোড যা করছে:

### `@SpringBootApplication` তিনটা annotation-এর shortcut

| Annotation | কাজ | Android analogy |
|---|---|---|
| `@Configuration` | এই class-টা bean definitions দিতে পারে | Hilt `@Module` class |
| `@EnableAutoConfiguration` | classpath-এ যা আছে সেই অনুযায়ী default setup করো | Hilt-এর pre-built `@InstallIn` modules |
| `@ComponentScan` | এই package ও sub-packages scan করে `@Component`/`@Service`/`@RestController` খোঁজো | Hilt-এর aggregating compiler যা সব `@AndroidEntryPoint` খুঁজে বের করে |

### `SpringApplication.run(...)` যা করে

1. `ApplicationContext` (IoC container) বানায় — এটাই আপনার app-এর "brain"
2. Component scan চালায় — সব `@RestController`, `@Service`, `@Repository`, `@Component` class খুঁজে বের করে
3. প্রত্যেকটার instance বানায় (singleton by default) — Spring ["bean"](bean = Spring-managed object) বলে
4. Dependency graph resolve করে — constructor-এ যার যা লাগে inject করে
5. Embedded Tomcat start করে port 8080-এ
6. আপনার app HTTP request নিতে প্রস্তুত

**Mental model:** Android-এ Hilt compile-time-এ code generate করে। Spring runtime-এ বেশিরভাগ কাজ করে — app boot-এর সময় reflection + classpath scanning দিয়ে। একটু slower startup, কিন্তু অনেক বেশি flexible।

### একটা critical rule

`@SpringBootApplication` class যে package-এ আছে (আপনার case-এ `com.yourcompany.portops`), component scan **সেই package আর তার সব sub-packages**-এ চলে। তাই `container/`, `shipment/` — সব ঠিক জায়গায় আছে।

যদি কোনো `@Service` বা `@RestController` component scan-এ ধরা না পড়ে (e.g., আপনি accidentally `com.other.package`-এ বানিয়েছেন), Spring সেটার existence জানবে না। এই ভুলটা খুব common প্রথম সপ্তাহে।

---

## Step 5 — `application.yml` সেট করুন

Spring Initializr আপনাকে `application.properties` দিয়েছে। এটা rename করুন → `application.yml`।

**কেন YAML?** Properties format flat key-value, অনেক duplication হয়। YAML hierarchical — nested config পড়তে সহজ। Enterprise projects-এ YAML default।

Location: `src/main/resources/application.yml`

লিখুন:

```yaml
spring:
  application:
    name: portops

  profiles:
    active: dev

server:
  port: 8080
  servlet:
    context-path: /

logging:
  level:
    root: INFO
    com.yourcompany.portops: DEBUG
    org.springframework.web: INFO
```

### Profile concept (Android build variant-এর সমতুল্য)

Android-এ `debug`, `release`, `staging` build variants থাকে — প্রত্যেকটায় আলাদা config (API base URL, logging level, feature flags)।

Spring-এ এটাকে বলে **profiles**। এখন তিনটা file বানান:

**`application.yml`** (ইতিমধ্যে আছে — common config সব profile-এর জন্য)

**`application-dev.yml`** (local development)
```yaml
# Local dev — Phase 2-এ এখানে Oracle/MSSQL connection আসবে
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

**`application-prod.yml`** (production)
```yaml
# Production — real secrets env variables থেকে আসবে
logging:
  level:
    root: WARN
    com.yourcompany.portops: INFO
```

**Active profile কীভাবে switch করবেন:**

- Development: `application.yml`-এ `spring.profiles.active: dev` (এখন যেটা করেছি)
- Runtime override: `java -jar portops.jar --spring.profiles.active=prod`
- Environment variable: `SPRING_PROFILES_ACTIVE=prod`

Spring active profile অনুযায়ী `application-{profile}.yml` load করে common `application.yml`-এর উপরে overlay করে। মানে: specific values override করে, বাকি values inherit করে। Android BuildConfig merging-এর মতোই।

---

## Step 6 — আপনার প্রথম REST Controller বানান

`container/api/` package-এ `ContainerController.java` বানান:

```java
package com.yourcompany.portops.container.api;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/containers")
public class ContainerController {

    @GetMapping("/ping")
    public String ping() {
        return "Container module alive";
    }
}
```

এই ছোট্ট class-এ কী হচ্ছে:

| Line | ব্যাখ্যা |
|---|---|
| `@RestController` | Spring-কে বলছে: এই class HTTP endpoints handle করে, return value JSON হিসেবে serialize হবে |
| `@RequestMapping("/api/v1/containers")` | class-level base path। সব method-এর path এর সাথে prefix হবে |
| `@GetMapping("/ping")` | এই method HTTP `GET /api/v1/containers/ping` handle করে |
| `return "Container module alive"` | String return হলে Spring `text/plain` response দেয় |

### Android থেকে mapping

- **ViewModel** = যে জায়গায় UI event আসে → Controller = যেখানে HTTP request আসে
- **Fragment `onClick()` method** = Controller-এর `@GetMapping` method
- **Retrofit interface** এর যেকোনো `@GET` method = এই Controller-এর একটা endpoint (কিন্তু এবার আপনি server side, client side না)

---

## Step 7 — App Run করুন

IntelliJ-এ `PortopsApplication` class খুলুন → `main` method-এর পাশে সবুজ ▶ button চাপুন → **Run 'PortopsApplication'**।

### Console-এ যা দেখবেন (এবং যা খুঁজবেন)

```
  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v3.3.x)

... Starting PortopsApplication using Java 21 ...
... The following 1 profile is active: "dev"
... Tomcat initialized with port 8080 (http)
... Starting service [Tomcat]
... Started PortopsApplication in 2.847 seconds
```

**যা verify করবেন:**

1. ✅ `profile is active: "dev"` — আপনার profile config ধরেছে
2. ✅ `Tomcat initialized with port 8080` — HTTP server চালু
3. ✅ `Started PortopsApplication in X seconds` — app boot complete
4. ❌ কোনো red stacktrace — থাকলে নিচের **Troubleshooting** section দেখুন

---

## Step 8 — Endpoint Hit করুন

**Option A: Browser**
`http://localhost:8080/api/v1/containers/ping`
"Container module alive" text দেখবেন।

**Option B: Postman** (recommended — Phase 2 থেকে আপনি অনেক API test করবেন)

1. Postman open করুন → New Request
2. Method: `GET`
3. URL: `http://localhost:8080/api/v1/containers/ping`
4. Send

**Expected response:**
- Status: `200 OK`
- Body: `Container module alive`

**Option C: Terminal (curl)**
```bash
curl -i http://localhost:8080/api/v1/containers/ping
```

Response:
```
HTTP/1.1 200
Content-Type: text/plain;charset=UTF-8
Content-Length: 22
Date: ...

Container module alive
```

---

## Step 9 — Constructor Injection Preview (Phase 2-এর জন্য Warmup)

এখন একটা খালি `@Service` বানাই শুধু DI flow টা দেখানোর জন্য।

`container/domain/` package-এ `ContainerService.java`:

```java
package com.yourcompany.portops.container.domain;

import org.springframework.stereotype.Service;

@Service
public class ContainerService {

    public String healthCheck() {
        return "Container service is wired up";
    }
}
```

এখন `ContainerController` update করুন:

```java
package com.yourcompany.portops.container.api;

import com.yourcompany.portops.container.domain.ContainerService;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/v1/containers")
public class ContainerController {

    private final ContainerService containerService;

    // Constructor injection — @Autowired লাগে না Spring 4.3+ থেকে
    public ContainerController(ContainerService containerService) {
        this.containerService = containerService;
    }

    @GetMapping("/ping")
    public String ping() {
        return "Container module alive";
    }

    @GetMapping("/health")
    public String health() {
        return containerService.healthCheck();
    }
}
```

App restart করুন (DevTools থাকায় auto-restart হবে), তারপর hit করুন:
`GET http://localhost:8080/api/v1/containers/health`

Response: `Container service is wired up`

### এখানে যা ঘটল — DI flow

1. App startup-এ Spring `@Service` চিহ্নিত `ContainerService` class-এর একটা instance বানালো
2. `ContainerController`-এর constructor scan করে দেখলো সেটা `ContainerService` চায়
3. Spring সেই pre-created instance inject করলো
4. একবারই instance বানায় (singleton scope by default) — সব requests একই instance ব্যবহার করে

### Hilt থেকে পার্থক্য

**Hilt:**
```kotlin
@HiltViewModel
class KycViewModel @Inject constructor(
    private val verifyKycUseCase: VerifyKycUseCase
) : ViewModel()
```

**Spring:**
```java
@RestController
public class ContainerController {
    private final ContainerService containerService;

    public ContainerController(ContainerService containerService) {
        this.containerService = containerService;
    }
}
```

পার্থক্য দুটো:
1. **Spring-এ `@Inject` লাগে না constructor-এ** (Spring 4.3+ থেকে) — শুধু একটাই constructor থাকলে Spring ধরে নেয় সেটাই injection target
2. **Hilt compile-time safe**, build fail হয় dependency missing হলে। **Spring runtime fail** — app startup-এ `NoSuchBeanDefinitionException` আসে। একটু পিছিয়ে, কিন্তু বেশি flexible

### যা কখনো করবেন না (Android dev-দের common mistake)

```java
// ❌ কখনো না — field injection
@Autowired
private ContainerService containerService;
```

কেন না:
- Testing-এ mock inject করা কষ্টকর
- Immutability break হয়
- Hidden dependency — constructor দেখে বোঝা যায় না class-টা কী কী লাগে

Android-এ যেভাবে property injection এড়িয়ে constructor injection ব্যবহার করতেন, Spring-এও একই discipline।

---

## Phase 1 Milestone Checklist

নিচের সব ✅ হলে Phase 1 সফল:

- [ ] App boot হয় error ছাড়া, console-এ "Started PortopsApplication" দেখা যায়
- [ ] `spring.profiles.active: "dev"` log-এ আসে
- [ ] `GET /api/v1/containers/ping` → `200 OK` + "Container module alive"
- [ ] `GET /api/v1/containers/health` → `200 OK` + "Container service is wired up"
- [ ] Package structure feature-first (`container/`, `shipment/`, `yard/`, `billing/`, `common/`)
- [ ] প্রত্যেক feature package-এর নিচে `api/`, `domain/`, `persistence/` sub-packages আছে

---

## Self-Check Questions (Phase 2-এ যাওয়ার আগে)

নিজেকে এই প্রশ্নগুলো করুন — না পারলে উপরে scroll করে দেখুন:

1. `@SpringBootApplication` আসলে কোন তিনটা annotation-এর combination?
2. Spring কীভাবে জানে `ContainerController` inject করার সময় কোন `ContainerService` instance দেবে?
3. Active profile switch করার তিনটা উপায় কী কী?
4. Feature-first package structure কেন `controllers/`, `services/` pattern-এর চেয়ে ভালো?
5. Controller-এ constructor injection-এ `@Autowired` কেন লাগে না?

---

## Troubleshooting — সাধারণ সমস্যা

### Problem: `Port 8080 was already in use`
**কারণ:** অন্য কোনো process 8080 ধরে রেখেছে।
**সমাধান:** `application.yml`-এ `server.port: 8081` করুন, অথবা যে process 8080 দখল করেছে সেটা kill করুন।

```bash
# Linux/Mac
lsof -i :8080
kill -9 <PID>

# Windows
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

### Problem: `Failed to configure a DataSource: 'url' attribute is not specified`
**কারণ:** Spring Data JPA classpath-এ দেখে auto-configure করতে চাচ্ছে DB connection, কিন্তু আপনি URL দেননি।

**সমাধান (Phase 1-এর জন্য):** `PortopsApplication.java`-এ temporary JPA auto-config exclude করুন:

```java
@SpringBootApplication(exclude = {
    org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration.class,
    org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration.class
})
public class PortopsApplication { ... }
```

Phase 2-এ DB connect করার সময় এই exclude তুলে দেব।

### Problem: Endpoint hit করলে 404
**Checklist:**
1. App কি আসলেই running? Console-এ "Started PortopsApplication" দেখেছেন?
2. URL ঠিক? `http://localhost:8080/api/v1/containers/ping` — slash-গুলো গুনে দেখুন
3. `ContainerController` কি `com.yourcompany.portops` বা তার sub-package-এ আছে? না হলে component scan খুঁজে পাবে না

### Problem: Lombok annotations কাজ করছে না (red underlines)
**সমাধান:** IntelliJ-এ Settings → Build → Compiler → Annotation Processors → "Enable annotation processing" check করুন। তারপর Lombok plugin install করুন (Settings → Plugins → search "Lombok")।

---

## Phase 1 শেষ — আমাকে জানান

এই তিনটা information পাঠান আমাকে:

1. **"Phase 1 done"** — সব checklist ✅ হলে
2. **আপনার final package structure-এর screenshot** (IntelliJ-এর Project view)
3. **কোনো step-এ আটকেছেন কি?** — error message বা confusion থাকলে বলুন

Phase 2-তে আমরা যাব:
- `Container` JPA entity design
- Oracle/MSSQL real connection setup
- `Shipment`, `MotherVessel`, `YardActivity`-এর সাথে relationships (`@ManyToOne`, `@OneToMany`)
- প্রথম real CRUD API with DTOs + validation
- N+1 problem এবং এড়ানোর উপায়

Phase 2 সবচেয়ে dense — ২ দিনের কাজ। তাই Phase 1-এর foundation পাকা থাকলে পরেরটা সহজ হবে।
