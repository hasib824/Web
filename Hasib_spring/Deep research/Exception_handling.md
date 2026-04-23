# Spring Boot — Exception Handling Complete Guide

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Level:** Beginner থেকে Intermediate
> **Topics:** Exception basics, `@ExceptionHandler`, `@ControllerAdvice`, Custom Exceptions (deep dive), Error Response DTO, Validation

---

## সূচিপত্র

1. [Prerequisites — Foundation](#পর্ব-০)
2. [Default Spring Behavior](#পর্ব-১)
3. [Try-Catch দিয়ে Handle (পুরনো উপায়)](#পর্ব-২)
4. [`@ExceptionHandler` — Controller Level](#পর্ব-৩)
5. [`@ControllerAdvice` — Global Level](#পর্ব-৪)
6. [`ResponseEntity` — Structured Response](#পর্ব-৫)
7. [Custom Exception — Deep Dive](#পর্ব-৬)
8. [Exception Hierarchy — Parent-Child](#পর্ব-৭)
9. [Error Response DTO — Professional Structure](#পর্ব-৮)
10. [Validation Exception Handle](#পর্ব-৯)
11. [`ResponseStatusException` — Built-in Shortcut](#পর্ব-১০)
12. [Complete Production Setup](#পর্ব-১১)
13. [Common Mistakes](#পর্ব-১২)

---

<a name="পর্ব-০"></a>

# পর্ব ০ — Prerequisites

## ১. Exception কী?

"Exception" মানে **ব্যতিক্রম** বা **সমস্যা।** Code চলার সময় কোনো unexpected ঘটনা ঘটলে Java একটা Exception throw করে।

### উদাহরণ

```java
int[] numbers = {1, 2, 3};
int value = numbers[10];    // ❌ 10 নম্বর index নেই!
```

Java এখানে `ArrayIndexOutOfBoundsException` throw করবে।

```java
String name = null;
name.length();    // ❌ null এর length নেই!
```

এখানে `NullPointerException`।

### Exception হলে কী হয়?

Program সেই মুহূর্তেই থেমে যায় — পরের লাইন চলে না।

```java
System.out.println("Line 1");    // ✅ চলবে
String name = null;
name.length();                    // ❌ Exception!
System.out.println("Line 3");    // এই line কখনো চলবে না
```

---

## ২. Checked vs Unchecked Exception

### Checked Exception

**Compile time এ** Java বলে — "এটা handle করতেই হবে।"

```java
public void readFile() throws IOException {
    FileReader reader = new FileReader("file.txt");
}
```

উদাহরণ: `IOException`, `SQLException`

### Unchecked Exception (Runtime Exception)

**Runtime এ** ঘটে। Java force করে না handle করতে।

```java
public void doSomething() {
    String name = null;
    name.length();    // Runtime এ Exception
}
```

উদাহরণ: `NullPointerException`, `IllegalArgumentException`, `RuntimeException`

### মনে রাখো

```
Checked    → File, Database, Network (external জিনিস)
Unchecked  → Null, Array, Wrong type (programming ভুল)
```

---

## ৩. Exception Handling কেন দরকার?

### একটা Scenario

```java
@GetMapping("/books/{id}")
public Book getBook(@PathVariable Long id) {
    return bookRepository.findById(id).get();
}
```

User call করলো `/books/999` — কিন্তু Book নেই!

### Handle না করলে Client কী পাবে

```json
{
  "status": 500,
  "error": "Internal Server Error",
  "trace": "java.util.NoSuchElementException: No value present\n
            at java.base/java.util.Optional.get(Optional.java:143)\n
            at com.example.BookController.getBook(BookController.java:25)\n
            ... আরো ২০০ লাইন stacktrace!",
  "path": "/books/999"
}
```

**সমস্যা —**
- Status 500 (actually 404 হওয়া উচিত — Client এর ভুল)
- ভয়ঙ্কর stacktrace
- Security risk (server internal exposed)
- Unprofessional

### Handle করলে যা আসা উচিত

```json
{
  "status": 404,
  "message": "Book with id 999 not found",
  "timestamp": "2026-04-23T10:30:45Z",
  "path": "/books/999"
}
```

Clean, clear, useful। ✅

---

## ৪. HTTP Status Code — Basic

| Code | নাম | কখন |
|---|---|---|
| **200** | OK | Request সফল |
| **201** | Created | নতুন resource তৈরি হলো |
| **400** | Bad Request | Client ভুল data পাঠিয়েছে |
| **401** | Unauthorized | Login ছাড়া access |
| **403** | Forbidden | Permission নেই |
| **404** | Not Found | Resource নেই |
| **409** | Conflict | Duplicate data |
| **500** | Internal Server Error | Server এর সমস্যা |

**Rule of thumb:**
- **4xx** = Client এর ভুল
- **5xx** = Server এর ভুল

---

## ৫. JSON Response কীভাবে client বুঝে?

Client (frontend, mobile app) server এর কাছ থেকে JSON response পায়। সেই JSON এ —

```json
{
  "status": 404,
  "message": "Book not found"
}
```

Client code চেক করে —
- `status` দেখে বুঝে — সফল না ব্যর্থ
- `message` দেখে user কে দেখায়

**তাই Structured JSON response দেয়া critical।**

---

<a name="পর্ব-১"></a>

# পর্ব ১ — Default Spring Behavior

Exception handle না করলে Spring নিজে থেকে কী করে?

## Example

```java
@RestController
public class BookController {

    @Autowired
    private BookRepository bookRepository;

    @GetMapping("/books/{id}")
    public Book getBook(@PathVariable Long id) {
        return bookRepository.findById(id).get();   // ⚠️ id না পেলে exception
    }
}
```

`/books/999` call করলে Exception হয় — কিন্তু তুমি handle করনি।

## Spring এর Default Response

```json
{
  "timestamp": "2026-04-23T10:30:45.123+00:00",
  "status": 500,
  "error": "Internal Server Error",
  "path": "/books/999"
}
```

Console এ পুরো stacktrace print হয় —

```
java.util.NoSuchElementException: No value present
    at java.util.Optional.get(Optional.java:143)
    at BookController.getBook(BookController.java:15)
    ... 50 more lines
```

## সমস্যা

- Status 500 — actually 404 হওয়া উচিত
- Client এ কোনো useful message নেই
- Stacktrace console এ log হয়ে গেল — production এ বিপজ্জনক

**তাই আমাদের নিজে handle করতে হবে।**

---

<a name="পর্ব-২"></a>

# পর্ব ২ — Try-Catch দিয়ে Handle (পুরনো উপায়)

## সবচেয়ে Primitive Way

```java
@GetMapping("/books/{id}")
public ResponseEntity<?> getBook(@PathVariable Long id) {
    try {
        Book book = bookRepository.findById(id).get();
        return ResponseEntity.ok(book);
    } catch (NoSuchElementException e) {
        Map<String, Object> error = new HashMap<>();
        error.put("status", 404);
        error.put("message", "Book not found");
        return ResponseEntity.status(404).body(error);
    }
}
```

## কাজ করবে, কিন্তু অনেক সমস্যা

### সমস্যা ১: Repetitive Code

প্রতিটা Controller method এ একই try-catch লিখতে হবে —

```java
getBook()      → try-catch
getAuthor()    → try-catch
createBook()   → try-catch
updateBook()   → try-catch
... ২০টা method = ২০টা try-catch!
```

### সমস্যা ২: Business Logic আর Error Handling mixed

Method এর main logic আর error handling একসাথে মিশে গেছে। পড়তে কঠিন।

### সমস্যা ৩: Maintainable না

Error response format change করতে চাইলে ২০ জায়গায় গিয়ে বদলাতে হবে।

**তাই এই approach ভুলে যাও। চলো better solution দেখি।**

---

<a name="পর্ব-৩"></a>

# পর্ব ৩ — `@ExceptionHandler` (Controller Level)

## এই Annotation কী করে?

`@ExceptionHandler` বলে —

> *"এই method টা নির্দিষ্ট Exception handle করার জন্য। Controller এ ওই Exception হলে সরাসরি এই method call হবে।"*

## উদাহরণ

```java
@RestController
public class BookController {

    @GetMapping("/books/{id}")
    public Book getBook(@PathVariable Long id) {
        return bookRepository.findById(id)
                .orElseThrow(() -> new NoSuchElementException("Book not found"));
    }

    // ⭐ এই method NoSuchElementException handle করবে
    @ExceptionHandler(NoSuchElementException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(NoSuchElementException e) {
        Map<String, Object> error = new HashMap<>();
        error.put("status", 404);
        error.put("message", e.getMessage());
        return ResponseEntity.status(404).body(error);
    }
}
```

## প্রতিটা Keyword বুঝি

### `@ExceptionHandler(NoSuchElementException.class)`

> *"আমি `NoSuchElementException` handle করবো।"*

এই annotation parameter এ বলো কোন Exception handle করবে।

### Method Parameter এ `NoSuchElementException e`

Spring automatically Exception object টা pass করে। তুমি `e.getMessage()` দিয়ে message পেতে পারো।

### Return Type `ResponseEntity<Map<String, Object>>`

- `ResponseEntity` = full HTTP response (status + body + headers)
- `Map<String, Object>` = body এর JSON structure

## কী সুবিধা হলো?

Controller এর main method এ আর try-catch লাগছে না —

```java
@GetMapping("/books/{id}")
public Book getBook(@PathVariable Long id) {
    return bookRepository.findById(id)
            .orElseThrow(() -> new NoSuchElementException("Book not found"));
}
```

Clean, readable ✅

## কিন্তু সমস্যা এখনো আছে

`@ExceptionHandler` **শুধু এই Controller এ কাজ করে।**

```
BookController → এখানে NoSuchElementException handle হবে ✅
AuthorController → একই Exception হলে handle হবে না ❌
```

প্রতিটা Controller এ আলাদা আলাদা `@ExceptionHandler` লিখতে হবে। আবার repetition!

**সমাধান → `@ControllerAdvice`।**

---

<a name="পর্ব-৪"></a>

# পর্ব ৪ — `@ControllerAdvice` (Global Level)

## এটা কী?

`@ControllerAdvice` একটা **global exception handler class** বানায়। সব Controller এর জন্য একসাথে।

> *"এই class এর ভেতরের `@ExceptionHandler` গুলো সব Controller এ apply হবে।"*

## "Advice" শব্দের মানে

"Advice" মানে **উপদেশ।** Spring AOP এর term। এটা মানে — Controller গুলো কে advice দেয়া "এই exception এ এভাবে handle কর।"

## উদাহরণ

```java
@ControllerAdvice                     // ⭐ Global handler declare করলাম
public class GlobalExceptionHandler {

    @ExceptionHandler(NoSuchElementException.class)
    public ResponseEntity<Map<String, Object>> handleNotFound(NoSuchElementException e) {
        Map<String, Object> error = new HashMap<>();
        error.put("status", 404);
        error.put("message", e.getMessage());
        return ResponseEntity.status(404).body(error);
    }

    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<Map<String, Object>> handleBadRequest(IllegalArgumentException e) {
        Map<String, Object> error = new HashMap<>();
        error.put("status", 400);
        error.put("message", e.getMessage());
        return ResponseEntity.status(400).body(error);
    }
}
```

## এখন কী হবে?

```
BookController → NoSuchElementException → ✅ Global handler handle করবে
AuthorController → NoSuchElementException → ✅ Global handler handle করবে
UserController → IllegalArgumentException → ✅ Global handler handle করবে
```

এক জায়গায় লিখলাম, সব জায়গায় কাজ করছে। ✅

## `@RestControllerAdvice`

REST API এর জন্য Spring একটা shortcut দিয়েছে —

```java
@RestControllerAdvice           // ⭐ @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler { }
```

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`

`@ResponseBody` মানে — return value JSON এ convert করো। REST API তে সবসময় এটা লাগে।

**Rule of thumb:** REST API হলে সবসময় `@RestControllerAdvice` use করো।

---

<a name="পর্ব-৫"></a>

# পর্ব ৫ — `ResponseEntity` — Structured Response

আমরা উপরে `ResponseEntity` use করেছি। এবার গভীরে বুঝি।

## `ResponseEntity` কী?

এটা একটা class যেটা পুরো HTTP Response represent করে —

```
HTTP Response:
├── Status Code  (200, 404, 500, etc.)
├── Headers      (Content-Type, etc.)
└── Body         (JSON data)
```

## কেন দরকার?

সাধারণভাবে —

```java
@GetMapping("/books")
public List<Book> getBooks() {
    return bookRepository.findAll();
    // → Status auto 200, body auto JSON
}
```

এখানে তুমি status control করতে পারো না। সবসময় 200।

কিন্তু Exception handling এ 404, 400 return করতে হয় — তাই `ResponseEntity` লাগে।

## Build করার উপায়

### উপায় ১ — Static Methods

```java
ResponseEntity.ok(body);                    // status 200
ResponseEntity.status(404).body(error);    // custom status
ResponseEntity.notFound().build();         // 404, no body
ResponseEntity.badRequest().body(error);   // 400
```

### উপায় ২ — Constructor

```java
new ResponseEntity<>(body, HttpStatus.NOT_FOUND);
```

### HttpStatus Enum

Hardcoded number না লিখে enum use করো —

```java
ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
// 404 এর বদলে HttpStatus.NOT_FOUND — readable ✅
```

| Enum | Code |
|---|---|
| `HttpStatus.OK` | 200 |
| `HttpStatus.CREATED` | 201 |
| `HttpStatus.BAD_REQUEST` | 400 |
| `HttpStatus.UNAUTHORIZED` | 401 |
| `HttpStatus.FORBIDDEN` | 403 |
| `HttpStatus.NOT_FOUND` | 404 |
| `HttpStatus.CONFLICT` | 409 |
| `HttpStatus.INTERNAL_SERVER_ERROR` | 500 |

---

<a name="পর্ব-৬"></a>

# পর্ব ৬ — Custom Exception (Deep Dive)

## কেন Custom Exception দরকার?

এতক্ষণ আমরা Java এর built-in Exception (`NoSuchElementException`, `IllegalArgumentException`) ব্যবহার করেছি। কিন্তু সমস্যা আছে —

### সমস্যা ১: Ambiguous (অস্পষ্ট)

```java
throw new IllegalArgumentException("Book not found");
```

`IllegalArgumentException` Java এর built-in। এটা দেখে কেউ বুঝবে না —
- এটা Book related?
- User related?
- Payment related?

### সমস্যা ২: Handle করা কঠিন

```java
@ExceptionHandler(IllegalArgumentException.class)
public ResponseEntity<?> handle(IllegalArgumentException e) {
    // Book error? User error? বুঝবো কীভাবে?
}
```

### সমস্যা ৩: Meaningful Information নেই

ভুল data কী? কোন id? কোন field? — সব শুধু message এ থাকে, structured না।

## সমাধান — নিজের Exception class

```java
throw new BookNotFoundException(id);
```

এখন নামেই বোঝা যায় — Book না পাওয়ার সমস্যা। Handle করাও সহজ।

---

## Step 1 — প্রথম Custom Exception

```java
public class BookNotFoundException extends RuntimeException {

    public BookNotFoundException(String message) {
        super(message);
    }
}
```

## এই Code এর প্রতিটা Keyword

### `extends RuntimeException`

আমাদের custom Exception কে Java এর `RuntimeException` থেকে inherit করতে হবে।

**কেন RuntimeException?**

- Unchecked Exception — `throws` declare করতে হয় না
- Method signature cleaner
- Spring এর default behavior এর সাথে match করে (RuntimeException এ auto rollback)

```java
// RuntimeException extend করলে
public void deleteBook(Long id) {
    throw new BookNotFoundException("Not found");   // ✅ throws লাগে না
}

// Exception extend করলে (Checked)
public void deleteBook(Long id) throws BookNotFoundException {  // ⚠️ throws লিখতে হয়
    throw new BookNotFoundException("Not found");
}
```

### `super(message)`

`super` মানে — **parent class এর constructor কে call করা।**

`RuntimeException` এর একটা constructor আছে যেটা message নেয়। আমরা সেটা কেই call করছি।

```java
public BookNotFoundException(String message) {
    super(message);    // RuntimeException(String) call
}
```

এতে `e.getMessage()` দিয়ে message পাওয়া যাবে।

---

## Step 2 — Exception কে আরো Meaningful করা

শুধু message না, extra field যোগ করি —

```java
public class BookNotFoundException extends RuntimeException {

    private final Long bookId;

    public BookNotFoundException(Long bookId) {
        super("Book with id " + bookId + " not found");
        this.bookId = bookId;
    }

    public Long getBookId() {
        return bookId;
    }
}
```

### এখন সুবিধা

```java
throw new BookNotFoundException(999L);
```

Handler এ —

```java
@ExceptionHandler(BookNotFoundException.class)
public ResponseEntity<?> handle(BookNotFoundException e) {
    Long missingId = e.getBookId();    // Extra info ✅
    // Log করতে পারো, error response এ add করতে পারো
}
```

---

## Step 3 — Multiple Constructor

অনেক সময় বিভিন্ন scenario এ exception throw করতে হয় —

```java
public class BookNotFoundException extends RuntimeException {

    // id দিয়ে
    public BookNotFoundException(Long id) {
        super("Book with id " + id + " not found");
    }

    // title দিয়ে
    public BookNotFoundException(String title) {
        super("Book with title '" + title + "' not found");
    }

    // custom message + cause
    public BookNotFoundException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### `Throwable cause` কী?

Exception এর "root cause" — মূল কারণ। যেমন —

```java
try {
    // database operation
} catch (SQLException e) {
    throw new BookNotFoundException("Database error", e);
    // ⭐ original SQLException কে cause হিসেবে পাঠাচ্ছি
}
```

এতে debug করা সহজ — Exception chain দেখা যায়।

---

## Step 4 — আরো Custom Exception

প্রতিটা type এর error এর জন্য আলাদা Exception —

```java
// Book না পেলে
public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(Long id) {
        super("Book with id " + id + " not found");
    }
}

// Author না পেলে
public class AuthorNotFoundException extends RuntimeException {
    public AuthorNotFoundException(Long id) {
        super("Author with id " + id + " not found");
    }
}

// Duplicate book
public class DuplicateBookException extends RuntimeException {
    public DuplicateBookException(String title) {
        super("Book with title '" + title + "' already exists");
    }
}

// Invalid data
public class InvalidBookDataException extends RuntimeException {
    public InvalidBookDataException(String field, String reason) {
        super("Invalid " + field + ": " + reason);
    }
}
```

## Use in Service

```java
@Service
public class BookService {

    public Book getBook(Long id) {
        return bookRepository.findById(id)
            .orElseThrow(() -> new BookNotFoundException(id));
    }

    public Book createBook(BookDTO dto) {
        if (bookRepository.existsByTitle(dto.getTitle())) {
            throw new DuplicateBookException(dto.getTitle());
        }

        if (dto.getPrice() < 0) {
            throw new InvalidBookDataException("price", "must be positive");
        }

        return bookRepository.save(book);
    }
}
```

## Handle করা

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(BookNotFoundException.class)
    public ResponseEntity<?> handleBookNotFound(BookNotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(Map.of("message", e.getMessage()));
    }

    @ExceptionHandler(DuplicateBookException.class)
    public ResponseEntity<?> handleDuplicate(DuplicateBookException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
                .body(Map.of("message", e.getMessage()));
    }

    @ExceptionHandler(InvalidBookDataException.class)
    public ResponseEntity<?> handleInvalid(InvalidBookDataException e) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(Map.of("message", e.getMessage()));
    }
}
```

প্রতিটা Exception এর জন্য আলাদা handler, আলাদা status code। ✅

---

## Step 5 — `@ResponseStatus` Shortcut

প্রতিটা Exception এ বার বার status code set করার ঝামেলা থেকে বাঁচতে —

```java
@ResponseStatus(HttpStatus.NOT_FOUND)                   // ⭐
public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(Long id) {
        super("Book with id " + id + " not found");
    }
}
```

এখন handle না করলেও Spring automatically 404 return করবে —

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Book with id 999 not found"
}
```

তবে **customized response** চাইলে `@ExceptionHandler` দিয়ে handle করাই ভালো।

---

<a name="পর্ব-৭"></a>

# পর্ব ৭ — Exception Hierarchy (Parent-Child)

## কেন Hierarchy দরকার?

আমরা অনেক Custom Exception বানিয়েছি — `BookNotFoundException`, `AuthorNotFoundException`, `UserNotFoundException`...

প্রতিটা 404 return করে। প্রতিটার জন্য আলাদা handler লিখা overkill।

## সমাধান — Base Exception বানাই

```java
// Base class — সব NotFound exception এর parent
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

// Child classes
public class BookNotFoundException extends ResourceNotFoundException {
    public BookNotFoundException(Long id) {
        super("Book with id " + id + " not found");
    }
}

public class AuthorNotFoundException extends ResourceNotFoundException {
    public AuthorNotFoundException(Long id) {
        super("Author with id " + id + " not found");
    }
}

public class UserNotFoundException extends ResourceNotFoundException {
    public UserNotFoundException(Long id) {
        super("User with id " + id + " not found");
    }
}
```

## এখন একটাই Handler

```java
@ExceptionHandler(ResourceNotFoundException.class)   // ⭐ Parent
public ResponseEntity<?> handleNotFound(ResourceNotFoundException e) {
    return ResponseEntity.status(404).body(Map.of("message", e.getMessage()));
}
```

এই handler **সব child exception** handle করে —
- `BookNotFoundException` → handle হবে ✅
- `AuthorNotFoundException` → handle হবে ✅
- `UserNotFoundException` → handle হবে ✅

## পুরো Hierarchy

```java
public class BusinessException extends RuntimeException { }    // Top level

public class ResourceNotFoundException extends BusinessException { }
public class DuplicateResourceException extends BusinessException { }
public class InvalidDataException extends BusinessException { }
public class UnauthorizedException extends BusinessException { }

public class BookNotFoundException extends ResourceNotFoundException { }
public class AuthorNotFoundException extends ResourceNotFoundException { }
```

## Visualization

```
RuntimeException (Java built-in)
    │
    └── BusinessException (আমার top-level)
            │
            ├── ResourceNotFoundException → 404
            │       ├── BookNotFoundException
            │       ├── AuthorNotFoundException
            │       └── UserNotFoundException
            │
            ├── DuplicateResourceException → 409
            │       └── DuplicateBookException
            │
            └── InvalidDataException → 400
                    └── InvalidBookDataException
```

## সুবিধা

```java
@ExceptionHandler(ResourceNotFoundException.class)    // 404 সব
public ResponseEntity<?> handleNotFound(ResourceNotFoundException e) { }

@ExceptionHandler(DuplicateResourceException.class)   // 409 সব
public ResponseEntity<?> handleDuplicate(DuplicateResourceException e) { }

@ExceptionHandler(InvalidDataException.class)         // 400 সব
public ResponseEntity<?> handleInvalid(InvalidDataException e) { }
```

৩টা handler এ সব exception cover হয়ে যায়। ✅

---

<a name="পর্ব-৮"></a>

# পর্ব ৮ — Error Response DTO (Professional Structure)

## এতক্ষণ Map ব্যবহার করলাম

```java
return ResponseEntity.status(404).body(Map.of("message", e.getMessage()));
```

কাজ করে, কিন্তু সমস্যা —
- Field এর consistency নেই
- IDE auto-complete নেই
- Type safety নেই
- বড় project এ mess হয়ে যায়

## সমাধান — Error Response DTO

একটা standard class —

```java
public class ErrorResponse {

    private int status;
    private String error;
    private String message;
    private String path;
    private LocalDateTime timestamp;

    public ErrorResponse(int status, String error, String message, String path) {
        this.status = status;
        this.error = error;
        this.message = message;
        this.path = path;
        this.timestamp = LocalDateTime.now();
    }

    // getters
}
```

## প্রতিটা Field এর মানে

| Field | কী |
|---|---|
| `status` | HTTP status code (404, 400, etc.) |
| `error` | Error type short form ("Not Found", "Bad Request") |
| `message` | Detailed message ("Book with id 999 not found") |
| `path` | কোন URL এ error হয়েছে (`/api/books/999`) |
| `timestamp` | কখন error হলো |

## Handler এ Use

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException e,
            HttpServletRequest request) {                 // ⭐ request inject

        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            e.getMessage(),
            request.getRequestURI()
        );

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
}
```

### `HttpServletRequest request` কেন?

Spring automatically inject করে। এর `getRequestURI()` দিয়ে current URL পাওয়া যায়।

## JSON Output

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Book with id 999 not found",
  "path": "/api/books/999",
  "timestamp": "2026-04-23T10:30:45"
}
```

Clean, consistent, professional। ✅

---

<a name="পর্ব-৯"></a>

# পর্ব ৯ — Validation Exception Handle

## Validation কী?

Client থেকে আসা data যাচাই করা — সঠিক কিনা, null কিনা, format ঠিক কিনা।

```java
public class BookCreateDTO {

    @NotBlank(message = "Title is required")
    private String title;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private Double price;

    @Email(message = "Invalid email format")
    private String authorEmail;
}
```

## Controller এ `@Valid`

```java
@PostMapping
public Book createBook(@Valid @RequestBody BookCreateDTO dto) {
    return bookService.createBook(dto);
}
```

`@Valid` বলে — *"এই DTO এর validation rules check করো।"*

## Validation Fail হলে কী হয়?

Spring automatically `MethodArgumentNotValidException` throw করে।

## এটা Handle করি

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<?> handleValidation(MethodArgumentNotValidException e) {

    Map<String, String> errors = new HashMap<>();

    e.getBindingResult().getFieldErrors().forEach(error -> {
        errors.put(error.getField(), error.getDefaultMessage());
    });

    return ResponseEntity.badRequest().body(Map.of(
        "status", 400,
        "error", "Validation Failed",
        "errors", errors
    ));
}
```

## প্রতিটা keyword বুঝি

### `e.getBindingResult()`

`BindingResult` object — সব validation errors এর container।

### `getFieldErrors()`

`FieldError` list — প্রতিটা field এর error।

### `error.getField()` এবং `error.getDefaultMessage()`

- `getField()` = কোন field এ error (`"title"`, `"price"`)
- `getDefaultMessage()` = কী message (`"Title is required"`)

## JSON Output

```json
{
  "status": 400,
  "error": "Validation Failed",
  "errors": {
    "title": "Title is required",
    "price": "Price must be positive",
    "authorEmail": "Invalid email format"
  }
}
```

Client এখন exactly জানে কোন field এ কী সমস্যা। ✅

---

<a name="পর্ব-১০"></a>

# পর্ব ১০ — `ResponseStatusException` (Built-in Shortcut)

## এটা কী?

Spring এর built-in Exception — Custom Exception class না বানিয়েই inline exception throw করা যায়।

## উদাহরণ

```java
@GetMapping("/books/{id}")
public Book getBook(@PathVariable Long id) {
    return bookRepository.findById(id)
            .orElseThrow(() -> new ResponseStatusException(
                HttpStatus.NOT_FOUND,
                "Book not found"
            ));
}
```

## কাজ করবে?

হ্যাঁ — Spring automatically handle করে। 404 + message return করে।

## কিন্তু এটা Best Practice না

### কেন?

- Business logic এ HTTP concept (status code) চলে আসে
- Custom Exception এর মতো reusable না
- Testing কঠিন
- Domain language reflect করে না

### Custom Exception এর সাথে তুলনা

```java
// ❌ ResponseStatusException
throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Book not found");

// ✅ Custom Exception
throw new BookNotFoundException(id);
```

দ্বিতীয়টা — clean, meaningful, reusable।

### কখন Use করবে?

- Small project / prototype
- Simple cases
- Custom Exception বানানোর সময় নেই

**বড় project এ সবসময় Custom Exception use করো।**

---

<a name="পর্ব-১১"></a>

# পর্ব ১১ — Complete Production Setup

এবার সব মিলিয়ে full setup দেখি।

## Step 1 — Base Exception

```java
public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }

    public BusinessException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

## Step 2 — Exception Hierarchy

```java
public class ResourceNotFoundException extends BusinessException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class DuplicateResourceException extends BusinessException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}

public class InvalidDataException extends BusinessException {
    public InvalidDataException(String message) {
        super(message);
    }
}

public class UnauthorizedException extends BusinessException {
    public UnauthorizedException(String message) {
        super(message);
    }
}
```

## Step 3 — Specific Exception

```java
public class BookNotFoundException extends ResourceNotFoundException {
    public BookNotFoundException(Long id) {
        super("Book with id " + id + " not found");
    }
}

public class DuplicateBookException extends DuplicateResourceException {
    public DuplicateBookException(String title) {
        super("Book with title '" + title + "' already exists");
    }
}
```

## Step 4 — Error Response DTO

```java
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private String path;
    private LocalDateTime timestamp;
    private Map<String, String> validationErrors;

    public ErrorResponse(int status, String error, String message, String path) {
        this.status = status;
        this.error = error;
        this.message = message;
        this.path = path;
        this.timestamp = LocalDateTime.now();
    }

    // getters, setters
}
```

## Step 5 — Global Exception Handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 404 Not Found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            "Not Found",
            e.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    // 409 Conflict
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(
            DuplicateResourceException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.CONFLICT.value(),
            "Conflict",
            e.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    // 400 Bad Request (business)
    @ExceptionHandler(InvalidDataException.class)
    public ResponseEntity<ErrorResponse> handleInvalid(
            InvalidDataException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Bad Request",
            e.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.badRequest().body(error);
    }

    // 403 Forbidden
    @ExceptionHandler(UnauthorizedException.class)
    public ResponseEntity<ErrorResponse> handleUnauthorized(
            UnauthorizedException e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.FORBIDDEN.value(),
            "Forbidden",
            e.getMessage(),
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }

    // 400 Validation (from @Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException e,
            HttpServletRequest request) {

        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage())
        );

        ErrorResponse error = new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation Failed",
            "Input validation failed",
            request.getRequestURI()
        );
        error.setValidationErrors(errors);

        return ResponseEntity.badRequest().body(error);
    }

    // 500 Catch-all — last resort
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(
            Exception e,
            HttpServletRequest request) {

        ErrorResponse error = new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "Internal Server Error",
            "Something went wrong",
            request.getRequestURI()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

## Step 6 — Service Layer Use

```java
@Service
public class BookService {

    public Book getBook(Long id) {
        return bookRepository.findById(id)
            .orElseThrow(() -> new BookNotFoundException(id));
    }

    public Book createBook(BookCreateDTO dto) {
        if (bookRepository.existsByTitle(dto.getTitle())) {
            throw new DuplicateBookException(dto.getTitle());
        }
        return bookRepository.save(toEntity(dto));
    }
}
```

## Step 7 — Controller Clean থাকবে

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping("/{id}")
    public Book getBook(@PathVariable Long id) {
        return bookService.getBook(id);    // try-catch লাগে না ✅
    }

    @PostMapping
    public Book createBook(@Valid @RequestBody BookCreateDTO dto) {
        return bookService.createBook(dto);    // clean ✅
    }
}
```

## Result — Professional Error Responses

### Book Not Found

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "Book with id 999 not found",
  "path": "/api/books/999",
  "timestamp": "2026-04-23T10:30:45"
}
```

### Duplicate Book

```json
{
  "status": 409,
  "error": "Conflict",
  "message": "Book with title 'Pother Pachali' already exists",
  "path": "/api/books",
  "timestamp": "2026-04-23T10:32:10"
}
```

### Validation Failed

```json
{
  "status": 400,
  "error": "Validation Failed",
  "message": "Input validation failed",
  "path": "/api/books",
  "timestamp": "2026-04-23T10:33:22",
  "validationErrors": {
    "title": "Title is required",
    "price": "Price must be positive"
  }
}
```

---

<a name="পর্ব-১২"></a>

# পর্ব ১২ — Common Mistakes

## Mistake 1: Controller এ try-catch রেখে দেয়া

```java
@GetMapping("/books/{id}")
public ResponseEntity<?> getBook(@PathVariable Long id) {
    try {                                              // ❌
        return ResponseEntity.ok(bookService.getBook(id));
    } catch (Exception e) {
        return ResponseEntity.status(500).build();
    }
}
```

**সমাধান:** `@RestControllerAdvice` দিয়ে global handle করো।

## Mistake 2: Exception Hierarchy ছাড়া prolifereration

```java
public class BookNotFoundException extends RuntimeException { }
public class AuthorNotFoundException extends RuntimeException { }
public class UserNotFoundException extends RuntimeException { }
// প্রতিটার জন্য আলাদা handler লাগে! 😫
```

**সমাধান:** Common parent (`ResourceNotFoundException`) বানাও।

## Mistake 3: Generic Exception throw

```java
throw new RuntimeException("Book not found");    // ❌
```

**সমাধান:** Specific custom exception throw করো —

```java
throw new BookNotFoundException(id);    // ✅
```

## Mistake 4: Stacktrace Client কে পাঠানো

```java
error.put("trace", ExceptionUtils.getStackTrace(e));    // ❌ Security risk!
```

**সমাধান:** Client কে শুধু message দাও। Stacktrace শুধু log এ।

## Mistake 5: সব Exception এ 500 Return

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<?> handle(Exception e) {
    return ResponseEntity.status(500).body(...);    // সব কিছু 500!
}
```

**সমাধান:** Specific exception এ specific status —

```
ResourceNotFoundException → 404
DuplicateResourceException → 409
InvalidDataException → 400
UnauthorizedException → 403
Exception (generic) → 500 (last resort)
```

## Mistake 6: Exception এ Business Logic

```java
public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(Long id) {
        super("Book not found");
        notifyAdmin();              // ❌ Business logic!
        logToDatabase();            // ❌
    }
}
```

**সমাধান:** Exception শুধু data ধরে রাখে। Logic handler এ থাকে।

## Mistake 7: Log না করা

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<?> handle(Exception e) {
    // কোনো log নেই!
    return ResponseEntity.status(500).body(...);
}
```

**সমাধান:** সবসময় log করো —

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<?> handle(Exception e) {
    log.error("Unexpected error", e);    // ✅ Full trace log এ
    return ResponseEntity.status(500).body("Something went wrong");
}
```

---

## চূড়ান্ত সারসংক্ষেপ

### Decision Flow

```
Controller এ Exception হলো
        ↓
@RestControllerAdvice খুঁজে handler
        ↓
Specific @ExceptionHandler match হলো?
├── YES → সেটা use করো
└── NO → Parent Exception handler খোঁজো
          ↓
          সেটাও না পেলে → Exception handler (catch-all)
          ↓
          সেটাও না থাকলে → Spring এর default (500)
```

### Golden Rules

```
1. প্রতিটা error এর জন্য নিজের Exception বানাও
   → Java built-in Exception throw করো না

2. Exception Hierarchy বানাও
   → Common parent → specific child

3. RuntimeException extend করো
   → Checked Exception বেশিরভাগ ক্ষেত্রে এড়াও

4. @RestControllerAdvice দিয়ে একটা Global Handler
   → প্রতি Controller এ আলাদা handler না

5. Structured Error Response DTO
   → Map না, proper class

6. Specific status code দাও
   → সব কিছু 500 না

7. Client এ stacktrace পাঠিও না
   → শুধু meaningful message

8. সবসময় log করো
   → Debug এর জন্য critical
```

### Project Structure

```
com.example.bookstore/
├── exception/
│   ├── BusinessException.java
│   ├── ResourceNotFoundException.java
│   ├── DuplicateResourceException.java
│   ├── InvalidDataException.java
│   ├── UnauthorizedException.java
│   ├── BookNotFoundException.java
│   ├── DuplicateBookException.java
│   ├── AuthorNotFoundException.java
│   ├── GlobalExceptionHandler.java
│   └── ErrorResponse.java
├── controller/
├── service/
└── repository/
```

---

## শেষ কথা

Exception Handling = তোমার API এর **professional face।**

- সাধারণ API → কাজ করে, কিন্তু error হলে ভয়ঙ্কর দেখায়
- Exception Handled API → error হলেও clean, professional response

এই tutorial এর সব concept মাথায় রেখে code লিখলে তোমার API enterprise-grade হবে। 🚀

পরবর্তী ধাপে — **Validation (Bean Validation + `@Valid`)** নিয়ে গভীরে যাবো। Exception Handling এর সাথে সেটা perfect jodi।

Happy coding!
