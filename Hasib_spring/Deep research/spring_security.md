# Spring Security Foundation — Concepts Only (Code নেই)

> **ভাষা:** বাংলা (Technical terms ইংরেজিতে)
> **Level:** একদম Beginner — কোনো prior knowledge ধরা হয়নি
> **Topics:** Authentication, Authorization, JWT, Bearer Token, Security Flow

> ⚠️ **এই tutorial এ কোনো Code নেই।** শুধু concepts। পরবর্তী tutorial এ code দেখাবো।
> Concepts পরিষ্কার না করে code দেখলে কিছুই বুঝবে না।

---

## সূচিপত্র

1. [Real World Analogy দিয়ে শুরু](#পর্ব-০)
2. [Authentication কী](#পর্ব-১)
3. [Authorization কী](#পর্ব-২)
4. [দুইটার পার্থক্য](#পর্ব-৩)
5. [Session-based Auth (পুরনো উপায়)](#পর্ব-৪)
6. [Token-based Auth (নতুন উপায়)](#পর্ব-৫)
7. [JWT কী](#পর্ব-৬)
8. [JWT এর Structure](#পর্ব-৭)
9. [Bearer Token কী](#পর্ব-৮)
10. [Login → Protected API: পুরো Flow](#পর্ব-৯)
11. [Role vs Permission](#পর্ব-১০)
12. [Real Scenario দিয়ে সব মিলিয়ে](#পর্ব-১১)

---

<a name="পর্ব-০"></a>

# পর্ব ০ — Real World Analogy দিয়ে শুরু

Security এর সব concept আগে real life দিয়ে বুঝি। তারপর technical এ যাবো।

## একটা Office Building এর কথা ভাবো

ধরো তুমি একটা বড় office এ কাজ করো।

### সকালে Office এ যাচ্ছো

```
১. Building এর গেট এ পৌঁছালে
২. Security Guard বললো — "ID কার্ড দেখান"
৩. তুমি ID কার্ড দিলে
৪. Guard verify করলো — "হ্যাঁ, ইনি আমাদের employee"
৫. তোমাকে ঢুকতে দিল
```

এই process টাই **Authentication** —
> *"তুমি কে?" — এটা verify করা।*

### এখন তুমি 3rd Floor এ Server Room এ যেতে চাও

```
১. Server Room এর দরজায় Card Scanner আছে
২. তুমি Card scan করলে
৩. System দেখলো — "এই card এর কি Server Room এ access আছে?"
৪. না থাকলে → "Access Denied"
   থাকলে → "Welcome"
```

এই process টাই **Authorization** —
> *"তোমার কী অনুমতি আছে?" — এটা check করা।*

---

## Web Application এ একই জিনিস

```
Office               =  Web Application
ID Card              =  Username + Password
Security Guard       =  Login System (Authentication)
Card Scanner         =  Permission Check (Authorization)
Different Floors     =  Different APIs
```

বাকি tutorial এ এই analogy বার বার আসবে।

---

<a name="পর্ব-১"></a>

# পর্ব ১ — Authentication কী?

## সরল সংজ্ঞা

> **"তুমি কে?" — এটা verify করার process।**

User claim করছে — "আমি Hasib।" System verify করে — "সত্যিই কি তুমি Hasib?"

## কীভাবে Verify করে?

বিভিন্ন উপায় আছে —

```
1. Password         → "আমি যা জানি"  (something you know)
2. OTP              → "আমার ফোনে যা আসে"
3. Fingerprint      → "আমি যা" (something you are)
4. Face ID          → "আমার চেহারা"
5. Google Login     → "Google verify করেছে যে আমি আমি"
```

আমরা শুরুতে শুধু **Username + Password** দিয়ে শুরু করবো।

## Authentication এর Steps

```
Step 1: User → Server: "আমি Hasib, password = 12345"
Step 2: Server → Database: "Hasib আছে কি?"
Step 3: Database → Server: "হ্যাঁ, password hash এটা"
Step 4: Server compares passwords
Step 5: Match হলে → "তুমি verified" ✅
        Match না হলে → "Wrong credentials" ❌
```

## Authentication সফল হলে কী হয়?

Server তোমাকে একটা **Token** দেয় — যেটা proof যে তুমি verified।

ঠিক যেমন office এ ID card pre-issued থাকে, তুমি সকালে show করো।

```
Login সফল
        ↓
Server: "এই নাও তোমার Token"
        ↓
User: Token সাথে রাখলো
        ↓
পরে যেকোনো API call এ এই Token পাঠাবে
```

---

<a name="পর্ব-২"></a>

# পর্ব ২ — Authorization কী?

## সরল সংজ্ঞা

> **"তোমার কী অনুমতি আছে?" — এটা check করার process।**

User authenticated হয়েছে — system জানে এটা Hasib। কিন্তু Hasib সব কিছু করতে পারবে?

```
Hasib  → /api/books         → ✅ পড়তে পারবে
Hasib  → POST /api/books    → ✅ Create করতে পারবে
Hasib  → DELETE /api/users  → ❌ Delete করতে পারবে না
```

কে কী পারবে — সেই rules ই Authorization।

## Authorization Check হয় কখন?

প্রতিটা API call এ —

```
User API call করলো
        ↓
Token verify হলো → "Hasib"
        ↓
"Hasib এর কি এই API access করার অনুমতি আছে?"
        ↓
আছে → API চলবে ✅
নেই → 403 Forbidden ❌
```

## Authorization কীভাবে Decide হয়?

দুইভাবে —

### Way 1: Role দিয়ে

```
ADMIN     → সব কিছু করতে পারে
EDITOR    → Edit করতে পারে
VIEWER    → শুধু দেখতে পারে
```

### Way 2: Permission দিয়ে (granular)

```
BOOK_READ
BOOK_CREATE
BOOK_DELETE
USER_READ
USER_CREATE
```

পরে এটা গভীরে দেখবো।

---

<a name="পর্ব-৩"></a>

# পর্ব ৩ — Authentication vs Authorization

## পার্থক্য একনজরে

| | Authentication | Authorization |
|---|---|---|
| **প্রশ্ন** | "তুমি কে?" | "তুমি কী করতে পারো?" |
| **কখন হয়** | Login এর সময় | প্রতিটা API call এ |
| **Verify করে** | Identity (পরিচয়) | Permission (অনুমতি) |
| **Fail হলে** | 401 Unauthorized | 403 Forbidden |
| **Office এ** | Building gate এ Guard | Floor এ Card scanner |

## সহজ মনে রাখো

```
Authentication = "Who are you?"  → পরিচয় দাও
Authorization  = "What can you do?" → কী পারবে

প্রথমে Authentication, তারপর Authorization
```

## Order সবসময় এই —

```
১. Authentication হবে (তুমি কে verify)
২. তারপর Authorization হবে (তোমার অনুমতি check)

Authentication না হলে Authorization এর প্রশ্নই নেই।
```

---

<a name="পর্ব-৪"></a>

# পর্ব ৪ — Session-based Auth (পুরনো উপায়)

আগে website গুলো এভাবে কাজ করতো। বুঝতে হবে — কারণ Token-based কেন এলো সেটা জানতে হবে।

## Session-based কীভাবে কাজ করে

```
১. User login করলো
২. Server verify করলো ✅
৩. Server একটা "Session ID" তৈরি করলো (যেমন: ABC123XYZ)
৪. Session ID server এর memory তে save করলো
       (Session: ABC123XYZ → User: Hasib)
৫. Browser কে Session ID দিল (Cookie হিসেবে)
৬. পরবর্তী request এ Browser automatically Session ID পাঠায়
৭. Server memory তে check করে কে এই user
```

## ছবি দিয়ে দেখো

```
Browser                          Server Memory
───────                          ─────────────
                                 Session: ABC123 → Hasib
Cookie: ABC123                   Session: XYZ789 → Karim
                                 Session: PQR456 → Rahim

Browser request পাঠায়:
"GET /books, Cookie: ABC123"
                                 Server check:
                                 "ABC123 = Hasib"
                                 "Hasib এর data ফেরত দাও"
```

## Session-based এর সমস্যা

### সমস্যা 1: Server Memory ভর্তি হয়ে যায়

প্রতিটা logged-in user এর জন্য server এ memory দরকার। ১০ লাখ user থাকলে — ১০ লাখ session save করতে হবে। 😱

### সমস্যা 2: Multiple Server এ কাজ করে না

আধুনিক apps এ অনেক server থাকে (load balancing) —

```
User login করলো → Server 1 এ session save হলো
পরের request → Load Balancer Server 2 এ পাঠালো
Server 2: "ABC123 কে? আমি জানি না!" ❌
```

### সমস্যা 3: Mobile App এ কাজ করে না

Mobile apps এ Cookie বেশ tricky। API based architecture এ session-based কাজ করানো কঠিন।

### সমস্যা 4: Scalability নেই

বড় system এ session-based ভেঙে পড়ে।

**তাই Token-based এলো।**

---

<a name="পর্ব-৫"></a>

# পর্ব ৫ — Token-based Auth (নতুন উপায়)

## মূল ধারণা

```
Server এ কিছু save করো না।
User এর handেই সব info রাখো — Token এর মধ্যে।
```

## কীভাবে কাজ করে

```
১. User login করলো
২. Server verify করলো ✅
৩. Server একটা Token তৈরি করলো — যেটাতে user info লেখা
       Token = encrypted version of:
              { user: "Hasib", role: "ADMIN", expires: "2026-12-31" }
৪. Token কে user দিল
       Server এ কিছুই save করলো না!
৫. পরের request এ user Token পাঠায়
৬. Server Token verify করে — "এই Token কি valid?"
৭. Valid হলে → user info Token থেকেই পাও
```

## ছবি দিয়ে দেখো

```
Browser                          Server
───────                          ──────
Token: eyJhbGc...                (memory তে কিছু নেই)
       (user info inside)

Browser request:
"GET /books,
 Authorization: Bearer eyJhbGc..."
                                 Server:
                                 1. Token verify করো
                                 2. Token থেকে user info বের করো
                                 3. "এটা Hasib"
                                 4. Data ফেরত দাও
```

## Token-based এর সুবিধা

```
✅ Server এ কিছু save করতে হয় না
✅ Multiple server এ কাজ করে (stateless)
✅ Mobile, Web, Desktop সব platform এ কাজ করে
✅ Scalable
```

## এই Token কী?

সবচেয়ে popular Token format = **JWT (JSON Web Token)**

পরের পর্বে JWT গভীরে দেখবো।

---

<a name="পর্ব-৬"></a>

# পর্ব ৬ — JWT কী?

## পূর্ণ নাম

**JSON Web Token** — উচ্চারণ "JOT"।

## সংজ্ঞা

> **JWT হলো একটা compact, encrypted string যেটাতে user এর info থাকে।**

## একটা Real JWT দেখো

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJIYXNpYiIsInJvbGUiOiJBRE1JTiIsImV4cCI6MTczNTY4MzIwMH0.kPeRjMq0bRPpTQVCV7Qyi0hAGEhJyTyJzPGhvOe3CQ4
```

দেখতে garbage মনে হচ্ছে? আসলে এর ভেতরে user এর information আছে — encrypted form এ।

---

<a name="পর্ব-৭"></a>

# পর্ব ৭ — JWT এর Structure

## তিনটা Part — Dot দিয়ে separated

```
HEADER.PAYLOAD.SIGNATURE
   ↓      ↓        ↓
 Algorithm Data  Verification
```

### একটা JWT example এ দেখো

```
eyJhbGciOiJIUzI1NiJ9 . eyJzdWIiOiJIYXNpYiJ9 . kPeRjMq0bRPpTQVCV7Qyi0
        ↑                       ↑                          ↑
     HEADER                  PAYLOAD                   SIGNATURE
```

## Part 1: Header

> *"এই Token কীভাবে encrypt করা হয়েছে।"*

Decode করলে JSON —

```json
{
  "alg": "HS256",      // কোন algorithm
  "typ": "JWT"         // type
}
```

## Part 2: Payload (সবচেয়ে Important!)

> *"User এর actual data — কে এই user, role কী, কখন expire হবে।"*

Decode করলে —

```json
{
  "sub": "Hasib",                    // subject (user identifier)
  "role": "ADMIN",                   // role
  "email": "hasib@mail.com",
  "iat": 1735596800,                 // issued at (Unix timestamp)
  "exp": 1735683200                  // expires at
}
```

এই data কে **Claims** বলে — JWT terminology তে।

## Part 3: Signature

> *"এই Token tamper করা হয়নি — সেটা verify করার জন্য।"*

Server এর কাছে একটা **secret key** থাকে। সেই key দিয়ে signature বানানো হয়।

```
Signature = encrypt(HEADER + PAYLOAD, secretKey)
```

## কেউ Token Tamper করার চেষ্টা করলে?

ধরো hacker payload এ role পরিবর্তন করলো —

```
Original Payload:           Hacker Modified:
{ role: "USER" }     →      { role: "ADMIN" }
```

কিন্তু Signature original payload এর জন্য ছিল। Server signature verify করতে গিয়ে দেখবে — match হচ্ছে না।

```
Server: "Signature mismatch! Token tampered. ❌"
```

**তাই JWT secure।**

## ⚠️ Important

Payload **encrypted না, encoded।** যে কেউ decode করে পড়তে পারে।

```
তাই Token এ password, sensitive data রাখো না!
শুধু non-sensitive info — user id, role, email।
```

Signature এর কারণে কেউ পরিবর্তন করতে পারে না — কিন্তু পড়তে পারে।

---

<a name="পর্ব-৮"></a>

# পর্ব ৮ — Bearer Token কী?

## সংজ্ঞা

> **Bearer Token** = Token যেটা যার কাছে আছে, সেই use করতে পারে।

"Bearer" শব্দের মানে — **বাহক / ধারক।**

## Real World Analogy

ধরো তোমার কাছে একটা bus ticket আছে।
- Bus driver ticket দেখে — কে এটা ধরে আছে সেটা দেখে না
- Ticket থাকলেই — bus এ উঠতে পারো

```
Ticket = Token
Bus Driver = Server
"Bearer" = তুমি (যে token ধরে আছে)
```

## তাই —

> **Token যত্ন করে রাখতে হয়। চুরি হয়ে গেলে সেই কেউ login হতে পারবে।**

## API তে Bearer Token কীভাবে পাঠাও?

HTTP Request এ একটা Header থাকে —

```
Authorization: Bearer <your-token-here>
```

### Example

```
GET /api/books HTTP/1.1
Host: localhost:8080
Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJIYXNpYiJ9.kPeRjM...
```

### Postman এ কেমন দেখায়

```
Headers:
┌──────────────────┬─────────────────────────────────┐
│ Key              │ Value                           │
├──────────────────┼─────────────────────────────────┤
│ Authorization    │ Bearer eyJhbGciOiJIUzI1NiJ9.... │
│ Content-Type     │ application/json                │
└──────────────────┴─────────────────────────────────┘
```

---

<a name="পর্ব-৯"></a>

# পর্ব ৯ — Login থেকে Protected API: পুরো Flow

এবার সব মিলিয়ে দেখি।

## Step 1: User Register করলো

```
POST /api/auth/register
{
  "username": "hasib",
  "email": "hasib@mail.com",
  "password": "SecurePass123!"
}
        ↓
Server:
১. Email duplicate কিনা check
২. Password hash করলো (bcrypt)
৩. Database এ user save
৪. "Registration successful" ✅
```

## Step 2: User Login করলো

```
POST /api/auth/login
{
  "username": "hasib",
  "password": "SecurePass123!"
}
        ↓
Server:
১. Database থেকে user find
২. Password match করো (hash compare)
৩. Match হলো ✅
৪. JWT Token তৈরি করো
        Payload: { sub: "hasib", role: "USER" }
৫. Response এ Token ফেরত দাও
```

```
Response:
{
  "token": "eyJhbGc...",
  "type": "Bearer",
  "expiresIn": 3600
}
```

## Step 3: User Token সংরক্ষণ করলো

Frontend (React/Mobile App) Token কে save করে —

```
Browser: localStorage এ save
Mobile: Secure storage এ save
```

## Step 4: Protected API Call

```
GET /api/books
Headers:
  Authorization: Bearer eyJhbGc...
```

## Step 5: Server Token Verify করলো

```
Server:
১. Authorization Header থেকে Token extract
২. Token signature verify করলো (secret key দিয়ে)
৩. Token expired কিনা check
৪. সব ঠিক → Payload থেকে user info বের
       { sub: "hasib", role: "USER" }
৫. "এটা Hasib, role USER"
```

## Step 6: Authorization Check

```
Server:
১. "এই API তে কে access করতে পারে?"
২. "Hasib এর কী permission আছে?"
৩. Match হলে → API চালাও ✅
   Match না হলে → 403 Forbidden ❌
```

## Step 7: Response

```
{
  "books": [
    { "id": 1, "title": "Pother Pachali" },
    { "id": 2, "title": "Aparajito" }
  ]
}
```

## পুরো Flow Visualization

```
┌──────────────┐                              ┌──────────────┐
│   Client     │                              │    Server    │
└──────┬───────┘                              └──────┬───────┘
       │                                             │
       │  POST /login (username, password)           │
       │────────────────────────────────────────────►│
       │                                             │ Verify
       │                                             │ Generate JWT
       │  { token: "eyJ..." }                        │
       │◄────────────────────────────────────────────│
       │                                             │
       │  Save Token                                 │
       │                                             │
       │  GET /api/books                             │
       │  Authorization: Bearer eyJ...               │
       │────────────────────────────────────────────►│
       │                                             │ Verify Token
       │                                             │ Check Permission
       │  { books: [...] }                           │ Run API
       │◄────────────────────────────────────────────│
       │                                             │
```

---

<a name="পর্ব-১০"></a>

# পর্ব ১০ — Role vs Permission

দুইটাই Authorization এ use হয়। পার্থক্য বুঝা important।

## Role

> **Role = User এর "type" বা "designation"।**

```
ADMIN
EDITOR
VIEWER
USER
```

ছোট system এ Role যথেষ্ট।

## Permission

> **Permission = Specific কাজ করার অনুমতি।**

```
BOOK_READ
BOOK_CREATE
BOOK_UPDATE
BOOK_DELETE
USER_READ
USER_CREATE
USER_DELETE
```

বড় system এ granular control এর জন্য Permission।

## দুইটার সম্পর্ক

```
Role এ Permissions থাকে:

ADMIN  →  [BOOK_READ, BOOK_CREATE, BOOK_UPDATE, BOOK_DELETE,
           USER_READ, USER_CREATE, USER_DELETE]

EDITOR →  [BOOK_READ, BOOK_CREATE, BOOK_UPDATE]

VIEWER →  [BOOK_READ]
```

User কে Role assign করলে — সেই Role এর সব Permission পেয়ে যায়।

## Database Structure

```
users:
┌────┬───────┐
│ id │ name  │
├────┼───────┤
│ 1  │ Hasib │
│ 2  │ Karim │
└────┴───────┘

roles:
┌────┬─────────┐
│ id │ name    │
├────┼─────────┤
│ 1  │ ADMIN   │
│ 2  │ EDITOR  │
│ 3  │ VIEWER  │
└────┴─────────┘

permissions:
┌────┬────────────────┐
│ id │ name           │
├────┼────────────────┤
│ 1  │ BOOK_READ      │
│ 2  │ BOOK_CREATE    │
│ 3  │ BOOK_DELETE    │
└────┴────────────────┘

user_roles (Many-to-Many):
┌─────────┬─────────┐
│ user_id │ role_id │
├─────────┼─────────┤
│ 1       │ 1       │  → Hasib has ADMIN
│ 2       │ 2       │  → Karim has EDITOR
└─────────┴─────────┘

role_permissions (Many-to-Many):
┌─────────┬──────────────┐
│ role_id │ permission_id│
├─────────┼──────────────┤
│ 1       │ 1            │  → ADMIN has BOOK_READ
│ 1       │ 2            │  → ADMIN has BOOK_CREATE
│ 1       │ 3            │  → ADMIN has BOOK_DELETE
│ 2       │ 1            │  → EDITOR has BOOK_READ
│ 2       │ 2            │  → EDITOR has BOOK_CREATE
└─────────┴──────────────┘
```

আগে JPA তে Many-to-Many শিখেছিলে — এখানেই কাজে লাগবে।

## কোনটা কখন Use করবে?

```
ছোট/Simple system   → Role দিয়ে চালিয়ে নাও
বড়/Complex system  → Role + Permission দুইটা
```

আমরা **Role + Permission** approach শিখবো — production grade।

---

<a name="পর্ব-১১"></a>

# পর্ব ১১ — Real Scenario দিয়ে সব মিলিয়ে

এবার একটা পুরো scenario দিয়ে সব মিলিয়ে দেখি।

## Scenario: Library Management System

তিন ধরনের user আছে —

```
ADMIN   → সব কিছু করতে পারে (book add, delete, user manage)
LIBRARIAN → Book add/edit করতে পারে, user manage করতে পারে না
MEMBER  → শুধু book browse করতে পারে
```

## Permission Matrix

| API | ADMIN | LIBRARIAN | MEMBER | Public |
|---|---|---|---|---|
| GET /books | ✅ | ✅ | ✅ | ✅ |
| POST /books | ✅ | ✅ | ❌ | ❌ |
| PUT /books | ✅ | ✅ | ❌ | ❌ |
| DELETE /books | ✅ | ❌ | ❌ | ❌ |
| GET /users | ✅ | ❌ | ❌ | ❌ |
| POST /users | ✅ | ❌ | ❌ | ❌ |
| DELETE /users | ✅ | ❌ | ❌ | ❌ |

## Flow Example 1: Member ব্যবহারকারী

```
1. Karim register করলো → MEMBER role assign
2. Karim login করলো → JWT Token পেলো
   Token Payload: { sub: "Karim", role: "MEMBER" }

3. Karim: GET /books → ✅ 200 OK (MEMBER can read)

4. Karim: POST /books (নতুন book add করতে চাইলো)
   → Server token verify ✅
   → Permission check: "MEMBER can BOOK_CREATE?"
   → ❌ 403 Forbidden
```

## Flow Example 2: Librarian ব্যবহারকারী

```
1. Hasib login → Token: { role: "LIBRARIAN" }

2. Hasib: POST /books → ✅ 201 Created

3. Hasib: DELETE /books/5 → ❌ 403 Forbidden
   (LIBRARIAN can't delete)

4. Hasib: GET /users → ❌ 403 Forbidden
   (LIBRARIAN can't manage users)
```

## Flow Example 3: Admin ব্যবহারকারী

```
1. Rahim login → Token: { role: "ADMIN" }

2. Rahim: যেকোনো API → ✅ সব Pass
```

## Flow Example 4: Token ছাড়া (Public)

```
1. কেউ Token ছাড়া POST /books call করলো
2. Server: "Authorization Header নেই"
3. ❌ 401 Unauthorized
```

## Flow Example 5: Expired Token

```
1. User এর Token 1 ঘণ্টা আগে expire হয়ে গেছে
2. সেই Token দিয়ে API call
3. Server: Token expire date check
4. ❌ 401 Unauthorized — "Token expired, please login again"
```

---

## চূড়ান্ত সারসংক্ষেপ

### Concepts Map

```
Authentication  →  Login → JWT Token দেওয়া
        ↓
Authorization   →  Token verify → Permission check
        ↓
JWT             →  HEADER.PAYLOAD.SIGNATURE
        ↓
Bearer Token    →  Authorization Header এ পাঠানো
        ↓
Role            →  User এর type
        ↓
Permission      →  Specific কাজের অনুমতি
```

### HTTP Status Codes

```
200 OK              →  সব ঠিক
401 Unauthorized    →  Authentication fail (Token নেই/invalid)
403 Forbidden       →  Authorization fail (Permission নেই)
```

দুইটার পার্থক্য —

```
401 → "তুমি কে?"          (login করো আগে)
403 → "তোমার অনুমতি নেই" (login করেছো, কিন্তু এই API তে access নেই)
```

### Industry Standard Flow

```
Register → Login → Token পাও → প্রতি request এ Token পাঠাও →
Server verify করে → Permission check করে → API চালায়
```

### Golden Rules

```
1. Authentication আগে, Authorization পরে
2. Server এ Session save করো না — Token use করো (JWT)
3. JWT Payload এ password রাখো না (পড়া যায়)
4. Token সবসময় HTTPS এ পাঠাও (HTTP তে চুরি হয়)
5. Token এর expire time দাও (security)
6. Bearer Token যত্ন করো — চুরি হলে কেউ login হতে পারবে
7. Role + Permission system দিয়ে granular control
```

---

## পরবর্তী পদক্ষেপ

এখন তুমি বুঝেছো —
- Authentication কী
- Authorization কী
- JWT কী
- Bearer Token কী
- পুরো flow কেমন

পরের tutorial এ — **Spring Security + JWT এর actual code।** সব concept মাথায় রাখো — code দেখে যেন আর confused না হও।

```
Tutorial 1 (এটা):  Concepts only ✅
Tutorial 2 (পরের): Code Implementation
                  - Spring Security setup
                  - User + Role + Permission tables
                  - JWT generate করা
                  - Login API
                  - Token verify filter
                  - Protected API
```

Concepts পরিষ্কার? পরের tutorial এ যাবো? 🚀
