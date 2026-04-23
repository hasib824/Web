# Spring Data JPA Relationships — Complete Beginner Guide

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Level:** Beginner থেকে Intermediate
> **Topics:** `@Entity`, `@Id`, `@GeneratedValue`, `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@JoinColumn`, `@JoinTable`, `mappedBy`

---

## সূচিপত্র

1. [Foundation — কিছু মৌলিক জিনিস আগে](#পর্ব-০)
2. [কেন Relationship Annotations দরকার?](#পর্ব-১)
3. [`@ManyToOne` — FK এর আসল ঘর](#পর্ব-২)
4. [`@JoinColumn` — গভীরে](#পর্ব-৩)
5. [`@OneToMany` — Reverse Side এর জাদু](#পর্ব-৪)
6. [`mappedBy` — পুরোপুরি Decode](#পর্ব-৫)
7. [Bidirectional vs Unidirectional](#পর্ব-৬)
8. [`@OneToOne`](#পর্ব-৭)
9. [`@ManyToMany` — Join Table কেন দরকার?](#পর্ব-৮)
10. [Common Mistakes](#পর্ব-৯)
11. [সারসংক্ষেপ](#পর্ব-১০)
12. [পরবর্তী পদক্ষেপ](#পর্ব-১১)

---

<a name="পর্ব-০"></a>

## পর্ব ০ — Foundation (এগুলো আগে বুঝতেই হবে)

Relationship এ যাওয়ার আগে কয়েকটা basic annotation বুঝে নাও। এগুলো ছাড়া কিছুই বুঝবে না।

### `@Entity` কী?

```java
@Entity
public class Author {
    // ...
}
```

`@Entity` মানে Hibernate কে বলছো —
> *"এই Java class টা আসলে database এর একটা table represent করে। এটাকে track করো।"*

এটা না দিলে Hibernate এই class টাকে চিনবেই না।

### `@Table` কী?

```java
@Entity
@Table(name = "authors")
public class Author {
    // ...
}
```

`@Table(name = "authors")` মানে —
> *"এই Entity এর জন্য database এ যে table তৈরি হবে, তার নাম হবে `authors`।"*

**না দিলে কী হয়?** Hibernate default এ class এর নাম দিয়ে table বানায় — `Author`। কিন্তু এটা inconsistent (case-sensitive issue)। তাই `@Table` দিয়ে explicit করে দেয়া best practice।

### `@Id` কী?

```java
@Id
private Long id;
```

`@Id` মানে —
> *"এই field টা database এর Primary Key (PK)। প্রতিটা row কে unique ভাবে identify করবে।"*

প্রতিটা Entity তে অবশ্যই একটা `@Id` থাকতে হবে — না দিলে error!

### `@GeneratedValue` কী?

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

`@GeneratedValue` মানে —
> *"এই id টা আমি manually দেবো না — database নিজে auto-generate করুক।"*

`GenerationType.IDENTITY` মানে database এর `AUTO_INCREMENT` feature ব্যবহার করো। প্রতিবার নতুন row insert হলে id automatically 1, 2, 3... হবে।

### একটা Complete Entity দেখি

```java
@Entity                              // Hibernate কে বলছি — এটা একটা database table
@Table(name = "authors")             // Table এর নাম হবে "authors"
public class Author {

    @Id                              // এটা Primary Key
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // Auto-increment
    private Long id;

    private String name;             // সাধারণ column

    // getters, setters...
}
```

Hibernate এই class দেখে database এ এই SQL run করবে —

```sql
CREATE TABLE authors (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);
```

এই foundation টা পাকা হলে এবার Relationship এ যাই।

---

<a name="পর্ব-১"></a>

## পর্ব ১ — কেন Relationship Annotations দরকার?

Real world এ সব কিছু একে অন্যের সাথে সম্পর্কযুক্ত —

- একজন **Author** অনেক **Book** লেখে
- একজন **User** এর একটাই **Profile**
- একজন **Student** অনেক **Course** নেয়, একটা **Course** এ অনেক **Student** থাকে

Database এ এই সম্পর্ক রাখা হয় **Foreign Key (FK)** দিয়ে। JPA এর relationship annotations এই FK based সম্পর্ক গুলোকে Java এ object হিসেবে দেখায়।

### Foreign Key (FK) আসলে কী?

ধরো `books` table এ একটা column আছে `author_id`। এই `author_id` column এর value হলো অন্য table (`authors`) এর কোনো row এর `id`। মানে —
> *"এই Book টা ওই Author এর — যার id হলো `author_id` এর value।"*

এই connecting column টাই **Foreign Key**।

### চারটা Relationship Type একনজরে

| Annotation | কখন ব্যবহার | উদাহরণ |
|---|---|---|
| `@OneToOne` | একটার সাথে একটা | User ↔ Profile |
| `@OneToMany` | একটার সাথে অনেক | Author → Books |
| `@ManyToOne` | অনেকের সাথে একটা | Books → Author |
| `@ManyToMany` | অনেকের সাথে অনেক | Students ↔ Courses |

**আজকের প্রধান Example:** Author এবং Book — একজন Author অনেক Book লেখে।

---

<a name="পর্ব-২"></a>

## পর্ব ২ — `@ManyToOne` (FK এর আসল ঘর)

### কেন `@ManyToOne` দিয়ে শুরু?

কারণ FK সবসময় **"Many" side এ বসে।** এটা মৌলিক একটা rule। চলো বুঝি কেন।

### "FK Many side এ বসে" — কেন?

ধরো আমি FK রাখি **Author এর side এ।** তাহলে `authors` table কেমন হবে?

```
authors table (FK Author side এ থাকলে):
┌──────────┬──────────┬───────────────────────────┐
│ id (PK)  │ name     │ book_id (FK)              │
├──────────┼──────────┼───────────────────────────┤
│ 1        │ Hasib    │ ???                       │
└──────────┴──────────┴───────────────────────────┘
```

কিন্তু Hasib তো **অনেক Book লিখেছে** — 5, 8, 12 ... একটা column এ এতগুলো id রাখা যায় না!

তাই FK রাখতে হবে সেই side এ যেখানে **এক single reference** আছে — মানে **Book এর side এ** —

```
books table (FK Book side এ থাকলে — সঠিক!):
┌──────────┬──────────────────┬──────────────────┐
│ id (PK)  │ title            │ author_id (FK)   │
├──────────┼──────────────────┼──────────────────┤
│ 5        │ Pother Pachali   │ 1                │
│ 8        │ Aparajito        │ 1                │
│ 12       │ Devdas           │ 2                │
└──────────┴──────────────────┴──────────────────┘
```

প্রতিটা Book এর **একজনই** Author — তাই `author_id` column এ এক single value রাখা সম্ভব। ✅

> **মূল কথা:** যে side এ "Many" থাকে (অনেক rows), সেই side এ FK থাকে। প্রতিটা Many row জানে সে কোন One এর সাথে যুক্ত।

### এবার Code

```java
@Entity
@Table(name = "authors")
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // এখনো কোনো relationship annotation নেই — আমরা পরে যোগ করবো
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

    @ManyToOne                           // ⭐ এই Book টার একজন Author
    @JoinColumn(name = "author_id")      // ⭐ FK column এর নাম দিচ্ছি
    private Author author;
}
```

### এই দুই annotation এর Breakdown

#### `@ManyToOne` কী বলে?

> *"আমি (Book) Many side এ আছি। আমার একজন Author আছে। তাই আমার table এ FK column থাকবে।"*

#### `@JoinColumn(name = "author_id")` কী বলে?

> *"আমার table এ যে FK column টা থাকবে, তার নাম দাও `author_id`।"*

### Hibernate এখন কী ভাবে?

```
1. Book class এ @Entity পেলাম → "books" table তৈরি করি
2. Book এর ভেতরে দেখলাম private Author author
3. @ManyToOne দেখলাম → "এটা একটা relationship, FK লাগবে"
4. @JoinColumn(name = "author_id") দেখলাম → "FK column এর নাম author_id"
5. Author class দেখলাম → "ওটার Primary Key field টা long id"
6. তাহলে FK column হবে: author_id BIGINT, references authors(id)
```

### Database এ যে Tables তৈরি হবে

```
┌─────────────────────┐                ┌──────────────────────────┐
│       authors       │                │          books            │
├─────────────────────┤                ├──────────────────────────┤
│ id (PK)             │◄───────────────│ author_id (FK)  ⭐        │
│ name                │                │ id (PK)                  │
└─────────────────────┘                │ title                    │
                                       └──────────────────────────┘
```

### Hibernate এর Generated SQL

```sql
CREATE TABLE authors (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE books (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    title     VARCHAR(255),
    author_id BIGINT,                               -- ⭐ FK column
    FOREIGN KEY (author_id) REFERENCES authors(id)  -- ⭐ FK constraint
);
```

### JSON Input/Output

**Book তৈরির Request (POST `/books`):**

```json
{
  "title": "Pother Pachali",
  "author": {
    "id": 1
  }
}
```

কেন শুধু `id` দিলে কাজ করে? কারণ Hibernate জানে — Book table এ গিয়ে যে FK save করতে হবে সেটা হলো author এর `id`। বাকি info (name) সে database থেকে নিজেই query করে নেবে।

**Response:**

```json
{
  "id": 5,
  "title": "Pother Pachali",
  "author": {
    "id": 1,
    "name": "Bibhutibhushan Bandyopadhyay"
  }
}
```

> **লক্ষ করো —** JSON এ পুরো `author` object nested আকারে আসে, কিন্তু database এ আসলে শুধু `author_id = 1` save হয়েছে। Hibernate দরকার মতো Author table থেকে Author এর সব info join করে nested object বানিয়ে দেয়।

---

<a name="পর্ব-৩"></a>

## পর্ব ৩ — `@JoinColumn` — গভীরে

`@JoinColumn` শুধু একটা কাজ করে — **FK column এর নাম এবং properties define করে।**

### না দিলে কী হয়?

Hibernate default নাম generate করে এই formula দিয়ে —

```
{field_name}_{referenced_PK_name}
```

```java
@ManyToOne
private Author author;   // @JoinColumn দিইনি!
```

Hibernate ভাববে —
- Field name = `author`
- Referenced class এর PK = `id`
- তাহলে FK column এর নাম হবে = `author_id`

So default এও কাজ করে। তবু **explicitly দেয়া best practice** — কারণ:
1. Schema পরিষ্কার থাকে
2. Future এ field name বদলালে FK column বদলায় না
3. Code reviewer-রা সহজে বোঝে

### Custom নাম দিতে চাইলে

```java
@ManyToOne
@JoinColumn(name = "writer_fk")   // FK column হবে "writer_fk"
private Author author;
```

এখন `books` table এ column টার নাম হবে `writer_fk` — `author_id` না।

### `@JoinColumn` এর Important Properties

| Property | কাজ | Example |
|---|---|---|
| `name` | FK column এর নাম | `name = "author_id"` |
| `nullable` | FK null হতে পারবে কি না | `nullable = false` |
| `unique` | FK unique হবে কি না | `unique = true` |
| `referencedColumnName` | কোন column কে reference করবে | `referencedColumnName = "id"` |

### Practical Example

```java
@ManyToOne(optional = false)
@JoinColumn(
    name = "author_id",
    nullable = false,                    // FK null হতে পারবে না
    referencedColumnName = "id"          // Author table এর "id" column কে reference করছে
)
private Author author;
```

Generated SQL —

```sql
author_id BIGINT NOT NULL,
FOREIGN KEY (author_id) REFERENCES authors(id)
```

---

<a name="পর্ব-৪"></a>

## পর্ব ৪ — `@OneToMany` — Reverse Side এর জাদু

এখন Author থেকে তার সব Books দেখতে চাইলে কী করবো? `@OneToMany` add করি Author class এ।

### Author Entity Update

```java
@Entity
@Table(name = "authors")
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "author")      // ⭐ নতুন যোগ করলাম
    private List<Book> books = new ArrayList<>();
}
```

### এখন Database এ কী পরিবর্তন হলো?

> ❗ **কিছুই না!**

`@OneToMany` কোনো **নতুন column বা table তৈরি করে না।** এটা শুধু Java Object level এ কাজ করে।

`authors` table এখনো এরকমই —

```
┌──────────┬──────────┐
│ id (PK)  │ name     │
└──────────┴──────────┘
```

`books` table এখনো এরকমই —

```
┌──────────┬──────────┬──────────────────┐
│ id (PK)  │ title    │ author_id (FK)   │
└──────────┴──────────┴──────────────────┘
```

### তাহলে `@OneToMany` কী করে?

এটা শুধু Java Object level এ একটা সুবিধা দেয় — তুমি যখন Author load করবে, Hibernate automatic ভাবে তার সব Books-ও query করে list এ ভরে দেবে।

```java
Author author = authorRepository.findById(1L).get();
List<Book> hisBooks = author.getBooks();   // ✅ List ready!
```

পর্দার পেছনে Hibernate এই query চালায় —

```sql
SELECT * FROM books WHERE author_id = 1;
```

---

<a name="পর্ব-৫"></a>

## পর্ব ৫ — `mappedBy` — পুরোপুরি Decode

এই অংশটা সবচেয়ে important। `mappedBy` কীভাবে কাজ করে সেটা একদম step by step বুঝি।

### `mappedBy = "author"` লিখে আমি কী বললাম?

```java
@OneToMany(mappedBy = "author")
private List<Book> books;
```

আমি Hibernate কে বললাম —

> *"আমি (Author এর `books` field) এই relationship এর owner না। আসল owner হলো **Book class এর `author` field**। সেটাকেই দেখো।"*

### Hibernate এই field খুঁজে পায় কী করে?

দুইটা step এ —

#### Step 1: কোন Class এ যাবো? — Generic Type থেকে বুঝে নেয়

```java
@OneToMany(mappedBy = "author")
private List<Book> books;       // ⭐ এখানে <Book> লেখা আছে
```

Hibernate চিন্তা করে —

```
1. List<Book> দেখলাম
2. তাহলে opposite class হলো → Book
3. ঠিক আছে, Book class এ যাবো
```

#### Step 2: Class এর কোন Field? — `mappedBy` এর string দেখে

```java
@OneToMany(mappedBy = "author")   // "author" = Book class এর field name
```

Hibernate Book class এ গিয়ে চিন্তা করে —

```
4. Book class এ আসলাম
5. "author" নামের কোনো field আছে কি দেখি?
6. পেলাম: private Author author; ✅
7. সেই field এ @JoinColumn(name = "author_id") আছে
8. So FK column হলো author_id, যা books table এ আছে
9. বুঝলাম — books.author_id দিয়ে relationship manage হচ্ছে!
```

### একটা Visual

```
Author class:
┌─────────────────────────────────────────┐
│ @OneToMany(mappedBy = "author")         │
│ private List<Book> books;               │
│           │                             │
│           │ Step 1: <Book> থেকে         │
│           │ বুঝলাম — Book class এ যাবো │
│           ▼                             │
└───────────┼─────────────────────────────┘
            │
            │ Step 2: "author" field খুঁজি
            ▼
Book class:
┌─────────────────────────────────────────┐
│ @ManyToOne                              │
│ @JoinColumn(name = "author_id")         │
│ private Author author;  ◄── পেলাম!     │
└─────────────────────────────────────────┘
```

### নাম মিলতেই হবে — ভুল হলে কী হবে?

```java
// Book.java
private Author author;             // field এর নাম "author"

// Author.java
@OneToMany(mappedBy = "author")    // ✅ same name — কাজ করবে
@OneToMany(mappedBy = "writer")    // ❌ "writer" নামের field Book এ নেই — error!
```

### Type ভুল দিলে কী হবে?

```java
@OneToMany(mappedBy = "author")
private List<Course> books;        // ❌ <Course> দিলে Hibernate Course class এ খুঁজবে
                                   //    Course এ "author" নেই → error!
```

### এক লাইনে মনে রাখো

> Hibernate **`List<Book>` এর `<Book>`** দেখে বুঝে — *"Book class এ যাবো।"*
> তারপর **`mappedBy = "author"`** দেখে বুঝে — *"সেখানে `author` field খুঁজবো।"*

---

<a name="পর্ব-৬"></a>

## পর্ব ৬ — Bidirectional vs Unidirectional

### Unidirectional (একদিকের relationship)

শুধু একটা side এ relationship define করা।

```java
// Book class এ আছে
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;

// Author class এ books এর কোনো reference নেই
public class Author {
    private Long id;
    private String name;
}
```

**সুবিধা:** Simple, কম complexity।
**অসুবিধা:** Author থেকে তার Books পেতে পারবে না Java code এ — আলাদা query লিখতে হবে।

### Bidirectional (দুইদিকের relationship)

দুইটা side এই relationship define করা।

```java
// Book class
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;

// Author class
@OneToMany(mappedBy = "author")
private List<Book> books = new ArrayList<>();
```

**সুবিধা:** দুই দিক থেকেই navigate করা যায়।
**অসুবিধা:** JSON serialize করার সময় infinite loop হতে পারে (এটা পরে সমাধান করবো)।

### Owner vs Inverse Side

Bidirectional এ একটা important concept — **এক side হবে owner, অন্য side হবে inverse।**

| Side | কীভাবে চিনবো? | কাজ |
|---|---|---|
| **Owner side** | যে side এ FK আছে (`@JoinColumn`) | FK manage করে |
| **Inverse side** | যে side এ `mappedBy` আছে | শুধু read এর জন্য |

আমাদের example এ —
- **Owner:** Book (এখানে FK column `author_id` আছে)
- **Inverse:** Author (এখানে শুধু `mappedBy`)

Database update সবসময় **Owner side থেকেই** হয়। Inverse side এ change করলে database এ সেটা reflect হয় না!

---

<a name="পর্ব-৭"></a>

## পর্ব ৭ — `@OneToOne`

প্রতিটা User এর একটাই Profile — এই scenario তে `@OneToOne` ব্যবহার করো।

### Code

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String email;

    @OneToOne
    @JoinColumn(name = "profile_id")    // ⭐ FK + UNIQUE constraint
    private Profile profile;
}
```

```java
@Entity
@Table(name = "profiles")
public class Profile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String bio;
}
```

### Database Structure

```
users table:                          profiles table:
┌──────────────────────────────┐      ┌─────────────────┐
│ id (PK)                      │      │ id (PK)         │
│ email                        │      │ bio             │
│ profile_id (FK, UNIQUE)  ⭐  │─────►│                 │
└──────────────────────────────┘      └─────────────────┘
```

### Generated SQL

```sql
CREATE TABLE profiles (
    id  BIGINT AUTO_INCREMENT PRIMARY KEY,
    bio VARCHAR(255)
);

CREATE TABLE users (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    email      VARCHAR(255),
    profile_id BIGINT UNIQUE,                          -- ⭐ UNIQUE constraint!
    FOREIGN KEY (profile_id) REFERENCES profiles(id)
);
```

### `@OneToOne` কেন `@ManyToOne` থেকে আলাদা?

দুইটাই FK তৈরি করে। কিন্তু পার্থক্য হলো —

| | `@ManyToOne` | `@OneToOne` |
|---|---|---|
| FK column এ UNIQUE? | না | ✅ হ্যাঁ |
| অনেক row একই FK value share করতে পারে? | হ্যাঁ | ❌ না |

মানে — দুইজন User কখনো একই Profile share করতে পারবে না, কারণ `profile_id` UNIQUE।

### Bidirectional `@OneToOne`

Profile থেকেও User access করতে চাইলে —

```java
@Entity
public class Profile {
    @Id
    @GeneratedValue
    private Long id;

    private String bio;

    @OneToOne(mappedBy = "profile")    // ⭐ User এর "profile" field কে point করছে
    private User user;
}
```

Database এ কোনো change নেই — শুধু Java level এ navigate করা যায়।

---

<a name="পর্ব-৮"></a>

## পর্ব ৮ — `@ManyToMany` — Join Table কেন দরকার?

একজন Student অনেক Course নেয়, একটা Course এ অনেক Student থাকে।

### আগে বুঝি — কেন এখানে Join Table দরকার?

আমরা শিখেছি FK রাখতে হয় "Many" side এ। কিন্তু এখানে দুই side ই Many!

```
দুই side ই many — FK কোথায় রাখবো?

students:                        courses:
- id                             - id
- name                           - title
- course_id?? ← অনেক course      - student_id?? ← অনেক student
                                   একসাথে কীভাবে রাখবো?
```

কোনো single column এ list রাখা যায় না। তাই middle এ একটা **third table** দরকার, যেটা **শুধু দুই দিকের id ধরে রাখবে।** এই third table কেই বলে **Join Table**।

```
┌──────────┐    ┌──────────────────────┐    ┌──────────┐
│ students │    │   student_courses    │    │ courses  │
├──────────┤    ├──────────────────────┤    ├──────────┤
│ id       │◄───│ student_id (FK)      │    │ id       │
│ name     │    │ course_id  (FK)   ───┼───►│ title    │
└──────────┘    └──────────────────────┘    └──────────┘
                       (JOIN TABLE)
```

প্রতিটা row একটা enrollment — কোন student কোন course এ আছে।

### Code

```java
@Entity
@Table(name = "students")
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_courses",                              // ⭐ join table এর নাম
        joinColumns = @JoinColumn(name = "student_id"),        // ⭐ এই side এর FK
        inverseJoinColumns = @JoinColumn(name = "course_id")   // ⭐ অন্য side এর FK
    )
    private List<Course> courses = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "courses")
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")    // ⭐ Student এ already define করা আছে
    private List<Student> students = new ArrayList<>();
}
```

### `@JoinTable` এর Breakdown

```java
@JoinTable(
    name = "student_courses",                              // join table এর নাম
    joinColumns = @JoinColumn(name = "student_id"),        // এই Entity এর FK
    inverseJoinColumns = @JoinColumn(name = "course_id")   // অন্য Entity এর FK
)
```

| Property | কী বলে |
|---|---|
| `name` | তৈরি করা join table এর নাম |
| `joinColumns` | যে Entity তে এই annotation আছে, সেটার FK column এর নাম |
| `inverseJoinColumns` | অপর side এর Entity এর FK column এর নাম |

### Generated SQL

```sql
CREATE TABLE students (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE courses (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255)
);

CREATE TABLE student_courses (              -- ⭐ Auto-created join table
    student_id BIGINT,
    course_id  BIGINT,
    PRIMARY KEY (student_id, course_id),    -- দুইটা মিলে composite PK
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id)  REFERENCES courses(id)
);
```

### Sample Data

**students table:**

| id | name |
|----|------|
| 1  | Hasib |
| 2  | Karim |

**courses table:**

| id | title |
|----|-------|
| 10 | Spring Boot |
| 20 | Kotlin |

**student_courses table (join):**

| student_id | course_id |
|------------|-----------|
| 1          | 10        |
| 1          | 20        |
| 2          | 10        |

> Hasib দুইটা course এ আছে (Spring Boot, Kotlin), Karim শুধু Spring Boot এ।

### JSON Output

```json
{
  "id": 1,
  "name": "Hasib",
  "courses": [
    { "id": 10, "title": "Spring Boot" },
    { "id": 20, "title": "Kotlin" }
  ]
}
```

---

<a name="পর্ব-৯"></a>

## পর্ব ৯ — Common Mistakes

### Mistake 1: `mappedBy` ভুলে যাওয়া

```java
// Author class
@OneToMany   // ❌ mappedBy দাওনি!
private List<Book> books;

// Book class
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;
```

**কী হবে?** Hibernate ভাববে দুইটা আলাদা relationship। একটা **extra join table** তৈরি করে ফেলবে — `authors_books`। সম্পূর্ণ schema নষ্ট!

**সমাধান:** Owner side ছাড়া অন্য side এ অবশ্যই `mappedBy` দাও।

### Mistake 2: দুই side কেই Owner বানানো

```java
// Author class
@OneToMany
@JoinColumn(name = "author_id")    // ❌ এখানেও @JoinColumn!
private List<Book> books;

// Book class
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;
```

**কী হবে?** Confusing schema — Hibernate কোনটা manage করবে বুঝতে পারে না।

**সমাধান:** Owner side এ `@JoinColumn`, Inverse side এ `mappedBy`।

### Mistake 3: Inverse side থেকে save করা

```java
Author author = new Author("Hasib");
Book book = new Book("Pother Pachali");

author.getBooks().add(book);    // ❌ শুধু Inverse side এ add করলে save হবে না!
authorRepository.save(author);
```

**কী হবে?** Database এ Book এর `author_id` null হয়ে save হবে।

**সমাধান:** Owner side থেকে set করো —

```java
book.setAuthor(author);          // ✅ Owner side এ set করলাম
bookRepository.save(book);
```

বা দুই side ই sync করে দাও (helper method দিয়ে) —

```java
public void addBook(Book book) {
    books.add(book);
    book.setAuthor(this);        // ✅ দুই side ই update
}
```

### Mistake 4: `@Entity` ভুলে যাওয়া

```java
public class Author {            // ❌ @Entity নেই
    @Id
    private Long id;
}
```

**কী হবে?** Hibernate এই class কে চিনবেই না। Repository inject করার সময় error।

**সমাধান:** প্রতিটা database table এর জন্য তৈরি class এ `@Entity` দাও।

---

<a name="পর্ব-১০"></a>

## পর্ব ১০ — সারসংক্ষেপ

### কোনটায় কী তৈরি হয়?

| Annotation | নতুন Table? | নতুন Column? | কোথায় বসে |
|---|---|---|---|
| `@ManyToOne` | ❌ না | ✅ FK column | "Many" side এ |
| `@OneToMany` | ❌ না | ❌ না | "One" side এ (mappedBy দিয়ে) |
| `@OneToOne` | ❌ না | ✅ FK column (UNIQUE) | যেকোনো এক side এ |
| `@ManyToMany` | ✅ হ্যাঁ — join table! | ❌ না (মূল table এ) | যেকোনো এক side এ owner |

### তিনটা Golden Rule

```
1. FK সবসময় "Many" side এ বসে
   → @ManyToOne এর সাথে @JoinColumn দাও

2. @OneToMany এর সাথে mappedBy অবশ্যই দাও
   → না দিলে extra join table তৈরি হয়ে যাবে

3. @ManyToMany ই একমাত্র যেটা নতুন table তৈরি করে
   → অন্য কোনোটা না!
```

### Quick Cheat Sheet

```java
// ManyToOne — এক Book এর এক Author
@ManyToOne
@JoinColumn(name = "author_id")
private Author author;

// OneToMany — এক Author এর অনেক Book
@OneToMany(mappedBy = "author")
private List<Book> books = new ArrayList<>();

// OneToOne — এক User এর এক Profile
@OneToOne
@JoinColumn(name = "profile_id")
private Profile profile;

// ManyToMany — অনেক Student অনেক Course
@ManyToMany
@JoinTable(
    name = "student_courses",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
private List<Course> courses = new ArrayList<>();
```

---

<a name="পর্ব-১১"></a>

## পর্ব ১১ — পরবর্তী পদক্ষেপ

এই tutorial শেষ করার পরে এগুলো শিখলে concept আরও পাকা হবে —

### 1. Cascade Types

Parent save/delete করলে Child এর কী হবে?

```java
@OneToMany(mappedBy = "author", cascade = CascadeType.ALL)
private List<Book> books;
```

`CascadeType.ALL` মানে — Author save করলে তার Books-ও auto-save হবে। Author delete করলে Books-ও delete হবে।

### 2. Fetch Types — LAZY vs EAGER

```java
@OneToMany(mappedBy = "author", fetch = FetchType.LAZY)   // Default for collections
private List<Book> books;

@ManyToOne(fetch = FetchType.EAGER)                       // Default for single ref
private Author author;
```

- **LAZY:** Books দরকার হলে তখন query হবে।
- **EAGER:** Author load করার সময়ই Books-ও load হবে।

### 3. N+1 Query Problem

LAZY এর কারণে অনেক বেশি query হয়ে যাওয়ার সমস্যা — `JOIN FETCH` দিয়ে সমাধান।

### 4. Bidirectional Infinite Recursion

JSON serialize করার সময় Author → Book → Author → Book ... infinite loop হয়। সমাধান — `@JsonManagedReference`, `@JsonBackReference` বা DTO ব্যবহার।

### 5. orphanRemoval

```java
@OneToMany(mappedBy = "author", orphanRemoval = true)
private List<Book> books;
```

Author এর list থেকে কোনো Book remove করলে সেটা database থেকেও delete হবে।

---

## শেষ কথা

এই Relationship annotations Spring Boot এর backbone। একবার পরিষ্কারভাবে বুঝে গেলে যেকোনো complex schema build করতে পারবে।

মনে রাখার মূল মন্ত্র —

> 🎯 **FK কোথায় থাকবে — সেটা আগে ভাবো। সব decision তার উপর নির্ভর করে।**

Happy coding! 🚀
