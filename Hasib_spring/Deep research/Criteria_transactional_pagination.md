# Spring Boot — Specifications, Transactional ও Pagination

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Previous Tutorials:** Basic Relationships, Advanced Concepts
> **Topics:** JPA Specifications, `@Transactional` in depth, Pagination, Sorting

---

## সূচিপত্র

1. [JPA Specifications / Criteria API](#পর্ব-১)
2. [Bridge — Specifications ও Transactional এর সম্পর্ক](#bridge)
3. [`@Transactional` গভীরে](#পর্ব-২)
4. [Pagination ও Sorting](#পর্ব-৩)

---

<a name="পর্ব-১"></a>

# পর্ব ১ — JPA Specifications / Criteria API

## ধাপ ১: সমস্যাটা আগে বুঝি

ধরো তুমি একটা Book search API বানাচ্ছো। User নানারকম filter দিয়ে search করবে —

- শুধু title দিয়ে
- শুধু author দিয়ে
- title + author একসাথে
- title + author + price range
- কোনো filter ই না (সব Books)
- আরো অনেক combination...

### পুরনো Approach — Method Explosion

প্রতিটা combination এর জন্য আলাদা method —

```java
public interface BookRepository extends JpaRepository<Book, Long> {

    List<Book> findByTitle(String title);
    List<Book> findByAuthor(String author);
    List<Book> findByTitleAndAuthor(String title, String author);
    List<Book> findByTitleAndAuthorAndPriceBetween(String title, String author, Double min, Double max);
    List<Book> findByTitleAndPriceBetween(String title, Double min, Double max);
    // ... ২০টা method!!! 😱
}
```

সমস্যা —
- প্রতিটা নতুন filter এ method বাড়ে
- Maintainable না
- User যদি কোনো filter না দেয়, তাহলে কোন method call করবো?

### আরেকটা পুরনো Approach — If-Else Hell

```java
public List<Book> search(String title, String author, Double minPrice) {
    if (title != null && author != null && minPrice != null) {
        return bookRepository.findByTitleAndAuthorAndPriceGreaterThan(title, author, minPrice);
    }
    if (title != null && author != null) {
        return bookRepository.findByTitleAndAuthor(title, author);
    }
    if (title != null) {
        return bookRepository.findByTitle(title);
    }
    // ... ১৬টা if-else!!! 😱
}
```

এটা **Dynamic Query** এর সমস্যা — **runtime এ filter change হয়**, কিন্তু আমরা compile time এ query লিখছি।

### সমাধান — Criteria API / Specifications

Runtime এ query বানানোর জন্য JPA দুইটা tool দেয় —

1. **Criteria API** (JPA এর built-in, low level)
2. **JPA Specifications** (Spring Data এর wrapper, easier)

আমরা মূলত Specifications শিখবো — কারণ সহজ এবং production এ এটাই use হয়।

---

## ধাপ ২: Criteria API কী (Base Technology)

Criteria API হলো **JPA এর built-in tool** যেটা দিয়ে Java code এ query বানানো যায় — SQL বা JPQL string না লিখে।

### String Query (পুরনো উপায়)

```java
@Query("SELECT b FROM Book b WHERE b.title = :title")
List<Book> findByTitle(String title);
```

এখানে query একটা **string**। Runtime এ পরিবর্তন করা কঠিন।

### Criteria API (নতুন উপায়)

```java
CriteriaBuilder cb = entityManager.getCriteriaBuilder();
CriteriaQuery<Book> query = cb.createQuery(Book.class);
Root<Book> book = query.from(Book.class);

query.select(book).where(cb.equal(book.get("title"), "Pother Pachali"));

List<Book> result = entityManager.createQuery(query).getResultList();
```

এখানে query একটা **Java Object**। Runtime এ if-else দিয়ে parts যোগ করা যায়।

### কিন্তু এটা Verbose!

Criteria API powerful, কিন্তু লিখতে অনেক code লাগে। তাই Spring এই জিনিস কে wrap করে **Specifications** বানিয়েছে।

---

## ধাপ ৩: JPA Specifications কী?

Specifications হলো Spring Data এর একটা interface যেটা Criteria API এর উপর একটা সহজ layer।

একটা **Specification** = একটা **filter condition**।

### Specification Interface

```java
public interface Specification<T> {
    Predicate toPredicate(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb);
}
```

দেখতে ভয়ঙ্কর কিন্তু use করা simple। একটা উদাহরণ দেখি —

---

## ধাপ ৪: Setup — Repository এ Specification Enable করা

সবার আগে Repository কে `JpaSpecificationExecutor` extend করাতে হবে —

```java
public interface BookRepository extends
        JpaRepository<Book, Long>,
        JpaSpecificationExecutor<Book> {    // ⭐ এই line যোগ করো
}
```

এটা না করলে Specification use করা যাবে না।

`JpaSpecificationExecutor` এই method গুলো দেয় —

```java
List<Book> findAll(Specification<Book> spec);
Page<Book> findAll(Specification<Book> spec, Pageable pageable);
Optional<Book> findOne(Specification<Book> spec);
long count(Specification<Book> spec);
```

---

## ধাপ ৫: প্রথম Specification লিখি

### Title দিয়ে Search

```java
public class BookSpecifications {

    public static Specification<Book> hasTitle(String title) {
        return (root, query, criteriaBuilder) ->
            criteriaBuilder.equal(root.get("title"), title);
    }
}
```

### এই Code কী করছে?

আগে prerequisite বুঝি — lambda expression এ `(root, query, cb) ->` এই তিনটা parameter কী?

| Parameter | কাজ |
|---|---|
| `root` | কোন Entity তে query চালাবো (এখানে Book) |
| `query` | পুরো query object (distinct, groupBy etc. এর জন্য) |
| `criteriaBuilder` | শর্ত (Predicate) বানানোর tool |

### Step by Step

```java
root.get("title")
// → "Book entity এর title field কে access করো"

criteriaBuilder.equal(root.get("title"), title)
// → "title field = parameter এর title" শর্ত বানাও
```

এই return value টাই **Predicate** — একটা WHERE শর্ত।

### Use করি

```java
Specification<Book> spec = BookSpecifications.hasTitle("Pother Pachali");
List<Book> books = bookRepository.findAll(spec);
```

Hibernate এর generated SQL —

```sql
SELECT * FROM books WHERE title = 'Pother Pachali';
```

---

## ধাপ ৬: একাধিক Specification

### আরো Specifications যোগ করি

```java
public class BookSpecifications {

    public static Specification<Book> hasTitle(String title) {
        return (root, query, cb) -> cb.equal(root.get("title"), title);
    }

    public static Specification<Book> hasAuthor(String authorName) {
        return (root, query, cb) -> cb.equal(root.get("author").get("name"), authorName);
        // root.get("author").get("name") = Book.author.name
    }

    public static Specification<Book> priceGreaterThan(Double minPrice) {
        return (root, query, cb) -> cb.greaterThan(root.get("price"), minPrice);
    }

    public static Specification<Book> titleContains(String keyword) {
        return (root, query, cb) ->
            cb.like(root.get("title"), "%" + keyword + "%");
    }
}
```

### CriteriaBuilder এর সব Useful Method

| Method | কাজ | SQL |
|---|---|---|
| `cb.equal(x, y)` | x = y | `x = y` |
| `cb.notEqual(x, y)` | x ≠ y | `x != y` |
| `cb.greaterThan(x, y)` | x > y | `x > y` |
| `cb.lessThan(x, y)` | x < y | `x < y` |
| `cb.greaterThanOrEqualTo(x, y)` | x ≥ y | `x >= y` |
| `cb.lessThanOrEqualTo(x, y)` | x ≤ y | `x <= y` |
| `cb.like(x, pattern)` | Pattern match | `LIKE '%val%'` |
| `cb.isNull(x)` | x IS NULL | `x IS NULL` |
| `cb.isNotNull(x)` | x IS NOT NULL | `x IS NOT NULL` |
| `cb.between(x, low, high)` | Range | `x BETWEEN low AND high` |
| `cb.in(x).value(v1).value(v2)` | IN clause | `x IN (v1, v2)` |

---

## ধাপ ৭: Specifications Combine করা

Specification গুলো মিলিয়ে complex query বানানো যায় —

### `and()` — দুই শর্ত ই সত্যি হতে হবে

```java
Specification<Book> spec = Specification
    .where(BookSpecifications.hasTitle("Pother Pachali"))
    .and(BookSpecifications.hasAuthor("Bibhutibhushan"));

List<Book> books = bookRepository.findAll(spec);
```

Generated SQL —

```sql
SELECT * FROM books b
JOIN authors a ON a.id = b.author_id
WHERE b.title = 'Pother Pachali'
  AND a.name = 'Bibhutibhushan';
```

### `or()` — যেকোনো একটা সত্যি হলেই চলবে

```java
Specification<Book> spec = Specification
    .where(BookSpecifications.hasTitle("Pother Pachali"))
    .or(BookSpecifications.hasTitle("Aparajito"));
```

### `not()` — উল্টো করা

```java
Specification<Book> spec = Specification
    .not(BookSpecifications.hasAuthor("Sharatchandra"));
```

### কয়েকটা একসাথে

```java
Specification<Book> spec = Specification
    .where(BookSpecifications.titleContains("Pother"))
    .and(BookSpecifications.priceGreaterThan(100.0))
    .and(BookSpecifications.hasAuthor("Bibhutibhushan"));
```

---

## ধাপ ৮: Dynamic Search (আসল Use Case)

এখন মূল সমস্যায় ফিরে যাই — runtime এ filter change হয়।

```java
@Service
public class BookSearchService {

    private final BookRepository bookRepository;

    public List<Book> search(String title, String author, Double minPrice) {

        Specification<Book> spec = Specification.where(null);
        // শুরুতে খালি spec (কোনো শর্ত নেই)

        if (title != null) {
            spec = spec.and(BookSpecifications.titleContains(title));
        }

        if (author != null) {
            spec = spec.and(BookSpecifications.hasAuthor(author));
        }

        if (minPrice != null) {
            spec = spec.and(BookSpecifications.priceGreaterThan(minPrice));
        }

        return bookRepository.findAll(spec);
    }
}
```

### এখন কী সুবিধা?

User যা যা পাঠাবে, শুধু সেই filter গুলোই apply হবে —

```
API call                                Generated SQL
─────────────────────────────────────────────────────────────
/books/search                          → SELECT * FROM books

/books/search?title=Pother              → SELECT * FROM books
                                         WHERE title LIKE '%Pother%'

/books/search?author=Bibhutibhushan     → SELECT * FROM books
                                         JOIN authors ...
                                         WHERE a.name = 'Bibhutibhushan'

/books/search?title=Pother&author=Bibhutibhushan&minPrice=100
                                        → SELECT * FROM books
                                          JOIN authors ...
                                          WHERE title LIKE '%Pother%'
                                            AND a.name = 'Bibhutibhushan'
                                            AND price > 100
```

কোনো if-else hell নেই, method explosion নেই। ✅

---

## ধাপ ৯: Controller এ Integration

```java
@RestController
@RequestMapping("/api/books")
@RequiredArgsConstructor
public class BookController {

    private final BookSearchService bookSearchService;

    @GetMapping("/search")
    public ResponseEntity<List<Book>> search(
            @RequestParam(required = false) String title,
            @RequestParam(required = false) String author,
            @RequestParam(required = false) Double minPrice) {

        List<Book> books = bookSearchService.search(title, author, minPrice);
        return ResponseEntity.ok(books);
    }
}
```

`required = false` দিয়ে বলছি — parameter optional।

---

## ধাপ ১০: Common Mistakes

### Mistake 1: `JpaSpecificationExecutor` extend না করা

```java
public interface BookRepository extends JpaRepository<Book, Long> {
    // ❌ JpaSpecificationExecutor extend করিনি
}
```

```java
bookRepository.findAll(spec);   // ❌ Compile error!
```

**সমাধান:** `JpaSpecificationExecutor<Book>` ও extend করো।

### Mistake 2: Root Field Name ভুল

```java
return (root, query, cb) -> cb.equal(root.get("titel"), title);
//                                          ^^^^^ typo!
```

**কী হবে?** Runtime exception — field পাওয়া যায়নি।

**সমাধান:** Entity class এর field name গুলো সাবধানে লেখো।

### Mistake 3: Specification এর combine ভুল

```java
Specification<Book> spec = BookSpecifications.hasTitle(title);
spec.and(BookSpecifications.hasAuthor(author));   // ❌ return value ফেলে দিলাম!
```

**কী হবে?** `and()` এর result save হয়নি — শুধু hasTitle apply হবে।

**সমাধান:** Return value assign করো —

```java
spec = spec.and(BookSpecifications.hasAuthor(author));   // ✅
```

### Mistake 4: Static Query যখন Dynamic দরকার না

যদি filter গুলো fixed হয় (runtime এ change হয় না), তাহলে Specification overkill। সাধারণ `@Query` ই যথেষ্ট।

**Specification শুধু dynamic query এর জন্য।**

---

<a name="bridge"></a>

# Bridge — Specifications ও Transactional এর সম্পর্ক

## কেন এই দুইটা একসাথে পড়ছি?

Specifications হলো **query বানানোর tool।** Transactional হলো **সেই query চালানোর environment।**

---

## Bridge Point 1: Query চালানোর জন্য Session লাগে

```java
Specification<Book> spec = BookSpecifications.titleContains("Pother");
List<Book> books = bookRepository.findAll(spec);
```

এই `findAll()` কোথায় query চালায়? → **Hibernate Session এ।**

Session ছাড়া query চালানো যায় না। আর Session খোলা থাকে **Transaction এর ভেতরে।**

Spring Data JPA method গুলো (যেমন `findAll`) **নিজে থেকেই ছোট একটা Transaction শুরু করে।** তাই single query চালাতে আলাদা `@Transactional` লাগে না।

---

## Bridge Point 2: LAZY Data এবং Specifications

ধরো তোমার Specification এ Book এর সাথে Author এর relationship check করছো —

```java
return (root, query, cb) -> cb.equal(root.get("author").get("name"), authorName);
```

Query result এ Book আসবে, কিন্তু Book এর `author` field **LAZY।**

```java
@Service
public class BookService {

    public List<String> getBookTitlesWithAuthorNames() {
        Specification<Book> spec = BookSpecifications.titleContains("Pother");
        List<Book> books = bookRepository.findAll(spec);
        // ↑ এখানে Session বন্ধ হয়ে গেছে!

        return books.stream()
            .map(b -> b.getTitle() + " - " + b.getAuthor().getName())
            // ↑ LazyInitializationException! 💥
            .toList();
    }
}
```

**সমাধান:** Method কে `@Transactional` দাও, Session পুরো method জুড়ে খোলা থাকবে।

```java
@Transactional(readOnly = true)   // ⭐
public List<String> getBookTitlesWithAuthorNames() {
    // এখন LAZY access safe ✅
}
```

এখানেই **Specifications → Transactional** এর সম্পর্ক।

---

## Bridge Point 3: Multiple Queries একসাথে

```java
@Transactional
public void bulkUpdate() {

    Specification<Book> oldBooks = BookSpecifications.createdBefore(2000);
    List<Book> books = bookRepository.findAll(oldBooks);     // Query 1

    books.forEach(b -> b.setStatus("ARCHIVED"));              // Update গুলো
                                                               // dirty checking এ track

    // Method শেষ হলে Hibernate auto UPDATE query চালাবে      // Query 2+
}
```

এখানে একাধিক query একসাথে atomic ভাবে চালাতে হবে। সব success হলে commit, না হলে rollback। তাই `@Transactional` দরকার।

---

এবার Transactional এ গভীরে যাই।

---

<a name="পর্ব-২"></a>

# পর্ব ২ — `@Transactional` গভীরে

## ধাপ ১: Transaction কী?

### বাস্তব Analogy — bKash Transfer

ধরো তুমি bKash থেকে Karim কে ৫০০ টাকা পাঠাচ্ছো —

```
Step 1: তোমার account থেকে ৫০০ টাকা কাটা
Step 2: Karim এর account এ ৫০০ টাকা যোগ করা
```

এখন ধরো **Step 1 হয়ে গেল, কিন্তু Step 2 এ server crash হলো** —

```
তোমার টাকা কেটে গেছে ❌
Karim এর টাকা যোগ হয়নি ❌
৫০০ টাকা হাওয়া! 😱
```

এটা হলে bKash ব্যবসা বন্ধ করতে হবে। তাই দুইটা step কে **একটা unit** হিসেবে treat করতে হয় —

> *"হয় দুইটাই সফল হবে, নাহলে কোনোটাই হবে না।"*

এই unit কেই বলে **Transaction**।

### Database এর context এ

```sql
BEGIN TRANSACTION;

    UPDATE accounts SET balance = balance - 500 WHERE user = 'Hasib';
    UPDATE accounts SET balance = balance + 500 WHERE user = 'Karim';

COMMIT;     -- দুইটাই সফল → permanently save
-- অথবা --
ROLLBACK;  -- কোনোটা fail → সব পরিবর্তন undo
```

---

## ধাপ ২: ACID Properties

একটা সত্যিকারের Transaction এ ৪টা property থাকতে হয় —

### A — Atomicity (অবিভাজ্যতা)

"Atom" মানে ভাঙা যায় না।

> *"Transaction এর সব step একসাথে হবে, নাহলে কোনোটাই হবে না।"*

### C — Consistency (সামঞ্জস্য)

> *"Transaction শুরু হওয়ার আগে database যেমন valid ছিল, শেষেও সেরকম valid থাকবে।"*

Transfer এর আগে total টাকা X ছিল, পরেও X। টাকা হারাবে না, তৈরি হবে না।

### I — Isolation (বিচ্ছিন্নতা)

> *"একই সময়ে দুইজন কাজ করলেও একজনের transaction অন্যজনকে disturb করবে না।"*

### D — Durability (স্থায়িত্ব)

> *"একবার commit হলে permanent — electricity চলে গেলেও data থাকবে।"*

Commit এর পর data hard disk এ save হয়।

---

## ধাপ ৩: Hibernate Session — আরো গভীরে

Session হলো database এর সাথে কথা বলার channel। এর ভেতরে কী থাকে?

```
Hibernate Session:
├── Database Connection       (live link)
├── Transaction              (current running)
├── Persistence Context      (cache — tracked objects)
└── Dirty Checking Engine    (change detection)
```

### Persistence Context কী?

Session এর ভেতরে load করা সব Entity এখানে track হয়।

```java
@Transactional
public void updateAuthor(Long id) {
    Author author = authorRepository.findById(id).get();
    // ↑ author এখন Persistence Context এ track

    author.setName("New Name");
    // ↑ save() call করোনি, তবু Hibernate জানে change হয়েছে!

    // Method শেষে Hibernate auto-detect করে → UPDATE query
}
```

এটা **dirty checking** — Hibernate নিজে বুঝে কোন object change হয়েছে।

---

## ধাপ ৪: `@Transactional` কীভাবে কাজ করে?

### Spring AOP Proxy এর জাদু

`@Transactional` আসলে **Spring AOP** এর মাধ্যমে কাজ করে। Spring একটা **proxy class** তৈরি করে।

```
Client  →  AuthorServiceProxy (Spring এর তৈরি)
              │
              ├── Step 1: BEGIN transaction
              ├── Step 2: তোমার method call করো
              ├── Step 3a: সব ঠিক → COMMIT
              └── Step 3b: Exception → ROLLBACK
```

### Internally যা হয়

Spring এরকম কিছু লিখে দেয় —

```java
public void createAuthor(AuthorDTO dto) {
    Transaction tx = beginTransaction();
    try {
        realService.createAuthor(dto);   // তোমার code
        tx.commit();
    } catch (Exception e) {
        tx.rollback();
        throw e;
    }
}
```

তুমি এই code লিখো না — Spring proxy দিয়ে করে।

### গুরুত্বপূর্ণ Limitation — Self Invocation

Same class এর ভেতরে একটা `@Transactional` method থেকে আরেকটা `@Transactional` method call করলে **Transaction কাজ করে না!**

```java
@Service
public class AuthorService {

    public void createAuthorAndBook(AuthorDTO dto) {
        this.createAuthor(dto);    // ❌ Transaction work করবে না!
    }

    @Transactional
    public void createAuthor(AuthorDTO dto) { }
}
```

**কেন?** `this.createAuthor()` proxy bypass করে সরাসরি real method call করে। Spring wrap করতে পারে না।

**সমাধান:**
- Outer method এ `@Transactional` দাও
- অথবা method গুলো আলাদা service class এ রাখো

---

## ধাপ ৫: Propagation Types

Propagation মানে — একটা transactional method আরেকটা transactional method call করলে কী হবে।

৭টা type আছে। মূল ৩টা —

### 1. `REQUIRED` (Default)

> *"Transaction আগে থেকে থাকলে use করো, না থাকলে নতুন শুরু করো।"*

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    methodB();   // same transaction এ চলবে
}

@Transactional(propagation = Propagation.REQUIRED)
public void methodB() { }
```

**কখন use:** ৯৯% ক্ষেত্রে। এটাই default।

### 2. `REQUIRES_NEW`

> *"সবসময় একটা নতুন transaction শুরু করো। চলমান transaction থাকলে pause করো।"*

```java
@Transactional(propagation = Propagation.REQUIRED)
public void placeOrder() {
    saveOrder();           // Transaction 1
    sendAuditLog();        // Transaction 2 (আলাদা!)
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void sendAuditLog() { }
```

**কখন use:** Audit logging, notification — main transaction fail হলেও এগুলো save চাইলে।

### 3. `NESTED`

> *"Parent এর ভেতরে nested transaction। Child rollback হলে শুধু child undo।"*

Database এর SAVEPOINT use করে। Rare।

### বাকি ৪টা (কম common)

| Type | মানে |
|---|---|
| `SUPPORTS` | Transaction থাকলে use, না থাকলেও চলবে |
| `NOT_SUPPORTED` | Transaction থাকলে pause, non-transactional চালাও |
| `MANDATORY` | Transaction অবশ্যই থাকতে হবে |
| `NEVER` | Transaction থাকলে error |

---

## ধাপ ৬: Isolation Levels

### সমস্যা টা আগে বুঝি

একই সময়ে দুইজন user একই data access করলে কিছু weird problem হতে পারে —

#### Problem 1: Dirty Read (অর্ধেক data পড়া)

```
User A: UPDATE balance SET value = 1000 WHERE id = 1;
        (commit হয়নি এখনো)

User B: SELECT balance FROM accounts WHERE id = 1;
        → 1000 পেল (commit হয়নি, তবু পড়ে ফেললো!)

User A: ROLLBACK;
        → A এর change undo হলো, কিন্তু B এর হাতে ভুল data!
```

#### Problem 2: Non-Repeatable Read

```
User A: SELECT balance FROM accounts WHERE id = 1;
        → 500 পেল

User B: UPDATE balance SET value = 1000 WHERE id = 1;
        COMMIT;

User A: SELECT balance FROM accounts WHERE id = 1;  (same transaction এ!)
        → 1000 পেল (আগের সাথে মিলল না!)
```

#### Problem 3: Phantom Read

```
User A: SELECT * FROM orders WHERE price > 100;
        → 5টা order পেল

User B: INSERT INTO orders VALUES (..., price = 200);
        COMMIT;

User A: SELECT * FROM orders WHERE price > 100;
        → 6টা order! (phantom row এলো)
```

### Isolation Levels — এই সমস্যার সমাধান

৪টা level আছে — weak থেকে strong —

| Level | Dirty Read | Non-Repeatable | Phantom | Performance |
|---|---|---|---|---|
| `READ_UNCOMMITTED` | ❌ হয় | ❌ হয় | ❌ হয় | Fastest |
| `READ_COMMITTED` | ✅ prevent | ❌ হয় | ❌ হয় | Fast |
| `REPEATABLE_READ` | ✅ prevent | ✅ prevent | ❌ হয় | Medium |
| `SERIALIZABLE` | ✅ prevent | ✅ prevent | ✅ prevent | Slowest |

### Code এ

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transfer() { }
```

### Default কী?

Database এর default level use হয় (সাধারণত `READ_COMMITTED`)। ৯৫% ক্ষেত্রে এটাই যথেষ্ট।

---

## ধাপ ৭: Rollback Rules

Default — **শুধু RuntimeException এ rollback হয়।** Checked Exception এ হয় না!

### Default Behavior

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);
    
    throw new RuntimeException("Error!");
    // ✅ Rollback হবে
}
```

```java
@Transactional
public void createOrder() throws IOException {
    orderRepository.save(order);
    
    throw new IOException("Error!");
    // ❌ Rollback হবে না!
}
```

### Checked Exception এও Rollback চাইলে

```java
@Transactional(rollbackFor = IOException.class)
public void createOrder() throws IOException { }
```

বা সব Exception এ —

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() throws Exception { }
```

### Specific Exception এ Rollback না চাইলে

```java
@Transactional(noRollbackFor = ValidationException.class)
public void createOrder() { }
```

---

## ধাপ ৮: read-only Transaction

শুধু data পড়বো, update করবো না — এমন ক্ষেত্রে —

```java
@Transactional(readOnly = true)
public List<Book> getAllBooks() {
    return bookRepository.findAll();
}
```

### সুবিধা

- Hibernate dirty checking skip করে (faster)
- Database optimization করতে পারে
- Accidental update prevent করে

**Rule of thumb:** GET API গুলোতে `readOnly = true` দাও।

---

## ধাপ ৯: কোথায় লাগাবো?

### Best Practice: Service Layer

```java
@Service
public class AuthorService {

    @Transactional
    public Author createAuthor(AuthorDTO dto) {
        // business logic
    }
}
```

### Controller এ না

```java
@RestController
public class AuthorController {

    @Transactional   // ❌ Bad practice
    @PostMapping
    public Author create(@RequestBody AuthorDTO dto) { }
}
```

**কেন?** Controller এর কাজ request/response handle করা। Transaction business logic এর concern — সেটা Service এর।

### Repository এ না

Spring Data JPA এর method গুলো নিজে থেকেই transactional। আলাদা দিতে হয় না।

---

## ধাপ ১০: Common Mistakes

### Mistake 1: `private` method এ `@Transactional`

```java
@Transactional
private void internalMethod() { }    // ❌ Work করবে না
```

**কেন?** Proxy শুধু public method wrap করতে পারে।

**সমাধান:** Public রাখো।

### Mistake 2: Self Invocation

আগে বললাম — same class এ self call করলে Transaction কাজ করে না।

### Mistake 3: Checked Exception এ Rollback আশা করা

```java
@Transactional
public void method() throws SQLException {
    throw new SQLException();   // ❌ Rollback হবে না (default এ)
}
```

**সমাধান:** `rollbackFor = SQLException.class` দাও।

### Mistake 4: Transaction এর ভেতরে External API Call

```java
@Transactional
public void createOrder() {
    orderRepository.save(order);

    restTemplate.postForObject(...);   // ❌ External API call!
    // যদি slow হয়, পুরো Transaction slow
    // Database connection ধরে রাখে
}
```

**সমাধান:** External call Transaction এর বাইরে করো।

---

<a name="পর্ব-৩"></a>

# পর্ব ৩ — Pagination ও Sorting

## ধাপ ১: সমস্যাটা কী?

ধরো তোমার `books` table এ **১০,০০,০০০ (১০ লাখ)** row আছে।

```java
List<Book> allBooks = bookRepository.findAll();
```

এই একটা line এ কী হবে?

1. Database থেকে ১০ লাখ row আসবে
2. ১০ লাখ Java object তৈরি হবে memory তে
3. Network এ ১০ লাখ row transfer হবে
4. Server memory out of memory → crash! 💥

### সমাধান — Pagination

> *"সব data একসাথে আনবো না। ছোট ছোট page এ ভাগ করে আনবো।"*

যেমন —
- Page 1 = প্রথম ২০ books
- Page 2 = পরের ২০ books
- ...

User যতটুকু চায় ততটুকুই load হবে।

---

## ধাপ ২: Page এর Concept

### Page Size ও Page Number

```
Total books = 100

Page size = 20 (প্রতি page এ ২০টা)

Page 0 → Book 1-20
Page 1 → Book 21-40
Page 2 → Book 41-60
Page 3 → Book 61-80
Page 4 → Book 81-100
```

**Note:** Spring এ page index **0 থেকে শুরু**, 1 থেকে না!

### SQL এ আসলে কী হয়?

```sql
-- Page 2, size 20
SELECT * FROM books
LIMIT 20 OFFSET 40;

-- LIMIT 20 = ২০টা row নাও
-- OFFSET 40 = প্রথম ৪০টা skip করো
```

Spring এটা তোমার জন্য auto generate করে।

---

## ধাপ ৩: `Pageable` Interface

Spring এ pagination করতে `Pageable` interface use করি।

### Pageable বানানো

```java
Pageable pageable = PageRequest.of(0, 20);
// Page 0, size 20
```

### Repository

```java
public interface BookRepository extends JpaRepository<Book, Long> {
    Page<Book> findAll(Pageable pageable);
}
```

**`JpaRepository` থেকেই এই method পাওয়া যায়** — আলাদা define করতে হয় না।

### Use

```java
Pageable pageable = PageRequest.of(0, 20);
Page<Book> bookPage = bookRepository.findAll(pageable);

List<Book> books = bookPage.getContent();     // এই page এর Books
int totalPages = bookPage.getTotalPages();    // মোট page
long totalElements = bookPage.getTotalElements();  // মোট Books
boolean hasNext = bookPage.hasNext();         // পরের page আছে?
```

---

## ধাপ ৪: `Page` বনাম `Slice`

Spring দুইরকম return type দেয় — Page এবং Slice। পার্থক্য কী?

### `Page<T>`

```java
Page<Book> findAll(Pageable pageable);
```

**Internally ২টা query চালায় —**
1. Data fetch করার query
2. `SELECT COUNT(*)` — total কতটা data আছে

সুবিধা — `getTotalPages()`, `getTotalElements()` পাওয়া যায়।
অসুবিধা — extra count query এর জন্য slow।

### `Slice<T>`

```java
Slice<Book> findAll(Pageable pageable);
```

**শুধু ১টা query চালায়** (count query নেই)।

সুবিধা — Faster।
অসুবিধা — `getTotalPages()` পাওয়া যায় না।

Slice এ শুধু জানা যায় — *"পরের page আছে কিনা"* (`hasNext()`)।

### কোনটা কখন?

| Scenario | Use |
|---|---|
| Total count দেখাতে হবে ("Page 5 of 20") | `Page` |
| শুধু "Next" button ("Infinite scroll") | `Slice` |
| Performance critical | `Slice` |

---

## ধাপ ৫: Sorting

### Single Field Sort

```java
Pageable pageable = PageRequest.of(0, 20, Sort.by("title"));
// title অনুযায়ী ascending sort
```

### Ascending / Descending

```java
Sort.by("title").ascending()       // A-Z
Sort.by("title").descending()      // Z-A
Sort.by(Sort.Direction.DESC, "title")
```

### Multiple Field Sort

```java
Sort sort = Sort.by("author").ascending()
                .and(Sort.by("title").ascending());
Pageable pageable = PageRequest.of(0, 20, sort);
```

Generated SQL —

```sql
SELECT * FROM books
ORDER BY author ASC, title ASC
LIMIT 20 OFFSET 0;
```

### Nested Field (Relationship এ)

```java
Sort.by("author.name").ascending()
// Book এর author এর name অনুযায়ী
```

---

## ধাপ ৬: Controller এ Integration

### Manual Way

```java
@GetMapping
public Page<Book> getBooks(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "title") String sortBy) {

    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
    return bookRepository.findAll(pageable);
}
```

API call —

```
GET /api/books?page=0&size=20&sortBy=title
```

### Spring Magic — `Pageable` সরাসরি parameter

```java
@GetMapping
public Page<Book> getBooks(Pageable pageable) {
    return bookRepository.findAll(pageable);
}
```

Spring automatically query parameter থেকে Pageable বানিয়ে দেয় —

```
GET /api/books?page=0&size=20&sort=title,asc
GET /api/books?page=0&size=20&sort=author.name,desc
```

**Multiple sort —**

```
GET /api/books?page=0&size=20&sort=author.name,asc&sort=title,desc
```

অনেক সহজ এবং clean। ✅

---

## ধাপ ৭: Response Format

Page object serialize হলে এরকম JSON আসে —

```json
{
  "content": [
    { "id": 1, "title": "Pother Pachali" },
    { "id": 2, "title": "Aparajito" }
  ],
  "pageable": {
    "pageNumber": 0,
    "pageSize": 20,
    "sort": { "sorted": true, "unsorted": false }
  },
  "totalElements": 100,
  "totalPages": 5,
  "first": true,
  "last": false,
  "number": 0,
  "numberOfElements": 20,
  "empty": false
}
```

### Custom Response (Best Practice)

Entity সরাসরি না দিয়ে custom format —

```java
public class PagedResponse<T> {
    private List<T> data;
    private int currentPage;
    private int pageSize;
    private long totalItems;
    private int totalPages;
    private boolean hasNext;
    private boolean hasPrevious;
}
```

```java
@GetMapping
public PagedResponse<BookDTO> getBooks(Pageable pageable) {
    Page<Book> page = bookRepository.findAll(pageable);

    return new PagedResponse<>(
        page.getContent().stream().map(this::toDTO).toList(),
        page.getNumber(),
        page.getSize(),
        page.getTotalElements(),
        page.getTotalPages(),
        page.hasNext(),
        page.hasPrevious()
    );
}
```

---

## ধাপ ৮: Specification + Pagination একসাথে

সবচেয়ে powerful combination — dynamic search with pagination —

```java
@Service
public class BookSearchService {

    private final BookRepository bookRepository;

    @Transactional(readOnly = true)
    public Page<Book> search(
            String title,
            String author,
            Double minPrice,
            Pageable pageable) {

        Specification<Book> spec = Specification.where(null);

        if (title != null) {
            spec = spec.and(BookSpecifications.titleContains(title));
        }
        if (author != null) {
            spec = spec.and(BookSpecifications.hasAuthor(author));
        }
        if (minPrice != null) {
            spec = spec.and(BookSpecifications.priceGreaterThan(minPrice));
        }

        return bookRepository.findAll(spec, pageable);
        // ⭐ Specification + Pageable একসাথে!
    }
}
```

```java
@RestController
@RequestMapping("/api/books")
public class BookController {

    @GetMapping("/search")
    public Page<Book> search(
            @RequestParam(required = false) String title,
            @RequestParam(required = false) String author,
            @RequestParam(required = false) Double minPrice,
            Pageable pageable) {

        return bookSearchService.search(title, author, minPrice, pageable);
    }
}
```

API call —

```
GET /api/books/search?title=Pother&minPrice=100&page=0&size=20&sort=title,asc
```

সব কিছু একসাথে — dynamic filter + pagination + sorting — কোনো if-else hell নেই। ✅

---

## ধাপ ৯: Common Mistakes

### Mistake 1: Page Index 1 থেকে ধরা

```java
PageRequest.of(1, 20);   // এটা page 2, page 1 না!
```

**মনে রাখো — Spring এ page index 0 থেকে শুরু।**

### Mistake 2: Huge Page Size

```java
PageRequest.of(0, 100000);   // ❌ Pagination এর মানে নেই!
```

**Rule:** Page size reasonable রাখো — ২০-১০০ এর মধ্যে।

### Mistake 3: Count Query বাদ দেয়া

```java
List<Book> findAll(Pageable pageable);   // ❌ Page না List!
```

`Page<Book>` ব্যবহার করো যাতে total info পাও।

### Mistake 4: Sort field Validation না করা

```java
Pageable pageable = PageRequest.of(0, 20, Sort.by(sortBy));
// sortBy = "../../etc/passwd" হলে?
```

User input sort field এ blindly দিও না। Whitelist check করো।

```java
List<String> allowedFields = List.of("title", "author", "price");
if (!allowedFields.contains(sortBy)) {
    throw new IllegalArgumentException("Invalid sort field");
}
```

---

## চূড়ান্ত সারসংক্ষেপ

### তিনটা Concept এর সম্পর্ক

```
Specifications    →  Dynamic query বানায়
Transactional     →  সেই query চালানোর environment
Pagination        →  Result কে page এ ভাগ করে
```

তিনটা মিলিয়েই production-grade search API তৈরি হয়।

### Master Setup

```java
@Service
@RequiredArgsConstructor
public class BookSearchService {

    private final BookRepository bookRepository;

    @Transactional(readOnly = true)                       // ⭐ Read-only transaction
    public Page<BookDTO> search(
            String title,
            String author,
            Double minPrice,
            Pageable pageable) {                           // ⭐ Pagination

        Specification<Book> spec = buildSpec(title, author, minPrice);   // ⭐ Specification
        return bookRepository.findAll(spec, pageable)
                             .map(this::toDTO);
    }

    private Specification<Book> buildSpec(String title, String author, Double minPrice) {
        Specification<Book> spec = Specification.where(null);
        if (title != null)    spec = spec.and(BookSpecifications.titleContains(title));
        if (author != null)   spec = spec.and(BookSpecifications.hasAuthor(author));
        if (minPrice != null) spec = spec.and(BookSpecifications.priceGreaterThan(minPrice));
        return spec;
    }
}
```

### ৫টা Golden Rule

```
1. Dynamic query = Specifications use করো
   → if-else hell থেকে বাঁচো

2. Service layer এ @Transactional দাও, GET এ readOnly = true
   → Database connection safe থাকবে

3. Page index 0 থেকে শুরু হয়, 1 না
   → এই ভুল সবাই করে

4. Count query চাইলে Page, না চাইলে Slice
   → Performance এর জন্য

5. সবসময় Entity return না করে DTO return করো
   → Security + flexibility
```

---

## পরবর্তী Steps

- **Exception Handling** (`@ControllerAdvice`)
- **Validation** (`@Valid`, Bean Validation)
- **Spring Security** (Authentication + Authorization)
- **Caching** (`@Cacheable`)
- **Asynchronous** (`@Async`)

এই tutorial এর concept গুলো পাকা হলে production-grade Spring Boot API লেখা শুরু করতে পারো। 🚀

Happy coding!
