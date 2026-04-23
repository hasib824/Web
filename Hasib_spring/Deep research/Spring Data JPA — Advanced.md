# Spring Data JPA — Advanced Relationship Concepts

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Previous Tutorial:** Basic Relationships (`@OneToMany`, `@ManyToOne`, etc.)
> **Topics:** Cascade Types, Fetch Types, N+1 Problem, Infinite Recursion, orphanRemoval

---

## সূচিপত্র

1. [Cascade Types — Parent এর কাজ Child পর্যন্ত পৌঁছানো](#পর্ব-১)
2. [Fetch Types — LAZY vs EAGER](#পর্ব-২)
3. [N+1 Query Problem — Performance এর বড় সমস্যা](#পর্ব-৩)
4. [Bidirectional Infinite Recursion — JSON এর Infinite Loop](#পর্ব-৪)
5. [orphanRemoval — অনাথ Entity মুছে ফেলা](#পর্ব-৫)
6. [চূড়ান্ত সারসংক্ষেপ](#পর্ব-৬)

---

<a name="পর্ব-১"></a>

## পর্ব ১ — Cascade Types

### প্রথমে "Cascade" শব্দের মানে বুঝি

"Cascade" একটা English word — মানে **"ঝর্ণার মতো একটার পর একটা নেমে যাওয়া।"**

ঝর্ণায় পানি যখন উপর থেকে নামে — একটা ধাপে পড়ে, তারপর সেখান থেকে পরের ধাপে পড়ে, তারপর আরেক ধাপে পড়ে। একটা কাজ automatically পরেরটাকে trigger করে।

JPA তে Cascade মানে —
> *"Parent এ যে operation করলাম, সেটা automatically Child এও apply হবে।"*

### Parent-Child সম্পর্ক কী?

আমাদের Author-Book example এ —
- **Author** = Parent (বড় entity)
- **Book** = Child (Author এর অধীনে)

Book ছাড়া Author থাকতে পারে, কিন্তু Author ছাড়া Book এর কোনো অস্তিত্ব নেই (orphan হয়ে যায়)।

```java
// Author (Parent)
@OneToMany(mappedBy = "author")
private List<Book> books;

// Book (Child)
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;
```

### Cascade ছাড়া কী সমস্যা হয়?

ধরো তুমি নতুন একজন Author তৈরি করলে, তার তিনটা Book সহ —

```java
Author author = new Author("Bibhutibhushan");

Book book1 = new Book("Pother Pachali");
Book book2 = new Book("Aparajito");
Book book3 = new Book("Aranyak");

book1.setAuthor(author);
book2.setAuthor(author);
book3.setAuthor(author);

author.getBooks().add(book1);
author.getBooks().add(book2);
author.getBooks().add(book3);

authorRepository.save(author);   // ❌ শুধু Author save হবে, Books হবে না!
```

Cascade না থাকলে Hibernate ভাবে —
> *"আমাকে শুধু Author save করতে বলেছে। Books গুলো আমি জানি না, save করার দরকার নেই।"*

Database এ —
```
authors table:                books table:
┌────┬────────────────┐       (empty — কিছুই save হয়নি!)
│ 1  │ Bibhutibhushan │
└────┴────────────────┘
```

### Cascade এর সমাধান

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.PERSIST)  // ⭐
private List<Book> books;
```

এখন Hibernate বুঝবে —
> *"Author save হলে Books-ও save করতে হবে।"*

Database এ —
```
authors table:                books table:
┌────┬────────────────┐       ┌────┬──────────────────┬───────────┐
│ 1  │ Bibhutibhushan │       │ 1  │ Pother Pachali   │ 1         │
└────┴────────────────┘       │ 2  │ Aparajito        │ 1         │
                              │ 3  │ Aranyak          │ 1         │
                              └────┴──────────────────┴───────────┘
```

---

### Cascade এর সব Type — একে একে

#### 1. `CascadeType.PERSIST`

**কাজ:** Parent save করলে Child-ও save হবে।

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.PERSIST)
private List<Book> books;
```

**কখন use করো?** নতুন Author create করার সময় তার নতুন Books ও একসাথে save করতে চাইলে।

#### 2. `CascadeType.MERGE`

**কাজ:** Parent update করলে Child-ও update হবে।

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.MERGE)
private List<Book> books;
```

**Scenario:**
```java
Author author = authorRepository.findById(1L).get();
author.setName("New Name");

Book book = author.getBooks().get(0);
book.setTitle("Updated Title");

authorRepository.save(author);   // দুইটাই update হবে ✅
```

#### 3. `CascadeType.REMOVE`

**কাজ:** Parent delete করলে সব Child-ও delete হবে।

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.REMOVE)
private List<Book> books;
```

**Scenario:**
```java
authorRepository.deleteById(1L);
// Author delete হবে
// + তার সব Books ও delete হবে ✅
```

⚠️ **সাবধান:** এটা dangerous। ভুল করে parent delete করলে সব child data হারিয়ে যাবে।

#### 4. `CascadeType.REFRESH`

**কাজ:** Parent কে database থেকে re-load করলে Child-ও re-load হবে।

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.REFRESH)
private List<Book> books;
```

খুব rare use case — সাধারণত দরকার হয় না।

#### 5. `CascadeType.DETACH`

**কাজ:** Parent কে Hibernate এর tracking থেকে সরিয়ে নিলে Child-ও সরবে।

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.DETACH)
private List<Book> books;
```

এটাও rare।

#### 6. `CascadeType.ALL`

**কাজ:** উপরের সব type একসাথে।

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
private List<Book> books;
```

**সবচেয়ে common** — যদি parent-child strongly linked হয় (মানে child parent ছাড়া বাঁচবে না)।

### একাধিক Cascade Type একসাথে

```java
@OneToMany(mappedBy = "author", cascade = {
    CascadeType.PERSIST,
    CascadeType.MERGE
})
private List<Book> books;
```

`CascadeType.REMOVE` বাদ দিতে চাইলে এভাবে করো — save/update cascade হবে কিন্তু delete cascade হবে না।

### সারসংক্ষেপ Table

| Type | কাজ | কখন use |
|---|---|---|
| `PERSIST` | Save cascade | নতুন parent + নতুন child |
| `MERGE` | Update cascade | Parent update = child update |
| `REMOVE` | Delete cascade | Parent delete = child delete |
| `REFRESH` | Reload cascade | খুব rare |
| `DETACH` | Detach cascade | খুব rare |
| `ALL` | সব একসাথে | Strong parent-child |

### Common Mistake

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
private List<Book> books;
```

তারপর —

```java
authorRepository.deleteById(1L);   // Author + সব Books delete হয়ে গেল!
```

কিন্তু হয়তো তুমি শুধু Author কে soft-delete করতে চাইছিলে। **ভেবেচিন্তে `CascadeType.ALL` use করো।**

---

<a name="পর্ব-২"></a>

## পর্ব ২ — Fetch Types (LAZY vs EAGER)

### "Fetch" মানে কী?

"Fetch" মানে **"আনা"** — database থেকে data এনে Java Object এ ভরা।

যখন তুমি `authorRepository.findById(1L)` করো, Hibernate database এ যায়, data fetch করে, একটা Java Author object এ সেট করে তোমাকে দেয়।

### কিন্তু Related Data কী হবে?

ধরো Author এর সাথে Books এর relationship আছে —

```java
@OneToMany(mappedBy = "author")
private List<Book> books;
```

Author fetch করার সময় তার Books-ও কি এখনই fetch হবে? নাকি পরে?

এটাই **Fetch Type** — কখন related data fetch হবে সেটা decide করে।

### দুইটা Type

#### 1. `FetchType.LAZY` (অলস — দরকার হলে তখন আনবো)

```java
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
private List<Book> books;
```

**কী হয়?**

```java
Author author = authorRepository.findById(1L).get();
// → SELECT * FROM authors WHERE id = 1
// Books এখনো load হয়নি ⏸️

List<Book> books = author.getBooks();
// → এখন query হলো:
// → SELECT * FROM books WHERE author_id = 1 ✅
```

Books actual access করা পর্যন্ত query হয় না।

#### 2. `FetchType.EAGER` (তাড়াতাড়ি — সাথে সাথেই আনবো)

```java
@OneToMany(mappedBy = "author", fetch = FetchType.EAGER)
private List<Book> books;
```

**কী হয়?**

```java
Author author = authorRepository.findById(1L).get();
// → SELECT * FROM authors a
//    LEFT JOIN books b ON b.author_id = a.id
//    WHERE a.id = 1
// Books ও সাথে সাথে load ✅
```

Author এর সাথেই Books load হয়ে যায়।

### Default Behavior

JPA এর default fetch type গুলো —

| Annotation | Default |
|---|---|
| `@OneToOne` | `EAGER` |
| `@ManyToOne` | `EAGER` |
| `@OneToMany` | `LAZY` |
| `@ManyToMany` | `LAZY` |

**কেন এই pattern?**

- **Single reference** (@OneToOne, @ManyToOne) → EAGER কারণ একটা মাত্র row, cheap
- **Collection** (@OneToMany, @ManyToMany) → LAZY কারণ হাজার হাজার row হতে পারে, costly

### কোনটা কখন use করবো?

#### LAZY use করো —

- Collection এর ক্ষেত্রে (default) ✅
- যখন related data সবসময় দরকার হয় না
- Performance এর জন্য important

```java
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)
private List<Book> books;
```

#### EAGER use করো —

- যখন parent এর সাথে child প্রায় সবসময়ই দরকার
- Very small related data
- Single reference (@ManyToOne এর সাথে)

```java
@ManyToOne(fetch = FetchType.EAGER)   // Default, কিন্তু explicit দেয়া ভালো
@JoinColumn(name = "author_id")
private Author author;
```

### LAZY এর Catch — `LazyInitializationException`

```java
public List<Book> getBooksOutside() {
    Author author;

    // Transaction এর ভেতরে
    // @Transactional
    author = authorRepository.findById(1L).get();
    // Transaction শেষ

    // Transaction এর বাইরে
    return author.getBooks();   // ❌ LazyInitializationException!
}
```

**কেন?** LAZY fetch এর query চালানোর জন্য Hibernate Session দরকার। Transaction শেষ হলে Session-ও শেষ। তাই তখন আর query করা যায় না।

**সমাধান:**
1. Method টাকে `@Transactional` দাও
2. অথবা Transaction এর ভেতরেই `getBooks()` call করে fetch করে নাও
3. অথবা `JOIN FETCH` ব্যবহার করো (পরের পর্বে)

### Common Mistake

সবকিছু EAGER করে দেওয়া —

```java
@OneToMany(mappedBy = "author", fetch = FetchType.EAGER)   // ❌
private List<Book> books;

@OneToMany(mappedBy = "book", fetch = FetchType.EAGER)     // ❌
private List<Review> reviews;
```

**কী হবে?** Author load করলে তার সব Books, প্রতিটা Book এর সব Reviews, সব Review এর সব Users ... একসাথে load হবে। Performance disaster!

**Rule of thumb:** LAZY হলো default, শুধু যেখানে সত্যিই দরকার সেখানে EAGER।

---

<a name="পর্ব-৩"></a>

## পর্ব ৩ — N+1 Query Problem

### "N+1" নামটা কোথা থেকে এলো?

Name টাই explain করে সমস্যাটা কী —
- **1 query** parent load করার জন্য
- **N queries** প্রতিটা parent এর child load করার জন্য
- মোট = **N+1 queries**

### সমস্যাটা আসলে কী?

ধরো ১০ জন Author আছে, প্রতিজনের ৫টা করে Book। তুমি সব Authors এবং তাদের সব Books দেখাতে চাও।

```java
List<Author> authors = authorRepository.findAll();

for (Author author : authors) {
    System.out.println(author.getName());
    for (Book book : author.getBooks()) {   // ⚠️ প্রতিটা iteration এ query!
        System.out.println("  " + book.getTitle());
    }
}
```

### পর্দার পেছনে কী হচ্ছে?

**Query 1** (সব Authors):
```sql
SELECT * FROM authors;
```

**Query 2** (1st Author এর Books):
```sql
SELECT * FROM books WHERE author_id = 1;
```

**Query 3** (2nd Author এর Books):
```sql
SELECT * FROM books WHERE author_id = 2;
```

**... এভাবে মোট 10 বার বেশি query!**

মোট query = **1 (authors) + 10 (books per author) = 11 queries** 😱

১০০ Author হলে = **101 queries**!

### কেন এটা এত খারাপ?

প্রতিটা query database এর সাথে communication। Network roundtrip আছে। **100 queries চালানো = 100 বার database এ যাওয়া-আসা।**

একটা joined query এ সব আনলে ১ বারেই শেষ।

### সমাধান 1: `JOIN FETCH` (JPQL)

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    List<Author> findAllWithBooks();
}
```

এখন মাত্র **১ query** হবে —

```sql
SELECT a.*, b.*
FROM authors a
LEFT JOIN books b ON b.author_id = a.id;
```

### সমাধান 2: `@EntityGraph`

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})
    List<Author> findAll();
}
```

একই কাজ — কিন্তু JPQL লিখতে হলো না। Annotation দিয়েই Hibernate join query generate করবে।

### সমাধান 3: Batch Fetching

```java
@OneToMany(mappedBy = "author")
@BatchSize(size = 10)
private List<Book> books;
```

Hibernate এখন 10 টা parent এর books একসাথে fetch করবে। N+1 এর বদলে N/10 + 1 queries।

### N+1 কখন হয়?

- LAZY fetch এ loop এর মধ্যে related data access করলে
- EAGER এও হতে পারে যদি proper join না হয়

### কীভাবে Detect করবো?

`application.properties` এ —

```properties
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

Console এ সব SQL দেখতে পাবে। একই pattern এর query বার বার হলে বুঝবে N+1 হচ্ছে।

---

<a name="পর্ব-৪"></a>

## পর্ব ৪ — Bidirectional Infinite Recursion

### JSON Serialize মানে কী?

যখন তুমি API থেকে Java Object return করো, Spring সেটাকে JSON এ convert করে client কে পাঠায়। এই convert করার process কে বলে **JSON Serialization**।

Jackson library এটা করে — field by field traverse করে JSON বানায়।

### সমস্যাটা কীভাবে তৈরি হয়?

Bidirectional relationship এ —

```java
// Author class
@OneToMany(mappedBy = "author")
private List<Book> books;

// Book class
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;
```

এখন Author return করলে —

```
Step 1: Author serialize করছি
  Step 2: Author এর books field দেখি
    Step 3: Book[0] serialize করি
      Step 4: Book[0] এর author field দেখি
        Step 5: আবার Author serialize করি
          Step 6: আবার Author এর books field দেখি
            Step 7: আবার Book[0] serialize করি
              ... ∞ (infinite loop!)
```

### Error টা কী?

```
StackOverflowError
  or
JsonMappingException: Infinite recursion
```

### সমাধান 1: `@JsonManagedReference` + `@JsonBackReference`

```java
// Author class
@OneToMany(mappedBy = "author")
@JsonManagedReference            // ⭐ "এটা সামনের দিক — এটা serialize হবে"
private List<Book> books;

// Book class
@ManyToOne
@JoinColumn(name = "author_id")
@JsonBackReference               // ⭐ "এটা পেছনের দিক — এটা skip হবে"
private Author author;
```

**JSON Output:**

```json
{
  "id": 1,
  "name": "Hasib",
  "books": [
    { "id": 5, "title": "Book 1" },   // ✅ author field নেই
    { "id": 8, "title": "Book 2" }
  ]
}
```

### সমাধান 2: `@JsonIgnore`

```java
// Book class
@ManyToOne
@JoinColumn(name = "author_id")
@JsonIgnore                      // ⭐ author field JSON এ যাবেই না
private Author author;
```

সরাসরি field টা JSON থেকে বাদ দাও।

**অসুবিধা:** Book API call করলে author এর তথ্য পাবে না — কিন্তু কখনো কখনো এটাই লাগতে পারে।

### সমাধান 3: DTO ব্যবহার (Best Practice!)

Entity সরাসরি return না করে DTO use করো —

```java
public class AuthorResponseDTO {
    private Long id;
    private String name;
    private List<BookSimpleDTO> books;    // Simple DTO, no back-reference
}

public class BookSimpleDTO {
    private Long id;
    private String title;
    // author field নেই — recursion এর সুযোগ নেই
}
```

**কেন এটা best?**
- No infinite recursion (structure ই allow করে না)
- Entity change হলে API response change হয় না
- Security (sensitive fields expose হয় না)

> **আগের tutorial এ DTO নিয়ে discuss করেছি — সেই reasoning এখানেও apply হয়।**

### কোনটা কখন Use করবো?

| Scenario | Solution |
|---|---|
| Quick fix, prototype | `@JsonIgnore` |
| Small project, direct entity return | `@JsonManagedReference` / `@JsonBackReference` |
| Production, serious project | **DTO ব্যবহার করো** ✅ |

---

<a name="পর্ব-৫"></a>

## পর্ব ৫ — orphanRemoval

### "Orphan" শব্দের মানে

"Orphan" মানে **অনাথ** — যার parent নেই।

Database এর context এ — যে Child row এর parent আর associate নেই।

### Scenario টা বুঝি

ধরো Author এর books list থেকে একটা Book remove করলাম —

```java
Author author = authorRepository.findById(1L).get();

Book firstBook = author.getBooks().get(0);
author.getBooks().remove(firstBook);       // Java এ remove করলাম

authorRepository.save(author);
```

### Database এ কী হবে?

**`orphanRemoval` ছাড়া:**

```
books table:
┌────┬──────────────────┬───────────┐
│ id │ title            │ author_id │
├────┼──────────────────┼───────────┤
│ 5  │ Pother Pachali   │ NULL      │  ← author_id null হয়ে গেল, book delete হয়নি
│ 8  │ Aparajito        │ 1         │
└────┴──────────────────┴───────────┘
```

Book orphan হয়ে গেল — কোনো Author এর সাথে associate নেই, কিন্তু database এ রয়ে গেল। এটা garbage data।

**`orphanRemoval = true` দিলে:**

```java
@OneToMany(mappedBy = "author", orphanRemoval = true)
private List<Book> books;
```

```
books table:
┌────┬──────────────────┬───────────┐
│ id │ title            │ author_id │
├────┼──────────────────┼───────────┤
│ 8  │ Aparajito        │ 1         │
└────┴──────────────────┴───────────┘
```

Orphan Book automatic ভাবে delete হয়ে গেল। ✅

### `CascadeType.REMOVE` এর সাথে পার্থক্য

দুইটাই delete করে — কিন্তু trigger আলাদা।

| | `CascadeType.REMOVE` | `orphanRemoval = true` |
|---|---|---|
| Trigger | Parent delete হলে child delete | Parent থেকে association cut হলে child delete |
| Parent delete করলে | Child delete ✅ | Child delete ✅ |
| শুধু list থেকে remove করলে | Child delete হয় না ❌ | Child delete হয় ✅ |

### কখন use করবো?

- যখন child parent ছাড়া বাঁচার কোনো মানে হয় না
- যেমন: Order এবং OrderItems — Order থেকে Item remove করলে সেটা delete হওয়া উচিত
- যেমন: Post এবং Comments — Post থেকে Comment remove করলে সেটাও delete হওয়া উচিত

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items;
```

দুইটা একসাথে — সবচেয়ে robust setup।

### Common Mistake

```java
@OneToMany(mappedBy = "author", orphanRemoval = true)
private List<Book> books;
```

```java
author.setBooks(null);    // ⚠️ সব Books একসাথে orphan!
authorRepository.save(author);
// সব Books delete হয়ে গেল!
```

সাবধানে use করো।

---

<a name="পর্ব-৬"></a>

## পর্ব ৬ — চূড়ান্ত সারসংক্ষেপ

### ৪টা concept এর Quick Overview

| Concept | কী করে | কখন দরকার |
|---|---|---|
| **Cascade** | Parent এর operation Child এ পৌঁছায় | Parent-Child tightly coupled |
| **Fetch Type** | Related data কখন load হবে | Performance control |
| **N+1 Fix** | Query কমানো | Loop এ related data access |
| **Infinite Recursion Fix** | JSON এর loop থামানো | Bidirectional relationship |
| **orphanRemoval** | অনাথ Child delete | Collection থেকে remove → delete |

### Master Setup (Production-ready)

```java
@Entity
@Table(name = "authors")
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(
        mappedBy = "author",
        cascade = CascadeType.ALL,              // Parent এর সব operation cascade
        orphanRemoval = true,                   // Orphan Books delete
        fetch = FetchType.LAZY                  // Performance এর জন্য LAZY
    )
    @JsonManagedReference                       // Infinite recursion prevent
    private List<Book> books = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "books")
public class Book {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToOne(fetch = FetchType.LAZY)          // LAZY এখানেও
    @JoinColumn(name = "author_id", nullable = false)
    @JsonBackReference                          // Back reference
    private Author author;
}
```

```java
public interface AuthorRepository extends JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = {"books"})    // N+1 prevent
    List<Author> findAll();
}
```

### Decision Flow

```
Parent-Child tightly coupled?
├─ YES → cascade = CascadeType.ALL + orphanRemoval = true
└─ NO  → শুধু PERSIST বা MERGE

Collection আছে?
├─ YES → fetch = LAZY (default, রাখো)
└─ NO  → fetch = EAGER (যদি সবসময় দরকার হয়)

Loop এ related data access হবে?
├─ YES → @EntityGraph বা JOIN FETCH use করো
└─ NO  → plain query ok

API এ Entity return করছো?
├─ YES → DTO এ convert করো (best)
└─ NO  → @JsonManagedReference/@JsonBackReference চলবে
```

### পাঁচটা Golden Rule

```
1. Collection field = FetchType.LAZY
   → EAGER করলে performance হত্যা

2. Bidirectional হলে Owner side চিনো (যেখানে @JoinColumn)
   → Inverse side এ mappedBy দাও

3. Parent-Child strongly linked হলে CascadeType.ALL + orphanRemoval
   → সবসময় দুইটা একসাথে

4. Loop এ related data access = N+1 এর জন্য ready থাকো
   → @EntityGraph বা JOIN FETCH use করো

5. API তে Entity না, DTO return করো
   → Infinite recursion + security + flexibility সবই solve
```

---

## শেষ কথা

Spring Data JPA এর basic relationships শেখা = ঘরের বাহির থেকে রং করা।
**এই Advanced concepts শেখা = ঘরের ভেতরের wiring ঠিক করা।**

এই ৫টা concept ছাড়া production-grade code লেখা যায় না। এগুলোই বাস্তবে bug, performance issue, crash তৈরি করে।

পরবর্তী step এ চাইলে — **JPA Specifications**, **Criteria API**, বা **Transactional behavior** নিয়ে যেতে পারি। 🚀

Happy coding!
