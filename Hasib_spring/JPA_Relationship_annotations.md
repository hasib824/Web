# Spring Data JPA Relationship Annotations — সম্পূর্ণ Tutorial

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Level:** Intermediate
> **Topics:** `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@JoinColumn`, `@JoinTable`, `mappedBy`

---

## এই Annotations গুলো কেন দরকার?

Real world এ সব কিছু একে অন্যের সাথে সম্পর্কযুক্ত —

- একজন **Author** অনেক **Book** লেখে
- একজন **User** এর একটাই **Profile**
- একজন **Student** অনেক **Course** নেয়, একটা **Course** এ অনেক **Student** থাকে

Database এও এই সম্পর্কগুলো রাখতে হয় (foreign key দিয়ে)। JPA এর relationship annotations এই সম্পর্ক গুলোকে Java Object এর মাধ্যমে express করতে দেয়।

---

## চারটা Relationship Type একনজরে

| Annotation | কখন ব্যবহার | উদাহরণ |
|---|---|---|
| `@OneToOne` | একটার সাথে একটা | User ↔ Profile |
| `@OneToMany` | একটার সাথে অনেক | Author → Books |
| `@ManyToOne` | অনেকের সাথে একটা | Books → Author |
| `@ManyToMany` | অনেকের সাথে অনেক | Students ↔ Courses |

**আজকের Example:** Author এবং Book (একজন Author অনেক Book লেখে)

---

## পর্ব ১ — `@ManyToOne` (এখান থেকে শুরু করো!)

**কেন এখান থেকে শুরু?**
Foreign Key (FK) সবসময় "Many" side এ বসে। অনেক Book এর একজন Author — তাই `books` table এ `author_id` column টা থাকবে।

### Entity Code

```java
@Entity
@Table(name = "authors")
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // এখন কোনো relationship annotation নেই
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

### Database এ তৈরি হওয়া Tables

```
┌─────────────────────┐         ┌──────────────────────────┐
│       authors       │         │          books            │
├─────────────────────┤         ├──────────────────────────┤
│ id (PK)             │◄────────│ author_id (FK)  ⭐        │
│ name                │         │ id (PK)                  │
└─────────────────────┘         │ title                    │
                                └──────────────────────────┘
```

### Hibernate এর তৈরি SQL

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

> **লক্ষ করো —** JSON এ পুরো `author` object nested আকারে আসে, কিন্তু database এ শুধু `author_id` column হিসেবে save হয়। এটাই Hibernate এর জাদু! ✨

---

## পর্ব ২ — `@OneToMany` (Reverse Side)

Author থেকে তার সব Books দেখতে চাইলে `@OneToMany` ব্যবহার করো।

### Author Entity Update

```java
@Entity
@Table(name = "authors")
public class Author {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    @OneToMany(mappedBy = "author")      // ⭐ "author" field টা Book entity তে আছে
    private List<Book> books = new ArrayList<>();
}
```

### এখন Database এ কী পরিবর্তন হলো?

> ❗ **কিছুই না!**

`@OneToMany` কোনো **নতুন column বা table তৈরি করে না।** এটা শুধু Java এর দিকে বলে —
*"আমি যখন Author load করবো, তার সব Books ও সাথে নিয়ে আসবো।"*

Database এ আগের মতই দুইটা table — `authors` এবং `books` (with `author_id` FK)।

### `mappedBy` মানে কী?

`mappedBy = "author"` মানে —

> *"আমি এই relationship এর owner না। আসল owner হলো Book class এর `author` field। সেটাকেই দেখো।"*

❌ **এটা না দিলে** Hibernate ভাববে দুইটা আলাদা relationship — তখন একটা **extra join table** তৈরি করে ফেলবে!

### Author এর JSON Output

```json
{
  "id": 1,
  "name": "Bibhutibhushan Bandyopadhyay",
  "books": [
    { "id": 5, "title": "Pother Pachali" },
    { "id": 8, "title": "Aparajito" }
  ]
}
```

---

## পর্ব ৩ — `@JoinColumn` কী এবং কেন?

`@JoinColumn` বলে দেয় — **FK column এর নাম কী হবে।**

```java
@ManyToOne
@JoinColumn(name = "author_id")   // ⭐ FK column এর নাম "author_id"
private Author author;
```

### না দিলে কী হয়?

Hibernate default নাম generate করে — `author_id` (field name + `_id`)।
তবু explicitly দেয়া ভালো practice — কারণ table schema পরিষ্কার থাকে।

### Custom নাম চাইলে?

```java
@ManyToOne
@JoinColumn(name = "writer_fk")   // FK column হবে "writer_fk"
private Author author;
```

### `@JoinColumn` এর Important Properties

| Property | কাজ | Example |
|---|---|---|
| `name` | FK column এর নাম | `name = "author_id"` |
| `nullable` | FK null হতে পারবে কি | `nullable = false` |
| `unique` | FK unique হবে কি | `unique = true` |
| `referencedColumnName` | কোন column কে reference করবে | `referencedColumnName = "id"` |

---

## পর্ব ৪ — `@OneToOne` (সংক্ষেপে)

প্রতিটা User এর একটাই Profile —

```java
@Entity
public class User {
    @Id @GeneratedValue
    private Long id;

    private String email;

    @OneToOne
    @JoinColumn(name = "profile_id")   // ⭐ FK এবং UNIQUE constraint
    private Profile profile;
}

@Entity
public class Profile {
    @Id @GeneratedValue
    private Long id;

    private String bio;
}
```

### Table Structure

```
users table:                       profiles table:
┌─────────────────────────┐       ┌─────────────────┐
│ id (PK)                 │       │ id (PK)         │
│ email                   │       │ bio             │
│ profile_id (FK, UNIQUE) │──────►│                 │
└─────────────────────────┘       └─────────────────┘
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

> `@OneToOne` মূলত `@ManyToOne` এর মতই — তবে FK column এ **`UNIQUE` constraint** থাকে, যাতে দুইজন User একই Profile share না করতে পারে।

---

## পর্ব ৫ — `@ManyToMany` (এখানেই Join Table আসে!)

একজন Student অনেক Course নেয়, একটা Course এ অনেক Student থাকে।

### Entity Code

```java
@Entity
@Table(name = "students")
public class Student {
    @Id @GeneratedValue
    private Long id;

    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_courses",                            // ⭐ join table name
        joinColumns = @JoinColumn(name = "student_id"),      // ⭐ this side FK
        inverseJoinColumns = @JoinColumn(name = "course_id") // ⭐ other side FK
    )
    private List<Course> courses = new ArrayList<>();
}
```

```java
@Entity
@Table(name = "courses")
public class Course {
    @Id @GeneratedValue
    private Long id;

    private String title;

    @ManyToMany(mappedBy = "courses")   // ⭐ Student এ already define করা আছে
    private List<Student> students = new ArrayList<>();
}
```

### Database এ তিনটা Table তৈরি হয়!

```
┌─────────────┐      ┌──────────────────────┐      ┌─────────────┐
│   students  │      │   student_courses    │      │   courses   │
├─────────────┤      │    (JOIN TABLE) ⭐    │      ├─────────────┤
│ id (PK)     │◄─────│ student_id (FK)      │      │ id (PK)     │
│ name        │      │ course_id (FK)   ────┼─────►│ title       │
└─────────────┘      └──────────────────────┘      └─────────────┘
```

### Hibernate এর তৈরি SQL

```sql
CREATE TABLE students (
    id   BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(255)
);

CREATE TABLE courses (
    id    BIGINT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(255)
);

CREATE TABLE student_courses (          -- ⭐ Auto-created join table
    student_id BIGINT,
    course_id  BIGINT,
    PRIMARY KEY (student_id, course_id),
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

> মানে — Hasib দুইটা course এ আছে (Spring Boot, Kotlin), Karim শুধু Spring Boot এ।

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

## চূড়ান্ত সারসংক্ষেপ

### কোনটায় কী তৈরি হয়?

| Annotation | নতুন Table? | নতুন Column? | কোথায় বসে |
|---|---|---|---|
| `@ManyToOne` | ❌ না | ✅ FK column | "Many" side এ |
| `@OneToMany` | ❌ না | ❌ না | "One" side এ (mappedBy দিয়ে) |
| `@OneToOne` | ❌ না | ✅ FK column (UNIQUE) | যেকোনো এক side এ |
| `@ManyToMany` | ✅ হ্যাঁ — join table! | ❌ না (মূল table এ) | যেকোনো এক side এ owner |

---

## তিনটা Golden Rule

```
1. FK সবসময় "Many" side এ বসে
   → @ManyToOne এর সাথে @JoinColumn দাও

2. @OneToMany এর সাথে mappedBy অবশ্যই দাও
   → না দিলে extra join table তৈরি হয়ে যাবে

3. @ManyToMany ই একমাত্র যেটা নতুন table তৈরি করে
   → অন্য কোনোটা না!
```

---

## পরবর্তী পদক্ষেপ (Next Steps)

এই tutorial শেষ করার পরে এগুলো শিখলে concept আরও পাকা হবে:

1. **Cascade Types** — `CascadeType.ALL`, `PERSIST`, `REMOVE` — Parent delete হলে Child এর কী হবে?
2. **Fetch Types** — `FetchType.LAZY` vs `FetchType.EAGER` — কখন data load হবে?
3. **Bidirectional Infinite Recursion** — JSON serialize করার সময় infinite loop এর সমস্যা ও সমাধান (`@JsonManagedReference`, `@JsonBackReference`)
4. **N+1 Query Problem** — Lazy loading এর কারণে extra queries, এবং `JOIN FETCH` দিয়ে সমাধান
