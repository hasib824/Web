# Phase 3 — Business Logic, Transactions & Error Handling

**Duration:** ১ দিন | ~৬–৭ ঘণ্টা

**Phase শেষে আপনার হাতে থাকবে:**
- Proper domain exception hierarchy (Phase 2-এর crude `IllegalArgumentException` সব gone)
- Global exception handler যা RFC 7807 ProblemDetail format-এ consistent error JSON return করে
- `ContainerMovementService.gateIn()` — একটা atomic business workflow যেটা status check + status update + yard activity log সব একই transaction-এ করে
- Status transition validation (state machine pattern)
- MapStruct দিয়ে auto-generated mapper (manual mapper replaced)
- Structured logging

---

## Phase 3 Overview

Phase 2-এ আপনি CRUD বানিয়েছেন — data in, data out। Phase 3-এ বানাবেন **business logic** — অর্থাৎ domain rules enforce করা, multi-step workflows atomic রাখা, এবং API consumers-কে (React team) consistent error messages দেওয়া।

### Android mental model

Android-এ আপনি সম্ভবত এরকম pattern use করেন:

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val exception: AppException) : Result<Nothing>()
}

sealed class AppException : Exception() {
    object NetworkError : AppException()
    data class ValidationError(val field: String) : AppException()
    data class NotFoundError(val resource: String) : AppException()
}

// ViewModel-এ:
when (result) {
    is Result.Success -> _state.update { it.copy(data = result.data) }
    is Result.Error -> _state.update { it.copy(error = result.exception) }
}
```

Spring-এ একই pattern, ভিন্ন mechanism:

| Android | Spring Boot |
|---|---|
| `sealed class AppException` hierarchy | `abstract class DomainException` + subclasses |
| `Result.Error` থেকে UI error message | Exception throw → `@RestControllerAdvice` catches → JSON response |
| `when` exhaustive check | `@ExceptionHandler` per exception type |
| Retrofit error interceptor | Global exception handler |

**Key difference:** Android-এ Result wrapping explicit, Spring-এ exception throw implicit — Service method-এ `throw new ContainerNotFoundException(id)` করলে framework সেটা catch করে HTTP response বানায়। Controller-এ try-catch লাগে না।

---

## Step 1 — Exception Hierarchy Design

Generic `IllegalArgumentException` throw করা একটা code smell। কারণ:

1. **Caller বুঝতে পারে না কী ভুল হয়েছে** — vessel missing, বা invalid status transition, বা duplicate BIC code?
2. **HTTP status map করা যায় না consistent-ভাবে** — সবকিছু 500 হয়ে যায়
3. **Testing-এ fragile** — exact message string check করতে হয়

**সমাধান:** Domain-specific exception classes, একটা common base class থেকে extend।

### Base Exception বানান

`common/exception/` package-এ `DomainException.java`:

```java
package com.yourcompany.portops.common.exception;

import org.springframework.http.HttpStatus;

/**
 * Base class for all domain-level exceptions.
 * Extends RuntimeException so @Transactional rolls back by default.
 */
public abstract class DomainException extends RuntimeException {

    protected DomainException(String message) {
        super(message);
    }

    protected DomainException(String message, Throwable cause) {
        super(message, cause);
    }

    /**
     * প্রতিটা subclass নিজেই decide করবে কোন HTTP status এ map হবে।
     */
    public abstract HttpStatus httpStatus();

    /**
     * Machine-readable error code — React team switch/case করতে পারবে।
     */
    public abstract String errorCode();
}
```

### Domain-specific Exceptions

`common/exception/ResourceNotFoundException.java`:

```java
package com.yourcompany.portops.common.exception;

import org.springframework.http.HttpStatus;

public class ResourceNotFoundException extends DomainException {

    private final String resourceType;
    private final Object resourceId;

    public ResourceNotFoundException(String resourceType, Object resourceId) {
        super("%s not found: %s".formatted(resourceType, resourceId));
        this.resourceType = resourceType;
        this.resourceId = resourceId;
    }

    @Override
    public HttpStatus httpStatus() {
        return HttpStatus.NOT_FOUND;
    }

    @Override
    public String errorCode() {
        return "RESOURCE_NOT_FOUND";
    }

    public String getResourceType() { return resourceType; }
    public Object getResourceId() { return resourceId; }
}
```

`container/domain/exception/ContainerNotFoundException.java`:

```java
package com.yourcompany.portops.container.domain.exception;

import com.yourcompany.portops.common.exception.ResourceNotFoundException;

public class ContainerNotFoundException extends ResourceNotFoundException {
    public ContainerNotFoundException(Long id) {
        super("Container", id);
    }

    public ContainerNotFoundException(String bicCode) {
        super("Container (BIC)", bicCode);
    }
}
```

`container/domain/exception/DuplicateBicCodeException.java`:

```java
package com.yourcompany.portops.container.domain.exception;

import com.yourcompany.portops.common.exception.DomainException;
import org.springframework.http.HttpStatus;

public class DuplicateBicCodeException extends DomainException {

    private final String bicCode;

    public DuplicateBicCodeException(String bicCode) {
        super("Container with BIC code '%s' already exists".formatted(bicCode));
        this.bicCode = bicCode;
    }

    @Override
    public HttpStatus httpStatus() {
        return HttpStatus.CONFLICT;   // 409 — resource conflict
    }

    @Override
    public String errorCode() {
        return "DUPLICATE_BIC_CODE";
    }

    public String getBicCode() { return bicCode; }
}
```

`container/domain/exception/InvalidStatusTransitionException.java`:

```java
package com.yourcompany.portops.container.domain.exception;

import com.yourcompany.portops.common.exception.DomainException;
import com.yourcompany.portops.container.domain.ContainerStatus;
import org.springframework.http.HttpStatus;

public class InvalidStatusTransitionException extends DomainException {

    private final ContainerStatus fromStatus;
    private final ContainerStatus toStatus;

    public InvalidStatusTransitionException(ContainerStatus from, ContainerStatus to) {
        super("Illegal container status transition: %s → %s".formatted(from, to));
        this.fromStatus = from;
        this.toStatus = to;
    }

    @Override
    public HttpStatus httpStatus() {
        return HttpStatus.CONFLICT;
    }

    @Override
    public String errorCode() {
        return "INVALID_STATUS_TRANSITION";
    }

    public ContainerStatus getFromStatus() { return fromStatus; }
    public ContainerStatus getToStatus() { return toStatus; }
}
```

`shipment/domain/exception/VesselNotFoundException.java`:

```java
package com.yourcompany.portops.shipment.domain.exception;

import com.yourcompany.portops.common.exception.ResourceNotFoundException;

public class VesselNotFoundException extends ResourceNotFoundException {
    public VesselNotFoundException(Long id) {
        super("MotherVessel", id);
    }
}
```

### Exception design rules

| Rule | কারণ |
|---|---|
| সব domain exception `RuntimeException`-এর subclass | Checked exception method signature-এ pollution করে; Spring-এ idiom হচ্ছে unchecked |
| প্রত্যেক exception-এ meaningful data field (e.g., `bicCode`, `fromStatus`) | Error response-এ এই data expose করা যাবে |
| Common base class (`DomainException`) | Handler-এ একটা generic branch বানানো যাবে |
| Namespace proper: feature-এর exception feature package-এ, cross-cutting টা `common/exception`-এ | বড় codebase-এ maintain easier |

---

## Step 2 — Global Exception Handler (@RestControllerAdvice)

এখন সব exception catch করে proper HTTP response বানানোর জন্য একটা central handler।

`common/exception/GlobalExceptionHandler.java`:

```java
package com.yourcompany.portops.common.exception;

import com.yourcompany.portops.container.domain.exception.InvalidStatusTransitionException;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.time.Instant;
import java.util.Map;
import java.util.stream.Collectors;

@RestControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    /**
     * Domain exceptions — 99% cases এই branch-এ handle হবে।
     */
    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ProblemDetail> handleDomainException(
            DomainException ex, HttpServletRequest request) {

        log.warn("Domain exception at [{}]: {}", request.getRequestURI(), ex.getMessage());

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(ex.httpStatus(), ex.getMessage());
        problem.setTitle(humanizeErrorCode(ex.errorCode()));
        problem.setType(URI.create("https://portops.example.com/errors/" + ex.errorCode().toLowerCase().replace('_', '-')));
        problem.setProperty("errorCode", ex.errorCode());
        problem.setProperty("timestamp", Instant.now().toString());
        problem.setProperty("path", request.getRequestURI());

        // ResourceNotFoundException-এ resource info add করি
        if (ex instanceof ResourceNotFoundException rnf) {
            problem.setProperty("resourceType", rnf.getResourceType());
            problem.setProperty("resourceId", rnf.getResourceId());
        }
        // InvalidStatusTransitionException-এ transition info add করি
        if (ex instanceof InvalidStatusTransitionException ist) {
            problem.setProperty("fromStatus", ist.getFromStatus());
            problem.setProperty("toStatus", ist.getToStatus());
        }

        return ResponseEntity.status(ex.httpStatus()).body(problem);
    }

    /**
     * @Valid validation errors — Spring থেকে আসে, আমাদের throw করা না।
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ProblemDetail> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {

        Map<String, String> fieldErrors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() == null ? "invalid" : fe.getDefaultMessage(),
                (a, b) -> a  // duplicate key হলে প্রথমটা রাখ
            ));

        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation failed");
        problem.setDetail("One or more fields failed validation");
        problem.setProperty("errorCode", "VALIDATION_FAILED");
        problem.setProperty("fieldErrors", fieldErrors);
        problem.setProperty("timestamp", Instant.now().toString());
        problem.setProperty("path", request.getRequestURI());

        log.warn("Validation failed at [{}]: {}", request.getRequestURI(), fieldErrors);
        return ResponseEntity.badRequest().body(problem);
    }

    /**
     * Catch-all — কিছু unexpected হলে। Stack trace log, client-কে generic message।
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ProblemDetail> handleUnexpected(
            Exception ex, HttpServletRequest request) {

        log.error("Unhandled exception at [{}]", request.getRequestURI(), ex);

        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
            HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred. Please contact support with the trace ID."
        );
        problem.setTitle("Internal server error");
        problem.setProperty("errorCode", "INTERNAL_ERROR");
        problem.setProperty("timestamp", Instant.now().toString());
        problem.setProperty("path", request.getRequestURI());

        return ResponseEntity.internalServerError().body(problem);
    }

    private String humanizeErrorCode(String code) {
        return code.replace('_', ' ').toLowerCase();
    }
}
```

### কীভাবে কাজ করছে

1. **`@RestControllerAdvice`** — Spring-কে বলছে এটা একটা cross-cutting exception handler যা সব `@RestController`-এর throw catch করবে।
2. Exception throw হলে Spring type hierarchy match করে সবচেয়ে specific handler খোঁজে। যেমন `ContainerNotFoundException` throw হলে → `DomainException` handler match, কারণ সেটা superclass।
3. `ProblemDetail` — Spring 6-এর built-in RFC 7807 implementation। HTTP response content-type automatically `application/problem+json` হবে।

### RFC 7807 ProblemDetail কেন?

Industry standard। React team একটা consistent shape পাবে:

```json
{
  "type": "https://portops.example.com/errors/container-not-found",
  "title": "resource not found",
  "status": 404,
  "detail": "Container not found: 42",
  "errorCode": "RESOURCE_NOT_FOUND",
  "resourceType": "Container",
  "resourceId": 42,
  "timestamp": "2026-04-20T10:15:30Z",
  "path": "/api/v1/containers/42"
}
```

React team একটা error interceptor লিখবে:
```js
if (response.status === 404 && error.errorCode === "RESOURCE_NOT_FOUND") { ... }
```

---

## Step 3 — Service Refactor: Domain Exceptions ব্যবহার করুন

`ContainerService.java` আপডেট করুন — Phase 2-এর `IllegalArgumentException` সব replace করুন:

```java
package com.yourcompany.portops.container.domain;

import com.yourcompany.portops.container.api.dto.*;
import com.yourcompany.portops.container.api.mapper.ContainerMapper;
import com.yourcompany.portops.container.domain.exception.ContainerNotFoundException;
import com.yourcompany.portops.container.domain.exception.DuplicateBicCodeException;
import com.yourcompany.portops.container.persistence.ContainerRepository;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import com.yourcompany.portops.shipment.domain.exception.VesselNotFoundException;
import com.yourcompany.portops.shipment.persistence.MotherVesselRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class ContainerService {

    private static final Logger log = LoggerFactory.getLogger(ContainerService.class);

    private final ContainerRepository containerRepository;
    private final MotherVesselRepository motherVesselRepository;

    public ContainerService(ContainerRepository containerRepository,
                            MotherVesselRepository motherVesselRepository) {
        this.containerRepository = containerRepository;
        this.motherVesselRepository = motherVesselRepository;
    }

    public ContainerResponse register(CreateContainerRequest request) {
        if (containerRepository.existsByBicCode(request.bicCode())) {
            throw new DuplicateBicCodeException(request.bicCode());
        }

        MotherVessel vessel = motherVesselRepository.findById(request.motherVesselId())
            .orElseThrow(() -> new VesselNotFoundException(request.motherVesselId()));

        Container container = new Container();
        container.setBicCode(request.bicCode());
        container.setType(request.type());
        container.setStatus(ContainerStatus.ON_VESSEL);
        container.setGrossWeightKg(request.grossWeightKg());
        container.setMotherVessel(vessel);

        Container saved = containerRepository.save(container);
        log.info("Registered container id={} bicCode={}", saved.getId(), saved.getBicCode());
        return ContainerMapper.toResponse(saved);
    }

    @Transactional(readOnly = true)
    public ContainerResponse findById(Long id) {
        return containerRepository.findById(id)
            .map(ContainerMapper::toResponse)
            .orElseThrow(() -> new ContainerNotFoundException(id));
    }

    @Transactional(readOnly = true)
    public Page<ContainerSummary> search(ContainerStatus status, Pageable pageable) {
        Page<Container> page = (status != null)
            ? containerRepository.findByStatus(status, pageable)
            : containerRepository.findAll(pageable);
        return page.map(ContainerMapper::toSummary);
    }

    public void delete(Long id) {
        if (!containerRepository.existsById(id)) {
            throw new ContainerNotFoundException(id);
        }
        containerRepository.deleteById(id);
        log.info("Deleted container id={}", id);
    }
}
```

**Note:** Phase 2-এর `updateStatus(...)` method টা সরিয়ে দিন — এই logic Phase 3-এ `ContainerMovementService`-এ যাবে proper transition validation সহ। Controller থেকেও `PATCH /status` endpoint টা temporarily remove করুন।

### `MotherVesselController`-ও update করুন

Phase 2-এ এটা direct repository ব্যবহার করছিল। এখন proper service বানান।

`shipment/domain/MotherVesselService.java`:

```java
package com.yourcompany.portops.shipment.domain;

import com.yourcompany.portops.shipment.api.dto.*;
import com.yourcompany.portops.shipment.domain.exception.VesselNotFoundException;
import com.yourcompany.portops.shipment.persistence.MotherVesselRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Transactional
public class MotherVesselService {

    private static final Logger log = LoggerFactory.getLogger(MotherVesselService.class);

    private final MotherVesselRepository repository;

    public MotherVesselService(MotherVesselRepository repository) {
        this.repository = repository;
    }

    public VesselResponse register(CreateVesselRequest request) {
        MotherVessel v = new MotherVessel();
        v.setImoNumber(request.imoNumber());
        v.setVesselName(request.vesselName());
        v.setFlagCountry(request.flagCountry());
        MotherVessel saved = repository.save(v);
        log.info("Registered vessel id={} imo={}", saved.getId(), saved.getImoNumber());
        return toResponse(saved);
    }

    @Transactional(readOnly = true)
    public List<VesselResponse> listAll() {
        return repository.findAll().stream().map(this::toResponse).toList();
    }

    @Transactional(readOnly = true)
    public VesselResponse findById(Long id) {
        return repository.findById(id)
            .map(this::toResponse)
            .orElseThrow(() -> new VesselNotFoundException(id));
    }

    private VesselResponse toResponse(MotherVessel v) {
        return new VesselResponse(v.getId(), v.getImoNumber(), v.getVesselName(),
                                  v.getFlagCountry(), v.getCreatedAt());
    }
}
```

Controller-এ `MotherVesselRepository` এর বদলে service inject করুন, endpoint গুলো service-এ delegate করান।

---

## Step 4 — Status Transition Validation (State Machine)

Container status যেকোনো status থেকে যেকোনো status-এ যেতে পারে না। Business rules আছে:

```
ON_VESSEL → ON_LIGHTER → AT_GATE → IN_YARD → LOADING → EXITED
```

(Import flow। Export-এ reverse: `EXITED` → `LOADING` → ... পরের phase-এ।)

এই rules enforce করার সবচেয়ে clean way — state machine pattern। Enum-এ এটা embedded করা যায়।

### `ContainerStatus` enum-এ transitions যোগ করুন

`container/domain/ContainerStatus.java` update:

```java
package com.yourcompany.portops.container.domain;

import java.util.EnumSet;
import java.util.Set;

public enum ContainerStatus {
    ON_VESSEL(EnumSet.of()),                      // start state (set at registration)
    ON_LIGHTER(EnumSet.of()),
    AT_GATE(EnumSet.of()),
    IN_YARD(EnumSet.of()),
    LOADING(EnumSet.of()),
    EXITED(EnumSet.of());                         // terminal state

    // নিচে static initializer-এ actual transitions wire করব
    private Set<ContainerStatus> allowedNextStates;

    ContainerStatus(Set<ContainerStatus> allowedNextStates) {
        this.allowedNextStates = allowedNextStates;
    }

    static {
        ON_VESSEL.allowedNextStates  = EnumSet.of(ON_LIGHTER);
        ON_LIGHTER.allowedNextStates = EnumSet.of(AT_GATE);
        AT_GATE.allowedNextStates    = EnumSet.of(IN_YARD);
        IN_YARD.allowedNextStates    = EnumSet.of(LOADING, EXITED);  // direct exit if no loading
        LOADING.allowedNextStates    = EnumSet.of(EXITED);
        EXITED.allowedNextStates     = EnumSet.noneOf(ContainerStatus.class);  // terminal
    }

    public boolean canTransitionTo(ContainerStatus next) {
        return allowedNextStates.contains(next);
    }

    public boolean isTerminal() {
        return allowedNextStates.isEmpty();
    }
}
```

> **Kotlin-এর `sealed class` pattern mimic করা হচ্ছে।** Kotlin-এ state machine আরো elegant হয় sealed class + `when`, কিন্তু Java enum-ও যথেষ্ট clean।

### কেন static initializer

Enum constants circular reference করতে পারে না constructor-এ (self-referential enum issue)। তাই `EnumSet.of()` দিয়ে empty initialize করে static block-এ actual values set করছি। এটা standard Java pattern।

### Test করুন (IDE-তে quick check)

```java
ContainerStatus.ON_VESSEL.canTransitionTo(ContainerStatus.ON_LIGHTER);  // true
ContainerStatus.ON_VESSEL.canTransitionTo(ContainerStatus.IN_YARD);     // false — skip not allowed
ContainerStatus.EXITED.isTerminal();                                    // true
```

---

## Step 5 — `ContainerMovementService`: Atomic Business Workflow

এটাই Phase 3-এর showcase। একটা single workflow — "gate in" — যেটা multiple DB operations atomic করে।

### Workflow: Gate-In

Container lighter ship থেকে port gate-এ পৌঁছেছে। আমাদের systems-এ:

1. Container status transition: `ON_LIGHTER` → `AT_GATE` (validate কর valid transition কিনা)
2. A `YardActivity` record তৈরি করো — activity type `GATE_IN`, operator info, timestamp
3. Container status আবার update: `AT_GATE` → `IN_YARD` (container slot-এ placed)
4. আরেকটা `YardActivity` — activity type yard placement
5. (Phase 4-5 এ) Billing entry তৈরি করো

সব step **একসাথে commit হবে, নয়তো কিছুই না**। মাঝখানে কোনো step fail হলে — আগের সবকিছু rollback।

### DTOs বানান

`yard/api/dto/GateInRequest.java`:

```java
package com.yourcompany.portops.yard.api.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;

public record GateInRequest(
    @NotBlank
    @Pattern(regexp = "[A-Z]-\\d{2}-\\d{2}-\\d", message = "Invalid yard slot format (e.g., A-07-03-2)")
    String yardSlotCode,

    String remarks
) {}
```

`yard/api/dto/YardActivityResponse.java`:

```java
package com.yourcompany.portops.yard.api.dto;

import com.yourcompany.portops.yard.domain.YardActivityType;
import java.time.Instant;

public record YardActivityResponse(
    Long id,
    Long containerId,
    String containerBicCode,
    YardActivityType activityType,
    String yardSlotCode,
    Instant occurredAt,
    String remarks
) {}
```

### Service-এ লেখেন

`container/domain/ContainerMovementService.java`:

```java
package com.yourcompany.portops.container.domain;

import com.yourcompany.portops.container.domain.exception.ContainerNotFoundException;
import com.yourcompany.portops.container.domain.exception.InvalidStatusTransitionException;
import com.yourcompany.portops.container.persistence.ContainerRepository;
import com.yourcompany.portops.yard.api.dto.GateInRequest;
import com.yourcompany.portops.yard.api.dto.YardActivityResponse;
import com.yourcompany.portops.yard.domain.YardActivity;
import com.yourcompany.portops.yard.domain.YardActivityType;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.Instant;

@Service
@Transactional
public class ContainerMovementService {

    private static final Logger log = LoggerFactory.getLogger(ContainerMovementService.class);

    private final ContainerRepository containerRepository;

    public ContainerMovementService(ContainerRepository containerRepository) {
        this.containerRepository = containerRepository;
    }

    /**
     * Gate-in workflow:
     * - Container-এর status validate: ON_LIGHTER থেকে AT_GATE-এ যেতে পারে কিনা
     * - Gate entry activity তৈরি
     * - Container status update: AT_GATE → IN_YARD
     * - Yard placement activity তৈরি
     *
     * সবকিছু এক transaction-এ। Yard slot invalid হলে পুরোটা rollback।
     */
    public YardActivityResponse gateIn(Long containerId, GateInRequest request) {
        Container container = containerRepository.findById(containerId)
            .orElseThrow(() -> new ContainerNotFoundException(containerId));

        // Step 1: Status transition validation
        if (!container.getStatus().canTransitionTo(ContainerStatus.AT_GATE)) {
            throw new InvalidStatusTransitionException(
                container.getStatus(), ContainerStatus.AT_GATE);
        }

        Instant now = Instant.now();

        // Step 2: Gate entry transition + activity log
        container.setStatus(ContainerStatus.AT_GATE);
        YardActivity gateActivity = new YardActivity();
        gateActivity.setActivityType(YardActivityType.GATE_IN);
        gateActivity.setYardSlotCode(request.yardSlotCode());
        gateActivity.setOccurredAt(now);
        gateActivity.setRemarks(request.remarks());
        container.addYardActivity(gateActivity);  // helper method — bidirectional link setup

        // Step 3: Immediate yard placement (AT_GATE → IN_YARD)
        if (!container.getStatus().canTransitionTo(ContainerStatus.IN_YARD)) {
            throw new InvalidStatusTransitionException(
                container.getStatus(), ContainerStatus.IN_YARD);
        }
        container.setStatus(ContainerStatus.IN_YARD);

        YardActivity moveActivity = new YardActivity();
        moveActivity.setActivityType(YardActivityType.YARD_MOVE);
        moveActivity.setYardSlotCode(request.yardSlotCode());
        moveActivity.setOccurredAt(now.plusSeconds(1));  // slight temporal ordering
        container.addYardActivity(moveActivity);

        // Note: save() call লাগছে না — container managed, transaction commit-এ Hibernate auto-flush করবে
        // যেহেতু Container.yardActivities-এ cascade = ALL, new YardActivity-গুলোও persist হবে

        log.info("Container {} gated in at slot {}, now IN_YARD",
                 container.getBicCode(), request.yardSlotCode());

        return toResponse(moveActivity, container);
    }

    private YardActivityResponse toResponse(YardActivity a, Container c) {
        return new YardActivityResponse(
            a.getId(),
            c.getId(),
            c.getBicCode(),
            a.getActivityType(),
            a.getYardSlotCode(),
            a.getOccurredAt(),
            a.getRemarks()
        );
    }
}
```

### এখানে কিছু subtle जिनिस

**1. `save()` call নেই container-এ**
- `findById()` থেকে returned entity **managed** (Hibernate persistence context track করছে)
- যেকোনো field modification (status, child collection) transaction commit-এ auto UPDATE/INSERT query generate করবে
- এটা Hibernate "dirty checking" — Room-এর সাথে সবচেয়ে বড় পার্থক্য

**2. `container.addYardActivity(...)` helper method**
- Phase 2-এ Container entity-তে এটা বানিয়েছিলেন:
  ```java
  public void addYardActivity(YardActivity activity) {
      yardActivities.add(activity);
      activity.setContainer(this);
  }
  ```
- Bidirectional relationship-এ **দুই side-ই set করতে হয়**। শুধু list-এ add করলে `YardActivity.container` null থাকবে, DB-তে FK orphan
- `cascade = ALL` থাকায় child-ও persist হবে parent save-এ

**3. যদি Step 3-এ exception হতো?**
- Status transition invalid বলে exception throw হতো
- `@Transactional` method থেকে RuntimeException bubble up
- Spring transaction ROLLBACK করত
- Step 2-এ container.status = AT_GATE set করা হয়েছিল — সেটাও rollback
- gateActivity যা in-memory add হয়েছিল — DB-তে যায়নি (flush হয়নি)
- Net effect: DB-তে কিছুই change হয়নি

এটাই atomic operation-এর জাদু।

### Controller endpoint যোগ করুন

`container/api/ContainerController.java`-এ `ContainerMovementService` inject করুন:

```java
@RestController
@RequestMapping("/api/v1/containers")
public class ContainerController {

    private final ContainerService containerService;
    private final ContainerMovementService movementService;

    public ContainerController(ContainerService containerService,
                               ContainerMovementService movementService) {
        this.containerService = containerService;
        this.movementService = movementService;
    }

    // ... existing endpoints ...

    @PostMapping("/{id}/gate-in")
    public YardActivityResponse gateIn(@PathVariable Long id,
                                       @Valid @RequestBody GateInRequest request) {
        return movementService.gateIn(id, request);
    }
}
```

Imports add করতে ভুলবেন না।

---

## Step 6 — `@Transactional` Deep Dive

এখন Phase 3-এর সবচেয়ে subtle topic — transaction behavior।

### Rollback rules

Default:
- **`RuntimeException` (unchecked)** → rollback
- **`Exception` (checked)** → **NO rollback** ⚠️

এটা Spring-এর quirk। আমাদের সব domain exceptions `DomainException extends RuntimeException`, তাই ঠিক আছে।

কিন্তু কখনো যদি checked exception throw করেন:

```java
@Transactional(rollbackFor = {IOException.class, SQLException.class})
public void importFromFile() throws IOException { ... }
```

`rollbackFor` explicit set করুন।

**Rule:** Domain exceptions সব `RuntimeException`-এ রাখুন — এই ঝামেলা এড়ানো যায়।

### Propagation

`@Transactional` method অন্য `@Transactional` method call করলে কী হবে?

```java
@Service
class ContainerService {
    @Transactional
    public void outerMethod() {
        innerService.innerMethod();   // অন্য service call
    }
}

@Service
class InnerService {
    @Transactional
    public void innerMethod() { ... }
}
```

Default propagation = `REQUIRED` মানে:
- যদি already একটা transaction চলমান থাকে → সেটা join করবে
- না থাকলে → নতুন শুরু করবে

### কখন `REQUIRES_NEW` লাগবে?

একটা real scenario: audit logging।

```java
@Service
class ContainerMovementService {
    @Transactional
    public YardActivityResponse gateIn(...) {
        // ... main logic
        auditService.logMovement(containerId, action);  // audit log করতে হবে
        // ... more main logic — এটা fail হলে?
    }
}

@Service
class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logMovement(Long containerId, String action) {
        // এই method নিজের আলাদা transaction-এ চলবে
        // Main transaction rollback হলেও audit log persist থাকবে
    }
}
```

**Use case:** Audit/compliance logs — business logic fail হলেও log-এ entry থাকা চাই "user tried to do X, failed"।

**Warning:** `REQUIRES_NEW` suspend করে outer transaction, নতুন connection ধার নেয়। Over-use করলে connection pool exhaust হতে পারে।

### `readOnly = true`

```java
@Transactional(readOnly = true)
public Page<ContainerSummary> search(...) { ... }
```

- Hibernate dirty-checking skip করে (performance)
- Some DBs-এ (Oracle) hint পাঠায় read-only transaction-এর
- Intent clear — code reviewer বুঝতে পারে এই method write করবে না

**All query methods-এ এটা দিন।**

### `@Transactional` কোথায় বসাবে?

| জায়গা | উচিত? |
|---|---|
| Controller method | ❌ না। Controller pure HTTP translator |
| Service method | ✅ হ্যাঁ। business logic atomic boundary |
| Repository method | ⚠️ সাধারণত লাগে না। Spring Data JPA auto-wraps |
| Entity/DTO | ❌ অপ্রাসঙ্গিক |

### Self-invocation trap

```java
@Service
class ContainerService {

    public void methodA() {
        methodB();  // ⚠️ @Transactional কাজ করবে না!
    }

    @Transactional
    public void methodB() { ... }
}
```

কেন? Spring transactions proxy-based। আপনি যখন `containerService.methodB()` call করেন external থেকে, আপনি আসলে proxy-কে call করেন যা transaction start করে তারপর real method-এ যায়। কিন্তু `this.methodB()` — proxy bypass, direct method call। No transaction।

**সমাধান:** Another bean-এ method move করুন, বা (advanced) `@EnableAspectJAutoProxy(exposeProxy=true)` + `AopContext.currentProxy()` use করুন — first option usually ভালো।

---

## Step 7 — MapStruct Introduction

Phase 2-এ আমরা manual mapper লিখেছি। Enterprise projects-এ MapStruct বা ModelMapper use হয়। MapStruct better because **compile-time code generation** — runtime overhead নেই।

### Dependency add করুন

`pom.xml`-এ:

```xml
<dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>1.5.5.Final</version>
</dependency>
```

আর `<build><plugins>` section-এ (maven-compiler-plugin ইতিমধ্যে থাকবে, সেটা extend করুন):

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <annotationProcessorPaths>
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </path>
            <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.5.5.Final</version>
            </path>
            <!-- IMPORTANT: MapStruct আর Lombok একসাথে use করলে এই binding লাগবে -->
            <path>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok-mapstruct-binding</artifactId>
                <version>0.2.0</version>
            </path>
        </annotationProcessorPaths>
    </configuration>
</plugin>
```

Maven reload।

### MapStruct দিয়ে Container Mapper

পুরানো `ContainerMapper.java` (manual) delete করুন বা rename `ContainerMapperManual`।

নতুন `container/api/mapper/ContainerMapper.java`:

```java
package com.yourcompany.portops.container.api.mapper;

import com.yourcompany.portops.container.api.dto.ContainerResponse;
import com.yourcompany.portops.container.api.dto.ContainerSummary;
import com.yourcompany.portops.container.domain.Container;
import com.yourcompany.portops.shipment.domain.MotherVessel;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;

@Mapper(componentModel = "spring")
public interface ContainerMapper {

    @Mapping(source = "motherVessel", target = "motherVessel", qualifiedByName = "vesselToSummary")
    ContainerResponse toResponse(Container entity);

    ContainerSummary toSummary(Container entity);

    @org.mapstruct.Named("vesselToSummary")
    default ContainerResponse.VesselSummary vesselToSummary(MotherVessel v) {
        if (v == null) return null;
        return new ContainerResponse.VesselSummary(v.getId(), v.getImoNumber(), v.getVesselName());
    }
}
```

### যা হচ্ছে

- `@Mapper(componentModel = "spring")` — MapStruct compile time-এ `ContainerMapperImpl` class generate করবে, সেটা Spring bean হিসেবে register হবে
- Field names যদি match করে (id, bicCode, type, status, ...), auto-map হবে। শুধু custom mapping (`motherVessel → VesselSummary`) specify করতে হবে

### Service-এ use

```java
@Service
public class ContainerService {
    private final ContainerMapper containerMapper;   // Spring inject করবে generated impl

    public ContainerService(..., ContainerMapper containerMapper) {
        ...
        this.containerMapper = containerMapper;
    }

    public ContainerResponse register(...) {
        ...
        return containerMapper.toResponse(saved);
    }
}
```

`ContainerMapper.toResponse(...)` static call-এর বদলে instance method। Service-এ inject করে use করুন।

### Build করুন এবং generated code দেখুন

`./mvnw compile` চালান। `target/generated-sources/annotations/.../ContainerMapperImpl.java` তে generated file দেখতে পাবেন — plain Java code, no reflection, no runtime cost।

---

## Step 8 — Structured Logging

SLF4J + Logback (Spring Boot default) ব্যবহার করছি। কয়েকটা rule:

### Logger instantiation

```java
private static final Logger log = LoggerFactory.getLogger(ContainerService.class);
```

Class-level `static final` — idiomatic।

### Log levels — কখন কোনটা

| Level | কখন |
|---|---|
| `ERROR` | Unexpected exception, external system down, retry exhausted |
| `WARN` | Validation failure, business rule violation, degraded behavior |
| `INFO` | Significant business events — container registered, gate-in done, invoice generated |
| `DEBUG` | Flow tracing, method entry/exit, intermediate values |
| `TRACE` | Very detailed — framework internals, SQL binding params |

### Parameterized logging — string concat না

```java
// ❌ বাজে — string concat সবসময় হয়, debug disabled হলেও
log.debug("Processing container " + containerId + " status " + status);

// ✅ ভালো — placeholders, debug disabled হলে concat skip
log.debug("Processing container {} status {}", containerId, status);

// ✅ Exception logging — last argument Throwable
log.error("Failed to persist container {}", bicCode, ex);
```

### Don't log sensitive data

- ❌ JWT tokens, passwords, credit card numbers
- ❌ Full request bodies (PII leak)
- ⚠️ BIC codes, vessel IMO — business data, usually OK

### Correlation IDs (preview, Phase 7)

Production-এ প্রত্যেক HTTP request-কে একটা unique ID দিতে হয় (trace করতে)। Spring-এ `MDC` (Mapped Diagnostic Context) ব্যবহার করে। Week 2-এ Micrometer Tracing integrate করব। এখন basic logging-ই যথেষ্ট।

---

## Step 9 — Postman-এ Error Responses Test করুন

App restart করে এই test cases চালান। এইগুলো Phase 3-এর real proof।

### Test 1: Non-existent container

```
GET http://localhost:8080/api/v1/containers/9999
```

**Expected response: 404 Not Found**
```json
{
  "type": "https://portops.example.com/errors/resource-not-found",
  "title": "resource not found",
  "status": 404,
  "detail": "Container not found: 9999",
  "errorCode": "RESOURCE_NOT_FOUND",
  "resourceType": "Container",
  "resourceId": 9999,
  "timestamp": "2026-04-20T...",
  "path": "/api/v1/containers/9999"
}
```

### Test 2: Duplicate BIC code

প্রথমে একটা container তৈরি করুন `MSCU1234567` দিয়ে। তারপর একই BIC code দিয়ে আরেকটা তৈরির চেষ্টা:

```
POST http://localhost:8080/api/v1/containers
Content-Type: application/json

{
  "bicCode": "MSCU1234567",
  "type": "DRY_20",
  "grossWeightKg": 15000,
  "motherVesselId": 1
}
```

**Expected: 409 Conflict**
```json
{
  "status": 409,
  "detail": "Container with BIC code 'MSCU1234567' already exists",
  "errorCode": "DUPLICATE_BIC_CODE",
  ...
}
```

### Test 3: Invalid vessel

```
POST http://localhost:8080/api/v1/containers
Content-Type: application/json

{
  "bicCode": "MSCU9999999",
  "type": "DRY_40",
  "grossWeightKg": 20000,
  "motherVesselId": 9999
}
```

**Expected: 404 Not Found**, errorCode `RESOURCE_NOT_FOUND`, resourceType `MotherVessel`.

### Test 4: Validation error

```
POST http://localhost:8080/api/v1/containers
Content-Type: application/json

{
  "bicCode": "invalid",
  "type": "DRY_40",
  "grossWeightKg": -100,
  "motherVesselId": null
}
```

**Expected: 400 Bad Request**
```json
{
  "status": 400,
  "detail": "One or more fields failed validation",
  "errorCode": "VALIDATION_FAILED",
  "fieldErrors": {
    "bicCode": "BIC code must be 4 letters followed by 7 digits",
    "grossWeightKg": "Weight must be positive",
    "motherVesselId": "Mother vessel ID is required"
  }
}
```

### Test 5: Gate-in happy path

আগে একটা container manually `ON_LIGHTER` status-এ নিন (DB-তে direct update, বা আপনি চাইলে একটা debug endpoint বানান Phase 2-এর মতো)। তারপর:

```
POST http://localhost:8080/api/v1/containers/1/gate-in
Content-Type: application/json

{
  "yardSlotCode": "A-07-03-2",
  "remarks": "Routine gate entry"
}
```

**Expected: 200 OK**
```json
{
  "id": 2,
  "containerId": 1,
  "containerBicCode": "MSCU1234567",
  "activityType": "YARD_MOVE",
  "yardSlotCode": "A-07-03-2",
  "occurredAt": "2026-04-20T...",
  "remarks": null
}
```

DB check:
- `CONTAINERS`-এ container status `IN_YARD`
- `YARD_ACTIVITIES`-এ 2টা new row (`GATE_IN` আর `YARD_MOVE`)

### Test 6: Invalid status transition

Container-কে status `EXITED` করুন manually। তারপর:

```
POST http://localhost:8080/api/v1/containers/1/gate-in
Content-Type: application/json

{
  "yardSlotCode": "A-07-03-2"
}
```

**Expected: 409 Conflict**
```json
{
  "status": 409,
  "detail": "Illegal container status transition: EXITED → AT_GATE",
  "errorCode": "INVALID_STATUS_TRANSITION",
  "fromStatus": "EXITED",
  "toStatus": "AT_GATE"
}
```

DB-তে কোনো change হয়নি — rollback proof।

### Test 7: Invalid yard slot format (validation)

```
POST http://localhost:8080/api/v1/containers/1/gate-in
Content-Type: application/json

{
  "yardSlotCode": "WRONG"
}
```

**Expected: 400 Bad Request** with `fieldErrors.yardSlotCode`।

---

## Phase 3 Milestone Checklist

- [ ] `DomainException` base class + domain-specific subclasses তৈরি
- [ ] `GlobalExceptionHandler` সব exception handle করে ProblemDetail response return করছে
- [ ] সব service-এ `IllegalArgumentException`/`IllegalStateException` replaced by domain exceptions
- [ ] `ContainerStatus` enum-এ state machine (canTransitionTo, isTerminal)
- [ ] `ContainerMovementService.gateIn()` কাজ করে — atomic, multi-step
- [ ] Invalid status transition rollback demonstrate করেছেন
- [ ] MapStruct integrate, generated `ContainerMapperImpl` দেখেছেন
- [ ] সব query method `@Transactional(readOnly = true)`
- [ ] Logging SLF4J parameterized format-এ, meaningful log messages
- [ ] Postman-এ 7টা test case সব expected response দিচ্ছে

---

## Self-Check Questions

1. `DomainException` কেন `RuntimeException`-এর subclass — `Exception`-এর না?
2. `@RestControllerAdvice` আর `@ControllerAdvice`-এর পার্থক্য?
3. RFC 7807 `ProblemDetail`-এ কোন fields standard, কোনগুলো custom (আপনার `errorCode`, `timestamp`, `resourceType`)?
4. `@Transactional` কেন controller-এ না বসিয়ে service-এ বসান?
5. Default-এ কোন exceptions rollback trigger করে, কোনটা করে না?
6. Self-invocation trap কী — `this.methodB()` call করলে `@Transactional` কাজ করে না কেন?
7. `Propagation.REQUIRED` vs `REQUIRES_NEW` — কখন কোনটা?
8. Managed entity-এর field change করলে `save()` ছাড়া DB update হয় কেন? এটার limitation কী?
9. MapStruct compile-time code generation — runtime reflection-based mapper (ModelMapper) থেকে কী benefit?
10. `container.addYardActivity(activity)` helper method-এ কেন দুই side সেট করতে হয়?

---

## Troubleshooting

### Problem: `ProblemDetail` JSON-এ extra properties show হচ্ছে না
**কারণ:** Jackson ProblemDetail-এর `properties` map serialize করে, কিন্তু default configuration-এ flat হয়ে যায়।
**সমাধান:** Spring Boot 3.2+ এ by default ঠিক কাজ করে। পুরানো version হলে `application.yml`-এ:
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

### Problem: MapStruct `NullPointerException` generated code-এ
**কারণ:** Source field null।
**সমাধান:** `@Mapper(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)` বা per-field null check।

### Problem: MapStruct processor IDE-তে generate করে না
**সমাধান:**
- IntelliJ: Settings → Build → Compiler → Annotation Processors → Enable
- `target/generated-sources/annotations` directory Mark as Generated Sources Root

### Problem: Exception handler fire হয় না, default 500 আসে
**Checklist:**
1. `GlobalExceptionHandler` class `@RestControllerAdvice` annotated?
2. Package scan-এর আওতায় আছে? (`com.yourcompany.portops.*`)
3. Your exception সত্যি `DomainException`-এর subclass?

### Problem: `@Transactional` কাজ করছে না (rollback হচ্ছে না)
**Checklist:**
1. Exception `RuntimeException`-এর subclass?
2. Self-invocation করছেন না তো?
3. Method `public`? (private/protected-এ proxy কাজ করে না)
4. Class Spring bean? (`@Service` etc.)

### Problem: `LazyInitializationException` gate-in response বানানোর সময়
**কারণ:** `toResponse()` method transaction-এর বাইরে lazy field access করছে।
**সমাধান:** Mapping service method-এর ভিতরে রাখুন (Phase 3-এর কোডে সেটা আছে), না হলে mapping-এর জন্য JOIN FETCH query।

### Problem: `DuplicateBicCodeException` throw হচ্ছে কিন্তু status 500 আসছে
**কারণ:** Handler registration সম্ভবত ভুল package-এ।
**সমাধান:** Stacktrace console-এ দেখুন — যদি DB-level `ConstraintViolationException` দেখেন, মানে আপনার `existsByBicCode` check miss করছে (race condition-এ সম্ভব)। এই case-এ `DataIntegrityViolationException` catch করে domain exception-এ convert করতে হবে handler-এ।

---

## Phase 3 Wrap-up

এই phase-এ আপনি Spring Boot service-এর "muscle" বানালেন — data plumbing (Phase 2) থেকে actual business logic-এ transition করলেন। এবার আপনার code-এ:

- **Controllers thin** — শুধু HTTP translation
- **Services own business rules** — transactions, validations, orchestration
- **Exceptions domain-aware** — HTTP status, error codes, context data
- **Responses consistent** — React team একটা shape expect করতে পারে

### Phase 4-এ যাব

- Spring Security 6 filter chain
- Stateless JWT authentication
- User entity + roles (OPERATOR, BILLING_CLERK, ADMIN)
- `POST /auth/login` endpoint
- Role-based endpoint access (`@PreAuthorize`)
- CORS configuration for React frontend

### আমাকে যা পাঠাবেন

1. **"Phase 3 done"** confirmation
2. **Postman screenshot** — Test 5 (gate-in happy path) এর successful response + Test 6 (invalid transition) এর 409 response
3. **Generated MapStruct file-এর path screenshot** — IntelliJ project view-এ `target/generated-sources/...ContainerMapperImpl.java` visible
4. **আপনার লেখা কোনো custom exception** — যেটা আমি list-এ দেইনি কিন্তু আপনি মনে করছেন domain-এ দরকার (e.g., `ContainerOverweightException`, `YardSlotOccupiedException`) — ideas শেয়ার করলে Phase 4-এ integrate করব

Phase 3-এর পর আপনার codebase এখন real enterprise-grade। Phase 4-এ security add করলে এটা প্রায় production-deployable হয়ে যাবে।

শুরু করুন। আটকালে জানান।
