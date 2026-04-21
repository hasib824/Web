# Phase 2 — REST API + Data Layer (Vertical Slice)

**Duration:** ২ দিন | মোট ~১২–১৪ ঘণ্টা
**Split:**
- **Day 1 (৬–৭ ঘণ্টা):** Data Layer — DB connection, entities, relationships, repositories
- **Day 2 (৬–৭ ঘণ্টা):** API Layer — DTOs, CRUD controller, validation, pagination, testing

**Phase শেষে আপনার হাতে থাকবে:** সম্পূর্ণ working Container feature — Oracle/MSSQL-এ data persist হচ্ছে, `Container ↔ MotherVessel ↔ YardActivity` relationships wired, full CRUD REST API with validation আর pagination, Postman collection দিয়ে test করা।

---

## Phase 2 Overview

এটাই সবচেয়ে দীর্ঘ phase। কারণ এখানে আপনি Spring Boot-এর **পুরো vertical slice** একবার বানাবেন — HTTP request আসে, DTO-তে validate হয়, Service-এ পৌঁছায়, JPA repository দিয়ে DB-তে persist হয়, আবার response হয়ে client-এ ফেরে। Phase 3, 4, 5 — সব এই slice-এর উপর layer add করবে।

**Android mental model:** এটা ঠিক যেভাবে আপনি Android-এ একটা feature module-এর প্রথমবার full flow বানান — Retrofit → Repository → UseCase → ViewModel → UI। একবার এটা ঠিকঠাক দাঁড়ালে, বাকি features টেমপ্লেট ধরে ধরে বানাতে পারেন।

---

# Day 1 — Data Layer

---

## Step 1 — Oracle/MSSQL Container চালু করুন

Local machine-এ real Oracle/MSSQL install বড় ঝামেলা। Docker দিয়ে চালানোই সহজ।

> **আপনার machine-এ Docker নেই?** → Docker Desktop install করুন: https://www.docker.com/products/docker-desktop
> যদি একদমই install করতে না পারেন, Phase 2-এর শেষে alternative দেওয়া আছে (H2 দিয়ে শুরু, পরে migrate)।

### Option A: Oracle (Free Developer Edition)

Terminal-এ run করুন:

```bash
docker run -d \
  --name portops-oracle \
  -p 1521:1521 \
  -e ORACLE_PASSWORD=PortOps@123 \
  -v portops-oracle-data:/opt/oracle/oradata \
  gvenzl/oracle-free:23-slim
```

Oracle ~2 মিনিট লাগে startup-এ। Ready হলো কিনা check:

```bash
docker logs portops-oracle | grep "DATABASE IS READY"
```

**Connection info:**
- Host: `localhost`
- Port: `1521`
- Service: `FREEPDB1`
- Username: `system`
- Password: `PortOps@123`

### Option B: MSSQL (SQL Server Developer)

```bash
docker run -d \
  --name portops-mssql \
  -p 1433:1433 \
  -e "ACCEPT_EULA=Y" \
  -e "MSSQL_SA_PASSWORD=PortOps@123" \
  mcr.microsoft.com/mssql/server:2022-latest
```

Ready-এর জন্য ~30 সেকেন্ড। Check:

```bash
docker logs portops-mssql | grep "Recovery is complete"
```

**Connection info:**
- Host: `localhost`
- Port: `1433`
- Database: (যেকোনো, আমরা `portops` বানাব)
- Username: `sa`
- Password: `PortOps@123`

MSSQL-এ database বানাতে হবে manually। DBeaver/JetBrains DataGrip খুলে connect করে run করুন:

```sql
CREATE DATABASE portops;
```

### GUI Client (optional কিন্তু recommended)

- **DBeaver** (free, universal) — https://dbeaver.io/
- **JetBrains DataGrip** — IntelliJ Ultimate-এ built-in

Setup-এ connect করে test করুন। দেখতে পেলে এগিয়ে যান।

---

## Step 2 — Spring Boot-কে DB-র সাথে Connect করান

`src/main/resources/application-dev.yml` update করুন।

### Oracle-এর জন্য:

```yaml
spring:
  datasource:
    url: jdbc:oracle:thin:@//localhost:1521/FREEPDB1
    username: system
    password: PortOps@123
    driver-class-name: oracle.jdbc.OracleDriver

  jpa:
    hibernate:
      ddl-auto: create-drop    # dev-only! প্রতি startup-এ schema drop + recreate
    properties:
      hibernate:
        dialect: org.hibernate.dialect.OracleDialect
        format_sql: true
        jdbc:
          batch_size: 50
    show-sql: true             # actual SQL console-এ দেখাবে

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE    # prepared statement parameters দেখাবে
```

### MSSQL-এর জন্য:

```yaml
spring:
  datasource:
    url: jdbc:sqlserver://localhost:1433;databaseName=portops;encrypt=false;trustServerCertificate=true
    username: sa
    password: PortOps@123
    driver-class-name: com.microsoft.sqlserver.jdbc.SQLServerDriver

  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.SQLServerDialect
        format_sql: true
        jdbc:
          batch_size: 50
    show-sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

### Phase 1-এর DataSource Exclude বাদ দিন

Phase 1-এ আপনি `PortopsApplication.java`-তে এটা করেছিলেন (troubleshooting section-এর instruction অনুযায়ী):

```java
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    HibernateJpaAutoConfiguration.class
})
```

এখন এটা সরিয়ে plain করুন:

```java
@SpringBootApplication
public class PortopsApplication {
    public static void main(String[] args) {
        SpringApplication.run(PortopsApplication.class, args);
    }
}
```

### ⚠️ `ddl-auto: create-drop` নিয়ে সতর্কতা

| Value | কখন |
|---|---|
| `create-drop` | Dev/learning — প্রতি startup-এ schema নতুন বানায়। Phase 2-এ এটাই ব্যবহার করব |
| `update` | ❌ কখনো prod-এ না। Silently schema change করে |
| `validate` | Prod-এ। Schema match না করলে startup fail |
| `none` | Migration tool (Flyway/Liquibase) দিয়ে schema control করলে |

**Week 2-এ প্রথম কাজ হবে Flyway integrate করা।** এখন learning-এ focus রাখুন।

---

## Step 3 — Domain Enums বানান

Entity বানানোর আগে enums। `container/domain/` package-এ দুটো enum class:

**`ContainerType.java`:**
```java
package com.yourcompany.portops.container.domain;

public enum ContainerType {
    DRY_20,         // 20-foot standard
    DRY_40,         // 40-foot standard
    DRY_40_HC,      // 40-foot high-cube
    REEFER_20,      // 20-foot refrigerated
    REEFER_40,      // 40-foot refrigerated
    TANK,           // tank container (liquids)
    OPEN_TOP,
    FLAT_RACK
}
```

**`ContainerStatus.java`:**
```java
package com.yourcompany.portops.container.domain;

public enum ContainerStatus {
    ON_VESSEL,      // এখনো mother vessel-এ, offshore
    ON_LIGHTER,     // lighter ship-এ, port-এ আসছে
    AT_GATE,        // port entry gate-এ, inspection
    IN_YARD,        // yard-এ stored
    LOADING,        // export-এর জন্য load হচ্ছে
    EXITED          // port ছেড়ে গেছে
}
```

`yard/domain/` package-এ:

**`YardActivityType.java`:**
```java
package com.yourcompany.portops.yard.domain;

public enum YardActivityType {
    GATE_IN,        // port entry
    YARD_MOVE,      // yard-এর মধ্যে slot change
    INSPECTION,
    GATE_OUT        // port exit
}
```

---

## Step 4 — `MotherVessel` Entity বানান

শুরু করি ছোট entity দিয়ে — no outgoing relationships।

`shipment/domain/` package-এ `MotherVessel.java`:

```java
package com.yourcompany.portops.shipment.domain;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import java.time.Instant;

@Entity
@Table(name = "MOTHER_VESSELS")
@Getter
@Setter
public class MotherVessel {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ID")
    private Long id;

    @Column(name = "IMO_NUMBER", nullable = false, unique = true, length = 10)
    private String imoNumber;       // IMO ship identifier, e.g., "IMO9811000"

    @Column(name = "VESSEL_NAME", nullable = false, length = 100)
    private String vesselName;

    @Column(name = "FLAG_COUNTRY", length = 50)
    private String flagCountry;

    @Column(name = "ARRIVED_AT")
    private Instant arrivedAt;

    @Column(name = "CREATED_AT", nullable = false, updatable = false)
    private Instant createdAt;

    @PrePersist
    void onCreate() {
        this.createdAt = Instant.now();
    }
}
```

### প্রত্যেকটা annotation কী করছে

| Annotation | কাজ |
|---|---|
| `@Entity` | এটা একটা JPA-managed persistent class | 
| `@Table(name = "MOTHER_VESSELS")` | DB table name explicitly দিলাম | 
| `@Id` | Primary key | 
| `@GeneratedValue(strategy = IDENTITY)` | DB auto-increment ব্যবহার করবে (Oracle-এর case-এ Hibernate sequence বানাবে behind the scenes) | 
| `@Column` | Column mapping — name, constraints | 
| `@PrePersist` | Entity save হওয়ার ঠিক আগে এই method call হবে — lifecycle callback | 
| `@Getter`/`@Setter` (Lombok) | compile time-এ getter/setter generate করবে, boilerplate কমায় | 

### Android থেকে mapping

| Android (Room) | Spring (JPA) |
|---|---|
| `@Entity` | `@Entity` (একই নাম, একই কাজ) |
| `@PrimaryKey(autoGenerate = true)` | `@Id` + `@GeneratedValue` |
| `@ColumnInfo(name = "...")` | `@Column(name = "...")` |
| Room-এ relationship: manual junction tables + `@Relation` queries | JPA-এ: `@ManyToOne`, `@OneToMany` directly, Hibernate joins generate করে |

**পার্থক্য:** Room intentionally simple, so আপনি relationship manual handle করেন। JPA অনেক powerful কিন্তু অনেক footgun — lazy loading, cascading, N+1 — এগুলোই আমরা এখন শিখব।

---

## Step 5 — `Container` Entity + ManyToOne Relationship

`container/domain/Container.java`:

```java
package com.yourcompany.portops.container.domain;

import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.yard.domain.YardActivity;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "CONTAINERS",
       indexes = {
           @Index(name = "IDX_CONTAINERS_STATUS", columnList = "STATUS"),
           @Index(name = "IDX_CONTAINERS_BIC_CODE", columnList = "BIC_CODE", unique = true)
       })
@Getter
@Setter
public class Container {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ID")
    private Long id;

    @Column(name = "BIC_CODE", nullable = false, unique = true, length = 11)
    private String bicCode;         // ISO 6346: 4 letters + 7 digits, e.g., "MSCU1234567"

    @Enumerated(EnumType.STRING)
    @Column(name = "TYPE", nullable = false, length = 20)
    private ContainerType type;

    @Enumerated(EnumType.STRING)
    @Column(name = "STATUS", nullable = false, length = 30)
    private ContainerStatus status;

    @Column(name = "GROSS_WEIGHT_KG", nullable = false, precision = 10, scale = 2)
    private BigDecimal grossWeightKg;

    // ManyToOne: অনেক container একটা mother vessel থেকে আসে
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "MOTHER_VESSEL_ID", nullable = false)
    private MotherVessel motherVessel;

    // OneToMany: একটা container-এর অনেক yard activity থাকে (gate-in, moves, gate-out)
    @OneToMany(mappedBy = "container",
               cascade = CascadeType.ALL,
               orphanRemoval = true,
               fetch = FetchType.LAZY)
    private List<YardActivity> yardActivities = new ArrayList<>();

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

    // Helper method — always use this to add YardActivity
    public void addYardActivity(YardActivity activity) {
        yardActivities.add(activity);
        activity.setContainer(this);
    }
}
```

### গুরুত্বপূর্ণ concept: `@ManyToOne` vs `@OneToMany` — কোনটা owning side?

JPA relationships-এ **একটা side "owning"**, অন্যটা "inverse"। Owning side-ই DB-তে foreign key column রাখে।

**Rule:** যে side-এ foreign key আছে, সেটা owning side।

আমাদের case-এ:
- `Container` table-এ `MOTHER_VESSEL_ID` column আছে → Container is owning side of this relationship
- `Container`-এ `@ManyToOne @JoinColumn(name = "MOTHER_VESSEL_ID")` — এটাই owning declaration
- `MotherVessel`-এ যদি `@OneToMany(mappedBy = "motherVessel")` রাখতে চাই (inverse side), সেটা optional। আমরা রাখিনি কারণ একটা mother vessel-এর সব container list দরকার সাধারণত না (performance)

### `mappedBy` কী মানে?

```java
@OneToMany(mappedBy = "container", ...)
private List<YardActivity> yardActivities;
```

এটা Hibernate-কে বলছে: "এই relationship-এর owning side হচ্ছে `YardActivity.container` field। তুমি আলাদা join table বানিয়ো না — `YARD_ACTIVITIES` table-এর `CONTAINER_ID` column-টাই এই relationship-এর foreign key।"

**ভুল যেটা করবেন না:** `mappedBy` ছাড়া `@OneToMany` দিলে Hibernate silently একটা extra join table বানাবে (`CONTAINERS_YARD_ACTIVITIES`) — যেটা আপনি চাননি, আর debug করা ঝামেলা।

### `CascadeType.ALL` + `orphanRemoval = true` — সাবধানে

```java
@OneToMany(mappedBy = "container", cascade = CascadeType.ALL, orphanRemoval = true)
```

- `cascade = ALL`: parent save/delete করলে children-ও save/delete হবে
- `orphanRemoval = true`: parent থেকে child list-এ একটা item remove করলে, সেটা DB থেকেও delete হবে

এটা ঠিক যখন **child-এর independent existence নেই** — `YardActivity` একটা `Container` ছাড়া meaningless। কিন্তু `Container`-এর `MotherVessel`-এ `cascade = ALL` দিলে বিপদ — container delete করলে vessel-ও delete হয়ে যাবে, যেটা আপনি চান না।

**Rule of thumb:** `ManyToOne`-এ cascade avoid করুন। `OneToMany`-তে (parent-child, ownership সম্পর্ক হলে) cascade + orphanRemoval দিন।

### `FetchType.LAZY` কেন সবসময়

- `ManyToOne` default `EAGER` — প্রতি Container fetch-এ automatically vessel-ও load হবে। 100 container = 101 query (N+1!)
- `OneToMany` default `LAZY` — ভালো, কিন্তু explicit লিখলে ভবিষ্যতে কেউ confusion-এ পড়বে না

**আমার recommendation:** সব relationship explicitly `LAZY`। যেখানে data লাগবে, `JOIN FETCH` দিয়ে specific query-তে load করবেন।

---

## Step 6 — `YardActivity` Entity

`yard/domain/YardActivity.java`:

```java
package com.yourcompany.portops.yard.domain;

import com.yourcompany.portops.container.domain.Container;
import jakarta.persistence.*;
import lombok.Getter;
import lombok.Setter;
import java.time.Instant;

@Entity
@Table(name = "YARD_ACTIVITIES",
       indexes = {
           @Index(name = "IDX_YARD_ACT_CONTAINER", columnList = "CONTAINER_ID"),
           @Index(name = "IDX_YARD_ACT_OCCURRED", columnList = "OCCURRED_AT")
       })
@Getter
@Setter
public class YardActivity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "ID")
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "CONTAINER_ID", nullable = false)
    private Container container;

    @Enumerated(EnumType.STRING)
    @Column(name = "ACTIVITY_TYPE", nullable = false, length = 20)
    private YardActivityType activityType;

    @Column(name = "YARD_SLOT_CODE", length = 20)
    private String yardSlotCode;    // e.g., "A-07-03-2" → block-A, row-07, bay-03, tier-2

    @Column(name = "OCCURRED_AT", nullable = false)
    private Instant occurredAt;

    @Column(name = "REMARKS", length = 500)
    private String remarks;
}
```

এটা Container-এর সাথে ManyToOne সম্পর্ক — `YARD_ACTIVITIES.CONTAINER_ID` foreign key।

---

## Step 7 — Repositories বানান

Spring Data JPA-এর সবচেয়ে powerful feature এখানে: **interface declare করলেই Spring implementation বানিয়ে দেয়।**

`container/persistence/ContainerRepository.java`:

```java
package com.yourcompany.portops.container.persistence;

import com.yourcompany.portops.container.domain.Container;
import com.yourcompany.portops.container.domain.ContainerStatus;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface ContainerRepository extends JpaRepository<Container, Long> {

    // Derived query — method name থেকে Spring SQL generate করে
    Optional<Container> findByBicCode(String bicCode);

    boolean existsByBicCode(String bicCode);

    Page<Container> findByStatus(ContainerStatus status, Pageable pageable);

    List<Container> findByMotherVesselId(Long motherVesselId);

    // Custom JPQL query with JOIN FETCH — N+1 avoid করার জন্য
    @Query("""
        SELECT c FROM Container c
        LEFT JOIN FETCH c.motherVessel
        WHERE c.status = :status
    """)
    List<Container> findByStatusWithVessel(@Param("status") ContainerStatus status);
}
```

`shipment/persistence/MotherVesselRepository.java`:

```java
package com.yourcompany.portops.shipment.persistence;

import com.yourcompany.portops.shipment.domain.MotherVessel;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;

@Repository
public interface MotherVesselRepository extends JpaRepository<MotherVessel, Long> {
    Optional<MotherVessel> findByImoNumber(String imoNumber);
}
```

`yard/persistence/YardActivityRepository.java`:

```java
package com.yourcompany.portops.yard.persistence;

import com.yourcompany.portops.yard.domain.YardActivity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface YardActivityRepository extends JpaRepository<YardActivity, Long> {
    List<YardActivity> findByContainerIdOrderByOccurredAtDesc(Long containerId);
}
```

### এখানে magic কী?

- আপনি **implementation লেখেননি**। Spring Data JPA runtime-এ proxy বানায় যেটা method name parse করে JPQL generate করে।
- `findByBicCode` → `SELECT c FROM Container c WHERE c.bicCode = ?`
- `findByStatusAndMotherVesselId` → `SELECT c FROM Container c WHERE c.status = ? AND c.motherVessel.id = ?`
- Rules: https://docs.spring.io/spring-data/jpa/reference/jpa/query-methods.html

### `JpaRepository<Container, Long>` কী দেয় free-তে

- `save(entity)` — insert বা update
- `findById(id)` — Optional<Container>
- `findAll()`, `findAll(Pageable)`, `findAll(Sort)`
- `deleteById(id)`, `delete(entity)`, `deleteAll()`
- `count()`, `existsById(id)`

### Android থেকে mapping

Room DAO-তে আপনি interface লিখে `@Query` দিতেন, annotation processor implementation generate করত। Spring Data JPA একই pattern, কিন্তু runtime proxy ব্যবহার করে + অনেক বেশি method auto-derivation।

| Room | Spring Data JPA |
|---|---|
| `@Dao interface ContainerDao` | `interface ContainerRepository extends JpaRepository<Container, Long>` |
| `@Query("SELECT * FROM containers WHERE bic_code = :code")` | `Optional<Container> findByBicCode(String bicCode);` (no query string needed!) |
| `@Insert suspend fun insert(c: Container)` | `containerRepository.save(container)` (free, no declaration) |
| Manual pagination | Built-in `Page<T>` with `Pageable` parameter |

---

## Step 8 — App Run করে Schema Generate দেখুন

এখন app start করুন। Console-এ যা দেখবেন:

```
... HikariPool-1 - Starting...
... HikariPool-1 - Added connection ...
... HHH000204: Processing PersistenceUnitInfo [name: default]
... drop table if exists CONTAINERS cascade
... create table CONTAINERS (ID number(19,0) not null, BIC_CODE varchar2(11 char) not null, ...)
... create table MOTHER_VESSELS (...)
... create table YARD_ACTIVITIES (...)
... alter table CONTAINERS add constraint FK... foreign key (MOTHER_VESSEL_ID) references MOTHER_VESSELS
... Started PortopsApplication in 4.231 seconds
```

✅ Tables generated, foreign keys wired। DBeaver/DataGrip দিয়ে DB-তে গিয়ে verify করুন — তিনটা table থাকবে proper columns ও FK সহ।

### Day 1 Checkpoint

নিচের সব হলে Day 1 পূর্ণ:

- [ ] Docker-এ Oracle/MSSQL চলছে
- [ ] Spring Boot app DB-তে connect হয়
- [ ] Startup log-এ `CREATE TABLE` statements আসে
- [ ] DB client দিয়ে `CONTAINERS`, `MOTHER_VESSELS`, `YARD_ACTIVITIES` দেখা যায়
- [ ] Foreign keys proper — Container → MotherVessel, YardActivity → Container

এখানে আটকালে আমাকে জানান, নয়তো Day 2-তে চলুন।

---

# Day 2 — API Layer

---

## Step 9 — DTOs কেন লাগে (এবং Entity expose কেন বিপদ)

**প্রলোভন:** "আমার কাছে Container entity আছে, সরাসরি controller থেকে return করি, DTO বানানোর কী দরকার?"

**যে সমস্যাগুলো হবে:**

1. **LazyInitializationException** — Controller থেকে `Container` return করলে Jackson JSON serialize করতে গিয়ে lazy `motherVessel` field access করবে। কিন্তু তখন transaction শেষ, Hibernate session closed → exception।

2. **Infinite loop** — `Container.yardActivities` serialize করতে গিয়ে প্রত্যেক `YardActivity.container` serialize করবে, আবার সেটা তার activities... infinite recursion।

3. **Over-exposure** — internal field (audit timestamp, soft-delete flag, internal IDs) সব API-তে leak।

4. **API বদলালে DB বদলাতে হবে** — আপনি field rename করলেন `grossWeightKg` → `weightKg`, সাথে সাথে DB column rename করতে হবে। Coupling।

**সমাধান:** আলাদা DTO class, controller-এ mapping।

---

## Step 10 — Container DTOs বানান

`container/api/dto/` package বানান। চারটা DTO:

**`CreateContainerRequest.java`:**
```java
package com.yourcompany.portops.container.api.dto;

import com.yourcompany.portops.container.domain.ContainerType;
import jakarta.validation.constraints.*;
import java.math.BigDecimal;

public record CreateContainerRequest(
    @NotBlank(message = "BIC code is required")
    @Pattern(regexp = "[A-Z]{4}[0-9]{7}",
             message = "BIC code must be 4 letters followed by 7 digits")
    String bicCode,

    @NotNull(message = "Container type is required")
    ContainerType type,

    @NotNull(message = "Gross weight is required")
    @DecimalMin(value = "0.0", inclusive = false, message = "Weight must be positive")
    @DecimalMax(value = "40000.00", message = "Weight exceeds container capacity")
    BigDecimal grossWeightKg,

    @NotNull(message = "Mother vessel ID is required")
    @Positive
    Long motherVesselId
) {}
```

**`UpdateContainerStatusRequest.java`:**
```java
package com.yourcompany.portops.container.api.dto;

import com.yourcompany.portops.container.domain.ContainerStatus;
import jakarta.validation.constraints.NotNull;

public record UpdateContainerStatusRequest(
    @NotNull ContainerStatus newStatus
) {}
```

**`ContainerResponse.java`:**
```java
package com.yourcompany.portops.container.api.dto;

import com.yourcompany.portops.container.domain.ContainerStatus;
import com.yourcompany.portops.container.domain.ContainerType;
import java.math.BigDecimal;
import java.time.Instant;

public record ContainerResponse(
    Long id,
    String bicCode,
    ContainerType type,
    ContainerStatus status,
    BigDecimal grossWeightKg,
    VesselSummary motherVessel,
    Instant createdAt,
    Instant updatedAt
) {
    public record VesselSummary(Long id, String imoNumber, String vesselName) {}
}
```

**`ContainerSummary.java`** (lightweight, list-এর জন্য):
```java
package com.yourcompany.portops.container.api.dto;

import com.yourcompany.portops.container.domain.ContainerStatus;
import com.yourcompany.portops.container.domain.ContainerType;

public record ContainerSummary(
    Long id,
    String bicCode,
    ContainerType type,
    ContainerStatus status
) {}
```

### Java `record` কী?

Java 14+ feature — immutable data class। Kotlin `data class`-এর সমতুল্য:

```kotlin
// Kotlin
data class ContainerResponse(val id: Long, val bicCode: String)
```

```java
// Java record (same thing, auto-generates equals/hashCode/toString + accessor methods)
public record ContainerResponse(Long id, String bicCode) {}
```

- No setters (immutable by design — DTOs-এ ideal)
- Accessor = `response.id()`, `response.bicCode()` (no `get` prefix)

---

## Step 11 — Mapper বানান (Manual for now)

`container/api/mapper/ContainerMapper.java`:

```java
package com.yourcompany.portops.container.api.mapper;

import com.yourcompany.portops.container.api.dto.*;
import com.yourcompany.portops.container.domain.Container;

public final class ContainerMapper {

    private ContainerMapper() {}  // utility class, no instances

    public static ContainerResponse toResponse(Container entity) {
        var vesselSummary = entity.getMotherVessel() == null ? null :
            new ContainerResponse.VesselSummary(
                entity.getMotherVessel().getId(),
                entity.getMotherVessel().getImoNumber(),
                entity.getMotherVessel().getVesselName()
            );

        return new ContainerResponse(
            entity.getId(),
            entity.getBicCode(),
            entity.getType(),
            entity.getStatus(),
            entity.getGrossWeightKg(),
            vesselSummary,
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }

    public static ContainerSummary toSummary(Container entity) {
        return new ContainerSummary(
            entity.getId(),
            entity.getBicCode(),
            entity.getType(),
            entity.getStatus()
        );
    }
}
```

Week 2-এ আমরা এটা **MapStruct** দিয়ে replace করব — compile-time generation, no runtime overhead। এখন manual ভালো কারণ আপনি দেখতে পাচ্ছেন exactly কী map হচ্ছে।

---

## Step 12 — `ContainerService` বানান

`container/domain/ContainerService.java` (Phase 1-এর stub replace করুন):

```java
package com.yourcompany.portops.container.domain;

import com.yourcompany.portops.container.api.dto.*;
import com.yourcompany.portops.container.api.mapper.ContainerMapper;
import com.yourcompany.portops.container.persistence.ContainerRepository;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.shipment.persistence.MotherVesselRepository;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional     // class-level default — সব public method transactional
public class ContainerService {

    private final ContainerRepository containerRepository;
    private final MotherVesselRepository motherVesselRepository;

    public ContainerService(ContainerRepository containerRepository,
                            MotherVesselRepository motherVesselRepository) {
        this.containerRepository = containerRepository;
        this.motherVesselRepository = motherVesselRepository;
    }

    public ContainerResponse register(CreateContainerRequest request) {
        // Duplicate check
        if (containerRepository.existsByBicCode(request.bicCode())) {
            throw new IllegalStateException(
                "Container with BIC code " + request.bicCode() + " already exists");
        }

        // Vessel lookup
        MotherVessel vessel = motherVesselRepository.findById(request.motherVesselId())
            .orElseThrow(() -> new IllegalArgumentException(
                "Mother vessel not found: " + request.motherVesselId()));

        // Build entity
        Container container = new Container();
        container.setBicCode(request.bicCode());
        container.setType(request.type());
        container.setStatus(ContainerStatus.ON_VESSEL);  // newly registered container
        container.setGrossWeightKg(request.grossWeightKg());
        container.setMotherVessel(vessel);

        Container saved = containerRepository.save(container);
        return ContainerMapper.toResponse(saved);
    }

    @Transactional(readOnly = true)
    public ContainerResponse findById(Long id) {
        return containerRepository.findById(id)
            .map(ContainerMapper::toResponse)
            .orElseThrow(() -> new IllegalArgumentException("Container not found: " + id));
    }

    @Transactional(readOnly = true)
    public Page<ContainerSummary> search(ContainerStatus status, Pageable pageable) {
        Page<Container> page = (status != null)
            ? containerRepository.findByStatus(status, pageable)
            : containerRepository.findAll(pageable);
        return page.map(ContainerMapper::toSummary);
    }

    public ContainerResponse updateStatus(Long id, UpdateContainerStatusRequest request) {
        Container container = containerRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Container not found: " + id));
        container.setStatus(request.newStatus());
        // save() call লাগবে না — managed entity, transaction commit-এ auto-flush হবে
        return ContainerMapper.toResponse(container);
    }

    public void delete(Long id) {
        if (!containerRepository.existsById(id)) {
            throw new IllegalArgumentException("Container not found: " + id);
        }
        containerRepository.deleteById(id);
    }
}
```

### এখানে কয়েকটা subtle জিনিস

**1. `@Transactional(readOnly = true)` on query methods**
Hibernate dirty-checking skip করে (performance gain), আর intent clearly express করে।

**2. `updateStatus` method-এ save() call নেই**
Entity যদি managed থাকে (transaction-এ `findById` থেকে এসেছে), field change করলে transaction commit-এ Hibernate automatically UPDATE query fire করে। এটাকে বলে **"dirty checking"**। Room-এ এই magic নেই — আপনাকে explicit update call করতে হতো।

**3. Exception হ্যান্ডলিং এখন crude**
`IllegalArgumentException`, `IllegalStateException` throw করছি। Phase 3-এ এগুলোকে proper domain exceptions দিয়ে replace করব (`ContainerNotFoundException`, `DuplicateBicCodeException`) আর global handler দিয়ে clean HTTP response দেব। এখন just focus data flow-এ।

---

## Step 13 — `ContainerController` Full CRUD

Phase 1-এর ping/health endpoints রেখে বাকি add করুন:

```java
package com.yourcompany.portops.container.api;

import com.yourcompany.portops.container.api.dto.*;
import com.yourcompany.portops.container.domain.ContainerService;
import com.yourcompany.portops.container.domain.ContainerStatus;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;

@RestController
@RequestMapping("/api/v1/containers")
public class ContainerController {

    private final ContainerService containerService;

    public ContainerController(ContainerService containerService) {
        this.containerService = containerService;
    }

    @GetMapping("/ping")
    public String ping() {
        return "Container module alive";
    }

    @PostMapping
    public ResponseEntity<ContainerResponse> create(
            @Valid @RequestBody CreateContainerRequest request) {
        ContainerResponse created = containerService.register(request);
        URI location = URI.create("/api/v1/containers/" + created.id());
        return ResponseEntity.created(location).body(created);
    }

    @GetMapping("/{id}")
    public ContainerResponse getById(@PathVariable Long id) {
        return containerService.findById(id);
    }

    @GetMapping
    public Page<ContainerSummary> search(
            @RequestParam(required = false) ContainerStatus status,
            Pageable pageable) {
        return containerService.search(status, pageable);
    }

    @PatchMapping("/{id}/status")
    public ContainerResponse updateStatus(
            @PathVariable Long id,
            @Valid @RequestBody UpdateContainerStatusRequest request) {
        return containerService.updateStatus(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        containerService.delete(id);
    }
}
```

### HTTP conventions যা follow করলাম

| Method | Path | Status | Use case |
|---|---|---|---|
| POST | `/containers` | 201 Created + Location header | নতুন container register |
| GET | `/containers/{id}` | 200 OK | একটা container দেখা |
| GET | `/containers?status=IN_YARD&page=0&size=20` | 200 OK | filtered + paginated list |
| PATCH | `/containers/{id}/status` | 200 OK | শুধু status field update (partial) |
| DELETE | `/containers/{id}` | 204 No Content | delete |

**`PATCH` vs `PUT`:** `PUT` পুরো entity replace করে; `PATCH` partial update। Status-only update হলে `PATCH` সঠিক। React team-ও ঠিক API-তে expect করবে।

---

## Step 14 — MotherVessel-এর জন্যও Minimal API

Container create করতে vessel লাগবে। তাই একটা simple vessel create endpoint।

`shipment/api/dto/CreateVesselRequest.java`:
```java
package com.yourcompany.portops.shipment.api.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;

public record CreateVesselRequest(
    @NotBlank @Pattern(regexp = "IMO\\d{7}") String imoNumber,
    @NotBlank String vesselName,
    String flagCountry
) {}
```

`shipment/api/dto/VesselResponse.java`:
```java
package com.yourcompany.portops.shipment.api.dto;

import java.time.Instant;

public record VesselResponse(
    Long id,
    String imoNumber,
    String vesselName,
    String flagCountry,
    Instant createdAt
) {}
```

`shipment/api/MotherVesselController.java`:
```java
package com.yourcompany.portops.shipment.api;

import com.yourcompany.portops.shipment.api.dto.*;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.shipment.persistence.MotherVesselRepository;
import jakarta.validation.Valid;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.net.URI;
import java.util.List;

@RestController
@RequestMapping("/api/v1/vessels")
public class MotherVesselController {

    private final MotherVesselRepository repository;

    public MotherVesselController(MotherVesselRepository repository) {
        this.repository = repository;
    }

    @PostMapping
    public ResponseEntity<VesselResponse> create(@Valid @RequestBody CreateVesselRequest request) {
        MotherVessel v = new MotherVessel();
        v.setImoNumber(request.imoNumber());
        v.setVesselName(request.vesselName());
        v.setFlagCountry(request.flagCountry());
        MotherVessel saved = repository.save(v);

        VesselResponse body = toResponse(saved);
        return ResponseEntity.created(URI.create("/api/v1/vessels/" + saved.getId())).body(body);
    }

    @GetMapping
    public List<VesselResponse> list() {
        return repository.findAll().stream().map(this::toResponse).toList();
    }

    private VesselResponse toResponse(MotherVessel v) {
        return new VesselResponse(v.getId(), v.getImoNumber(), v.getVesselName(),
                                  v.getFlagCountry(), v.getCreatedAt());
    }
}
```

> ⚠️ এটা intentionally repository সরাসরি controller-এ inject করেছি — quick shortcut for test data setup। **Production code-এ service layer skip করবেন না।** Phase 3-এ Container-এর pattern-এ vessel-এর জন্যও proper service বানাবেন।

---

## Step 15 — Postman-এ Test করুন

App restart করুন। Postman খুলে এই requests run করুন **exactly এই order-এ**:

### 1. Vessel বানান

```
POST http://localhost:8080/api/v1/vessels
Content-Type: application/json

{
  "imoNumber": "IMO9811000",
  "vesselName": "MAERSK HONAM",
  "flagCountry": "Singapore"
}
```

Expected: `201 Created`, response body-তে `id: 1`।

### 2. Container register করুন

```
POST http://localhost:8080/api/v1/containers
Content-Type: application/json

{
  "bicCode": "MSCU1234567",
  "type": "DRY_40",
  "grossWeightKg": 24500.00,
  "motherVesselId": 1
}
```

Expected: `201 Created`, `Location: /api/v1/containers/1` header, response body-তে nested `motherVessel` object।

### 3. Invalid request পাঠিয়ে validation test করুন

```
POST http://localhost:8080/api/v1/containers
Content-Type: application/json

{
  "bicCode": "invalid",
  "type": "DRY_40",
  "grossWeightKg": -100,
  "motherVesselId": 1
}
```

Expected: `400 Bad Request`। Response body-তে Spring-এর default error structure (Phase 3-এ এটা আমরা improve করব)।

### 4. Container retrieve করুন

```
GET http://localhost:8080/api/v1/containers/1
```

Expected: `200 OK`, full ContainerResponse with nested vessel summary।

### 5. Status update করুন

```
PATCH http://localhost:8080/api/v1/containers/1/status
Content-Type: application/json

{
  "newStatus": "IN_YARD"
}
```

Expected: `200 OK`, updated status।

### 6. Listing with pagination

আরো ৪-৫টা container বানান (different BIC codes), তারপর:

```
GET http://localhost:8080/api/v1/containers?page=0&size=3&sort=createdAt,desc
```

Expected response structure:
```json
{
  "content": [
    { "id": 5, "bicCode": "...", "type": "...", "status": "..." },
    { "id": 4, ... },
    { "id": 3, ... }
  ],
  "pageable": { "pageNumber": 0, "pageSize": 3, ... },
  "totalElements": 5,
  "totalPages": 2,
  "first": true,
  "last": false
}
```

### 7. Filter by status

```
GET http://localhost:8080/api/v1/containers?status=IN_YARD
```

শুধু `IN_YARD` status-এর container আসবে।

### 8. Delete

```
DELETE http://localhost:8080/api/v1/containers/1
```

Expected: `204 No Content`, empty body।

---

## Step 16 — N+1 Problem লাইভ দেখুন এবং Fix করুন

এটা JPA-এর সবচেয়ে famous footgun। Live demo করি।

### Problem দেখুন

একটা debug endpoint temporary add করুন `ContainerController`-এ:

```java
@GetMapping("/debug/all")
public List<ContainerResponse> debugAll() {
    return containerService.findAllDebug();
}
```

`ContainerService`-এ:

```java
@Transactional(readOnly = true)
public List<ContainerResponse> findAllDebug() {
    return containerRepository.findAll().stream()
        .map(ContainerMapper::toResponse)
        .toList();
}
```

DB-তে ৫টা container আছে ধরে নিন। Console-এ `show-sql: true` active থাকায় SQL দেখতে পাবেন। Endpoint hit করুন এবং SQL count করুন:

```
Hibernate: select c1_0.ID, c1_0.BIC_CODE, ... from CONTAINERS c1_0   ← 1টা query
Hibernate: select mv1_0.ID, mv1_0.VESSEL_NAME, ... from MOTHER_VESSELS mv1_0 where mv1_0.ID=?  ← container 1 এর vessel
Hibernate: select mv1_0.ID, ... from MOTHER_VESSELS mv1_0 where mv1_0.ID=?  ← container 2 এর vessel
Hibernate: select mv1_0.ID, ... from MOTHER_VESSELS mv1_0 where mv1_0.ID=?  ← container 3 এর vessel
Hibernate: select mv1_0.ID, ... from MOTHER_VESSELS mv1_0 where mv1_0.ID=?  ← container 4 এর vessel
Hibernate: select mv1_0.ID, ... from MOTHER_VESSELS mv1_0 where mv1_0.ID=?  ← container 5 এর vessel
```

**১টা query + N টা query = N+1 problem।** 100 containers হলে 101 queries, 1000 হলে 1001। DB-তে roundtrip নষ্ট।

### কেন হলো?

`Container.motherVessel` field-এ `FetchType.LAZY` ছিল। `findAll()` শুধু containers fetch করল। তারপর mapper-এ `entity.getMotherVessel().getImoNumber()` access করতে গেলে Hibernate lazy-load trigger করল — প্রত্যেক container-এর জন্য আলাদা query।

### Fix 1: JOIN FETCH (একটাই query)

`ContainerRepository`-তে add করুন:

```java
@Query("""
    SELECT c FROM Container c
    LEFT JOIN FETCH c.motherVessel
""")
List<Container> findAllWithVessel();
```

Service method update:

```java
public List<ContainerResponse> findAllDebug() {
    return containerRepository.findAllWithVessel().stream()
        .map(ContainerMapper::toResponse)
        .toList();
}
```

আবার endpoint hit করুন। Console-এ এখন **একটাই query**:

```
Hibernate: select c1_0.ID, ..., mv1_0.ID, mv1_0.VESSEL_NAME, ... from CONTAINERS c1_0 left join MOTHER_VESSELS mv1_0 on mv1_0.ID=c1_0.MOTHER_VESSEL_ID
```

### Fix 2: `@EntityGraph` (annotation-based)

Alternative syntax যেটা cleaner:

```java
@EntityGraph(attributePaths = {"motherVessel"})
@Override
List<Container> findAll();
```

একই effect। Choice of preference।

### কখন N+1 চিন্তা করবেন

**Rule:** যখনই list return করছেন এবং সেই list-এর items-এর lazy association access করা হবে, JOIN FETCH বা EntityGraph ব্যবহার করুন।

- Single entity fetch (`findById`) → lazy লোড usually ঠিক
- List/Page fetch + association access → JOIN FETCH/EntityGraph মাস্ট

Debug endpoint এখন delete করে দিন — ওটা শুধু N+1 demo-এর জন্য ছিল।

---

## Phase 2 Milestone Checklist

- [ ] Docker-এ Oracle/MSSQL running
- [ ] Spring Boot DB-তে connect করে schema auto-generate করছে
- [ ] DB-তে 3 tables visible (CONTAINERS, MOTHER_VESSELS, YARD_ACTIVITIES) proper FK সহ
- [ ] `POST /api/v1/vessels` → 201 Created
- [ ] `POST /api/v1/containers` → 201 Created with Location header
- [ ] Invalid BIC code / negative weight → 400 Bad Request (validation working)
- [ ] `GET /api/v1/containers/{id}` → full response with nested vessel
- [ ] `GET /api/v1/containers?status=IN_YARD&page=0&size=10` → proper Page response
- [ ] `PATCH /api/v1/containers/{id}/status` → status update, auto dirty-check save
- [ ] `DELETE /api/v1/containers/{id}` → 204 No Content
- [ ] N+1 problem দেখেছেন, JOIN FETCH দিয়ে fix করেছেন

---

## Self-Check Questions (Phase 3-এ যাওয়ার আগে)

1. `@ManyToOne` relationship-এ owning side কে? `@JoinColumn` কোথায় বসে?
2. `mappedBy` attribute কী কাজ করে — না দিলে কী হয়?
3. Entity সরাসরি controller থেকে return করলে কী ৩টা সমস্যা হতে পারে?
4. `FetchType.LAZY` আর `EAGER`-এর পার্থক্য — default কোনটা `ManyToOne`-এ, কোনটা `OneToMany`-তে?
5. N+1 query problem কী — কখন হয়, কীভাবে detect করবেন, কীভাবে fix করবেন?
6. Managed entity-এর field change করলে `save()` call ছাড়াই DB update হয় কেন?
7. `@Transactional(readOnly = true)` query method-এ দিলে কী benefit?
8. Spring Data JPA-এর derived query method (e.g., `findByStatusAndMotherVesselId`) কীভাবে কাজ করে?

---

## Troubleshooting

### Problem: `ORA-00942: table or view does not exist`
**কারণ:** Oracle user-এর permission কম, বা schema mismatch।
**সমাধান:** `application-dev.yml`-এ explicit schema set করুন:
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_schema: SYSTEM
```

### Problem: `LazyInitializationException: could not initialize proxy`
**কারণ:** Transaction-এর বাইরে lazy field access।
**সমাধান:** mapping logic Service-এর `@Transactional` method-এর ভিতরে রাখুন, অথবা `JOIN FETCH` দিয়ে eager load করুন।

### Problem: Validation annotation কাজ করছে না
**Checklist:**
1. Controller parameter-এ `@Valid` আছে?
2. `spring-boot-starter-validation` dependency আছে `pom.xml`-এ?
3. Import ঠিক? `jakarta.validation.constraints.*` (not `javax.validation`)

### Problem: `JSON parse error: Cannot construct instance`
**কারণ:** Jackson record-এর constructor resolve করতে পারছে না।
**সমাধান:** Spring Boot 3.x-এ by default কাজ করে। পুরানো version হলে Jackson parameter-names module add করুন।

### Problem: Pageable request-এ sort কাজ করছে না
**সমাধান:** field name exact match হতে হবে entity field-এর সাথে। `?sort=createdAt,desc` — DB column `CREATED_AT` না, entity field `createdAt`।

### Problem: Startup-এ `Schema-validation: missing table`
**কারণ:** `ddl-auto: validate` আছে, কিন্তু schema তৈরি নেই।
**সমাধান:** Dev profile-এ `ddl-auto: create-drop` আছে তা verify করুন।

---

## Phase 2 Wrap-up

এই ২ দিনে আপনি Spring Boot-এর **মূল vertical slice** বানিয়ে ফেলেছেন। এবার প্রতিটা feature (Shipment, Billing, Yard Operations) একই pattern-এ বানাতে পারবেন:

1. Entity + relationships
2. Repository (interface only)
3. DTOs (request/response)
4. Mapper (manual এখন, MapStruct পরে)
5. Service with `@Transactional`
6. Controller with proper HTTP verbs + validation

### Phase 3-এ আমরা যাব

- এই crude `IllegalArgumentException`-গুলো proper domain exceptions-এ refactor
- `@RestControllerAdvice` + `ProblemDetail` — clean, consistent API error responses
- Business logic Service-এ — `ContainerMovementService.gateIn()` atomic operation
- Transaction propagation, rollback rules
- DTO mapping-এ MapStruct introduction
- Structured logging

### আমাকে যা পাঠাবেন Phase 2 শেষে

1. **"Phase 2 done"** confirmation
2. **Postman screenshot** — একটা successful `POST /containers` response দেখিয়ে (status 201, body সহ)
3. **Console log snippet** — N+1 problem demo + JOIN FETCH fix-এর SQL comparison
4. **কোথাও আটকালে** — full error stacktrace + আপনার yml config

আপনার codebase এখন Phase 2 শেষে যা দেখাবে সেটাই এনটারপ্রাইজ Spring Boot project-এর baseline। Phase 3 থেকে আমরা production-quality তে নিয়ে যাব।

শুরু করুন। আটকালে জানান।
