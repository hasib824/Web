# 🚢 DSTOS Module 1 — Vessel & Berthing Management

**Complete Deep Dive Report**

> এই document Module 1 কে ভেতর থেকে বুঝতে সাহায্য করবে — কেন এই module দরকার, কারা কারা use করে, প্রতিটা feature কিভাবে কাজ করে, real-world workflow কেমন, data কিভাবে flow হয়, এবং অন্য modules-এর সাথে কিভাবে connect হয়।

---

## 📋 Table of Contents

1. [Overview — এই Module কেন আছে?](#1-overview--এই-module-কেন-আছে)
2. [Real-World Context — Chittagong Port](#2-real-world-context--chittagong-port)
3. [Actors — কারা ব্যবহার করে?](#3-actors--কারা-ব্যবহার-করে)
4. [Core Concepts — Foundation](#4-core-concepts--foundation)
5. [Use Cases — Detailed Breakdown](#5-use-cases--detailed-breakdown)
6. [Complete Workflow — Step by Step](#6-complete-workflow--step-by-step)
7. [Data Model — কি Store হয়?](#7-data-model--কি-store-হয়)
8. [Business Rules & Edge Cases](#8-business-rules--edge-cases)
9. [Integration Points — অন্য Modules এর সাথে](#9-integration-points--অন্য-modules-এর-সাথে)
10. [External System Dependencies](#10-external-system-dependencies)
11. [Failure Scenarios — কি ভুল হতে পারে?](#11-failure-scenarios--কি-ভুল-হতে-পারে)
12. [Implementation Considerations](#12-implementation-considerations)

---

## 1. Overview — এই Module কেন আছে?

### Big Picture

Chittagong port-এ প্রতিদিন **11+ vessels** আসে। প্রতিটা vessel আসার আগে তাকে allocate করতে হয় একটা **berth** (jetty/parking spot)। কিন্তু port-এ berth সংখ্যা সীমিত — only **19 main berths** আছে। প্রতিটা berth-এর আছে নিজস্ব **size, depth, equipment**। সব vessel সব berth-এ যেতে পারে না।

এর সাথে আছে আরো জটিলতা:
- **Tide** — Bay of Bengal-এ tide significant। Big vessel শুধু high tide-এ ঢুকতে পারে
- **Multiple shipping agents** — Maersk, MSC, CMA CGM, etc. সবাই একসাথে berth চায়
- **Priority cargo** — Perishables, hazmat-এর জন্য priority handling
- **Berth occupancy** — একটা vessel berth-এ আছে, কখন release হবে?

এই সব factor মিলিয়ে **manually decide করা impossible**। তাই Module 1।

### এই Module কি Solve করে?

```
+------------------------------------------------------------+
|  Problem: কোন ship কোন berth-এ কখন park করবে?              |
|                                                            |
|  Without Module 1:                                         |
|    - Phone calls, emails, paper schedules                  |
|    - Conflicts, double-bookings                            |
|    - Wasted berth time                                     |
|    - Vessels waiting at sea (demurrage cost)               |
|                                                            |
|  With Module 1:                                            |
|    - Centralized berthing schedule                         |
|    - Auto-suggestion based on rules                        |
|    - Real-time visibility for all parties                  |
|    - Dispute resolution via audit trail                    |
+------------------------------------------------------------+
```

### Module 1-এর Core Responsibilities

```
+------------------------------------------------------------+
|  1. Shipping Agent Management                              |
|     - Register, verify, maintain agent records             |
|                                                            |
|  2. Vessel Master Data Management                          |
|     - Track every vessel that visits Chittagong            |
|     - Specifications, capabilities, history                |
|                                                            |
|  3. Berth (Jetty) Master Data                              |
|     - Configure each berth's properties                    |
|     - Depth, length, equipment, restrictions               |
|                                                            |
|  4. Tidal Information Management                           |
|     - Store tide forecasts                                 |
|     - Used in berthing decisions                           |
|                                                            |
|  5. Berthing Application Processing                        |
|     - Receive applications from agents                     |
|     - Validate, queue, schedule                            |
|                                                            |
|  6. Auto-Berthing Suggestion                               |
|     - Algorithm-based berth assignment                     |
|     - Considers all constraints                            |
|                                                            |
|  7. Manual Override                                        |
|     - Marine Dept can override system suggestions          |
|     - Reason tracking for audit                            |
|                                                            |
|  8. Cancellation & Rescheduling                            |
|     - Handle cancellations gracefully                      |
|     - Rebalance schedule                                   |
|                                                            |
|  9. Sail Out Recording                                     |
|     - Track vessel departures                              |
|     - Release berth for next vessel                        |
|                                                            |
|  10. Reporting                                             |
|      - Berthing reports for management                     |
|      - Vessel declaration reports for customs/authorities  |
+------------------------------------------------------------+
```

---

## 2. Real-World Context — Chittagong Port

### Port-এর Physical Layout

Chittagong port-এ multiple terminals এবং berths আছে। বুঝার জন্য simplified view:

```
                      Bay of Bengal
                            |
                            | (vessels approach from sea)
                            v
        +-----------------------------------------------+
        |              Outer Anchorage                  |
        |      (vessels wait here for berth)            |
        +-----------------------------------------------+
                            |
                            | (when berth ready, pilot guides ship in)
                            v
        +-----------------------------------------------+
        |              Inner Channel                    |
        |          (Karnaphuli River)                   |
        +-----------------------------------------------+
                            |
                            v
   +---------+---------+----+----+---------+---------+
   |         |         |         |         |         |
   v         v         v         v         v         v
+------+ +------+ +------+ +------+ +------+ +------+
|Berth | |Berth | |Berth | | NCT  | | PCT  | |GCB   |
|  1   | |  2   | | ...  | |Term. | |Term. |  ...  |
+------+ +------+ +------+ +------+ +------+ +------+
                  (different berth types & equipment)
```

### Key Berth Types

```
+----------------------------------------------------+
| GCB (General Cargo Berth)                          |
|   - Berth 1-13                                     |
|   - Mixed cargo: containers, breakbulk, vehicles   |
|   - Older infrastructure                           |
+----------------------------------------------------+
| NCT (New Mooring Container Terminal)               |
|   - Specialized container handling                 |
|   - Modern STS cranes                              |
|   - Higher draft capacity                          |
+----------------------------------------------------+
| PCT (Patenga Container Terminal)                   |
|   - Newest, deepest berth                          |
|   - Largest vessels                                |
|   - Modern automated equipment                     |
+----------------------------------------------------+
```

### Daily Numbers

```
+----------------------------------------------------+
| Daily Operations at Chittagong Port (FY 2024-25)   |
+----------------------------------------------------+
| Vessels arriving           : ~11                   |
| Vessels departing          : ~11                   |
| Total TEU handled          : ~9,000                |
| Active berths              : 19                    |
| Average berth occupancy    : 75-85%                |
| Average ship turnaround    : 2.5-3 days            |
+----------------------------------------------------+
```

### Why Berthing is Critical

প্রতি ঘণ্টা berth-এ vessel না থাকলে port-এর revenue loss। আবার vessel সমুদ্রে wait করলে shipping line-এর demurrage cost ($10,000-30,000 per day per vessel)। এই সব মিলিয়ে berthing efficiency = port-এর জাতীয় economy-তে impact।

---

## 3. Actors — কারা ব্যবহার করে?

Module 1-এর actors এবং তাদের responsibilities:

### Primary Actors (Direct Users)

```
+---------------------------------------------------------+
| 1. Shipping Agent                                       |
+---------------------------------------------------------+
| Examples : Maersk Bangladesh, MSC Bangladesh, etc.      |
| Role     : Vessels-এর local representative              |
| Access   : Web Community Portal (external interface)    |
| Actions  : - Register their company                     |
|            - Submit vessel info before arrival          |
|            - Apply for berthing                         |
|            - Cancel/modify applications                 |
|            - View own vessels' status                   |
+---------------------------------------------------------+

+---------------------------------------------------------+
| 2. Marine Department                                    |
+---------------------------------------------------------+
| Role     : Port-এর marine operations authority         |
| Access   : DSTOS internal interface                     |
| Actions  : - Configure berth master data               |
|            - Upload tidal information                   |
|            - Approve/override berth suggestions         |
|            - Cancel berthing schedules                  |
|            - Record sail-out events                     |
|            - Generate berthing reports                  |
|            - Generate vessel declaration reports        |
+---------------------------------------------------------+

+---------------------------------------------------------+
| 3. Traffic Department                                   |
+---------------------------------------------------------+
| Role     : Vessel scheduling and queue management       |
| Access   : DSTOS internal interface                     |
| Actions  : - Register shipping agents                   |
|            - Manage vessel master data                  |
|            - Coordinate with marine dept                |
|            - Handle priority decisions                  |
+---------------------------------------------------------+

+---------------------------------------------------------+
| 4. Radio Control Section                                |
+---------------------------------------------------------+
| Role     : Communication with vessels                   |
| Access   : DSTOS internal interface                     |
| Actions  : - Receive vessel arrival info                |
|            - Communicate berth assignments              |
|            - Coordinate vessel movements                |
+---------------------------------------------------------+
```

### Secondary Actors (External Systems)

```
+---------------------------------------------------------+
| 5. VTMIS / AIS                                          |
|    (Vessel Traffic Management Information System)       |
|    (Automatic Identification System)                    |
+---------------------------------------------------------+
| Role     : Real-time vessel tracking globally           |
| Interface: XML data exchange with DSTOS                 |
| Provides : - Vessel position (latitude, longitude)      |
|            - Speed, heading, status                     |
|            - ETA refinement                             |
+---------------------------------------------------------+
```

### Actor Interaction Map

```
                    +------------------+
                    | Shipping Agent   |
                    | (Web Portal)     |
                    +--------+---------+
                             |
                             | submit/apply
                             v
        +---------------------------------------+
        |              DSTOS Module 1            |
        |                                        |
        |   +-------+    +-------+    +-------+ |
        |   |Vessel |    |Berth  |    |Tide   | |
        |   |Master |    |Master |    |Data   | |
        |   +-------+    +-------+    +-------+ |
        |                                        |
        |       +---------------------+          |
        |       | Auto-Suggestion     |          |
        |       | Engine              |          |
        |       +---------------------+          |
        +---------------+------------------------+
                        |
        +---------------+----------------+
        |               |                |
        v               v                v
+-------------+  +-------------+  +-------------+
| Marine Dept |  | Traffic Dept|  | Radio       |
| (approve/   |  | (coordinate)|  | Control     |
|  override)  |  |             |  | (comm)      |
+-------------+  +-------------+  +-------------+
        ^
        | XML data exchange
        |
+---------------+
| VTMIS / AIS   |
| (external)    |
+---------------+
```

---

## 4. Core Concepts — Foundation

আগে কিছু **fundamental concepts** clear করি যেগুলো ছাড়া বাকিটা বুঝবে না।

### 4.1 Vessel Visit (এক ship-এর এক trip)

একটা vessel multiple times Chittagong আসতে পারে। প্রতিটা trip = একটা **vessel visit**।

```
Vessel: "MAERSK SHANGHAI"
   |
   +-- Visit #1 (Jan 5-7, 2026)    [completed]
   +-- Visit #2 (Jan 18-20, 2026)  [completed]
   +-- Visit #3 (Feb 2-4, 2026)    [in progress]
   +-- Visit #4 (Feb 15, 2026)     [scheduled]

Vessel = stable identity (IMO number)
Visit  = specific arrival event (visit ID)
```

### 4.2 Vessel Visit States

একটা visit কতগুলো state-এর মধ্যে দিয়ে যায়:

```
+---------------------------------------------------------+
|                Vessel Visit Lifecycle                    |
+---------------------------------------------------------+

   Inbound          (Application submitted, ETA known)
      |
      v
   Arrived          (Vessel reached outer anchorage)
      |
      v
   Working          (Vessel berthed, cargo operations on)
      |
      v
   Completed        (Cargo operations done)
      |
      v
   Departed         (Vessel sailed out)
      |
      v
   Closed           (All paperwork done, audit complete)


   Alternative path:
   Cancelled        (Application cancelled before berth)
```

প্রতিটা state-এ allowed actions আলাদা। যেমন:
- **Inbound** state-এ vessel info edit করা যায়
- **Working** state-এ berth change করা যায় না (vessel already at berth)
- **Closed** state-এ কিছুই change করা যায় না (audit locked)

### 4.3 Key Vessel Specifications

```
+---------------------------------------------------------+
| LOA (Length Overall)                                    |
|   - Vessel-এর total length, bow থেকে stern             |
|   - Berth length-এর চেয়ে কম হতে হবে                   |
|   - Example: 350 meters                                 |
+---------------------------------------------------------+
| Beam (Width)                                            |
|   - Vessel-এর width                                     |
|   - Channel width-এর সাথে compatible হতে হবে           |
+---------------------------------------------------------+
| Draft (Draught)                                         |
|   - Vessel-এর underwater depth                         |
|   - Berth depth + tide-এর চেয়ে কম হতে হবে              |
|   - Example: 12 meters                                  |
+---------------------------------------------------------+
| GRT (Gross Registered Tonnage)                          |
|   - Vessel-এর total internal volume                     |
|   - Charges calculation-এ use হয়                       |
+---------------------------------------------------------+
| NRT (Net Registered Tonnage)                            |
|   - Cargo-carrying capacity                             |
+---------------------------------------------------------+
| Vessel Type                                             |
|   - Geared (vessel-এর নিজের crane আছে)                 |
|   - Gearless (port-এর crane লাগবে)                     |
+---------------------------------------------------------+
```

### 4.4 Berth (Jetty) Specifications

```
+---------------------------------------------------------+
| Berth Length                                            |
|   - Total quay length                                   |
|   - LOA-এর চেয়ে বেশি হতে হবে                          |
+---------------------------------------------------------+
| Berth Depth (Draught)                                   |
|   - Water depth at low tide                             |
|   - Vessel draft-এর চেয়ে বেশি হতে হবে                  |
|   - Tide adds extra depth                               |
+---------------------------------------------------------+
| Jetty Type                                              |
|   - Container terminal                                  |
|   - General cargo                                       |
|   - Bulk cargo                                          |
|   - Specialized (oil, gas, etc.)                        |
+---------------------------------------------------------+
| Equipment Available                                     |
|   - STS cranes (number, capacity)                       |
|   - Mooring facilities                                  |
|   - Power supply (for reefers)                          |
+---------------------------------------------------------+
```

### 4.5 Tidal Concepts

```
+---------------------------------------------------------+
|                 Sea Level Variation                      |
+---------------------------------------------------------+

   Sea Level
       ^
       |          /\          /\          /\
   HW  |---+---/    \---+---/    \---+---/    \---
       |   |  /        \  |  /        \  |  /
   MSL |---+/-----------\-+/-----------\-+/--------- (Mean)
       |  /|             \|             \|
   LW  |-/ +---------+---+---------+---+---------+--
       |/                                            
       +----------------------------------------> Time
        12:00      00:00      12:00      00:00

   HW  = High Water (tide top)
   MSL = Mean Sea Level
   LW  = Low Water (tide bottom)

   Tidal cycle ~12.5 hours (twice daily)
   Bay of Bengal tide range: ~3-5 meters in Chittagong
```

**কেন এটা matter করে?**

```
Vessel draft: 12 meters
Berth depth at LW: 10 meters

At LW: vessel can't enter (12m > 10m)
At HW: depth becomes 14m, vessel CAN enter (12m < 14m)

Tidal Window: HW ± 2 hours = ~4 hour window
```

---

## 5. Use Cases — Detailed Breakdown

SRS-এ Module 1-এ মোট **15টা Use Cases** আছে। প্রতিটা detailed explain করি।

### UC-001: Shipping Agent Registration

**Actor:** Traffic Department

**Purpose:** নতুন shipping agent system-এ add করা।

**Real-world context:** যখন কোনো নতুন shipping line Chittagong-এ operate করতে চায়, তাদের local agent must register করতে হয়। এটা one-time setup।

**Process:**

```
+------------------------------------------------------------+
| Step 1: Traffic Dept officer login করে                     |
|   |                                                        |
|   v                                                        |
| Step 2: "Shipping Agent Registration" form open করে       |
|   |                                                        |
|   v                                                        |
| Step 3: Required info enter করে:                          |
|   - Company name                                           |
|   - License number                                         |
|   - Address                                                |
|   - Contact details (phone, email)                         |
|   - Authorized representative                              |
|   - Principal company (যদি agent হয়)                       |
|   |                                                        |
|   v                                                        |
| Step 4: System validates + saves                           |
|   |                                                        |
|   v                                                        |
| Step 5: Agent receives login credentials                   |
+------------------------------------------------------------+
```

**Important business rule:**
> "Shipping agents works on behalf of principal or Principal can acts itself directly."

মানে কখনো actual shipping company নিজেই act করে, কখনো local agent। দুটোই accommodate করতে হবে।

---

### UC-002: Vessel Management — Getting Vessel Details

**Actors:** Marine Department, Radio Control Section, Traffic Department

**Purpose:** প্রতিটা vessel-এর master data store ও maintain করা।

**Real-world context:** একটা vessel যখন প্রথম Chittagong আসে, তার সম্পূর্ণ specifications system-এ enter হয়। পরবর্তী visits-এ এই data reuse হয় (just visit details add হয়)।

**Data Captured:**

```
+------------------------------------------------------------+
| Field              | Type      | Example                   |
+------------------------------------------------------------+
| vessel_id          | unique ID | 12345                     |
| vessel_name        | string    | "Maersk Shanghai"         |
| active             | boolean   | Y/N                       |
| radio_call_sign    | string    | "9V8765"                  |
| length             | meters    | 350                       |
| GRT                | tonnage   | 95000                     |
| NRT                | tonnage   | 50000                     |
| draught            | meters    | 12.5                      |
| vessel_type        | enum      | "Container/Geared"        |
| flag               | country   | "Singapore"               |
| local_agent        | string    | "Maersk Bangladesh"       |
| agency_period_from | date      | 2025-01-01                |
| agency_period_to   | date      | 2026-12-31                |
+------------------------------------------------------------+
```

**Important nuance:**

`local_agent` change হতে পারে! একই vessel কখনো Maersk Bangladesh-এর under, পরে অন্য agent-এর under আসতে পারে। তাই `agency_period_from/to` track করা।

---

### UC-003: Berth Master (Jetty) Configuration

**Actor:** Marine Department

**Purpose:** Port-এর প্রতিটা berth-এর specifications system-এ define করা।

**SRS Quote:**
> "Berth Configuration in DSTOS is one of the main KPI and basically Operation starts its journey from here."

মানে — এটাই foundation। ভুল হলে পুরো system-এ ripple effect।

**Data Captured:**

```
+------------------------------------------------------------+
| Field            | Description                              |
+------------------------------------------------------------+
| jetty_id         | Unique identifier                        |
| jetty_name       | "Berth 7", "NCT-1", "PCT-A", etc.        |
| draught          | Available water depth (m)                |
| length           | Total quay length (m)                    |
| jetty_type       | Container/General/Specialized            |
| jetty_description| Current status, "empty until X date"     |
+------------------------------------------------------------+
```

**Critical thinking — কেন description field important?**

```
Berth-এর status complex হতে পারে:
   - "Available now"
   - "Empty after Jan 15 (current vessel departing)"
   - "Under maintenance until Feb 1"
   - "Dredging in progress, depth temporarily 9m"

এই dynamic info description-এ track করা।
```

---

### UC-004: Storing Tidal Information

**Actor:** Marine Department

**Purpose:** Future 6 months-এর tide data store করা।

**Real-world context:** Bangladesh-এর Meteorological Department predicts tide তথ্য। Marine Dept Excel file পায়, DSTOS-এ upload করে।

**Process:**

```
+------------------------------------------------------------+
| Step 1: Weather dept থেকে tide forecast Excel আসে         |
|         (port community portal এর মাধ্যমে)                |
|   |                                                        |
|   v                                                        |
| Step 2: Marine Dept officer DSTOS-এ login                 |
|   |                                                        |
|   v                                                        |
| Step 3: "Tidal Data Upload" page-এ Excel file select       |
|   |                                                        |
|   v                                                        |
| Step 4: System parses Excel:                               |
|   - Date                                                   |
|   - Time                                                   |
|   - Day name                                               |
|   - Tide height (LW, HW)                                   |
|   |                                                        |
|   v                                                        |
| Step 5: Upload date wise track করে                         |
|         (যাতে wrong upload trace করা যায়)                  |
|   |                                                        |
|   v                                                        |
| Step 6: Berthing decisions-এ এই data use হবে              |
+------------------------------------------------------------+
```

**Data Structure:**

```
+------------------------------------------------------------+
| Tide Records (per day, multiple entries)                  |
+------------------------------------------------------------+
| tide_id        | Unique ID                                  |
| date           | 2026-05-10                                 |
| time_period    | 06:00 / 12:30 / 18:45                      |
| day_name       | Sunday/Monday/etc.                         |
| tide_height_LW | 1.2 meters                                 |
| tide_height_HW | 4.8 meters                                 |
| upload_date    | When this data was uploaded                |
+------------------------------------------------------------+
```

**Nuance:** Per day **2-4 tide events** থাকে (semi-diurnal pattern)। প্রতিটা separate record।

---

### UC-005: Apply for Berthing

**Actor:** Shipping Agent (via Web Community Portal)

**Purpose:** Shipping agent তাদের vessel-এর জন্য berth request করে।

**SRS Quote:**
> "The Shipping Agents has to be able to apply / cancel for berthing against their declared vessel through the built-in web community portal."

**Real-world flow:**

```
Maersk Bangladesh-এর office, 3 দিন আগে:

  Officer-এর কাছে info:
    - Vessel: "Maersk Shanghai"
    - ETA: 2026-05-15, 06:00
    - Cargo: 1200 TEU containers
    - Quota: Container terminal-এ চাই

  Officer DSTOS Web Portal-এ login করে:
    - Application form fill করে
    - Submit করে
    - Confirmation receipt পায়

  System-এ:
    - Application queue-এ যায়
    - Marine Dept review-এর জন্য pending
```

**Data Captured:**

```
+------------------------------------------------------------+
| Field                    | Description                      |
+------------------------------------------------------------+
| berthing_id              | Auto-generated unique ID         |
| imp_rotation_number      | Import rotation tracking         |
| exp_rotation_number      | Export rotation tracking         |
| vessel_visit_id          | Link to vessel visit record      |
| local_agent              | Submitting agent                 |
| outer_anchorage_dated    | When vessel will reach anchorage |
| ETA                      | Estimated Time of Arrival        |
| ETD                      | Estimated Time of Departure      |
| ATA                      | Actual Time of Arrival           |
| ATD                      | Actual Time of Departure         |
| work_start_date          | When operations begin            |
| common_discharged_date   | All discharge complete           |
| common_loaded_date       | All loading complete             |
| sailed_out_datetime      | Vessel departure                 |
| status                   | Inbound/Arrived/Working/etc.     |
+------------------------------------------------------------+
```

**Lifecycle field — `status`:**

```
At application:    status = "Inbound"
Vessel reaches:    status = "Arrived"
Berthed + working: status = "Working"
Operations done:   status = "Completed"
Vessel sailed:     status = "Departed"
Final closure:     status = "Closed"
Application drop:  status = "Cancelled"
```

**Quota concept:**

> "At the time of application, Shipping Agents have to choose the quota against what berth he wants to apply."

মানে agent বলে — "আমি container terminal-এ চাই" বা "GCB-1-এ চাই"। Quota = preferred berth category। System এই preference consider করে।

---

### UC-006: Auto-Suggestion of Berthing

**Actor:** Marine Department (triggers algorithm)

**Purpose:** System automatic suggest করে কোন vessel কোন berth-এ কখন যাবে।

**SRS Quote:**
> "The DSTOS will be able to suggest berth for the applied vessels. It has to consider berthing rule"
> "Berthing matrix/rules must be pre-configured in DSTOS"

**এই UC-ই Module 1-এর crown jewel।** Algorithm কেমন হবে সেটা SRS-এ specifically defined না, কিন্তু standard port operations-এ এই factors consider হয়:

**Algorithm Inputs:**

```
+------------------------------------------------------------+
| Input Type          | Examples                              |
+------------------------------------------------------------+
| Vessel specs        | Length, beam, draft, type             |
| Berth specs         | Length, depth, equipment              |
| Tidal data          | High tide windows                     |
| Berth occupancy     | Current vessels and their ETD         |
| Pending applications| Other vessels waiting                 |
| Cargo type priority | Perishables, hazmat, regular         |
| Quota preference    | Agent's preferred berth               |
| Equipment ready     | STS cranes available                  |
| Historical patterns | Vessel's typical berth                |
+------------------------------------------------------------+
```

**Algorithm Logic (Simplified):**

```
For each pending vessel application:

  Step 1: FILTER compatible berths
    - Berth length >= Vessel LOA
    - Berth depth + max tide >= Vessel draft + safety margin
    - Berth type compatible with vessel type

  Step 2: APPLY tidal constraints
    - If vessel needs high tide:
        Find HW windows in next 48 hours
        Filter berths only available during those windows

  Step 3: FIND time slots
    - Identify when each compatible berth becomes free
    - Consider current vessel's expected ETD

  Step 4: SCORE each option
    - Earliest available time: higher score
    - Berth-equipment match: higher score
    - Quota match: higher score
    - Proximity to yard area: higher score
    - Historical preference: bonus

  Step 5: SUGGEST top 3 options
    - With scores and reasons
    - Marine Dept selects or overrides

  Step 6: CHECK conflicts
    - No two vessels at same berth same time
    - Channel traffic constraints

  Step 7: SAVE suggestion
    - Status: "Suggested" (pending approval)
```

**Visualization:**

```
                    Pending Applications
                          |
                          v
            +-------------+-------------+
            |   Algorithm Engine        |
            |                           |
            |   1. Filter compatible    |
            |   2. Apply tide rules     |
            |   3. Find time slots      |
            |   4. Score options        |
            |   5. Rank top 3           |
            +-------------+-------------+
                          |
                          v
                   Suggestion List
                          |
            +-------------+-------------+
            |             |             |
            v             v             v
        Berth 7     Berth NCT-2    Berth PCT-1
        Score: 95   Score: 87      Score: 75
        15 May      16 May         17 May
        06:00       08:00          14:00
                          |
                          v
                  Marine Dept reviews
                  Approve or Override
```

---

### UC-007: Changing Precedence (Manual Override)

**Actor:** Marine Department (super user)

**Purpose:** Emergency situation-এ system suggestion override করা।

**SRS Quote:**
> "The System Will be able super user to change the berth suggested by the system in emergency case. The Change reason has to specify."

**Real-world scenarios for override:**

```
+------------------------------------------------------------+
| Scenario                | Why Override                      |
+------------------------------------------------------------+
| VIP vessel              | Government vessel, military       |
| Emergency cargo         | Medical supplies, disaster relief |
| Shipping line priority  | Long-term contract preference     |
| Equipment breakdown     | Original berth's crane down       |
| Weather change          | Storm, need shelter               |
| Strategic decision      | Marine Dept knows local context   |
+------------------------------------------------------------+
```

**Process:**

```
+------------------------------------------------------------+
| Step 1: Marine Dept officer-এর override authority verify  |
|   |                                                        |
|   v                                                        |
| Step 2: Override form open করে                            |
|   |                                                        |
|   v                                                        |
| Step 3: Reason specify করে (mandatory):                    |
|   - Free text + category select                            |
|   - "Emergency: Medical supplies vessel"                   |
|   - "VIP: Bangladesh Navy vessel"                          |
|   |                                                        |
|   v                                                        |
| Step 4: New berth assignment provide করে                   |
|   |                                                        |
|   v                                                        |
| Step 5: System validate করে:                               |
|   - Berth still compatible?                                |
|   - No double-booking?                                     |
|   - Tide constraint OK?                                    |
|   |                                                        |
|   v                                                        |
| Step 6: Save with audit trail                              |
|   - Who overrode                                           |
|   - When                                                   |
|   - Reason                                                 |
|   - Original suggestion vs. final                          |
+------------------------------------------------------------+
```

**Audit trail importance:** Port operations regulated। সব override-এর accountability থাকতে হবে। Future inquiry-তে evidence।

---

### UC-008: Cancel Berthing Schedule

**Actor:** Marine Department

**Purpose:** Confirmed berthing schedule cancel করা।

**SRS Quote:**
> "The Port Authority will able to perform cancellation of berth for a particular vessel due to non-performance of vessel or misinformation by shipping agents or shipping agent's reluctance to take the berth."

**Real cancellation reasons:**

```
+------------------------------------------------------------+
| Reason                       | Description                  |
+------------------------------------------------------------+
| Vessel non-performance       | Mechanical failure, late     |
| Misinformation by agent      | Wrong vessel specs           |
| Agent reluctance             | Cost dispute, change plan    |
| Vessel diverted to other port| Strategic decision           |
| Force majeure                | Weather, strike, accident    |
+------------------------------------------------------------+
```

**Important: Cascade effects of cancellation:**

```
Cancel Berth 7 booking for Maersk Shanghai (was scheduled 6:00 AM)
              |
              v
     +--------+--------+
     |                 |
     v                 v
 Other vessels    Modules affected:
 in queue can    - Module 2: Pre-arrival planning canceled
 move up         - Module 3: Equipment unallocated
                 - Module 5: Yard space released
                 - Auto-reschedule queue
```

**Cancellation must:**
1. Update berthing schedule
2. Notify dependent modules
3. Trigger re-suggestion for waiting vessels
4. Maintain audit log
5. Update agent (notification)

---

### UC-009: Sailed Out Vessel

**Actor:** Marine Department

**Purpose:** Vessel-এর departure record করা।

**Process:**

```
+------------------------------------------------------------+
| Step 1: Vessel cargo operations complete                  |
|         (Module 2 indicates: all containers handled)      |
|   |                                                        |
|   v                                                        |
| Step 2: Vessel ready to sail                              |
|         (Pilotage requested, tug boats arranged)          |
|   |                                                        |
|   v                                                        |
| Step 3: Vessel actually departs berth                     |
|   |                                                        |
|   v                                                        |
| Step 4: Marine Dept officer enters in DSTOS:              |
|   - Sailed out date/time                                   |
|   - Any notes                                              |
|   |                                                        |
|   v                                                        |
| Step 5: Status changes:                                    |
|         "Completed" -> "Departed"                          |
|   |                                                        |
|   v                                                        |
| Step 6: Berth becomes available                           |
|         Triggers next vessel notification                  |
+------------------------------------------------------------+
```

**Why distinct from cancellation?**

- Cancel = vessel never used the berth
- Sailed out = vessel completed trip successfully

দুটোর downstream effect আলাদা। Sailed out = success path। Cancel = exception path।

---

### UC-010: Booking Item Form

**Actor:** Traffic

**Purpose:** Stowage planning preparation — vessel arrival-এর আগে container booking position, number, weight, destinations track করা।

**SRS Quote:**
> "Before the arrival of the ships, the accurate booking position, number and weight of Containers, destination of each container among others must be known."
> 
> "The stowage planning must take all information on other calling ports into consideration and has to be very complex."

**Critical insight:**

এই UC actually **bridge** Module 1 এবং Module 2-এর মধ্যে। Vessel আসার আগে যা যা container আসছে তার preliminary info। এই info থেকে Module 2-এর detailed planning হবে।

```
Module 1                      Module 2
   |                              |
   | Booking info captured        |
   | (preliminary)                |
   |  +-- Container count         |
   |  +-- Weight estimate         |
   |  +-- Origin/destination      |
   |                              |
   |---- Hand off ---->           |
                                  |
                                  | EDI Baplie processed
                                  | (detailed)
                                  | Stowage planning
```

---

### UC-011: Empty Loadout Order Form

**Actor:** Mechanical Department / Equipment Operator / Traffic

**Purpose:** Empty container handling orders manage করা।

**Concept clarification:**

```
+------------------------------------------------------------+
| Loaded Container : Cargo full container                    |
| Empty Container  : Empty container being repositioned     |
+------------------------------------------------------------+
```

Shipping operations-এ empty containers significant। Bangladesh primarily exports (garments, jute) — cargo full containers go out, empty containers come in for next export cycle।

**Holds & Permissions logic:**

```
+------------------------------------------------------------+
| Action            | Purpose                                |
+------------------------------------------------------------+
| Add Hold          | Block movement (under inspection)      |
| Release Hold      | Allow movement                         |
| Grant Permission  | Special handling allowed               |
| Cancel Permission | Revoke special handling                |
+------------------------------------------------------------+
```

**Multi-entity action:**
> "If you select multiple entities in a list view and a validation prevents you from applying the hold/permission to one or more selected entities, N4 still applies the hold/permission to the other selected entities."

মানে batch operation — কয়েকটা container-এ hold apply করতে গেলে, যেগুলোতে valid সেগুলোতে apply হবে, যেগুলোতে invalid সেগুলো skip।

---

### UC-012: Marine Events Charge

**Actor:** Mechanical Department / Equipment Operator / Traffic

**Purpose:** Berthing-related fees calculate এবং record করা।

**Charge components:**

```
+------------------------------------------------------------+
| Charge Type            | Basis                              |
+------------------------------------------------------------+
| Berth occupancy        | Per hour at berth                 |
| Pilotage               | Per pilotage event                |
| Tug boat               | Per tug, per hour                 |
| Mooring/Unmooring      | Fixed fee                         |
| Light dues             | Vessel size based                 |
| Port dues              | GRT based                         |
| Conservancy            | Channel maintenance fee           |
+------------------------------------------------------------+
```

**Why integrated in Module 1?**

Berthing operations এর সাথে these charges directly tied। Vessel arrives → charges start accruing → vessel departs → final charge calculated।

---

### UC-013: Hold/Release Inbound Unit Form

**Actor:** Traffic

**Purpose:** Specific containers temporarily hold বা release করা।

**Use cases:**

```
+------------------------------------------------------------+
| Scenario                       | Action                    |
+------------------------------------------------------------+
| Customs inspection needed      | Hold container            |
| Inspection complete            | Release container         |
| Documentation pending          | Hold container            |
| Damage assessment              | Hold container            |
| Hazmat verification            | Hold until cleared        |
| Court order                    | Hold indefinitely         |
+------------------------------------------------------------+
```

**Cascade impact:**

```
Container held in Module 1
   |
   v
Module 4 (Yard): Container marked hold, no movement allowed
Module 5 (Planning): Excluded from move plans
Module 6 (Gate): Cannot leave port
```

---

### UC-014: Generate Report for Berthing

**Actor:** Marine Department

**Purpose:** Berthing operations summary report।

**Typical reports:**

```
+------------------------------------------------------------+
| Report Type              | Content                          |
+------------------------------------------------------------+
| Daily berthing schedule  | Today's vessels, berths          |
| Weekly summary           | Total vessels, occupancy %       |
| Monthly performance      | Avg turnaround, delays           |
| Vessel-wise history      | Specific vessel's all visits     |
| Berth-wise utilization   | Per berth statistics             |
| Agent-wise activity      | Per agent's vessels              |
+------------------------------------------------------------+
```

**Parameters typically:**
- Date range (from-to)
- Specific berth filter
- Specific agent filter
- Vessel type filter
- Status filter

---

### UC-015: Generate Vessel Declaration Report

**Actor:** Marine Department

**Purpose:** Detailed vessel declaration report — for customs, port authority, audit।

**Vessel Declaration Report contents:**

```
+------------------------------------------------------------+
| Section                  | Information                      |
+------------------------------------------------------------+
| Vessel particulars       | Name, IMO, flag, specs           |
| Voyage details           | Origin, destination, route       |
| Agent information        | Local representative             |
| Cargo summary            | Total containers, weight         |
| Crew information         | Captain, officers count          |
| Bunker details           | Fuel onboard                     |
| Berthing history         | This visit's full timeline       |
| Charges summary          | All fees applicable              |
+------------------------------------------------------------+
```

**Why this matters:**
- Customs uses for duty calculation
- Port authority for record keeping
- Insurance claims if needed
- Statistical reporting to government

---

## 6. Complete Workflow — Step by Step

এখন সব UC গুলো mil ke একটা **complete real-world scenario** দিয়ে trace করি।

### Scenario: Maersk Shanghai-এর একটা Visit

```
+------------------------------------------------------------+
|  Vessel: Maersk Shanghai (IMO: 9876543)                    |
|  Agent: Maersk Bangladesh                                  |
|  Cargo: 1,200 TEU containers                               |
|  Origin: Singapore                                         |
|  Destination: Chittagong                                   |
+------------------------------------------------------------+
```

### Phase 1: Pre-Application (T-7 days)

```
DAY -7 (May 8, 2026)
+------------------------------------------------------------+
| Maersk Bangladesh office:                                  |
|   - Singapore HQ থেকে advance notice পেলো                  |
|   - Vessel আসবে May 15                                     |
|   - DSTOS-এ vessel info update prepare করতে হবে            |
|                                                            |
| If vessel new -> UC-002: Vessel Master Data create        |
| If vessel known -> Just verify existing data              |
+------------------------------------------------------------+
```

### Phase 2: Berthing Application (T-3 days)

```
DAY -3 (May 12, 2026)
+------------------------------------------------------------+
| Web Community Portal:                                      |
|   Maersk Bangladesh officer login                          |
|        |                                                   |
|        v                                                   |
|   UC-005: Apply for Berthing                              |
|        |                                                   |
|        +--> Vessel: Maersk Shanghai                        |
|        +--> ETA: May 15, 06:00                             |
|        +--> ETD: May 17, 18:00                             |
|        +--> Quota: Container Terminal                      |
|        +--> Cargo: 1200 TEU                                |
|        +--> Submit                                         |
|                                                            |
|   System: Application received                             |
|           Status = "Inbound"                               |
|           Queued for review                                |
+------------------------------------------------------------+
```

### Phase 3: Auto-Suggestion (T-2 days)

```
DAY -2 (May 13, 2026, 09:00 AM)
+------------------------------------------------------------+
| Marine Dept officer login:                                 |
|        |                                                   |
|        v                                                   |
|   UC-006: Auto-Berthing Suggestion triggered               |
|        |                                                   |
|        v                                                   |
|   Algorithm runs:                                          |
|        |                                                   |
|        v                                                   |
|   Inputs:                                                  |
|     - Vessel: 350m LOA, 12.5m draft                       |
|     - ETA: May 15, 06:00                                   |
|     - Tide May 15:                                         |
|         06:00 - HW (4.8m total depth on shallow berths)   |
|         12:00 - LW                                         |
|     - Berth status:                                        |
|         Berth 7 (NCT): free after May 14, 14:00           |
|         Berth NCT-2: occupied until May 16                 |
|         Berth PCT-1: free anytime                          |
|                                                            |
|   Suggestions ranked:                                      |
|     1. Berth NCT-7  Score: 95  May 15, 06:00              |
|        - Compatible size                                   |
|        - Tidal window matches ETA                          |
|        - Modern container handling                         |
|     2. Berth PCT-1  Score: 78  May 15, 14:00              |
|        - Best equipment but later                          |
|     3. Berth NCT-2  Score: 65  May 16, 08:00              |
|        - Delayed, not preferred                            |
+------------------------------------------------------------+
```

### Phase 4: Marine Dept Review

```
DAY -2 (May 13, 2026, 10:00 AM)
+------------------------------------------------------------+
| Marine Dept officer reviews suggestions:                   |
|        |                                                   |
|        v                                                   |
|   Decision tree:                                           |
|        |                                                   |
|        +--> Accept top suggestion (NCT-7)                 |
|        |        |                                          |
|        |        v                                          |
|        |    Berthing schedule confirmed                    |
|        |    Status = "Inbound" (still)                    |
|        |                                                   |
|        +--> Override (UC-007)                             |
|        |        |                                          |
|        |        v                                          |
|        |    Provide reason                                 |
|        |    Manual berth selection                         |
|        |                                                   |
|        +--> Cancel application (UC-008)                   |
|                 |                                          |
|                 v                                          |
|             Reason required                                |
|             Status = "Cancelled"                          |
|                                                            |
| In our scenario: Accept NCT-7                              |
+------------------------------------------------------------+
```

### Phase 5: Notifications & Module Triggers

```
+------------------------------------------------------------+
| Once berth confirmed:                                      |
|                                                            |
|   Notifications sent:                                      |
|     -> Maersk Bangladesh (agent)                          |
|     -> Radio Control (for vessel comm)                    |
|     -> Module 2 (start pre-arrival prep)                  |
|     -> Module 3 (equipment planning)                      |
|     -> Module 5 (yard allocation)                         |
|                                                            |
|   Other modules begin their work                           |
|   Module 1's work paused until vessel arrives              |
+------------------------------------------------------------+
```

### Phase 6: Vessel Arrival

```
DAY 0 (May 15, 2026, 04:30 AM)
+------------------------------------------------------------+
| VTMIS detects vessel approaching:                          |
|        |                                                   |
|        v                                                   |
|   Updates Module 1:                                        |
|        |                                                   |
|        v                                                   |
|   Vessel position tracking                                 |
|        |                                                   |
|        v                                                   |
| 05:30 AM - Vessel reaches outer anchorage                  |
|   Status = "Arrived"                                       |
|   ATA recorded                                             |
|        |                                                   |
|        v                                                   |
| 05:45 AM - Pilot dispatched                                |
|        |                                                   |
|        v                                                   |
| 06:00 AM - Vessel berthed at NCT-7                         |
|   Status = "Working"                                       |
|   Cargo operations begin (Module 2 takes over)             |
+------------------------------------------------------------+
```

### Phase 7: During Operations

```
DAY 0 - DAY 2 (May 15-17)
+------------------------------------------------------------+
| Module 1 mostly observer mode:                             |
|                                                            |
|   - UC-012 charges accumulating (per hour)                 |
|   - UC-013 if any container hold needed                    |
|   - Status updates from other modules:                     |
|       - Discharge progress                                 |
|       - Loading progress                                   |
|                                                            |
| Other modules doing the heavy lifting                      |
+------------------------------------------------------------+
```

### Phase 8: Departure

```
DAY +2 (May 17, 2026, 18:00)
+------------------------------------------------------------+
| Cargo operations complete:                                 |
|   Status = "Completed"                                     |
|        |                                                   |
|        v                                                   |
|   UC-009: Sailed Out Vessel                               |
|        |                                                   |
|        v                                                   |
| 19:00 - Vessel departs berth                               |
|   Marine Dept records:                                     |
|     - Sailed out time                                      |
|     - Final tonnage handled                                |
|        |                                                   |
|        v                                                   |
|   Status = "Departed"                                      |
|        |                                                   |
|        v                                                   |
| 19:15 - Vessel exits port                                  |
|        |                                                   |
|        v                                                   |
|   UC-012: Final charges calculated                         |
|        |                                                   |
|        v                                                   |
|   UC-015: Vessel Declaration Report generated              |
|        |                                                   |
|        v                                                   |
|   Status = "Closed" (after audit)                         |
+------------------------------------------------------------+
```

### Phase 9: Berth Available Again

```
+------------------------------------------------------------+
| NCT-7 berth released                                       |
|        |                                                   |
|        v                                                   |
| System automatically:                                      |
|   - Updates berth availability                             |
|   - Re-runs suggestion for queued vessels                  |
|   - Notifies relevant agents                               |
|        |                                                   |
|        v                                                   |
| Next vessel scheduled                                      |
+------------------------------------------------------------+
```

---

## 7. Data Model — কি Store হয়?

Module 1-এর core entities এবং তাদের relationships:

### Entity Relationship Overview

```
+----------------------------------------------------------------+
|                     Core Entities Map                           |
+----------------------------------------------------------------+

  +-------------------+
  | ShippingAgent     |
  +-------------------+
  | id (PK)           |
  | name              |
  | license           |
  | contact_info      |
  | active            |
  +---------+---------+
            |
            | (many vessels per agent)
            v
  +-------------------+        +-------------------+
  | Vessel            |<-------| VesselType        |
  +-------------------+        +-------------------+
  | vessel_id (PK)    |        | type_id (PK)      |
  | imo_number        |        | type_name         |
  | name              |        | (Container/Bulk/  |
  | radio_call_sign   |        |  Tanker/...)      |
  | length            |        +-------------------+
  | GRT, NRT          |
  | draught           |
  | flag              |
  | active            |
  +---------+---------+
            |
            | (each visit links to vessel)
            v
  +-------------------+
  | VesselVisit       |
  +-------------------+
  | visit_id (PK)     |
  | vessel_id (FK)    |
  | agent_id (FK)     |
  | imp_rotation_no   |
  | exp_rotation_no   |
  | ETA, ETD          |
  | ATA, ATD          |
  | status            |
  | sailed_out        |
  +---------+---------+
            |
            | (one berthing per visit)
            v
  +-------------------+        +-------------------+
  | BerthingApplication|<------| Berth             |
  +-------------------+        +-------------------+
  | berthing_id (PK)  |        | jetty_id (PK)     |
  | visit_id (FK)     |        | jetty_name        |
  | berth_id (FK)     |        | length            |
  | requested_at      |        | draught           |
  | suggested_at      |        | jetty_type        |
  | confirmed_at      |        | description       |
  | quota             |        | active            |
  | status            |        +-------------------+
  +-------------------+

  +-------------------+
  | TidalRecord       |
  +-------------------+
  | tide_id (PK)      |
  | date              |
  | time_period       |
  | day_name          |
  | tide_height_LW    |
  | tide_height_HW    |
  | upload_date       |
  +-------------------+

  +-------------------+
  | MarineCharge      |
  +-------------------+
  | charge_id (PK)    |
  | visit_id (FK)     |
  | charge_type       |
  | amount            |
  | charged_at        |
  +-------------------+

  +-------------------+
  | OverrideAudit     |
  +-------------------+
  | audit_id (PK)     |
  | berthing_id (FK)  |
  | original_berth    |
  | new_berth         |
  | reason            |
  | overridden_by     |
  | overridden_at     |
  +-------------------+
```

### Key Relationships Explained

```
+------------------------------------------------------------+
| Relationship              | Cardinality | Meaning           |
+------------------------------------------------------------+
| Agent -> Vessels          | 1:N         | Agent represents  |
|                           |             | many vessels      |
+------------------------------------------------------------+
| Vessel -> Visits          | 1:N         | Same vessel many  |
|                           |             | trips             |
+------------------------------------------------------------+
| Visit -> Berthing         | 1:1         | Each visit has    |
|                           |             | one berth         |
+------------------------------------------------------------+
| Berth -> Berthings        | 1:N         | Same berth used   |
|                           |             | by many visits    |
|                           |             | (over time)       |
+------------------------------------------------------------+
| Visit -> Charges          | 1:N         | Multiple charges  |
|                           |             | per visit         |
+------------------------------------------------------------+
| Berthing -> Override      | 1:0..1      | Optional override |
|                           |             | per berthing      |
+------------------------------------------------------------+
```

### Data Volume Estimates

```
+------------------------------------------------------------+
| Entity              | Records (Annual) | Growth Rate         |
+------------------------------------------------------------+
| Shipping Agents     | ~50 active       | +5/year            |
| Vessels             | ~500-700 unique  | +50/year           |
| Vessel Visits       | ~4,000           | linear             |
| Berthing Apps       | ~4,500           | linear             |
| Tidal Records       | ~2,920 (4/day)   | constant           |
| Marine Charges      | ~50,000          | linear             |
| Override Audits     | ~500             | linear             |
+------------------------------------------------------------+
```

---

## 8. Business Rules & Edge Cases

প্রতিটা enterprise system-এ business rules অনেক subtle। চলো Module 1-এর critical rules:

### Rule 1: Vessel-Berth Compatibility

```
+------------------------------------------------------------+
| Rule: A vessel can only berth at compatible berth          |
+------------------------------------------------------------+
| Constraints:                                               |
|   1. Berth length >= Vessel LOA + safety margin (10m)     |
|   2. Berth depth + min tide >= Vessel draft + 0.5m        |
|   3. Berth type compatible with vessel type               |
|                                                            |
| Edge cases:                                                |
|   - Vessel slightly too long: Manual override possible     |
|   - Marginal depth: Wait for higher tide                   |
|   - Special vessels (oil tanker): Only specialized berth   |
+------------------------------------------------------------+
```

### Rule 2: Tide Window Calculation

```
+------------------------------------------------------------+
| Rule: Big vessels need tide windows                        |
+------------------------------------------------------------+
| If vessel_draft > berth_depth:                             |
|     tide_needed = vessel_draft - berth_depth + 0.5         |
|                                                            |
| Find HW windows where tide_height >= tide_needed           |
|                                                            |
| Window duration: typically 2-3 hours around HW             |
|                                                            |
| Edge cases:                                                |
|   - No suitable HW for 24+ hours: Reject application       |
|   - Vessel arrival before window: Anchor and wait          |
|   - Vessel arrival after window: Wait for next HW          |
+------------------------------------------------------------+
```

### Rule 3: Berth Conflict Prevention

```
+------------------------------------------------------------+
| Rule: No two vessels at same berth at same time            |
+------------------------------------------------------------+
| Logic:                                                     |
|   For new application at Berth X, time T:                  |
|     Check existing schedules at Berth X                    |
|     If any vessel: arrival < T < departure                 |
|         CONFLICT - reject or queue                         |
|                                                            |
| Buffer time:                                               |
|   30-60 minutes between vessels                            |
|   For pilotage, mooring/unmooring                          |
+------------------------------------------------------------+
```

### Rule 4: Cancellation Cascading

```
+------------------------------------------------------------+
| Rule: Cancellation triggers re-evaluation                  |
+------------------------------------------------------------+
| When berthing cancelled:                                   |
|   1. Update berth as "available from cancellation time"    |
|   2. Find waiting applications for same/similar berth      |
|   3. Re-run suggestion algorithm                           |
|   4. Notify affected agents                                |
|                                                            |
| Edge cases:                                                |
|   - Cancelled while vessel approaching: Notify vessel      |
|   - Cancelled while alongside: Different process           |
|   - Multiple cancellations cascade: Avoid loops            |
+------------------------------------------------------------+
```

### Rule 5: Status Transitions

```
+------------------------------------------------------------+
| Rule: Strict state machine                                 |
+------------------------------------------------------------+
|                                                            |
|   Inbound -----> Arrived -----> Working                   |
|       |             |              |                       |
|       |             |              v                       |
|       |             |          Completed                   |
|       |             |              |                       |
|       v             v              v                       |
|   Cancelled    Cancelled        Departed                  |
|                                     |                      |
|                                     v                      |
|                                  Closed                    |
|                                                            |
| Forbidden transitions:                                     |
|   - Working -> Cancelled (vessel already at berth!)       |
|   - Departed -> Working (cannot reverse departure)        |
|   - Closed -> anything (audit locked)                     |
+------------------------------------------------------------+
```

### Rule 6: Agency Period Validity

```
+------------------------------------------------------------+
| Rule: Agent can only act during agency period              |
+------------------------------------------------------------+
| Check:                                                     |
|   For vessel V with agent A:                               |
|     A must have valid agency_period                        |
|     current_date BETWEEN period_from AND period_to         |
|                                                            |
| Edge cases:                                                |
|   - Agent change mid-visit: Update agency_period           |
|   - Expired agency: Renew before next vessel               |
|   - Multiple agents historical: Track per visit            |
+------------------------------------------------------------+
```

### Rule 7: Tidal Data Freshness

```
+------------------------------------------------------------+
| Rule: Tidal data must be current                           |
+------------------------------------------------------------+
| Check:                                                     |
|   For berthing on date D:                                  |
|     Tidal data must exist for D and D-1, D+1               |
|     Data uploaded within last 6 months                     |
|                                                            |
| If missing or stale:                                       |
|   Block auto-suggestion                                    |
|   Alert Marine Dept                                        |
|   Allow manual override only                               |
+------------------------------------------------------------+
```

### Common Edge Cases

```
+------------------------------------------------------------+
| Edge Case                | Handling                         |
+------------------------------------------------------------+
| Vessel arrives early     | Wait at anchorage                |
+------------------------------------------------------------+
| Vessel arrives late      | Lose berth slot, re-queue        |
+------------------------------------------------------------+
| Berth equipment failure  | Re-suggest different berth       |
+------------------------------------------------------------+
| Sudden weather event     | Mass cancellation cascade        |
+------------------------------------------------------------+
| Agent changes mid-visit  | Update record, audit             |
+------------------------------------------------------------+
| Vessel info incorrect    | Block, request correction        |
+------------------------------------------------------------+
| Two vessels same time    | Algorithm conflict, manual       |
+------------------------------------------------------------+
| VTMIS data unavailable   | Manual position entry            |
+------------------------------------------------------------+
| Application after ETA    | Special priority, manual         |
+------------------------------------------------------------+
| Override audit missing   | System block, require reason     |
+------------------------------------------------------------+
```

---

## 9. Integration Points — অন্য Modules এর সাথে

Module 1 isolated না — এটা DSTOS-এর **trigger module**। এর events থেকে অন্য modules-এর কাজ শুরু হয়।

### Integration Map

```
+----------------------------------------------------------------+
|                      Module 1 Integration Map                   |
+----------------------------------------------------------------+

                  +-------------------+
                  |    Module 1       |
                  | Vessel & Berthing |
                  +---------+---------+
                            |
        +-------------------+-------------------+
        |                   |                   |
        v                   v                   v
+---------------+   +---------------+   +---------------+
|   Module 2    |   |   Module 3    |   |   Module 5    |
| Pre-arrival & |   |   Equipment   |   |   Yard        |
|   Discharge   |   |   Control     |   |   Planning    |
+---------------+   +---------------+   +---------------+

Triggers from Module 1 to others:

  -> Vessel arrival confirmed:
       -> Module 2: Start pre-arrival processing
       -> Module 3: Allocate equipment
       -> Module 5: Reserve yard space

  -> Berthing cancelled:
       -> All modules: Roll back allocations

  -> Vessel sailed out:
       -> Module 2: Close vessel manifest
       -> Module 3: Release equipment
       -> Module 5: Clear yard reservations
```

### Detailed Integration with Module 2 (Pre-arrival & Discharge)

```
+------------------------------------------------------------+
| Module 1                  Module 2                         |
+------------------------------------------------------------+
| Vessel registered ----> Receive vessel basic info         |
| Visit created      ----> Track visit number               |
| Berthing confirmed ----> Start pre-arrival activities    |
|                          - Customs registration           |
|                          - EDI Baplie expected            |
|                          - IGM expected                   |
|                          - Berthing meeting scheduled     |
| Vessel arrived     ----> Visit phase change to "Arrived"  |
| Vessel berthed     ----> Begin discharge planning         |
| Vessel departing   ----> Finalize manifest                |
+------------------------------------------------------------+

Data shared:
   - vessel_visit_id (link key)
   - berth assignment
   - cargo information (preliminary from UC-010)
```

### Detailed Integration with Module 3 (Equipment Control)

```
+------------------------------------------------------------+
| Module 1                  Module 3                         |
+------------------------------------------------------------+
| Berthing confirmed ----> Equipment demand calculation     |
|                          - How many cranes needed         |
|                          - How many tractors              |
|                          - How many RTGs at yard          |
| ETA approaching    ----> Pre-position equipment           |
| Vessel berthed     ----> Allocate specific equipment      |
| Equipment failure  <---- Re-evaluate berthing if critical |
| Vessel departing   ----> Release equipment                |
+------------------------------------------------------------+

Data shared:
   - vessel_visit_id
   - berth location
   - cargo volume
   - vessel type (geared = own crane, gearless = need port crane)
```

### Detailed Integration with Module 5 (Yard Planning)

```
+------------------------------------------------------------+
| Module 1                  Module 5                         |
+------------------------------------------------------------+
| Berthing confirmed ----> Yard space pre-allocation       |
|                          - Reserve blocks                  |
|                          - Plan based on cargo type        |
| ETA approaching    ----> Pre-clear yard areas             |
|                          - Move existing if needed         |
| Vessel berthed     ----> Confirm yard allocation         |
| Cargo discharged   <---- Containers placed in yard        |
| Vessel departing   ----> Yard reservation released        |
+------------------------------------------------------------+

Data shared:
   - vessel_visit_id
   - cargo volume
   - cargo types (dry/reefer/hazmat)
   - berth location (proximity matters)
```

### Integration Patterns

```
+------------------------------------------------------------+
|              Module 1 Integration Patterns                 |
+------------------------------------------------------------+

Pattern 1: Event Broadcasting
   Module 1 publishes event ----> Multiple modules subscribe

   Events:
   - vessel.registered
   - berthing.confirmed
   - vessel.arrived
   - vessel.berthed
   - vessel.departed
   - berthing.cancelled

Pattern 2: Data Lookup
   Other modules query Module 1 ----> Get current vessel status

Pattern 3: Update Notifications
   Module 1 status changes ----> Notify subscribers

   Status changes:
   - Inbound -> Arrived
   - Arrived -> Working
   - Working -> Completed
   - Completed -> Departed
```

---

## 10. External System Dependencies

Module 1 multiple external systems-এর সাথে integrate করে।

### External System: VTMIS / AIS

```
+------------------------------------------------------------+
| VTMIS = Vessel Traffic Management Information System       |
| AIS = Automatic Identification System                      |
+------------------------------------------------------------+
| Provided by: Bangladesh Port Authority                     |
| Purpose: Real-time vessel tracking                         |
| Interface: XML data exchange (pre-defined schema)         |
| Frequency: Continuous (every 30 seconds typically)        |
+------------------------------------------------------------+
| Data received:                                             |
|   - Vessel identification (IMO, MMSI)                      |
|   - Position (latitude, longitude)                         |
|   - Speed and heading                                      |
|   - Status (sailing, anchored, moored)                    |
|   - ETA refinement                                         |
+------------------------------------------------------------+
| DSTOS uses to:                                             |
|   - Update ATA (actual time of arrival)                   |
|   - Refine ETA dynamically                                 |
|   - Confirm vessel position before berthing               |
|   - Verify departure                                       |
+------------------------------------------------------------+
```

### External System: Weather/Tidal Service

```
+------------------------------------------------------------+
| Source: Bangladesh Meteorological Department               |
| Channel: Port Community Portal                             |
| Frequency: Every 6 months bulk upload                      |
| Format: Excel file                                         |
+------------------------------------------------------------+
| Data received:                                             |
|   - Tidal predictions for next 6 months                    |
|   - Per-day high water and low water times                 |
|   - Tide heights                                           |
+------------------------------------------------------------+
| DSTOS uses to:                                             |
|   - Berthing window calculation                            |
|   - Vessel sailing recommendations                         |
+------------------------------------------------------------+
```

### External System: Web Community Portal

```
+------------------------------------------------------------+
| Type: Self-service web portal                              |
| Users: Shipping agents (external)                          |
| Hosting: Same DSTOS infrastructure (different UI)          |
+------------------------------------------------------------+
| Functions:                                                 |
|   - Agent self-registration                                |
|   - Vessel info submission                                 |
|   - Berthing application                                   |
|   - Status tracking                                        |
|   - Document upload                                        |
+------------------------------------------------------------+
| Security:                                                  |
|   - Separate authentication from internal                  |
|   - Role-based access (agent only sees own vessels)        |
|   - HTTPS required                                         |
|   - Audit logging                                          |
+------------------------------------------------------------+
```

---

## 11. Failure Scenarios — কি ভুল হতে পারে?

Production system-এ failures inevitable। চলো critical scenarios এবং handling:

### Scenario 1: VTMIS Data Unavailable

```
+------------------------------------------------------------+
| What happens:                                              |
|   VTMIS service down or network issue                      |
|   No real-time vessel position                             |
+------------------------------------------------------------+
| Impact:                                                    |
|   - Cannot auto-detect vessel arrival                      |
|   - ETA refinement stuck                                   |
|   - Position-based decisions blocked                       |
+------------------------------------------------------------+
| Mitigation:                                                |
|   1. Manual position entry by Radio Control                |
|   2. Use last known position                               |
|   3. Phone call to vessel for position                     |
|   4. Continue operations with manual updates               |
|   5. Reconcile when VTMIS restored                         |
+------------------------------------------------------------+
```

### Scenario 2: Tidal Data Missing

```
+------------------------------------------------------------+
| What happens:                                              |
|   Tidal data not uploaded for upcoming dates               |
+------------------------------------------------------------+
| Impact:                                                    |
|   - Auto-suggestion algorithm cannot run                   |
|   - Risky for deep-draft vessels                           |
+------------------------------------------------------------+
| Mitigation:                                                |
|   1. Alert Marine Dept urgently                            |
|   2. Block auto-suggestion until data uploaded             |
|   3. Manual override allowed with reason                   |
|   4. Use backup tidal source if available                  |
|   5. Conservative scheduling (high tide assumed)           |
+------------------------------------------------------------+
```

### Scenario 3: Concurrent Berthing Conflict

```
+------------------------------------------------------------+
| What happens:                                              |
|   Two officers simultaneously assign same berth            |
+------------------------------------------------------------+
| Impact:                                                    |
|   - Database constraint violation                          |
|   - Inconsistent state                                     |
+------------------------------------------------------------+
| Mitigation:                                                |
|   1. Database-level unique constraints                     |
|   2. Optimistic locking on schedule updates                |
|   3. Proper transaction isolation                          |
|   4. UI lock during edit                                   |
|   5. Real-time conflict notification                       |
+------------------------------------------------------------+
```

### Scenario 4: Wrong Vessel Specs

```
+------------------------------------------------------------+
| What happens:                                              |
|   Agent submitted wrong draft (12m instead of 14m)        |
|   Vessel cannot fit assigned berth                         |
+------------------------------------------------------------+
| Impact:                                                    |
|   - Vessel arrives, cannot enter                           |
|   - Demurrage cost                                         |
|   - Schedule disruption                                    |
+------------------------------------------------------------+
| Mitigation:                                                |
|   1. Validate vessel specs against IMO database            |
|   2. Cross-check with previous visits                      |
|   3. Captain must confirm specs before entry               |
|   4. Emergency reschedule procedure                        |
|   5. Penalty/audit for misinformation                      |
+------------------------------------------------------------+
```

### Scenario 5: Cascade Cancellations

```
+------------------------------------------------------------+
| What happens:                                              |
|   Major weather event, multiple cancellations              |
|   System struggling to re-suggest                          |
+------------------------------------------------------------+
| Impact:                                                    |
|   - Many waiting vessels                                   |
|   - Re-suggestion algorithm under load                     |
|   - Notification storm                                     |
+------------------------------------------------------------+
| Mitigation:                                                |
|   1. Emergency mode: Pause auto-suggestions                |
|   2. Manual prioritization by Marine Dept                  |
|   3. Batch notification to agents                          |
|   4. Phased re-scheduling                                  |
|   5. Communication coordination                            |
+------------------------------------------------------------+
```

---

## 12. Implementation Considerations

Building this module-এর সময় যা মাথায় রাখতে হবে:

### Technical Considerations

```
+------------------------------------------------------------+
| Performance                                                |
+------------------------------------------------------------+
| - Auto-suggestion algorithm: <2 seconds response          |
| - Berthing schedule queries: indexed by date, berth       |
| - VTMIS data ingestion: handle 100s msg/sec               |
| - Concurrent users: 50-100 typical                         |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Data Integrity                                             |
+------------------------------------------------------------+
| - Strict foreign keys                                      |
| - Transaction boundaries for critical ops                  |
| - Audit trail for all changes                              |
| - Soft delete (never hard delete)                          |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Security                                                   |
+------------------------------------------------------------+
| - Role-based access (Marine, Traffic, Agent)              |
| - Field-level permissions                                  |
| - Web portal vs internal isolation                         |
| - All write actions audited                                |
| - Sensitive data (charges) encrypted at rest               |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Reliability                                                |
+------------------------------------------------------------+
| - 24/7 operation (port never sleeps)                      |
| - High availability (active-passive at minimum)            |
| - Backup every hour                                        |
| - Disaster recovery plan                                   |
| - Manual fallback procedures                               |
+------------------------------------------------------------+
```

### Business Considerations

```
+------------------------------------------------------------+
| User Experience                                            |
+------------------------------------------------------------+
| - Marine Dept: Quick decision interface                    |
| - Agents: Self-service, simple workflow                    |
| - Mobile-friendly for field operations                     |
| - Multilingual (Bangla + English)                          |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Compliance                                                 |
+------------------------------------------------------------+
| - SOLAS regulations (vessel safety)                        |
| - Port authority regulations                               |
| - Customs requirements                                     |
| - Audit requirements                                       |
| - Data retention (7+ years typical)                        |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Operational                                                |
+------------------------------------------------------------+
| - Training program for users                               |
| - Standard operating procedures                            |
| - Escalation procedures                                    |
| - Help desk integration                                    |
| - Change management                                        |
+------------------------------------------------------------+
```

### Phased Implementation Recommendation

```
+------------------------------------------------------------+
| Phase 1 (Months 1-2): Foundation                          |
+------------------------------------------------------------+
| - Master data (UC-001, 002, 003, 004)                     |
| - Manual berth assignment                                  |
| - Basic reporting (UC-014)                                 |
| - User authentication & roles                              |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Phase 2 (Months 2-3): Core Workflow                       |
+------------------------------------------------------------+
| - Berthing application (UC-005)                            |
| - Manual approval workflow                                 |
| - Cancel/sail out (UC-008, 009)                            |
| - Status tracking                                          |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Phase 3 (Months 3-4): Intelligence                        |
+------------------------------------------------------------+
| - Auto-suggestion algorithm (UC-006)                       |
| - Override workflow (UC-007)                               |
| - Audit logging                                            |
| - Advanced reporting (UC-015)                              |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Phase 4 (Months 4-5): Integration                         |
+------------------------------------------------------------+
| - VTMIS integration                                        |
| - Web Community Portal                                     |
| - Notification system                                      |
| - Charges (UC-012)                                         |
+------------------------------------------------------------+

+------------------------------------------------------------+
| Phase 5 (Months 5-6): Module 2-5 Integration              |
+------------------------------------------------------------+
| - Event publishing to other modules                        |
| - Cascade workflows                                        |
| - End-to-end testing                                       |
| - Performance optimization                                 |
+------------------------------------------------------------+
```

---

## 🎯 Summary — Key Takeaways

Module 1 হলো DSTOS-এর **ignition system** — পুরো port operation এই module থেকে trigger হয়।

### এক কথায় Module 1

> **"কোন ship কোন berth-এ কখন park করবে — এই decision systematically নেওয়ার system।"**

### Mental Model

```
Module 1 জিজ্ঞেস করে এবং উত্তর দেয়:

   "Ship আসছে?"           -> Vessel master data
   "কে আনছে?"             -> Shipping agent
   "কত বড় ship?"          -> Vessel specs
   "কোথায় park করবে?"     -> Berth selection
   "Tide ঠিক আছে?"         -> Tidal data
   "অন্য কেউ ছিল কি?"      -> Berth schedule
   "কখন আসবে?"             -> ETA tracking
   "কখন গেলো?"             -> Sail out record
   "কত charge?"            -> Marine charges
```

### Critical Success Factors

```
+------------------------------------------------------------+
| 1. Master data accuracy                                    |
|    - Wrong berth specs = wrong assignments                 |
|    - Wrong vessel data = scheduling chaos                  |
+------------------------------------------------------------+
| 2. Algorithm tuning                                        |
|    - Must match real port operations                       |
|    - Iterative improvement based on Marine Dept feedback   |
+------------------------------------------------------------+
| 3. Integration reliability                                 |
|    - Other modules depend on Module 1 events               |
|    - Failures cascade                                      |
+------------------------------------------------------------+
| 4. Audit completeness                                      |
|    - Port operations heavily regulated                     |
|    - Every decision must be traceable                      |
+------------------------------------------------------------+
| 5. User adoption                                           |
|    - Marine Dept must trust system suggestions             |
|    - Agents must find portal easy                          |
+------------------------------------------------------------+
```

### What's Next?

Module 1 master হলে natural progression:
- **Module 2:** Pre-arrival এবং Discharge planning — Module 1-এর events থেকে trigger হয়
- Then Module 5 (Yard Planning) — Allocation decisions
- Then Module 3 (Equipment) — Resource deployment

প্রতিটা module previous-এর foundation-এর উপর build হয়।

---

**Document Version:** 1.0  
**Module:** 1 of 6  
**For:** Hasibuzzaman, DataSoft Systems Bangladesh Limited