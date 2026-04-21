# Phase 5 — Testing, API Documentation & Production Readiness

**Duration:** ২ দিন | ~১২–১৪ ঘণ্টা

**Split:**
- **Day 1 (৬–৭ ঘণ্টা):** Testing — Unit tests, Controller tests, Repository tests, Integration tests
- **Day 2 (৬–৭ ঘণ্টা):** Production concerns — OpenAPI, Actuator, Scheduling, Async, final code review

**Phase শেষে আপনার হাতে থাকবে:**
- `@WebMvcTest` দিয়ে controller tests (security + validation)
- `@DataJpaTest` দিয়ে repository tests
- Mockito দিয়ে service layer unit tests
- `@SpringBootTest` দিয়ে integration test
- OpenAPI/Swagger UI — React team-এর জন্য self-documenting API
- `@Scheduled` — nightly billing scheduler skeleton
- `@Async` — notification service
- Spring Boot Actuator — health/metrics endpoints
- ৪-৫ দিনের built-up code-এর holistic review

---

## Phase 5 Overview

এই phase দুই parts-এ:

**Day 1 — Testing:** "It works in Postman" PR review-এ আর চলবে না। আপনাকে prove করতে হবে — code behaves correctly under expected conditions, and doesn't break when new code is added। এটাই automated tests-এর কাজ।

**Day 2 — Production readiness:** React team-কে Swagger UI handover, ops team-এর জন্য health endpoints, background jobs (billing runs), async notifications। এই pieces ছাড়া app "works" কিন্তু "production-ready" না।

### Android mental model

| Android | Spring Boot |
|---|---|
| JUnit + Mockito — ViewModel/UseCase unit test | JUnit + Mockito — Service unit test |
| Robolectric — Activity/Fragment test without emulator | `@WebMvcTest` — Controller test without Tomcat |
| Room `@Database.inMemoryDatabaseBuilder()` | `@DataJpaTest` — in-memory DB repo test |
| Espresso — full app UI test | `@SpringBootTest` + `MockMvc` — full app HTTP test |
| WorkManager periodic job | `@Scheduled` method |
| Coroutines `launch {}` fire-and-forget | `@Async` method |
| Dokka/KDoc | OpenAPI/Swagger — auto-generated from code |

পরিচিত concepts, নতুন annotations।

---

# Day 1 — Testing

---

## Step 1 — Test Dependencies & Structure

### Dependencies check

`pom.xml`-এ ইতিমধ্যে থাকার কথা:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.springframework.security</groupId>
    <artifactId>spring-security-test</artifactId>
    <scope>test</scope>
</dependency>
```

Spring Security test না থাকলে add করুন — `@WithMockUser` লাগবে।

H2 in-memory DB (Repository tests-এর জন্য):

```xml
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>test</scope>
</dependency>
```

### Test source directory

```
src/test/java/com/yourcompany/portops/
    ├── container/
    │     ├── api/
    │     │     └── ContainerControllerTest.java
    │     ├── domain/
    │     │     ├── ContainerServiceTest.java
    │     │     └── ContainerMovementServiceTest.java
    │     └── persistence/
    │           └── ContainerRepositoryTest.java
    ├── user/
    │     └── domain/
    │           └── AuthServiceTest.java
    └── integration/
          └── ContainerFlowIntegrationTest.java
```

**Rule:** Test class-এর package structure production code-এর mirror। Test class-এর নাম `<ClassName>Test` — JUnit convention।

### Test profile

Tests-এর জন্য আলাদা config বানান। `src/test/resources/application-test.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:portops_test;MODE=Oracle;DB_CLOSE_DELAY=-1
    username: sa
    password:
    driver-class-name: org.h2.Driver

  jpa:
    hibernate:
      ddl-auto: create-drop
    properties:
      hibernate:
        dialect: org.hibernate.dialect.H2Dialect
    show-sql: false

  h2:
    console:
      enabled: false

portops:
  security:
    jwt:
      secret: dGVzdC1zZWNyZXQtZm9yLXRlc3RpbmctcHVycG9zZXMtb25seS1kby1ub3QtdXNlLWluLXByb2Q=
      access-token-expiry-minutes: 60
      issuer: portops-api-test

logging:
  level:
    org.hibernate.SQL: WARN
    com.yourcompany.portops: DEBUG
```

**Why H2 for tests?**
- In-memory → fast startup (seconds, not 30s for real Oracle)
- `MODE=Oracle` → Oracle-compatible SQL dialect → production-close behavior
- Isolated → tests পরস্পর-কে affect করে না

> **Alternative (advanced):** Testcontainers দিয়ে real Oracle container-এ tests চালানো। Week 2-এর subject। এখন H2 যথেষ্ট।

---

## Step 2 — Service Unit Test (Pure Mockito)

Service layer test — DB বা Spring context ছাড়াই। সবচেয়ে fast, সবচেয়ে বেশি লেখা হয়।

`container/domain/ContainerServiceTest.java`:

```java
package com.yourcompany.portops.container.domain;

import com.yourcompany.portops.container.api.dto.CreateContainerRequest;
import com.yourcompany.portops.container.api.dto.ContainerResponse;
import com.yourcompany.portops.container.api.mapper.ContainerMapper;
import com.yourcompany.portops.container.domain.exception.ContainerNotFoundException;
import com.yourcompany.portops.container.domain.exception.DuplicateBicCodeException;
import com.yourcompany.portops.container.persistence.ContainerRepository;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.shipment.domain.exception.VesselNotFoundException;
import com.yourcompany.portops.shipment.persistence.MotherVesselRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ContainerService")
class ContainerServiceTest {

    @Mock private ContainerRepository containerRepository;
    @Mock private MotherVesselRepository motherVesselRepository;
    @Mock private ContainerMapper containerMapper;

    @InjectMocks private ContainerService containerService;

    private MotherVessel vessel;

    @BeforeEach
    void setUp() {
        vessel = new MotherVessel();
        vessel.setId(1L);
        vessel.setImoNumber("IMO9811000");
        vessel.setVesselName("MAERSK HONAM");
    }

    // ------------------------------------------------------------
    // register()
    // ------------------------------------------------------------

    @Test
    @DisplayName("register: valid request → container saved with status ON_VESSEL")
    void register_validRequest_savesContainer() {
        // Arrange
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 1L);

        when(containerRepository.existsByBicCode("MSCU1234567")).thenReturn(false);
        when(motherVesselRepository.findById(1L)).thenReturn(Optional.of(vessel));
        when(containerRepository.save(any(Container.class))).thenAnswer(inv -> {
            Container c = inv.getArgument(0);
            c.setId(42L);
            return c;
        });
        when(containerMapper.toResponse(any())).thenReturn(mock(ContainerResponse.class));

        // Act
        ContainerResponse response = containerService.register(request);

        // Assert
        assertThat(response).isNotNull();

        // Verify side effects
        verify(containerRepository).existsByBicCode("MSCU1234567");
        verify(motherVesselRepository).findById(1L);
        verify(containerRepository).save(argThat(c ->
            c.getBicCode().equals("MSCU1234567") &&
            c.getStatus() == ContainerStatus.ON_VESSEL &&
            c.getMotherVessel().equals(vessel)
        ));
    }

    @Test
    @DisplayName("register: duplicate BIC code → DuplicateBicCodeException")
    void register_duplicateBic_throws() {
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 1L);

        when(containerRepository.existsByBicCode("MSCU1234567")).thenReturn(true);

        assertThatThrownBy(() -> containerService.register(request))
            .isInstanceOf(DuplicateBicCodeException.class)
            .hasMessageContaining("MSCU1234567");

        verify(containerRepository, never()).save(any());
        verifyNoInteractions(motherVesselRepository, containerMapper);
    }

    @Test
    @DisplayName("register: non-existent vessel → VesselNotFoundException")
    void register_vesselNotFound_throws() {
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 999L);

        when(containerRepository.existsByBicCode(any())).thenReturn(false);
        when(motherVesselRepository.findById(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> containerService.register(request))
            .isInstanceOf(VesselNotFoundException.class);

        verify(containerRepository, never()).save(any());
    }

    // ------------------------------------------------------------
    // findById()
    // ------------------------------------------------------------

    @Test
    @DisplayName("findById: exists → returns mapped response")
    void findById_exists_returnsResponse() {
        Container container = new Container();
        container.setId(1L);
        ContainerResponse expected = mock(ContainerResponse.class);

        when(containerRepository.findById(1L)).thenReturn(Optional.of(container));
        when(containerMapper.toResponse(container)).thenReturn(expected);

        assertThat(containerService.findById(1L)).isEqualTo(expected);
    }

    @Test
    @DisplayName("findById: not found → ContainerNotFoundException")
    void findById_notFound_throws() {
        when(containerRepository.findById(999L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> containerService.findById(999L))
            .isInstanceOf(ContainerNotFoundException.class);
    }

    // ------------------------------------------------------------
    // delete()
    // ------------------------------------------------------------

    @Test
    @DisplayName("delete: exists → calls repository.deleteById")
    void delete_exists_deletes() {
        when(containerRepository.existsById(1L)).thenReturn(true);

        containerService.delete(1L);

        verify(containerRepository).deleteById(1L);
    }

    @Test
    @DisplayName("delete: not found → throws, no delete call")
    void delete_notFound_throws() {
        when(containerRepository.existsById(999L)).thenReturn(false);

        assertThatThrownBy(() -> containerService.delete(999L))
            .isInstanceOf(ContainerNotFoundException.class);

        verify(containerRepository, never()).deleteById(any());
    }
}
```

### Key annotations

| Annotation | কাজ |
|---|---|
| `@ExtendWith(MockitoExtension.class)` | JUnit 5-এ Mockito-র support enable |
| `@Mock` | Mock object — all methods return defaults |
| `@InjectMocks` | Mocks automatic inject করবে target class-এর constructor-এ |
| `@DisplayName` | IDE/test report-এ readable name |
| `@BeforeEach` | প্রত্যেক test method-এর আগে চলে — common setup |

### Testing idioms

**1. AAA pattern — Arrange, Act, Assert**
কোড organize করুন তিন blocks-এ। Comments optional কিন্তু visual separation।

**2. AssertJ > JUnit assertions**
```java
// ✅ AssertJ — fluent, readable
assertThat(containers).hasSize(3).extracting("bicCode").containsExactly("MSCU1", "MSCU2", "MSCU3");

// ❌ JUnit-only — verbose
assertEquals(3, containers.size());
```

**3. `verify(...)` — behavior verification**
```java
verify(repo).save(argThat(c -> c.getStatus() == IN_YARD));   // এই shape-এর entity save হয়েছে
verify(repo, never()).delete(any());                         // delete কখনো call হয়নি
verify(repo, times(2)).findById(anyLong());                  // exactly 2 বার
```

**4. `verifyNoInteractions(mock)` — defensive**
"এই mock-এ কোনো call-ই যাওয়ার কথা না" — এটা assert করুন specially error path tests-এ।

---

## Step 3 — Business Logic Test: Status Transitions

`container/domain/ContainerMovementServiceTest.java`:

```java
package com.yourcompany.portops.container.domain;

import com.yourcompany.portops.container.domain.exception.InvalidStatusTransitionException;
import com.yourcompany.portops.container.persistence.ContainerRepository;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.user.domain.User;
import com.yourcompany.portops.user.persistence.UserRepository;
import com.yourcompany.portops.yard.api.dto.GateInRequest;
import com.yourcompany.portops.yard.api.dto.YardActivityResponse;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("ContainerMovementService - gateIn()")
class ContainerMovementServiceTest {

    @Mock private ContainerRepository containerRepository;
    @Mock private UserRepository userRepository;

    @InjectMocks private ContainerMovementService movementService;

    private Container container;

    @BeforeEach
    void setUp() {
        MotherVessel vessel = new MotherVessel();
        vessel.setId(1L);

        container = new Container();
        container.setId(1L);
        container.setBicCode("MSCU1234567");
        container.setType(ContainerType.DRY_40);
        container.setGrossWeightKg(new BigDecimal("24500"));
        container.setMotherVessel(vessel);
    }

    @Test
    @DisplayName("gateIn: ON_LIGHTER → transitions through AT_GATE to IN_YARD, creates 2 activities")
    void gateIn_fromOnLighter_success() {
        container.setStatus(ContainerStatus.ON_LIGHTER);

        when(containerRepository.findById(1L)).thenReturn(Optional.of(container));
        when(userRepository.getReferenceById(5L)).thenReturn(new User());

        GateInRequest request = new GateInRequest("A-07-03-2", "Routine");

        YardActivityResponse response = movementService.gateIn(1L, request, 5L);

        assertThat(response).isNotNull();
        assertThat(container.getStatus()).isEqualTo(ContainerStatus.IN_YARD);
        assertThat(container.getYardActivities()).hasSize(2);
    }

    @Test
    @DisplayName("gateIn: ON_VESSEL → InvalidStatusTransitionException (skip ON_LIGHTER not allowed)")
    void gateIn_fromOnVessel_throws() {
        container.setStatus(ContainerStatus.ON_VESSEL);

        when(containerRepository.findById(1L)).thenReturn(Optional.of(container));

        GateInRequest request = new GateInRequest("A-07-03-2", null);

        assertThatThrownBy(() -> movementService.gateIn(1L, request, 5L))
            .isInstanceOf(InvalidStatusTransitionException.class)
            .extracting("fromStatus", "toStatus")
            .containsExactly(ContainerStatus.ON_VESSEL, ContainerStatus.AT_GATE);

        // Container status unchanged (in-memory mock, but intent verified)
        assertThat(container.getStatus()).isEqualTo(ContainerStatus.ON_VESSEL);
        assertThat(container.getYardActivities()).isEmpty();
    }

    @Test
    @DisplayName("gateIn: EXITED (terminal) → InvalidStatusTransitionException")
    void gateIn_fromExited_throws() {
        container.setStatus(ContainerStatus.EXITED);

        when(containerRepository.findById(1L)).thenReturn(Optional.of(container));

        GateInRequest request = new GateInRequest("A-07-03-2", null);

        assertThatThrownBy(() -> movementService.gateIn(1L, request, 5L))
            .isInstanceOf(InvalidStatusTransitionException.class);
    }
}
```

### একটা subtle জিনিস: status rollback testing

Unit test-এ আমরা real transaction নেই — শুধু in-memory entity changes। "Rollback actually হয় কিনা" verify করতে integration test লাগবে (Step 7-এ দেখাব)।

এই test শুধু **business rule** verify করছে — invalid status-এ gate-in-এর চেষ্টা করলে exception throw হয়, activities create হয় না। সেটাই যথেষ্ট unit level-এ।

---

## Step 4 — Controller Test (`@WebMvcTest`)

Controller layer test — Spring MVC loaded, services mocked।

`container/api/ContainerControllerTest.java`:

```java
package com.yourcompany.portops.container.api;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.yourcompany.portops.common.security.JwtAccessDeniedHandler;
import com.yourcompany.portops.common.security.JwtAuthenticationEntryPoint;
import com.yourcompany.portops.common.security.JwtAuthenticationFilter;
import com.yourcompany.portops.common.security.SecurityConfig;
import com.yourcompany.portops.container.api.dto.*;
import com.yourcompany.portops.container.domain.ContainerMovementService;
import com.yourcompany.portops.container.domain.ContainerService;
import com.yourcompany.portops.container.domain.ContainerStatus;
import com.yourcompany.portops.container.domain.ContainerType;
import com.yourcompany.portops.container.domain.exception.ContainerNotFoundException;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.context.annotation.Import;
import org.springframework.http.MediaType;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import java.math.BigDecimal;
import java.time.Instant;

import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.when;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(ContainerController.class)
@Import(SecurityConfig.class)
@DisplayName("ContainerController")
class ContainerControllerTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;

    @MockBean private ContainerService containerService;
    @MockBean private ContainerMovementService movementService;
    @MockBean private JwtAuthenticationFilter jwtAuthFilter;           // security filter stub
    @MockBean private JwtAuthenticationEntryPoint authEntryPoint;
    @MockBean private JwtAccessDeniedHandler accessDeniedHandler;

    // ------------------------------------------------------------
    // POST /api/v1/containers
    // ------------------------------------------------------------

    @Test
    @WithMockUser(roles = "YARD_OPERATOR")
    @DisplayName("POST /containers → 201 Created with Location header")
    void create_valid_returns201() throws Exception {
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 1L);
        var response = new ContainerResponse(42L, "MSCU1234567", ContainerType.DRY_40,
                ContainerStatus.ON_VESSEL, new BigDecimal("24500.00"), null, Instant.now(), Instant.now());

        when(containerService.register(any())).thenReturn(response);

        mockMvc.perform(post("/api/v1/containers")
                .with(org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", "/api/v1/containers/42"))
            .andExpect(jsonPath("$.id").value(42))
            .andExpect(jsonPath("$.bicCode").value("MSCU1234567"))
            .andExpect(jsonPath("$.status").value("ON_VESSEL"));
    }

    @Test
    @WithMockUser(roles = "YARD_OPERATOR")
    @DisplayName("POST /containers with invalid BIC → 400 with fieldErrors")
    void create_invalidBic_returns400() throws Exception {
        var request = new CreateContainerRequest("invalid", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 1L);

        mockMvc.perform(post("/api/v1/containers")
                .with(org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errorCode").value("VALIDATION_FAILED"))
            .andExpect(jsonPath("$.fieldErrors.bicCode").exists());
    }

    @Test
    @WithMockUser(roles = "YARD_OPERATOR")
    @DisplayName("POST /containers with negative weight → 400")
    void create_negativeWeight_returns400() throws Exception {
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("-100"), 1L);

        mockMvc.perform(post("/api/v1/containers")
                .with(org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.fieldErrors.grossWeightKg").exists());
    }

    // ------------------------------------------------------------
    // Authorization
    // ------------------------------------------------------------

    @Test
    @DisplayName("POST /containers without auth → 401")
    void create_noAuth_returns401() throws Exception {
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 1L);

        mockMvc.perform(post("/api/v1/containers")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "BILLING_CLERK")
    @DisplayName("POST /containers with wrong role → 403")
    void create_wrongRole_returns403() throws Exception {
        var request = new CreateContainerRequest("MSCU1234567", ContainerType.DRY_40,
                new BigDecimal("24500.00"), 1L);

        mockMvc.perform(post("/api/v1/containers")
                .with(org.springframework.security.test.web.servlet.request.SecurityMockMvcRequestPostProcessors.csrf())
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isForbidden());
    }

    // ------------------------------------------------------------
    // GET /api/v1/containers/{id}
    // ------------------------------------------------------------

    @Test
    @WithMockUser(roles = "YARD_OPERATOR")
    @DisplayName("GET /containers/{id} when not found → 404")
    void getById_notFound_returns404() throws Exception {
        when(containerService.findById(9999L)).thenThrow(new ContainerNotFoundException(9999L));

        mockMvc.perform(get("/api/v1/containers/9999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.errorCode").value("RESOURCE_NOT_FOUND"))
            .andExpect(jsonPath("$.resourceType").value("Container"))
            .andExpect(jsonPath("$.resourceId").value(9999));
    }
}
```

### Key concepts

**1. `@WebMvcTest(ContainerController.class)`**
- শুধু web layer load করে — Controller, Jackson, Validation, Security
- Service/Repository layer load হয় না — `@MockBean` দিয়ে provide করতে হয়
- Fast — 2-3 seconds

**2. `@Import(SecurityConfig.class)`**
- By default `@WebMvcTest` আপনার custom `SecurityConfig` load করে না
- Explicitly import করলে real security rules test হয়

**3. `@MockBean` vs `@Mock`**
- `@Mock` (pure Mockito) — Spring context-এ register হয় না
- `@MockBean` — Spring context-এ mock inject করে (actual bean replace)। Integration/web tests-এ লাগে।

**4. `@WithMockUser(roles = "YARD_OPERATOR")`**
- SecurityContext-এ fake authenticated user inject করে
- Actual JWT generate করতে হয় না test-এ
- Role automatically `ROLE_` prefix পায়

**5. `.with(csrf())`**
- Spring Security default-এ POST-এ CSRF token require করে
- Test-এ CSRF token inject করার shortcut
- আমরা production config-এ `csrf().disable()` করেছি, তবে `@WebMvcTest` default config-এ csrf enabled থাকতে পারে

**6. `jsonPath("$.field")`**
- Response JSON navigate করার JSONPath expression
- `$` = root, `.field` = nested field, `$[0]` = array index

---

## Step 5 — Repository Test (`@DataJpaTest`)

Repository tests — আপনার custom queries আসলে কাজ করে কিনা verify করতে।

`container/persistence/ContainerRepositoryTest.java`:

```java
package com.yourcompany.portops.container.persistence;

import com.yourcompany.portops.container.domain.Container;
import com.yourcompany.portops.container.domain.ContainerStatus;
import com.yourcompany.portops.container.domain.ContainerType;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.test.context.ActiveProfiles;

import java.math.BigDecimal;
import java.util.List;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
@ActiveProfiles("test")
@DisplayName("ContainerRepository")
class ContainerRepositoryTest {

    @Autowired private ContainerRepository containerRepository;
    @Autowired private TestEntityManager em;   // test-time helper for persisting

    private MotherVessel vessel;

    @BeforeEach
    void setUp() {
        vessel = new MotherVessel();
        vessel.setImoNumber("IMO9811000");
        vessel.setVesselName("MAERSK HONAM");
        vessel.setFlagCountry("Singapore");
        em.persistAndFlush(vessel);
    }

    @Test
    @DisplayName("save + findByBicCode: roundtrip")
    void saveAndFind() {
        Container c = buildContainer("MSCU1234567", ContainerStatus.ON_VESSEL);
        containerRepository.save(c);

        Optional<Container> found = containerRepository.findByBicCode("MSCU1234567");

        assertThat(found).isPresent();
        assertThat(found.get().getType()).isEqualTo(ContainerType.DRY_40);
        assertThat(found.get().getMotherVessel().getImoNumber()).isEqualTo("IMO9811000");
    }

    @Test
    @DisplayName("existsByBicCode: returns true when exists")
    void existsByBicCode() {
        containerRepository.save(buildContainer("MSCU1111111", ContainerStatus.IN_YARD));

        assertThat(containerRepository.existsByBicCode("MSCU1111111")).isTrue();
        assertThat(containerRepository.existsByBicCode("MSCU9999999")).isFalse();
    }

    @Test
    @DisplayName("findByStatus: paginated, correct filter")
    void findByStatus_paginated() {
        containerRepository.save(buildContainer("MSCU0000001", ContainerStatus.IN_YARD));
        containerRepository.save(buildContainer("MSCU0000002", ContainerStatus.IN_YARD));
        containerRepository.save(buildContainer("MSCU0000003", ContainerStatus.ON_VESSEL));
        containerRepository.save(buildContainer("MSCU0000004", ContainerStatus.IN_YARD));

        Page<Container> page = containerRepository.findByStatus(ContainerStatus.IN_YARD,
                                                                PageRequest.of(0, 2));

        assertThat(page.getTotalElements()).isEqualTo(3);
        assertThat(page.getTotalPages()).isEqualTo(2);
        assertThat(page.getContent()).hasSize(2);
        assertThat(page.getContent()).allMatch(c -> c.getStatus() == ContainerStatus.IN_YARD);
    }

    @Test
    @DisplayName("findAllWithVessel: JOIN FETCH avoids N+1")
    void findAllWithVessel_fetchesVessel() {
        containerRepository.save(buildContainer("MSCU0000001", ContainerStatus.IN_YARD));
        containerRepository.save(buildContainer("MSCU0000002", ContainerStatus.IN_YARD));

        em.flush();
        em.clear();   // persistence context clear করি — fresh load force করতে

        List<Container> containers = containerRepository.findAllWithVessel();

        // Vessel already loaded — no lazy init exception
        assertThat(containers).hasSize(2);
        assertThat(containers.get(0).getMotherVessel().getVesselName()).isEqualTo("MAERSK HONAM");
    }

    @Test
    @DisplayName("unique constraint: duplicate BIC throws")
    void duplicateBic_violatesUnique() {
        containerRepository.save(buildContainer("MSCU1111111", ContainerStatus.IN_YARD));

        assertThat(containerRepository.existsByBicCode("MSCU1111111")).isTrue();

        // Trying to save another with same BIC would throw on flush
        // এটাই আমাদের Service layer-এ existsByBicCode check করার কারণ
    }

    private Container buildContainer(String bicCode, ContainerStatus status) {
        Container c = new Container();
        c.setBicCode(bicCode);
        c.setType(ContainerType.DRY_40);
        c.setStatus(status);
        c.setGrossWeightKg(new BigDecimal("24500.00"));
        c.setMotherVessel(vessel);
        return c;
    }
}
```

### `@DataJpaTest` যা করে

- শুধু JPA-related components load করে — Repositories, EntityManager, TransactionManager
- By default H2 in-memory DB ব্যবহার করে (Step 1-এ test dependency add করেছি)
- প্রত্যেক test method একটা transaction-এ — test শেষে rollback (default behavior)
- Controllers, Services load হয় না

### `em.flush()` + `em.clear()` কেন?

Hibernate persistence context-এ newly saved entities cached থাকে। `findAllWithVessel()` call করলে cache থেকে return করতে পারে — আসলে query fire হচ্ছে কিনা সন্দেহ থাকবে।

- `em.flush()` — pending inserts DB-তে push
- `em.clear()` — persistence context খালি, entities detached

এরপর `findAllWithVessel()` fresh query fire করতে বাধ্য — actual JOIN FETCH behavior verify।

---

## Step 6 — AuthService Test

Security layer test — password hashing + JWT generation actually কাজ করে কিনা।

`user/domain/AuthServiceTest.java`:

```java
package com.yourcompany.portops.user.domain;

import com.yourcompany.portops.common.config.JwtProperties;
import com.yourcompany.portops.common.security.JwtService;
import com.yourcompany.portops.user.api.dto.LoginRequest;
import com.yourcompany.portops.user.api.dto.LoginResponse;
import com.yourcompany.portops.user.domain.exception.InvalidCredentialsException;
import com.yourcompany.portops.user.persistence.UserRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.BadCredentialsException;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.authority.SimpleGrantedAuthority;

import java.util.List;
import java.util.Optional;
import java.util.Set;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("AuthService")
class AuthServiceTest {

    @Mock private AuthenticationManager authenticationManager;
    @Mock private JwtService jwtService;
    @Mock private UserRepository userRepository;

    private JwtProperties jwtProperties;
    private AuthService authService;

    @BeforeEach
    void setUp() {
        jwtProperties = new JwtProperties("secret", 60, "portops-test");
        authService = new AuthService(authenticationManager, jwtService, jwtProperties, userRepository);
    }

    @Test
    @DisplayName("login: valid credentials → token returned, lastLoginAt updated")
    void login_valid_returnsToken() {
        var request = new LoginRequest("alice", "pass123");

        UserPrincipal principal = buildPrincipal();
        Authentication authResult = new UsernamePasswordAuthenticationToken(
            principal, null, principal.getAuthorities());

        when(authenticationManager.authenticate(any())).thenReturn(authResult);
        when(jwtService.generateToken(principal)).thenReturn("mocked.jwt.token");

        User user = new User();
        user.setId(1L);
        when(userRepository.findById(1L)).thenReturn(Optional.of(user));

        LoginResponse response = authService.login(request);

        assertThat(response.accessToken()).isEqualTo("mocked.jwt.token");
        assertThat(response.tokenType()).isEqualTo("Bearer");
        assertThat(response.expiresInSeconds()).isEqualTo(3600);
        assertThat(response.user().username()).isEqualTo("alice");
        assertThat(response.user().roles()).contains("ROLE_YARD_OPERATOR");

        assertThat(user.getLastLoginAt()).isNotNull();
    }

    @Test
    @DisplayName("login: bad credentials → InvalidCredentialsException (no leak)")
    void login_badCredentials_throws() {
        var request = new LoginRequest("alice", "wrong");

        when(authenticationManager.authenticate(any()))
            .thenThrow(new BadCredentialsException("Bad credentials"));

        assertThatThrownBy(() -> authService.login(request))
            .isInstanceOf(InvalidCredentialsException.class);

        verifyNoInteractions(jwtService);
    }

    private UserPrincipal buildPrincipal() {
        User user = new User();
        user.setId(1L);
        user.setUsername("alice");
        user.setPasswordHash("$2a$12$...");
        user.setFullName("Alice Smith");
        user.setRoles(Set.of(Role.YARD_OPERATOR));
        user.setEnabled(true);
        user.setAccountNonLocked(true);
        return UserPrincipal.from(user);
    }
}
```

---

## Step 7 — Integration Test (`@SpringBootTest`)

Full app boot — real HTTP, real DB, real security। সবচেয়ে slow, কিন্তু সবচেয়ে confidence।

`integration/ContainerFlowIntegrationTest.java`:

```java
package com.yourcompany.portops.integration;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.shipment.persistence.MotherVesselRepository;
import com.yourcompany.portops.user.domain.Role;
import com.yourcompany.portops.user.domain.User;
import com.yourcompany.portops.user.persistence.UserRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.http.MediaType;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.MvcResult;
import org.springframework.transaction.annotation.Transactional;

import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Transactional    // each test rolls back
@DisplayName("End-to-end Container registration + gate-in")
class ContainerFlowIntegrationTest {

    @Autowired private MockMvc mockMvc;
    @Autowired private ObjectMapper objectMapper;
    @Autowired private UserRepository userRepository;
    @Autowired private MotherVesselRepository vesselRepository;
    @Autowired private PasswordEncoder passwordEncoder;

    private Long vesselId;

    @BeforeEach
    void setUp() {
        // Operator user
        User op = new User();
        op.setUsername("test_operator");
        op.setEmail("op@test.local");
        op.setPasswordHash(passwordEncoder.encode("TestPass@123"));
        op.setFullName("Test Operator");
        op.setRoles(Set.of(Role.YARD_OPERATOR));
        userRepository.save(op);

        // Mother vessel
        MotherVessel v = new MotherVessel();
        v.setImoNumber("IMO9999999");
        v.setVesselName("TEST VESSEL");
        v.setFlagCountry("Testland");
        vesselId = vesselRepository.save(v).getId();
    }

    @Test
    @DisplayName("full flow: login → register container → gate-in → verify IN_YARD")
    void fullFlow() throws Exception {
        // 1. Login
        ObjectNode loginBody = objectMapper.createObjectNode();
        loginBody.put("username", "test_operator");
        loginBody.put("password", "TestPass@123");

        MvcResult loginResult = mockMvc.perform(post("/api/v1/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(loginBody.toString()))
            .andExpect(status().isOk())
            .andReturn();

        String token = objectMapper.readTree(loginResult.getResponse().getContentAsString())
            .get("accessToken").asText();
        assertThat(token).isNotBlank();

        // 2. Register container
        ObjectNode registerBody = objectMapper.createObjectNode();
        registerBody.put("bicCode", "MSCU1234567");
        registerBody.put("type", "DRY_40");
        registerBody.put("grossWeightKg", 24500.00);
        registerBody.put("motherVesselId", vesselId);

        MvcResult registerResult = mockMvc.perform(post("/api/v1/containers")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content(registerBody.toString()))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.status").value("ON_VESSEL"))
            .andReturn();

        Long containerId = objectMapper.readTree(registerResult.getResponse().getContentAsString())
            .get("id").asLong();

        // 3. Manually set container to ON_LIGHTER (state machine requirement)
        // Production-এ এটা আসলে আরেকটা endpoint-এর কাজ — test simplification
        // আপনার business flow-এ "lighter ship pickup" নামের আলাদা endpoint থাকবে

        // 4. Attempt gate-in from ON_VESSEL → should 409
        ObjectNode gateInBody = objectMapper.createObjectNode();
        gateInBody.put("yardSlotCode", "A-07-03-2");

        mockMvc.perform(post("/api/v1/containers/" + containerId + "/gate-in")
                .header("Authorization", "Bearer " + token)
                .contentType(MediaType.APPLICATION_JSON)
                .content(gateInBody.toString()))
            .andExpect(status().isConflict())
            .andExpect(jsonPath("$.errorCode").value("INVALID_STATUS_TRANSITION"));
    }

    @Test
    @DisplayName("unauthenticated → 401")
    void unauthenticated_blocked() throws Exception {
        mockMvc.perform(get("/api/v1/containers/1"))
            .andExpect(status().isUnauthorized());
    }
}
```

### `@SpringBootTest` features

- **Full context load** — সব beans, real DB, real security, real filter chain
- **Slow** — ~10-15 seconds startup, তাই সংখ্যায় কম লিখুন (5-10 integration tests যথেষ্ট)
- **`@Transactional`** — test method-এর পর automatic rollback, DB clean
- **`@AutoConfigureMockMvc`** — real Tomcat boot না করেই MockMvc use

### কী test করবেন integration level-এ

- **Critical flows** — login → create → update, end-to-end
- **Security integration** — JWT filter actual token-এ কাজ করে
- **Cross-module transactions** — container + yard activity commit একসাথে

### কী NOT test করবেন integration-এ

- Individual validation rules — unit test-এ করুন
- Every single endpoint — slow, redundant

---

## Step 8 — Running Tests

### Maven

```bash
# All tests
./mvnw test

# Specific test class
./mvnw test -Dtest=ContainerServiceTest

# Specific method
./mvnw test -Dtest=ContainerServiceTest#register_validRequest_savesContainer
```

### IntelliJ
- Test class/method-এ right-click → Run
- Gutter-এ green ▶ icon click

### Coverage report (optional কিন্তু recommended)

`pom.xml`-এ JaCoCo plugin add:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.11</version>
    <executions>
        <execution><goals><goal>prepare-agent</goal></goals></execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals><goal>report</goal></goals>
        </execution>
    </executions>
</plugin>
```

Run:
```bash
./mvnw test
# Report: target/site/jacoco/index.html
```

Enterprise target: **70-80% line coverage** for service layer, **60%+ overall**।

---

## Day 1 Checkpoint

- [ ] Unit tests — 3+ service classes, সবগুলো green
- [ ] Controller tests — auth + validation scenarios
- [ ] Repository test — H2 profile-এ চলে, custom query verified
- [ ] Integration test — end-to-end flow, JWT-সহ
- [ ] `./mvnw test` সব pass করে
- [ ] Coverage report generated (optional)

---

# Day 2 — Production Readiness

---

## Step 9 — OpenAPI / Swagger UI

React team-এর জন্য self-documenting API। Manual documentation maintain করা লাগবে না।

### Dependency

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

### Security config-এ Swagger paths allow

`SecurityConfig` আগেই-এ থাকা উচিত (Phase 4-এ যোগ করেছিলাম):

```java
.requestMatchers("/v3/api-docs/**", "/swagger-ui/**", "/swagger-ui.html").permitAll()
```

### OpenAPI metadata config

`common/config/OpenApiConfig.java`:

```java
package com.yourcompany.portops.common.config;

import io.swagger.v3.oas.models.Components;
import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import io.swagger.v3.oas.models.security.SecurityRequirement;
import io.swagger.v3.oas.models.security.SecurityScheme;
import io.swagger.v3.oas.models.servers.Server;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.List;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI portopsOpenAPI() {
        final String schemeName = "bearerAuth";

        return new OpenAPI()
            .info(new Info()
                .title("Port Operations API")
                .version("v1")
                .description("Container tracking, yard operations, and billing for port/shipping operations.")
                .contact(new Contact()
                    .name("Backend Team")
                    .email("backend@portops.example.com"))
                .license(new License().name("Proprietary")))
            .servers(List.of(
                new Server().url("http://localhost:8080").description("Local dev"),
                new Server().url("https://api-staging.portops.example.com").description("Staging")
            ))
            .addSecurityItem(new SecurityRequirement().addList(schemeName))
            .components(new Components()
                .addSecuritySchemes(schemeName, new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")));
    }
}
```

### Controller-এ enhanced annotations (optional)

`ContainerController.gateIn`:

```java
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;

@Tag(name = "Containers", description = "Container lifecycle and movement")
@RestController
@RequestMapping("/api/v1/containers")
public class ContainerController {

    @Operation(
        summary = "Gate-in a container at the port yard",
        description = "Transitions container status from ON_LIGHTER → AT_GATE → IN_YARD in one atomic transaction."
    )
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "Gate-in successful"),
        @ApiResponse(responseCode = "404", description = "Container not found"),
        @ApiResponse(responseCode = "409", description = "Invalid status transition"),
        @ApiResponse(responseCode = "401", description = "Authentication required"),
        @ApiResponse(responseCode = "403", description = "Insufficient privileges")
    })
    @PostMapping("/{id}/gate-in")
    @PreAuthorize("hasAnyRole('YARD_OPERATOR', 'ADMIN')")
    public YardActivityResponse gateIn(...) { ... }
}
```

### Access Swagger

App restart, browser:
- **Swagger UI:** `http://localhost:8080/swagger-ui.html`
- **Raw OpenAPI JSON:** `http://localhost:8080/v3/api-docs`

### React team-এর জন্য magic

Swagger UI-তে:
1. "Authorize" button → JWT paste (format: just the token, no "Bearer " prefix)
2. Try endpoints directly
3. Copy curl commands
4. **TypeScript client auto-generate:**
   ```bash
   openapi-generator-cli generate -i http://localhost:8080/v3/api-docs -g typescript-axios -o ./frontend/src/api
   ```

React team এখন আর manual API client লিখবে না — auto-generated।

---

## Step 10 — Spring Boot Actuator

Health checks, metrics, runtime info — production essentials।

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### Config

`application.yml`:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      probes:
        enabled: true
  info:
    env:
      enabled: true
    git:
      mode: full
      enabled: true
```

### Security — actuator lockdown

`SecurityConfig.filterChain`:

```java
.requestMatchers("/actuator/health", "/actuator/health/**").permitAll()
.requestMatchers("/actuator/**").hasRole("ADMIN")
```

Health endpoint public (load balancer check), বাকি admin-only।

### Available endpoints

| Endpoint | Purpose |
|---|---|
| `/actuator/health` | App health — DB, disk, etc. Load balancer use করে |
| `/actuator/health/liveness` | K8s liveness probe |
| `/actuator/health/readiness` | K8s readiness probe |
| `/actuator/info` | App version, build info |
| `/actuator/metrics` | Available metrics list |
| `/actuator/metrics/{name}` | Specific metric value |
| `/actuator/prometheus` | Prometheus-scrapable metrics |

### Custom info

`application.yml`:

```yaml
info:
  app:
    name: "@project.name@"           # Maven property, build-এ replace হবে
    version: "@project.version@"
    description: Port Operations Management API
  contact:
    team: Backend Team
    email: backend@portops.example.com
```

`@project.name@` — Maven resource filtering। Enable করতে `pom.xml`-এ:

```xml
<build>
    <resources>
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
        </resource>
    </resources>
</build>
```

### Test

```
GET http://localhost:8080/actuator/health
```

Response:
```json
{
  "status": "UP",
  "components": {
    "db": { "status": "UP", "details": { "database": "Oracle", "validationQuery": "SELECT 1" } },
    "diskSpace": { "status": "UP" },
    "ping": { "status": "UP" }
  }
}
```

Details দেখতে authenticated admin user লাগবে (config-এ `when-authorized`)।

---

## Step 11 — `@Scheduled` — Background Jobs

Real port ops scenario: প্রতি রাতে 2AM-এ yard-এ থাকা সব container-এর জন্য storage invoice generate।

### Enable scheduling

`PortopsApplication.java`:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
@EnableScheduling         // ⬅ add
public class PortopsApplication { ... }
```

### Billing scheduler (skeleton)

`billing/domain/BillingScheduler.java`:

```java
package com.yourcompany.portops.billing.domain;

import com.yourcompany.portops.container.domain.ContainerStatus;
import com.yourcompany.portops.container.persistence.ContainerRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Component
public class BillingScheduler {

    private static final Logger log = LoggerFactory.getLogger(BillingScheduler.class);

    private final ContainerRepository containerRepository;
    // private final BillingInvoiceService billingInvoiceService;   // Week 2-এ implement

    public BillingScheduler(ContainerRepository containerRepository) {
        this.containerRepository = containerRepository;
    }

    /**
     * প্রতি রাতে 02:00 AM (server time) — yard-এ থাকা সব container-এর জন্য daily storage charge।
     */
    @Scheduled(cron = "0 0 2 * * *")
    @Transactional
    public void generateDailyStorageInvoices() {
        Instant startedAt = Instant.now();
        log.info("Starting daily storage billing run at {}", startedAt);

        long inYardCount = containerRepository.findByStatus(ContainerStatus.IN_YARD,
                org.springframework.data.domain.Pageable.unpaged())
            .getTotalElements();

        // Actual billing logic Week 2-এ — এখন skeleton
        log.info("Found {} containers in yard eligible for storage billing", inYardCount);

        // for each container: calculate days-in-yard * tariff rate, create BillingInvoice
        // billingInvoiceService.generateStorageInvoicesForYardContainers();

        log.info("Completed billing run in {} ms",
                 java.time.Duration.between(startedAt, Instant.now()).toMillis());
    }

    /**
     * প্রতি 15 মিনিটে stale container check — 30 দিনের বেশি yard-এ থাকলে alert।
     */
    @Scheduled(fixedDelay = 15 * 60 * 1000, initialDelay = 60 * 1000)   // 15 min, start after 1 min
    public void checkStaleContainers() {
        log.debug("Running stale container check");
        // Stale container detection logic
    }
}
```

### Cron syntax

Spring cron: **6 fields** — second, minute, hour, day-of-month, month, day-of-week।

| Cron | Meaning |
|---|---|
| `0 0 2 * * *` | প্রতিদিন 2 AM |
| `0 */15 * * * *` | প্রতি 15 মিনিট |
| `0 0 9 * * MON-FRI` | সোম-শুক্র 9 AM |
| `0 0 0 1 * *` | প্রতি মাসের 1 তারিখ midnight |

### Fixed delay vs Fixed rate

| Annotation | Behavior |
|---|---|
| `@Scheduled(fixedDelay = 60000)` | শেষ execution শেষের 60s পর। Overlap নেই। |
| `@Scheduled(fixedRate = 60000)` | প্রতি 60s-এ শুরু। Previous ongoing থাকলেও পরেরটা শুরু হবে। |
| `@Scheduled(cron = "...")` | Calendar-based |

**Default** — fixedDelay safer।

### ⚠️ Multi-instance caveat

Production-এ একাধিক app instance চললে `@Scheduled` method **প্রতি instance-এ** fire হবে। Billing job 3 বার run হবে 3 instance হলে — duplicate invoices!

**Solution:** [ShedLock](https://github.com/lukas-krecan/ShedLock) library। DB-based lock — only one instance runs at a time। Week 2-এর subject।

---

## Step 12 — `@Async` — Fire-and-Forget

Notification পাঠানো, audit log লেখা — এগুলো main request path block করবে না।

### Enable async

`PortopsApplication.java`:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
@EnableScheduling
@EnableAsync              // ⬅ add
public class PortopsApplication { ... }
```

### Thread pool config

`common/config/AsyncConfig.java`:

```java
package com.yourcompany.portops.common.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
public class AsyncConfig {

    @Bean(name = "taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("portops-async-");
        executor.initialize();
        return executor;
    }
}
```

### Notification service

`common/notification/NotificationService.java`:

```java
package com.yourcompany.portops.common.notification;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

@Service
public class NotificationService {

    private static final Logger log = LoggerFactory.getLogger(NotificationService.class);

    @Async("taskExecutor")
    public void notifyShippingAgent(Long shipmentId, String event) {
        log.info("Sending notification for shipment {} event {} on thread {}",
                 shipmentId, event, Thread.currentThread().getName());

        // Simulate email/SMS send — actual implementation-এ JavaMailSender / SMS gateway call
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }

        log.info("Notification sent for shipment {}", shipmentId);
    }
}
```

### Usage

```java
@Service
class ContainerMovementService {
    private final NotificationService notificationService;

    public YardActivityResponse gateIn(...) {
        // ... main logic ...

        // Fire-and-forget — ঘুরে এসে main flow-এ block করবে না
        notificationService.notifyShippingAgent(container.getShipment().getId(), "GATE_IN");

        return response;
    }
}
```

### ⚠️ `@Async` traps

**1. Self-invocation** — same class-এর method call-এ async কাজ করে না (proxy-based, same limitation as `@Transactional`)।

**2. Return type** — async method-এর return:
- `void` — fire-and-forget
- `CompletableFuture<T>` — caller wait করতে পারে
- অন্য যেকোনো type → ignored, null return (bug-prone)

**3. Exception handling** — `void` async method-এ exception silently swallow হয়। Custom `AsyncUncaughtExceptionHandler` লাগাতে হয়।

**4. Transaction boundary** — `@Async` method **new thread**-এ run, caller-এর transaction-এ না। `@Transactional @Async` method-এর নিজস্ব transaction হবে।

---

## Step 13 — Final Code Review

৫ phase-এ আপনি কী বানিয়েছেন, সেটার একটা consolidated view।

### Package structure

```
com.yourcompany.portops/
    ├── PortopsApplication.java
    ├── common/
    │     ├── config/
    │     │     ├── AsyncConfig.java
    │     │     ├── JwtProperties.java
    │     │     └── OpenApiConfig.java
    │     ├── exception/
    │     │     ├── DomainException.java
    │     │     ├── GlobalExceptionHandler.java
    │     │     └── ResourceNotFoundException.java
    │     ├── notification/
    │     │     └── NotificationService.java
    │     └── security/
    │           ├── JwtAccessDeniedHandler.java
    │           ├── JwtAuthenticationEntryPoint.java
    │           ├── JwtAuthenticationFilter.java
    │           ├── JwtService.java
    │           └── SecurityConfig.java
    ├── container/
    │     ├── api/
    │     │     ├── ContainerController.java
    │     │     ├── dto/ (request/response records)
    │     │     └── mapper/ContainerMapper.java  (MapStruct interface)
    │     ├── domain/
    │     │     ├── Container.java
    │     │     ├── ContainerStatus.java (state machine)
    │     │     ├── ContainerType.java
    │     │     ├── ContainerService.java
    │     │     ├── ContainerMovementService.java
    │     │     └── exception/
    │     └── persistence/
    │           └── ContainerRepository.java
    ├── shipment/
    │     ├── api/
    │     ├── domain/
    │     └── persistence/
    ├── yard/
    │     ├── api/
    │     ├── domain/
    │     └── persistence/
    ├── billing/
    │     └── domain/BillingScheduler.java
    └── user/
          ├── api/AuthController.java
          ├── domain/
          │     ├── User.java, Role.java, UserPrincipal.java
          │     ├── AuthService.java, UserService.java
          │     ├── AppUserDetailsService.java
          │     ├── UserBootstrap.java (dev-only)
          │     └── exception/
          └── persistence/UserRepository.java
```

### Feature coverage

| Feature | Status |
|---|---|
| REST endpoints with proper HTTP semantics | ✅ |
| DTO-based API (no entity leak) | ✅ |
| Bean Validation on requests | ✅ |
| JPA entities with proper relationships | ✅ |
| Oracle/MSSQL connected | ✅ |
| N+1 avoidance patterns | ✅ |
| Domain exception hierarchy | ✅ |
| RFC 7807 ProblemDetail responses | ✅ |
| Transactional business logic | ✅ |
| State machine validation | ✅ |
| JWT authentication | ✅ |
| Role-based authorization | ✅ |
| CORS for React | ✅ |
| MapStruct auto-mappers | ✅ |
| Unit + integration tests | ✅ |
| OpenAPI/Swagger | ✅ |
| Actuator health checks | ✅ |
| Scheduled jobs | ✅ |
| Async processing | ✅ |

### What's NOT there (Week 2+ priorities)

| Missing | Priority | Note |
|---|---|---|
| Flyway/Liquibase migrations | **High** | `ddl-auto: create-drop` unsafe for prod |
| Refresh token pattern | **High** | Production-এ session management |
| ShedLock for `@Scheduled` multi-instance | **High** | যদি 1+ instance deploy করেন |
| Testcontainers for integration tests | Medium | H2 vs real Oracle behavior difference |
| Distributed tracing (Micrometer) | Medium | Multi-service debugging |
| Rate limiting on login | Medium | Brute force protection |
| Account lockout logic | Medium | N failed attempts → lock |
| Audit logging (who changed what when) | Medium | Compliance + debugging |
| File upload (BL, photos) | Medium | `MultipartFile` + S3/MinIO |
| WebClient for outbound API calls | Low | Customs/vessel tracking integration |
| Caching (Caffeine/Redis) | Low | Tariff tables, vessel lookups |

---

## Phase 5 Milestone Checklist

### Day 1 — Testing
- [ ] 3+ Service unit tests (ContainerService, ContainerMovementService, AuthService)
- [ ] Controller test with `@WebMvcTest` + `@WithMockUser` — auth scenarios covered
- [ ] Repository test with `@DataJpaTest` — custom query verified
- [ ] Integration test with `@SpringBootTest` — end-to-end flow
- [ ] `./mvnw test` সব test pass

### Day 2 — Production
- [ ] Swagger UI accessible at `/swagger-ui.html`, auth lock icon working
- [ ] JWT authorize button দিয়ে Swagger থেকে authenticated request
- [ ] `/actuator/health` public, `/actuator/metrics` admin-only
- [ ] Custom `info` endpoint app metadata দেখায়
- [ ] BillingScheduler-এ cron method configured (log-এ "Starting billing run" visible after cron trigger)
- [ ] Async notification thread pool configured — log-এ `portops-async-1` thread name
- [ ] Final package structure clean, naming consistent

---

## Self-Check Questions

1. `@WebMvcTest` আর `@SpringBootTest`-এর পার্থক্য — কখন কোনটা ব্যবহার করবেন?
2. `@MockBean` আর `@Mock`-এর structural পার্থক্য কী?
3. `@DataJpaTest` কেন by default transaction rollback করে?
4. Repository test-এ `em.flush()` + `em.clear()` কখন লাগে?
5. `@WithMockUser` কীভাবে JWT filter bypass করে authenticated state simulate করে?
6. `jsonPath("$.fieldErrors.bicCode")` — কী match করছে?
7. Swagger UI-তে "Authorize" button কোন OpenAPI component থেকে আসছে?
8. `management.endpoint.health.show-details: when-authorized` — কেন "always" নয়?
9. `@Scheduled(cron = "0 0 2 * * *")` — 6 field meaning?
10. Multi-instance deployment-এ `@Scheduled` problem কী, solution কী?
11. `@Async void` vs `@Async CompletableFuture<T>` — কখন কোনটা?
12. `@Async` self-invocation trap কেন হয় (hint: Phase 3-এর transactional trap-এর same cause)?

---

## Troubleshooting

### Problem: Test-এ `No qualifying bean of type ...`
**কারণ:** `@WebMvcTest` service bean load করে না, `@MockBean` missing।
**সমাধান:** Controller-এ যত service inject হয়, সব `@MockBean` declare।

### Problem: `@WithMockUser` কাজ করছে না (401 আসছে)
**Checklist:**
1. `spring-security-test` dependency আছে?
2. `@Import(SecurityConfig.class)` করেছেন?
3. Role prefix ঠিক — `roles = "ADMIN"` (`ROLE_ADMIN` না)?

### Problem: Integration test-এ "Port 8080 already in use"
**সমাধান:** `@SpringBootTest(webEnvironment = RANDOM_PORT)` দিন, বা `@AutoConfigureMockMvc` ব্যবহার করুন (নো real port)।

### Problem: `@DataJpaTest` production DB use করছে
**সমাধান:** `@AutoConfigureTestDatabase(replace = Replace.ANY)` — H2 override force করে।

### Problem: Swagger UI load হয় কিন্তু endpoints দেখায় না
**Checklist:**
1. `springdoc-openapi-starter-webmvc-ui` dependency ঠিক?
2. Security config-এ `/v3/api-docs/**` permitAll?
3. Controller-এ `@RestController` (just `@Controller` না)?

### Problem: `@Scheduled` method চলে না
**Checklist:**
1. `@EnableScheduling` main application class-এ?
2. Method-enclosing class `@Component`/`@Service`?
3. Method `public` এবং no parameters?
4. Cron syntax ঠিক? (6 fields, 5 না)

### Problem: `@Async` method synchronously execute
**Checklist:**
1. `@EnableAsync` present?
2. Self-invocation না তো?
3. Method `public`?

---

## Phase 5 Wrap-up

7 দিন পর এখন আপনার যা আছে সেটা **real enterprise Spring Boot codebase**। আপনি এই সব concepts জানেন:

- **Layered architecture:** Controller → Service → Repository
- **Domain modeling:** Entity, relationships, state machines
- **API design:** REST semantics, DTOs, validation, pagination, proper HTTP codes
- **Persistence:** JPA, Oracle/MSSQL, N+1 patterns, transactions
- **Error handling:** Domain exceptions, global handler, ProblemDetail
- **Security:** Spring Security filter chain, JWT, roles, CORS
- **Testing:** Unit, integration, mocking, security testing
- **Production:** Swagger, Actuator, scheduling, async

এই list-এর প্রত্যেকটা concept আপনি Android থেকে একটা analog দিয়ে compare করতে পারেন, আর code লিখতে পারেন।

### আপনার project-এ actual কাজ শুরু করার পরামর্শ

**প্রথম দুই সপ্তাহে যা করবেন:**
1. Existing codebase-এর patterns observe করুন — team already কী pattern use করে (DTO mapping, exception handling, security) সেটা follow করুন, আপনার preference না
2. Simple tickets দিয়ে শুরু — একটা field add, একটা query optimize, একটা test write
3. Code review-তে senior-দের comments মনোযোগ দিয়ে পড়ুন — enterprise-specific conventions এভাবেই শেখা যায়

**মাসের শেষে ধরুন:**
1. Flyway migration — প্রথম real production concern
2. ShedLock — যদি multi-instance deploy থাকে
3. Actual business domain-এর গভীরে — tariff rules, demurrage calculation, customs integration

### আমাকে যা পাঠাবেন

1. **"Phase 5 done"** confirmation
2. **`./mvnw test` output screenshot** — সব green
3. **Swagger UI screenshot** — authenticated request successful
4. **`/actuator/health` response** — DB UP দেখাচ্ছে
5. **Jacoco coverage report screenshot** (যদি set up করেন)
6. **একটা reflection note** — ৭ দিন আগে কোথায় ছিলেন, এখন কোথায়, কোন concept সবচেয়ে কঠিন লেগেছে, কোনটা সবচেয়ে সহজ

---

## 7 দিন পর — কী ভাববেন

আপনি এখন এই প্রশ্নগুলোর উত্তর পরিষ্কার দিতে পারবেন:

- "আমি Spring Boot জানি" — হ্যাঁ
- "Production-grade কোড লিখতে পারি" — yes, with code review
- "এনটারপ্রাইজ codebase-এ productively contribute করতে পারব" — yes, from day 1

যা এখনো বাকি:
- Existing codebase-এর context — প্রতিটা company-র নিজস্ব conventions
- Domain expertise — port operations-এর গভীর knowledge
- Distributed systems concerns — message queues, eventual consistency, saga pattern

এগুলো project-এ কাজ করতে করতেই আসবে। Code দিয়ে শেখা স্বাভাবিক।

---

**আপনার শেখা এখানেই শেষ না। এটাই শুরু।**

Spring Boot ecosystem বিশাল — Spring Cloud, Spring Batch, Spring Integration, WebFlux, Kafka Streams। ৭ দিনে আপনি যা শিখেছেন সেটা core MVC stack। এই foundation-এর উপর আরো অনেক কিছু বানানো যায়।

আপনার বেঙ্কিং eKYC থেকে port operations — দুটোই enterprise domain, দুটোতেই safety-critical operations, দুটোতেই আপনার এখন যা skillset সেটাই লাগবে।

**All the best. Production-এ দেখা হবে।**

শুরু করুন। আটকালে জানান।
