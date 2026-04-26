# Spring Boot — Bean Validation Complete Guide

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Level:** Beginner থেকে Intermediate
> **Previous:** Exception Handling (এটা পরের)
> **Topics:** `@Valid`, Built-in annotations, Custom Validators, Group Validation

---

## সূচিপত্র

1. [Prerequisites — Foundation](#পর্ব-০)
2. [Setup — Dependency](#পর্ব-১)
3. [`@Valid` — কীভাবে কাজ করে](#পর্ব-২)
4. [Built-in Validation Annotations](#পর্ব-৩)
5. [`@NotNull` vs `@NotEmpty` vs `@NotBlank` — সবচেয়ে Confusing!](#পর্ব-৪)
6. [Number Validations](#পর্ব-৫)
7. [String Validations](#পর্ব-৬)
8. [Date Validations](#পর্ব-৭)
9. [Validation Fail হলে কী হয়](#পর্ব-৮)
10. [Exception Handler এ ধরা](#পর্ব-৯)
11. [Nested Object Validation](#পর্ব-১০)
12. [List Validation](#পর্ব-১১)
13. [Custom Validator (Deep Dive)](#পর্ব-১২)
14. [Group Validation](#পর্ব-১৩)
15. [Common Mistakes](#পর্ব-১৪)

---

<a name="পর্ব-০"></a>

# পর্ব ০ — Prerequisites

## ১. Validation কী?

### সাধারণ ভাষায়

"Validation" মানে **যাচাই করা।** Client থেকে আসা data সঠিক কিনা — সেটা চেক করা।

### একটা সহজ উদাহরণ

User registration form —

```json
POST /users
{
  "name": "",
  "email": "abc",
  "age": -5,
  "password": "123"
}
```

এই data accept করা যায়?
- name খালি ❌
- email format ভুল ❌
- age negative ❌
- password ৩ character (too short) ❌

এই check গুলোই Validation।

---

## ২. Validation কেন দরকার?

### Scenario — Validation না থাকলে

```java
@PostMapping("/users")
public User createUser(@RequestBody UserDTO dto) {
    User user = new User();
    user.setName(dto.getName());        // "" save হবে
    user.setEmail(dto.getEmail());       // "abc" save হবে
    user.setAge(dto.getAge());           // -5 save হবে
    return userRepository.save(user);
}
```

**সমস্যা —**

```
Database এ:
- নাম ছাড়া User
- ভুল email
- Negative age
- Garbage data!
```

পরে যখন email পাঠাতে যাবে — fail হবে। App crash হবে।

### Validation থাকলে

```
Invalid data এলো
        ↓
Spring সাথে সাথে reject করলো
        ↓
Database এ যাবেই না ✅
        ↓
Client কে সুন্দর error message
```

---

## ৩. Validation কোথায় করতে হয়?

### দুই জায়গায় — কিন্তু কেন?

```
Frontend Validation:
  ✅ User experience ভালো (instant feedback)
  ❌ Security নেই (developer tools দিয়ে bypass করা যায়)

Backend Validation:
  ✅ Security guaranteed (bypass করা যায় না)
  ❌ Server এ extra processing

দুইটাই দরকার:
  Frontend → User এর জন্য
  Backend  → Security এর জন্য (must-have)
```

আমরা **Backend Validation** শিখবো — Spring Boot এ।

---

## ৪. Bean Validation কী?

**Bean Validation** = Java এর একটা **standard specification** (JSR 380)।

> *"Java Object এর field গুলো কীভাবে validate করতে হবে তার নিয়ম।"*

### Implementation — Hibernate Validator

Bean Validation শুধু specification (নিয়ম)। Implementation হলো **Hibernate Validator** (Spring Boot এ default)।

```
Bean Validation     =  Rules (interface)
Hibernate Validator =  Implementation (actual code)
```

ঠিক যেমন আগে JPA আর Hibernate এর সম্পর্ক দেখেছিলে।

---

<a name="পর্ব-১"></a>

# পর্ব ১ — Setup

## Dependency

`pom.xml` এ এই dependency add করো —

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

ব্যস! Hibernate Validator automatically add হয়ে যাবে।

> Spring Boot এর কিছু version এ এটা automatically থাকে — `spring-boot-starter-web` এর সাথে। তবু explicit add করা best practice।

---

<a name="পর্ব-২"></a>

# পর্ব ২ — `@Valid` কীভাবে কাজ করে?

## প্রথম Example

### DTO তে Validation Rules দাও

```java
public class UserCreateDTO {

    @NotBlank(message = "Name is required")      // ⭐ Rule 1
    private String name;

    @Email(message = "Invalid email format")     // ⭐ Rule 2
    private String email;

    // getters, setters
}
```

### Controller এ `@Valid` দাও

```java
@PostMapping("/users")
public User createUser(@Valid @RequestBody UserCreateDTO dto) {    // ⭐ @Valid
    return userService.createUser(dto);
}
```

### `@Valid` এর কাজ কী?

`@Valid` Spring কে বলে —

> *"এই DTO এর সব validation rules check করো। ভুল হলে exception throw করো।"*

---

## পর্দার পেছনে কী হয়?

```
Client → POST /users { "name": "", "email": "abc" }
        ↓
Spring DTO তে data fill করলো
        ↓
@Valid দেখলো → Hibernate Validator কে call করলো
        ↓
Validator প্রতিটা field check করলো:
  - name: @NotBlank fail (empty) ❌
  - email: @Email fail (format wrong) ❌
        ↓
সব error collect করলো
        ↓
MethodArgumentNotValidException throw করলো
        ↓
Controller এর method চলবে না
        ↓
Spring Exception Handler খুঁজলো
        ↓
সুন্দর error response দিলো
```

**এই পুরো flow automatic — তুমি কিছু লিখো না।**

---

<a name="পর্ব-৩"></a>

# পর্ব ৩ — Built-in Validation Annotations

Bean Validation এ অনেক ready-made annotation আছে। আমরা এগুলো বিভাগ করে শিখবো —

```
1. Null/Empty check    → @NotNull, @NotEmpty, @NotBlank
2. String                → @Size, @Pattern, @Email
3. Number                → @Min, @Max, @Positive, @Negative
4. Date                  → @Past, @Future
5. Boolean               → @AssertTrue, @AssertFalse
```

প্রতিটা একটা একটা করে দেখবো — গভীরে।

---

<a name="পর্ব-৪"></a>

# পর্ব ৪ — `@NotNull` vs `@NotEmpty` vs `@NotBlank` (সবচেয়ে Confusing!)

এই তিনটা annotation দেখতে একই, কিন্তু behavior আলাদা। **৯০% beginner এখানে ভুল করে।**

---

## `@NotNull`

> *"Field null হতে পারবে না। কিন্তু empty string ("") চলবে।"*

```java
@NotNull
private String name;
```

| Input | Result |
|---|---|
| `null` | ❌ Fail |
| `""` (empty string) | ✅ Pass |
| `"  "` (only spaces) | ✅ Pass |
| `"Hasib"` | ✅ Pass |

---

## `@NotEmpty`

> *"Null হতে পারবে না, AND empty ও হতে পারবে না।"*

```java
@NotEmpty
private String name;
```

| Input | Result |
|---|---|
| `null` | ❌ Fail |
| `""` (empty string) | ❌ Fail |
| `"  "` (only spaces) | ✅ Pass (😱) |
| `"Hasib"` | ✅ Pass |

`@NotEmpty` **String, Collection, Array, Map** — সবগুলোতে কাজ করে।

```java
@NotEmpty
private List<String> hobbies;   // List খালি হতে পারবে না
```

---

## `@NotBlank`

> *"Null না, empty না, শুধু space ও না।"*

```java
@NotBlank
private String name;
```

| Input | Result |
|---|---|
| `null` | ❌ Fail |
| `""` (empty string) | ❌ Fail |
| `"  "` (only spaces) | ❌ Fail ✅ |
| `"Hasib"` | ✅ Pass |

`@NotBlank` **শুধু String এ কাজ করে।**

---

## একনজরে পার্থক্য

| Annotation | `null` | `""` | `"  "` | `"text"` | কোথায় কাজ করে |
|---|---|---|---|---|---|
| `@NotNull` | ❌ | ✅ | ✅ | ✅ | যেকোনো type |
| `@NotEmpty` | ❌ | ❌ | ✅ | ✅ | String, Collection, Array, Map |
| `@NotBlank` | ❌ | ❌ | ❌ | ✅ | শুধু String |

---

## কোনটা কখন Use করবো?

```
String এর জন্য → @NotBlank (সবচেয়ে strict)
List/Set এর জন্য → @NotEmpty
Number/Boolean → @NotNull
```

### Practical Example

```java
public class UserCreateDTO {

    @NotBlank(message = "Name is required")           // String
    private String name;

    @NotBlank(message = "Email is required")          // String
    private String email;

    @NotNull(message = "Age is required")             // Integer
    private Integer age;

    @NotEmpty(message = "At least one hobby required") // List
    private List<String> hobbies;

    @NotNull(message = "Active flag is required")     // Boolean
    private Boolean active;
}
```

---

<a name="পর্ব-৫"></a>

# পর্ব ৫ — Number Validations

## `@Min` ও `@Max`

```java
@Min(value = 18, message = "Age must be at least 18")
@Max(value = 100, message = "Age must be at most 100")
private Integer age;
```

| Input | Result |
|---|---|
| 17 | ❌ Fail |
| 18 | ✅ Pass |
| 50 | ✅ Pass |
| 100 | ✅ Pass |
| 101 | ❌ Fail |

## `@Positive`

> *"0 এর বড় হতে হবে।"*

```java
@Positive(message = "Price must be positive")
private Double price;
```

| Input | Result |
|---|---|
| -5 | ❌ Fail |
| 0 | ❌ Fail |
| 5 | ✅ Pass |

## `@PositiveOrZero`

> *"0 বা positive।"*

```java
@PositiveOrZero
private Integer count;
```

| Input | Result |
|---|---|
| -1 | ❌ Fail |
| 0 | ✅ Pass |
| 5 | ✅ Pass |

## `@Negative` ও `@NegativeOrZero`

`@Positive` এর উল্টো।

## `@DecimalMin` ও `@DecimalMax`

Decimal numbers এর জন্য —

```java
@DecimalMin(value = "0.01", message = "Price must be at least 0.01")
@DecimalMax(value = "999.99")
private BigDecimal price;
```

## `@Digits`

> *"কতগুলো integer digit এবং কতগুলো fraction digit allowed।"*

```java
@Digits(integer = 5, fraction = 2)
private BigDecimal salary;
```

| Input | Result | কেন |
|---|---|---|
| `12345.67` | ✅ Pass | 5 integer + 2 fraction |
| `123456.78` | ❌ Fail | 6 integer (5 allowed) |
| `123.456` | ❌ Fail | 3 fraction (2 allowed) |

---

<a name="পর্ব-৬"></a>

# পর্ব ৬ — String Validations

## `@Size`

> *"String এর length কত হতে পারে।"*

```java
@Size(min = 3, max = 50, message = "Name must be 3-50 characters")
private String name;
```

| Input | Length | Result |
|---|---|---|
| `"Hi"` | 2 | ❌ Fail |
| `"Hasib"` | 5 | ✅ Pass |
| `"a".repeat(51)` | 51 | ❌ Fail |

`@Size` **Collection, Array, Map এও** কাজ করে —

```java
@Size(min = 1, max = 5)
private List<String> tags;   // 1-5 items
```

## `@Email`

```java
@Email(message = "Invalid email format")
private String email;
```

| Input | Result |
|---|---|
| `"abc"` | ❌ Fail |
| `"abc@"` | ❌ Fail |
| `"abc@gmail"` | ❌ Fail |
| `"abc@gmail.com"` | ✅ Pass |

⚠️ **সাবধান:** `@Email` null অথবা empty string accept করে। তাই `@NotBlank` এর সাথে দাও —

```java
@NotBlank
@Email
private String email;
```

## `@Pattern`

> *"Regex দিয়ে custom validation।"*

```java
@Pattern(
    regexp = "^[A-Za-z0-9]{6,20}$",
    message = "Username: 6-20 alphanumeric characters"
)
private String username;
```

### Useful Regex Examples

```java
// বাংলাদেশ phone number
@Pattern(regexp = "^\\+8801[0-9]{9}$")
private String phone;

// Strong password
@Pattern(regexp = "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d)(?=.*[@$!%*?&]).{8,}$")
private String password;

// Only letters
@Pattern(regexp = "^[a-zA-Z]+$")
private String firstName;
```

---

<a name="পর্ব-৭"></a>

# পর্ব ৭ — Date Validations

## `@Past`

> *"Date অতীতে হতে হবে।"*

```java
@Past(message = "Birth date must be in the past")
private LocalDate dateOfBirth;
```

আজকের তারিখ এর আগের যেকোনো date pass হবে।

## `@PastOrPresent`

> *"আজ বা অতীতের।"*

```java
@PastOrPresent
private LocalDate joinDate;
```

## `@Future`

> *"ভবিষ্যতে হতে হবে।"*

```java
@Future(message = "Event date must be in the future")
private LocalDate eventDate;
```

## `@FutureOrPresent`

> *"আজ বা ভবিষ্যতের।"*

---

<a name="পর্ব-৮"></a>

# পর্ব ৮ — Validation Fail হলে কী হয়?

## Trigger Flow

```java
@PostMapping("/users")
public User createUser(@Valid @RequestBody UserCreateDTO dto) {
    return userService.createUser(dto);
}
```

### Invalid Request

```json
POST /users
{
  "name": "",
  "email": "abc",
  "age": 15
}
```

### পর্দার পেছনে

```
1. Spring DTO তে data fill করলো
2. @Valid দেখলো → Hibernate Validator চালু
3. সব validation check হলো:
   - name: @NotBlank fail
   - email: @Email fail
   - age: @Min(18) fail
4. সব error collect করলো
5. MethodArgumentNotValidException throw করলো
6. Controller এর method চলবেই না!
7. Exception handler খুঁজলো
```

## Default Behavior (Handler না থাকলে)

```json
{
  "timestamp": "2026-04-23T10:30:45.123+00:00",
  "status": 400,
  "error": "Bad Request",
  "trace": "...অনেক বড় stacktrace...",
  "path": "/users"
}
```

ভয়ঙ্কর এবং unhelpful। তাই handler লিখতে হবে।

---

<a name="পর্ব-৯"></a>

# পর্ব ৯ — Exception Handler এ ধরা

আগে Exception Handling এ যা শিখেছিলে — এখন সেটা use করি।

## Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            MethodArgumentNotValidException e) {

        Map<String, String> errors = new HashMap<>();

        e.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage())
        );

        return ResponseEntity.badRequest().body(Map.of(
            "status", 400,
            "message", "Validation failed",
            "errors", errors
        ));
    }
}
```

## প্রতিটা Method Decode

### `e.getBindingResult()`

`BindingResult` object — সব validation errors এর container।

### `getFieldErrors()`

`FieldError` list — প্রতিটা field এর error।

### `error.getField()` এবং `error.getDefaultMessage()`

- `getField()` = কোন field এ error ("name", "email")
- `getDefaultMessage()` = কী message ("Name is required")

## Clean Response

```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": {
    "name": "Name is required",
    "email": "Invalid email format",
    "age": "Age must be at least 18"
  }
}
```

Client এখন exactly জানে কোন field এ কী সমস্যা। ✅

---

<a name="পর্ব-১০"></a>

# পর্ব ১০ — Nested Object Validation

ধরো DTO এর ভেতরে আরেকটা DTO আছে —

```java
public class UserCreateDTO {

    @NotBlank
    private String name;

    private AddressDTO address;     // Nested DTO
}

public class AddressDTO {

    @NotBlank(message = "City is required")
    private String city;

    @NotBlank(message = "Country is required")
    private String country;
}
```

## সমস্যা — Nested Object Validate হয় না!

```java
@PostMapping
public User createUser(@Valid @RequestBody UserCreateDTO dto) { }
```

`@Valid` শুধু **outer DTO** এর rules check করে। AddressDTO এর `@NotBlank` চেক করে না!

## সমাধান — Nested Field এও `@Valid`

```java
public class UserCreateDTO {

    @NotBlank
    private String name;

    @Valid                          // ⭐ এই line যোগ করো
    private AddressDTO address;
}
```

এখন AddressDTO এর rules ও check হবে।

## Practical Example

```json
POST /users
{
  "name": "Hasib",
  "address": {
    "city": "",
    "country": ""
  }
}
```

### Response

```json
{
  "status": 400,
  "message": "Validation failed",
  "errors": {
    "address.city": "City is required",
    "address.country": "Country is required"
  }
}
```

`address.city` — nested path দেখাচ্ছে। ✅

---

<a name="পর্ব-১১"></a>

# পর্ব ১১ — List Validation

## List এর প্রতিটা item validate করা

```java
public class UserCreateDTO {

    @NotBlank
    private String name;

    @Valid                                    // ⭐ List এর প্রতিটা item validate
    @NotEmpty(message = "At least one address required")
    private List<AddressDTO> addresses;
}
```

`@Valid` দিলে List এর প্রতিটা AddressDTO এর rules check হবে।

## Practical Example

```json
POST /users
{
  "name": "Hasib",
  "addresses": [
    { "city": "Dhaka", "country": "BD" },
    { "city": "", "country": "" }
  ]
}
```

### Response

```json
{
  "errors": {
    "addresses[1].city": "City is required",
    "addresses[1].country": "Country is required"
  }
}
```

Index দেখাচ্ছে — `addresses[1]` মানে দ্বিতীয় address। ✅

---

<a name="পর্ব-১২"></a>

# পর্ব ১২ — Custom Validator (Deep Dive)

## কেন Custom Validator?

Built-in annotation দিয়ে সব কিছু check করা যায় না। যেমন —

- "Username already exists কিনা" check করা
- "Phone number বাংলাদেশের format এ" check করা
- "Two passwords match করে" check করা

এসবের জন্য নিজের annotation বানাতে হয়।

---

## একটা Custom Validator বানাই — Step by Step

আমাদের লক্ষ্য — `@StrongPassword` বানানো যেটা check করবে —
- কমপক্ষে ৮ character
- ১টা uppercase
- ১টা digit
- ১টা special character

### Step 1 — Annotation তৈরি করো

```java
@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {

    String message() default "Password is not strong enough";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

### এই Code এর প্রতিটা keyword

#### `@Target({ ElementType.FIELD })`

> *"এই annotation কোথায় বসানো যাবে।"*

`ElementType.FIELD` মানে — শুধু field এ। Method এ বা parameter এ না।

#### `@Retention(RetentionPolicy.RUNTIME)`

> *"এই annotation কতক্ষণ থাকবে।"*

`RUNTIME` মানে — runtime এও থাকবে, যাতে Spring read করতে পারে।

#### `@Constraint(validatedBy = StrongPasswordValidator.class)`

> *"এই annotation এর actual logic কোন class এ আছে।"*

`StrongPasswordValidator` class টা পরবর্তী step এ বানাবো।

#### `String message() default "..."`

Default error message। User চাইলে override করতে পারে।

#### `groups()` ও `payload()`

Advanced usage এর জন্য — Beginner হিসেবে এখন না বুঝলেও চলবে। শুধু copy করে রাখো।

---

### Step 2 — Validator Class বানাও

```java
public class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {

    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {

        if (password == null) return false;

        // কমপক্ষে ৮ character
        if (password.length() < 8) return false;

        // ১টা uppercase
        if (!password.matches(".*[A-Z].*")) return false;

        // ১টা digit
        if (!password.matches(".*\\d.*")) return false;

        // ১টা special character
        if (!password.matches(".*[!@#$%^&*].*")) return false;

        return true;   // সব ঠিক
    }
}
```

### এই Code এর প্রতিটা Keyword

#### `implements ConstraintValidator<StrongPassword, String>`

`ConstraintValidator` একটা interface। দুইটা generic type —
- `StrongPassword` = কোন annotation এর Validator
- `String` = কোন type এর field validate করছি

#### `isValid(String password, ConstraintValidatorContext context)`

Spring automatic এই method call করে। আমরা logic এখানে লিখি।

- Return `true` → Validation pass
- Return `false` → Validation fail

#### `ConstraintValidatorContext`

Advanced features এর জন্য — error message customize করতে। এখন not used।

---

### Step 3 — DTO তে Use করো

```java
public class UserCreateDTO {

    @NotBlank
    private String username;

    @StrongPassword(message = "Password must be strong")    // ⭐ Custom annotation!
    private String password;
}
```

### Step 4 — Test করি

```json
POST /users
{
  "username": "hasib",
  "password": "abc"
}
```

### Response

```json
{
  "errors": {
    "password": "Password must be strong"
  }
}
```

✅ Validator কাজ করছে।

---

## আরেকটা Example — Phone Number Validator

```java
// Annotation
@Target({ ElementType.FIELD })
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = BangladeshPhoneValidator.class)
public @interface BangladeshPhone {
    String message() default "Invalid Bangladesh phone number";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator
public class BangladeshPhoneValidator
        implements ConstraintValidator<BangladeshPhone, String> {

    @Override
    public boolean isValid(String phone, ConstraintValidatorContext context) {
        if (phone == null) return false;
        return phone.matches("^\\+8801[0-9]{9}$");
        // +8801xxxxxxxxx format
    }
}

// Use
public class UserCreateDTO {
    @BangladeshPhone
    private String phone;
}
```

---

<a name="পর্ব-১৩"></a>

# পর্ব ১৩ — Group Validation

## সমস্যাটা কী?

ধরো একটা User DTO আছে —

```java
public class UserDTO {

    @NotNull
    private Long id;

    @NotBlank
    private String name;
}
```

- **Create এর সময়** — id null থাকবে (database auto-generate করবে)
- **Update এর সময়** — id লাগবে (কোন user update করবো)

কীভাবে handle করি? — Group Validation দিয়ে।

---

## Step by Step

### Step 1 — Group Interface বানাও

```java
public interface CreateGroup {}
public interface UpdateGroup {}
```

এগুলো শুধু marker — কোনো logic নেই।

### Step 2 — DTO তে Group Assign করো

```java
public class UserDTO {

    @NotNull(groups = UpdateGroup.class)        // ⭐ শুধু Update এ check
    private Long id;

    @NotBlank(groups = { CreateGroup.class, UpdateGroup.class })   // দুইটাতেই
    private String name;
}
```

### Step 3 — Controller এ Group Specify করো

```java
@PostMapping("/users")
public User create(@Validated(CreateGroup.class) @RequestBody UserDTO dto) {
    // CreateGroup এর rules check হবে
    // id null হলেও pass করবে ✅
}

@PutMapping("/users")
public User update(@Validated(UpdateGroup.class) @RequestBody UserDTO dto) {
    // UpdateGroup এর rules check হবে
    // id null হলে fail করবে ❌
}
```

⚠️ **লক্ষ করো:** Group এর সাথে `@Valid` না, **`@Validated`** ব্যবহার করি।

---

<a name="পর্ব-১৪"></a>

# পর্ব ১৪ — Common Mistakes

## Mistake 1: `@Email` এর সাথে `@NotBlank` না দেয়া

```java
@Email   // ❌ null অথবা "" pass করে যাবে!
private String email;
```

**সমাধান:**

```java
@NotBlank
@Email
private String email;
```

## Mistake 2: `@Valid` ভুলে যাওয়া

```java
@PostMapping
public User create(@RequestBody UserDTO dto) {   // ❌ @Valid নেই!
    // Validation চলবেই না
}
```

**সমাধান:**

```java
@PostMapping
public User create(@Valid @RequestBody UserDTO dto) { }
```

## Mistake 3: Nested Object এ `@Valid` না দেয়া

```java
public class UserDTO {

    @Valid                  // outer
    private String name;

    private AddressDTO addr;   // ❌ @Valid নেই — nested check হবে না
}
```

**সমাধান:**

```java
@Valid
private AddressDTO addr;
```

## Mistake 4: `@NotNull` vs `@NotBlank` confuse

```java
@NotNull
private String name;   // ❌ "" pass করে যাবে!
```

**সমাধান:** String এর জন্য সবসময় `@NotBlank`।

## Mistake 5: Validation Logic Service এ লেখা

```java
@Service
public class UserService {
    public User create(UserDTO dto) {
        if (dto.getName() == null || dto.getName().isBlank()) {   // ❌
            throw new RuntimeException("Name required");
        }
        // ...
    }
}
```

**সমাধান:** DTO তে `@NotBlank` দাও — Service clean থাকবে।

## Mistake 6: Null check করার সময় Wrapper vs Primitive

```java
@NotNull
private int age;     // ❌ primitive int কখনো null হতে পারে না!

// সঠিক:
@NotNull
private Integer age;   // ✅ wrapper class
```

---

## চূড়ান্ত সারসংক্ষেপ

### Decision Flow

```
String validate করতে হবে?
├── @NotBlank দাও
├── @Size, @Pattern, @Email নিজের ইচ্ছামতো

Number validate করতে হবে?
├── @NotNull (wrapper class হলে)
├── @Min, @Max, @Positive, @Negative

Date validate করতে হবে?
├── @NotNull
├── @Past, @Future

Collection/List?
├── @NotEmpty
├── @Valid (যদি object list হয়)

Custom logic দরকার?
└── Custom Validator বানাও
```

### Master Setup

```java
public class UserCreateDTO {

    @NotBlank(message = "Name is required")
    @Size(min = 3, max = 50)
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Must be 18+")
    @Max(value = 100, message = "Must be under 100")
    private Integer age;

    @StrongPassword                        // Custom validator
    private String password;

    @NotEmpty(message = "At least one address required")
    @Valid                                 // Nested validation
    private List<AddressDTO> addresses;

    @NotNull
    @Past(message = "Birth date must be in past")
    private LocalDate dateOfBirth;
}
```

### Golden Rules

```
1. String এ @NotBlank, Number এ @NotNull, List এ @NotEmpty

2. Nested DTO তে @Valid দিতে ভুলো না

3. Custom Validator বানাও complex check এর জন্য

4. Validation logic DTO তে — Service এ না

5. Always pair: @Email + @NotBlank, @NotNull + @Min etc.

6. Group Validation use করো একই DTO দিয়ে create/update করতে

7. MethodArgumentNotValidException always handle করো
```

---

## পরবর্তী পদক্ষেপ

এই tutorial শেষ। পরে যা শিখবো —

- **Spring Security + JWT** (Authentication & Authorization)
- **Testing** (Unit + Integration)
- **Caching** (`@Cacheable`)
- **Async Operations** (`@Async`)

Validation এ পাকা হলে production-grade input handling করতে পারবে। 🚀

Happy coding!
