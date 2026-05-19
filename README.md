# Functional Specification: Court Reserve Booking Platform

## 1. Product Overview

**Court Reserve** is a configurable booking platform for sports facilities that manage court reservations, fixed class packages, memberships, and punch cards. **Policy numbers** (group size, booking windows, cancel notice, late threshold, and so on) are **per-tenant settings** (see §4.0.1); examples in this document use common defaults.

The product supports two major booking scenarios:

1. **Class package registration**  
   Members register for a `ClassPackage` with a fixed set of class dates and times. The same product type supports **weekly group lessons** (e.g. every Wednesday for 4 weeks) and **summer camp** (e.g. every weekday for 1–2 weeks at the same daily time) via a **schedule policy** — not a separate camp product.

2. **Practice court booking**  
   Members book a court for personal practice via **`PracticeBooking`** + **`Payment`** (commerce), which creates a **`PRACTICE` Session** on the calendar when confirmed. Price and payment are **not** stored on `Session`.

The system uses **`Session`** for all calendar reservations (class and practice).

A `Session` is the **operational** slot (member, court, time, status) — **no price or payment fields**.

A session can be:

```text
CLASS session
- Created from a class package registration (commerce on PackageRegistration / Payment)
- Subtypes:
  - GROUP_CLASS — shared class with multiple members (e.g. group lesson, summer camp block)
  - PRIVATE_CLASS — one member (or one household) with the coach

PRACTICE session
- Created when a PracticeBooking is confirmed (commerce on PracticeBooking / Payment)
- Linked via practiceBookingId
```

---

## 2. Business Goals

The platform should help a sports facility:

- Sell fixed class packages (group and private).
- Run seasonal programs such as Summer Camp (July–August).
- Allow members to register before or after a package starts.
- Support prorated pricing for late registration.
- Assign members to courts/resources for each class.
- Allow members to book practice courts within each tenant’s **practice booking window** (outside of classes).
- Support payment by one-time payment, punch card, or membership.
- Prevent court/resource double-booking.
- Enforce per-tenant **maximum concurrent resources**, **session slot length**, and **policy thresholds** (all tenant-configurable).
- Run group lessons within each tenant’s configured **min/max group size**.
- Track attendance, cancellations, payment status, and session history.
- Support multiple tenants/facilities in the future.

---

## 3. User Roles

### 3.1 Admin / Facility Manager

The admin manages business operations.

Admin can:

- Create and manage class packages (group and private).
- Create summer camp `ClassPackage` rows (`DAILY_CAMP`, July/August).
- Create concrete class instances inside a package.
- Manage instructors (coaches) and their **weekly availability schedules** in fixed slot format (§4.8) **before** publishing class packages.
- When creating a class package, **assign a coach** by choosing from instructors who already cover every generated instance (`coachCoversPackage` — §4.8.4); on assign, the system **reserves those slots** via `PackageCoachHold` (§4.8.6). No separate coach acceptance step.
- Manage courts/resources and **tenant policy settings** (scheduling, group size, booking/cancel windows, late threshold).
- Register members for packages.
- View package registrations.
- View class attendance.
- View court bookings.
- Create punch card products.
- Create membership plans.
- Manage payment and booking status.
- Cancel or **update (reschedule)** practice court bookings and sessions (§6.5).
- Cancel class sessions per product rules (group/private).

### 3.2 Member / Customer

The member uses the platform to book and manage activities.

Member can:

- View available class packages with coach name and information.
- Register for a group or private class package; for private packages without a fixed coach, use **match-instructors** to choose a coach (§5.6).
- Register late for an active package if allowed.
- Book practice courts only within the tenant’s **practice booking window** (`practiceBookingMinLeadHours`–`practiceBookingMaxLeadHours`).
- Use punch card credits.
- Use membership benefits.
- View upcoming sessions.
- Cancel **practice court** sessions per tenant **`practiceCancelNoticeHours`**.
- **Update (reschedule)** a confirmed practice booking — change court, date, or slot — per §6.5.
- Cancel **private class** sessions per tenant **`privateClassCancelNoticeHours`**.
- **Cannot** cancel individual **group class** sessions or registrations after purchase (dates are fixed; no refund).
- View booking/payment history.

### 3.3 Instructor

The instructor teaches classes.

Instructor can:

- View and (if permitted) update own **weekly availability** (§4.8).
- **Claim** open class packages when the facility allows instructor self-assignment (must pass coach match rules in §4.8.4).
- View assigned class instances.
- View members attending each class.
- View assigned courts/resources.
- Mark attendance.
- Mark no-shows.

---

## 4. Core Product Concepts

### 4.0 Tenant

A `Tenant` represents one sports facility (or business) on the platform.

**All policy numbers in this specification are tenant-configurable** unless noted as derived (e.g. `sessionCount` from schedule policy). The document uses **default values** as examples; each facility sets its own values in **tenant settings**. Application code must read **`tenant.*`** (or `tenant_policies`) at booking, registration, confirmation, and cancellation time — not hardcoded constants.

#### 4.0.1 Tenant configuration (reference)

| Setting | Field | Default (example) | Used for |
|--------|--------|-------------------|----------|
| Max concurrent courts | `maxResources` | 6 | §10.2 capacity |
| Session slot length | `sessionSlotMinutes` | 30 | §4.6 slot grid, camp/lesson instance split |
| Facility open | `facilityOpenTime` | 06:00 | Slot grid alignment |
| Facility close | `facilityCloseTime` | 22:00 | Slot grid alignment |
| Group min to run | `groupMinMembers` | 2 | §5.5 T-N confirmation, `PACKAGE_FULL` floor |
| Group max enrollment | `groupMaxMembers` | 4 | Registration cap |
| Group confirm lead time | `groupConfirmationLeadHours` | 24 | §5.5 confirm/cancel job before first class |
| Practice book — earliest | `practiceBookingMinLeadHours` | 2 | §6.1 minimum advance booking |
| Practice book — latest | `practiceBookingMaxLeadHours` | 24 | §6.1 maximum advance booking |
| Practice cancel notice | `practiceCancelNoticeHours` | 2 | §11.3 practice cancellation / refund |
| Private class cancel notice | `privateClassCancelNoticeHours` | 24 | §11.2 private cancellation |
| Class late threshold | `classLateThresholdMinutes` | 20 | §12.2 late forfeiture |

```text
Tenant (scheduling + policies)
  maxResources
  sessionSlotMinutes
  facilityOpenTime
  facilityCloseTime
  groupMinMembers
  groupMaxMembers
  groupConfirmationLeadHours
  practiceBookingMinLeadHours
  practiceBookingMaxLeadHours
  practiceCancelNoticeHours
  privateClassCancelNoticeHours
  classLateThresholdMinutes
```

**`maxResources`** — cap on concurrent sessions facility-wide (typically equals physical court count).

**`sessionSlotMinutes`** — every `Session` and bookable slot uses this duration for the tenant. Members cannot book arbitrary ranges (e.g. 10:15–10:35 when slot is 30 minutes); they book **aligned slots**.

**Examples (one tenant’s settings):**

```text
Tenant: Downtown Tennis Club
maxResources: 6
sessionSlotMinutes: 30
groupMinMembers: 2
groupMaxMembers: 4
practiceBookingMinLeadHours: 2
practiceBookingMaxLeadHours: 24
```

**Admin** configures all tenant policy fields when onboarding or updating the facility. **Class packages** use the tenant’s group settings at publish and confirmation time (values may be **snapshotted** on the package record for audit, but must match tenant policy unless the product later adds explicit per-package overrides).

**Validation:** The system should reject invalid tenant config (e.g. `groupMinMembers > groupMaxMembers`, `practiceBookingMinLeadHours > practiceBookingMaxLeadHours`).

---

### 4.1 ClassPackage

A `ClassPackage` is the class product being sold.

Example:

```text
Beginner Tennis 4-Class Package
Every Wednesday, 7:00 PM - 8:00 PM
May 1 - May 22
Price: $200
```

A class package includes package-level schedule information:

```text
startDate
endDate                   (optional for camp — may be derived from startDate + durationWeeks)
startTime
endTime                   (same clock time on each occurrence day; half-day vs full-day = length of window)
schedulePolicy            (how ClassInstances are generated — see below)
programType               (STANDARD | SUMMER_CAMP — catalog/reporting tag; summer camp uses SUMMER_CAMP)
durationWeeks             (1 | 2 — for DAILY_CAMP; optional for WEEKLY_RECURRING)
campWeekdays              (for DAILY_CAMP — default Mon–Fri)
dayOfWeek                 (for WEEKLY_RECURRING — e.g. Wednesday)
sessionCount              (derived from schedulePolicy — admin may preview, system calculates)
price
classSessionType          (GROUP_CLASS | PRIVATE_CLASS)
coachId                   (assigned instructor — optional until match/claim; see §4.8, §5.6)
coachName                 (display name)
coachInformation          (bio, credentials, photo URL, or short profile text)
allowLateRegistration
prorateLateRegistration
minMembersToRun           (group — from tenant.groupMinMembers; may snapshot on package)
maxMembers                (group — from tenant.groupMaxMembers; may snapshot on package)
confirmationLeadTimeHours (group — from tenant.groupConfirmationLeadHours; may snapshot on package)
```

#### Schedule policy (same ClassPackage, different session definitions)

Summer camp does **not** use a separate product class. It is a `ClassPackage` with a different **`schedulePolicy`** that defines how `ClassInstance` rows are built.

```text
schedulePolicy:
  WEEKLY_RECURRING   — group/private series: same day of week, same time, once per week
  DAILY_CAMP         — summer camp: every camp weekday in range, same time each day
```

| Policy | Typical use | How instances are placed |
|--------|-------------|---------------------------|
| `WEEKLY_RECURRING` | 4-week group lesson, private series | Same **day of week** + **startTime/endTime**, spaced **7 days** apart |
| `DAILY_CAMP` | Summer camp (July–August), 1–2 weeks | Every **weekday** in camp range + **same daily startTime/endTime** |

**Half-day vs full-day** (camp) is defined by the package’s daily **`startTime` and `endTime`**. The system splits that range into **consecutive `sessionSlotMinutes` windows** — one `ClassInstance` (and one `Session` per member) per slot (see §4.6, §5.2.2).

```text
Half-day AM example:  09:00 – 12:00  → 6 slots @ 30 min, or 3 slots @ 60 min
Half-day PM example:  13:00 – 16:00
Full-day example:     09:00 – 16:00  → 14 slots @ 30 min, or 7 slots @ 60 min (minus lunch block if configured)
```

**Weekly lessons** use one or more consecutive slots per occurrence (e.g. one 60-minute slot, or two 30-minute slots); `startTime`/`endTime` on the package must align to the tenant slot grid.

**Group lesson enrollment limits (from tenant):**

```text
minMembersToRun = tenant.groupMinMembers
maxMembers      = tenant.groupMaxMembers
confirmationLeadTimeHours = tenant.groupConfirmationLeadHours
```

The package itself does not represent one member’s registration. It is the product definition.

**Group package run status** (facility-level, not per member):

```text
OPEN_FOR_REGISTRATION     — accepting paid sign-ups before confirmation checkpoint
PENDING_CONFIRMATION      — at least one paid member; awaiting confirmation job
CONFIRMED                 — confirmation job passed (paid count ≥ tenant.groupMinMembers); class will run
FULL                      — paid count = tenant.groupMaxMembers; no further registrations
CANCELLED_INSUFFICIENT    — below min at checkpoint; cancelled and registrants refunded
```

**Group vs private packages**

| Aspect | Group Class Package | Private Class Package |
|---|---|---|
| `classSessionType` | `GROUP_CLASS` | `PRIVATE_CLASS` |
| Enrollment | **tenant min–max** (defaults 2–4); **system confirms** per `groupConfirmationLeadHours` before first session | Usually 1 (member + coach) |
| Minimum to run | `tenant.groupMinMembers` at confirmation job | 1 paid member |
| Maximum enrollment | `tenant.groupMaxMembers` (`PACKAGE_FULL` when reached) | Per product config |
| Coach | Required on package **or** assigned via match/claim (§4.8, §5.6) | **Fixed coach** on package **or** member selects via match (§5.6) |
| Coach matching | Same availability rules as private when assigning coach at publish or claim | **§5.6** — validate coach covers all instances before pay |
| Schedule | `WEEKLY_RECURRING` or `DAILY_CAMP` | Usually `WEEKLY_RECURRING` |

---

### 4.2 ClassInstance

A `ClassInstance` is one concrete class occurrence inside a package. Instances are **generated from the package’s `schedulePolicy`** (system-built or admin-triggered); they are not a different product type for camp vs weekly lessons.

**Weekly recurring example** (`schedulePolicy = WEEKLY_RECURRING`):

```text
ClassInstance 1: May 1 (Wed), 7:00 PM - 8:00 PM
ClassInstance 2: May 8 (Wed), 7:00 PM - 8:00 PM
ClassInstance 3: May 15 (Wed), 7:00 PM - 8:00 PM
ClassInstance 4: May 22 (Wed), 7:00 PM - 8:00 PM
```

**Daily camp example** (`DAILY_CAMP`, half-day 9:00–12:00, `sessionSlotMinutes = 30`):

```text
Per camp day (Mon): 6 ClassInstances — 9:00–9:30, 9:30–10:00, …, 11:30–12:00
1 week Mon–Fri → 5 × 6 = 30 ClassInstances for the package
Each registered member → 30 CLASS Sessions (one per instance)
```

**Daily camp example** (`DAILY_CAMP`, half-day 9:00–12:00, `sessionSlotMinutes = 60`):

```text
Per camp day: 3 ClassInstances — 9:00–10:00, 10:00–11:00, 11:00–12:00
1 week Mon–Fri → 15 ClassInstances; ~4–5 slots per day if daily window is slightly longer (e.g. 9:00–12:30)
```

Relationship:

```text
One ClassPackage has many ClassInstances (each instance = exactly one sessionSlotMinutes window).
sessionCount = count(ClassInstances)  (derived when instances are generated)
```

Each `ClassInstance` `startDateTime`/`endDateTime` span **exactly `tenant.sessionSlotMinutes`**. Registration, sessions, coach, capacity, and tenant confirmation rules use the same flows for all policies.

---

### 4.3 PackageRegistration

A `PackageRegistration` represents one member’s registration for one class package.

It tracks:

```text
member
package
registration date
coachId                     (instructor for this registration — from package or assignment)
coachName                   (snapshot for receipts and member UI)
coachInformation            (snapshot or live link to instructor profile)
original class count
registered class count
skipped class count
original price
final price
payment status
registration status
```

Coach fields on the registration ensure members see who will teach their series even if the facility later reassigns instructors on the catalog package. Snapshots are recommended at confirmation time.

Example:

```text
Member A registers before the package starts:
originalClassCount = 4
registeredClassCount = 4
finalPrice = $200

Member B registers late after 2 classes have passed:
originalClassCount = 4
registeredClassCount = 2
skippedClassCount = 2
finalPrice = $100
```

---

### 4.4 Session

A `Session` is the **operational** reserved slot on the calendar (who, which court, when). It does **not** store price, payment status, or Stripe identifiers.

It contains:

```text
member
resource / court
startDateTime
endDateTime
sessionType                 (CLASS | PRACTICE)
classSessionSubtype         (GROUP_CLASS | PRIVATE_CLASS — only when sessionType = CLASS)
coachId                     (CLASS only)
coachName                   (CLASS only)
practiceBookingId           (PRACTICE only — FK to commerce record; set when session is created)
status
attendanceStatus
checkInTime                 (optional — used for late attendance policy)
```

**Explicitly not on Session:** `amount`, `listedPrice`, `paymentStatus`, `stripeTransactionId`, `punchCardId`, `membershipId`. Those live on **`PracticeBooking`**, **`PackageRegistration`**, **`Order`**, and **`Payment`**.

There are two top-level session types:

```text
CLASS
PRACTICE
```

Each `CLASS` session also has a **class session subtype**:

```text
GROUP_CLASS
PRIVATE_CLASS
```

The subtype is inherited from the class package at creation time.

#### CLASS Session (Group or Private)

A `CLASS` session is created from a package registration (weekly or camp package).

**Group Class Session** — member attends a shared class with other registrants.

Example:

```text
Member A
Beginner Tennis ClassInstance 1
classSessionSubtype: GROUP_CLASS
Coach: Jane Smith
Court 1
May 1, 7:00 PM - 8:00 PM
```

**Private Class Session** — member has dedicated time with the coach.

Example:

```text
Member B
Private Tennis Lesson 1
classSessionSubtype: PRIVATE_CLASS
Coach: John Lee
Court 2
May 3, 4:00 PM - 5:00 PM
```

For a 4-class package, one member registration creates multiple `CLASS` sessions (all with the same subtype and coach as the package/registration).

#### PRACTICE Session

A **`PRACTICE` session** is still a **`Session`** — the court reservation on the schedule. It is created when a **practice booking** is successfully completed (payment confirmed or entitlement applied).

Commerce (price, how it was paid) is stored on **`PracticeBooking`** + **`Payment`**, not on the session.

Example:

```text
Member A
Court 3
May 10, 6:00 PM - 7:00 PM
sessionType: PRACTICE
practiceBookingId: PB-123
(no price fields on Session)
```

This has no relationship to a class package or class instance. See **§4.7 PracticeBooking**.

**CLASS sessions** follow the same principle: commercial data on **`PackageRegistration`** / **`Order`** / **`Payment`**; the session row is scheduling and attendance only.

---

### 4.5 Resource

A `Resource` represents a bookable facility asset.

Examples:

```text
Tennis Court 1
Tennis Court 2
Badminton Court 3
Pickleball Court A
```

A resource can be used by both class sessions and practice sessions.

Main rule:

```text
A resource cannot be double-booked for overlapping active sessions.
```

---

### 4.6 Session Time Slots (Tenant Grid)

All sessions — **CLASS** and **PRACTICE** — must use the tenant’s fixed slot length:

```text
sessionDuration = tenant.sessionSlotMinutes
slotEnd = slotStart + sessionDuration
```

**Slot alignment rule:** `slotStart` must fall on the facility’s slot grid (derived from `facilityOpenTime` and `sessionSlotMinutes`). Requests with non-aligned start/end times are rejected (`INVALID_TIME_SLOT`).

**Why:** Discretizing time removes ambiguous overlaps (e.g. 10:15–10:35 vs 10:00–10:30). Conflict checks compare **identical or overlapping slot windows** on the same resource; members book **court + slot**, not arbitrary intervals.

**Camp slot count per member per day:**

```text
slotsPerCampDay = floor((campDayEnd - campDayStart) / sessionSlotMinutes)
```

| Camp day window | `sessionSlotMinutes` | Instances / sessions per member per day |
|-----------------|----------------------|----------------------------------------|
| Half-day 9:00–12:00 (3 h) | 30 | **6** |
| Half-day 9:00–12:00 (3 h) | 60 | **3** (often described as ~4–5 if window is 4–4.5 h) |
| Full-day 9:00–16:00 (7 h) | 30 | **14** |
| Full-day 9:00–16:00 (7 h) | 60 | **7** |

**Practice booking:** Member selects **resource**, **date**, and a **slot start time** from available aligned slots (not a free-form range). System creates one `PRACTICE` session for that single window.

**Weekly class:** Package `startTime`/`endTime` must equal one slot or an integer number of consecutive slots; each `ClassInstance` is **one slot** (same as camp).

**Coach availability:** `InstructorAvailability` uses the **same slot grid** as courts and classes (§4.8). Admin defines when a coach can teach using **aligned slot boundaries**, not arbitrary minute ranges.

---

### 4.7 PracticeBooking (Commerce for Practice Courts)

A **`PracticeBooking`** is the **commercial record** for one practice court checkout. It exists so **`Session`** stays free of payment fields.

**Practice is still a `Session` on the calendar; `PracticeBooking` is how that session was sold.**

```text
PracticeBooking
  member
  resource
  startDateTime / endDateTime     (intended slot; aligned to sessionSlotMinutes)
  listedPrice
  chargedAmount
  paymentSource                   (DIRECT | PUNCH_CARD | MEMBERSHIP)
  punchCardId                     (optional)
  membershipId                    (optional)
  status                          (PENDING_PAYMENT | CONFIRMED | CANCELLED | REFUNDED)
  sessionId                       (set when PRACTICE Session is created — after pay/confirm)
```

```text
Payment
  practiceBookingId               (FK — not sessionId for money)
  amount
  listedAmount
  paymentProvider
  status                          (PENDING | PAID | FAILED | REFUNDED | ...)
  transactionId
```

Optional **`Order`** may wrap `PracticeBooking` for reporting (`orderType = PRACTICE_BOOKING`).

#### Separation of concerns

| Concern | Model |
|--------|--------|
| Calendar, conflicts, `maxResources`, cancellation window | **`Session`** (`PRACTICE`) |
| Checkout, price, entitlement used, booking status | **`PracticeBooking`** |
| Stripe charge, refund, payment status | **`Payment`** |

#### What goes where

| Field | Session | PracticeBooking | Payment |
|--------|---------|-----------------|---------|
| Court, start, end | ✓ | ✓ (until `sessionId` set) | |
| Member | ✓ | ✓ | |
| Listed / charged price | | ✓ | ✓ (authoritative for Stripe) |
| Punch card / membership used | | ✓ (`paymentSource`) | |
| Refund / transaction id | | | ✓ |

---

### 4.8 Instructor (Coach) Availability Schedule

Facilities need a **coach schedule** so the system can answer: *“Can this instructor teach every session in this package?”* That applies to **private and group** class packages whenever a coach is assigned, matched, or claimed.

#### 4.8.0 Entity: `CoachSchedule` (slot-level `AVAILABLE` | `BOOKED`)

The **`CoachSchedule`** is the canonical **per-slot** model for a coach’s calendar. Each row is one aligned slot on the tenant grid (§4.6) for one instructor. Weekly templates live in **`InstructorAvailability`** (§4.8.1); **`CoachSchedule`** is the **materialized bookable state** for concrete dates.

```text
CoachSchedule
  tenantId
  instructorId              (coach — User with role INSTRUCTOR)
  slotStartDateTime           (facility local time, slot-aligned)
  slotEndDateTime             (slotStart + tenant.sessionSlotMinutes)
  status                      (AVAILABLE | BOOKED)
  classPackageId              (nullable — set when slot is held for a package)
  classInstanceId             (nullable — FK to ClassInstance when known)
  packageRegistrationId       (nullable — set when hold is tied to a specific registration checkout)
  packageRunStatus            (nullable — snapshot of ClassPackage run status when BOOKED:
                                 PENDING_CONFIRMATION | CONFIRMED | OPEN_FOR_REGISTRATION)
  updatedAt
```

**Status semantics:**

| `status` | Meaning |
|----------|---------|
| `AVAILABLE` | Slot lies inside weekly `InstructorAvailability` and is not blocked by another package, registration checkout, or active `CLASS` session |
| `BOOKED` | Slot is reserved for a `ClassPackage` and/or `PackageRegistration`; other packages and match flows must not use it |

**Materialization:** When admin saves `InstructorAvailability`, the system upserts future `CoachSchedule` rows for each aligned slot in the window with `status = AVAILABLE` (unless already `BOOKED`). Past slots may be omitted or archived per implementation.

**Uniqueness:** `(tenantId, instructorId, slotStartDateTime)`.

**Validation helper** (used at package create and private registration):

```text
function validateCoachScheduleForPackage(instructorId, classInstances[], excludePackageId = null):
  for each instance in classInstances:
    schedule = CoachSchedule for (tenant, instructorId, instance.slotStart)
    if schedule is missing OR schedule.status != AVAILABLE:
      if schedule.status == BOOKED
         AND schedule.classPackageId == excludePackageId:
        continue   // re-validate same package on edit
      return false
    if NOT slotInsideWeeklyAvailability(instructorId, instance.start, instance.end):  // §4.8.2
      return false
    if overlapping active CLASS Session for instructor (§4.8.3):
      return false
  return true
```

`validateCoachScheduleForPackage` is equivalent to **`coachCoversPackage`** (§4.8.4) when `CoachSchedule` is kept in sync with holds and sessions.

**Update `CoachSchedule` when package or registration state changes:**

| Trigger | `CoachSchedule` update |
|---------|-------------------------|
| Admin creates/publishes package with `coachId` | After instances generated: run `validateCoachScheduleForPackage`; on success set matching rows → `BOOKED`, set `classPackageId`, `classInstanceId`, `packageRunStatus` from package |
| Admin clears or changes `coachId` | Release prior package slots → `AVAILABLE` (if no other blockers); re-validate and `BOOKED` for new coach |
| Group package → `PENDING_CONFIRMATION` (first paid registration) | Rows for all future instances on that package remain `BOOKED`; set `packageRunStatus = PENDING_CONFIRMATION` |
| Group package → `CONFIRMED` (§5.5 job) | Same rows stay `BOOKED`; set `packageRunStatus = CONFIRMED` |
| Group package → `CANCELLED_INSUFFICIENT` | Release all package slots → `AVAILABLE` |
| Private: member starts registration / checkout (`registration` pending payment) | `validateCoachScheduleForPackage` immediately before allowing pay; set rows → `BOOKED` with `packageRegistrationId` and `packageRunStatus` reflecting registration (pending until paid) |
| Private: payment `PAID` / registration confirmed | Keep `BOOKED`; clear `packageRegistrationId` hold flag or set `packageRunStatus = CONFIRMED` on package |
| Private: payment abandoned / registration cancelled before pay | Release registration-scoped holds → `AVAILABLE` if no package-level assignment remains |
| `CLASS` session created for instance | Row stays `BOOKED` (session also blocks per §4.8.3) |
| Session or package cancelled (§11.6) | Release slot → `AVAILABLE` when weekly availability still covers the slot and no other blocker exists |

**Implementation note:** `PackageCoachHold` (§4.8.6) may be implemented as `CoachSchedule` rows with `status = BOOKED` and the foreign keys above; one physical table is preferred to avoid duplicate blocking logic.

#### 4.8.1 Entity: `InstructorAvailability` (fixed time slot format)

Coach schedules use the tenant’s **fixed session slot grid** (§4.6) — the same discretization as practice court booking and `ClassInstance` generation. Availability is **not** stored as free-form minute ranges (e.g. 09:17–11:43).

Recurring **weekly** rows per instructor (not one row per calendar date).

```text
InstructorAvailability
  tenantId
  instructorId              (coach — User with role INSTRUCTOR)
  dayOfWeek                 (0 = Monday … 6 = Sunday)
  startTime                 (local facility time, "HH:MM" — slot-aligned)
  endTime                   (local facility time, "HH:MM" — slot-aligned)
```

**Slot alignment (required):**

```text
slotMinutes = tenant.sessionSlotMinutes

startTime and endTime must lie on the facility slot grid:
  gridOrigin = tenant.facilityOpenTime
  validStart = gridOrigin + (n * slotMinutes)   for integer n >= 0

(endTime - startTime) must be a positive multiple of slotMinutes
```

**Examples** (`sessionSlotMinutes = 30`, facility opens 06:00):

| Valid window | Invalid |
|--------------|---------|
| Wed 09:00–12:00 (6 consecutive slots) | Wed 09:15–12:00 |
| Wed 14:00–18:00 (8 slots) | Wed 14:00–18:20 |
| Wed 09:00–09:30 (single slot) | Wed 09:10–09:40 |

**Semantics:** A window means the coach is available for **every** slot boundary from `startTime` up to (but not including) `endTime`, stepping by `sessionSlotMinutes`. Multiple windows per day are allowed (e.g. 09:00–12:00 and 14:00–18:00).

**Admin UI:** Pick availability by **toggling or selecting aligned slots** on the weekly grid (same visual model as practice slot picker), not by typing arbitrary times. API rejects non-aligned `startTime`/`endTime` with `INVALID_TIME_SLOT`.

**Uniqueness:** `(tenantId, instructorId, dayOfWeek, startTime)` — one window start per day per coach.

**Admin** creates and maintains rows when onboarding coaches or updating schedules (`POST /admin/instructor-availability` or equivalent admin UI).

**Equivalent representation (implementation option):** Store one row per **slot** instead of a range:

```text
InstructorAvailabilitySlot
  tenantId, instructorId, dayOfWeek, slotStartTime   ("HH:MM")
  (implicit duration = tenant.sessionSlotMinutes)
```

Range rows and per-slot rows are equivalent if ranges are always slot-aligned; the product standard is **slot-aligned times** either way.

**Not stored here:** one-off PTO, holidays, or sick days (future extension: `InstructorTimeOff` with specific dates). Until then, admin blocks time by removing slot windows or cancelling sessions.

#### 4.8.2 Slot-level match (`slotInsideWeeklyAvailability`)

Each `ClassInstance` spans **exactly one** `sessionSlotMinutes` window. Matching checks **instance against slot grid**:

```text
function slotInsideWeeklyAvailability(instructorId, slotStart, slotEnd):
  dow = dayOfWeek(slotStart)   // local facility timezone
  assert (slotEnd - slotStart) == tenant.sessionSlotMinutes

  for each InstructorAvailability row for (tenant, instructorId, dow):
    if row.startTime and row.endTime are slot-aligned (§4.8.1)
    AND slotStart.time >= row.startTime
    AND slotEnd.time   <= row.endTime:
      return true
  return false
```

**Package match:** Every generated instance must pass `slotInsideWeeklyAvailability` for the assigned coach.

#### 4.8.3 Second layer: existing bookings (coach conflict)

Weekly availability alone is not enough. A coach may already be scheduled on a `CLASS` session.

Before assigning or confirming a coach for a package, for **each** `ClassInstance` (or derived slot) the system checks:

```text
coachCoversSlot(instructor, slotStart, slotEnd, excludePackageId = null) =
  slotInsideWeeklyAvailability(instructor, slotStart, slotEnd)    // §4.8.2
  AND
  no overlapping active CLASS Session where session.coachId = instructor
      AND session.status in (ACTIVE_COACH_CONSUMING_STATUSES)   // see below
  AND
  no overlapping PackageCoachHold where hold.instructorId = instructor
      AND hold.classPackageId != excludePackageId               // §4.8.6
```

**Statuses that consume coach capacity** (block matching for that slot):

```text
ACTIVE_COACH_CONSUMING_STATUSES =
  BOOKED | CONFIRMED | IN_PROGRESS | PENDING_CONFIRMATION
```

**Statuses that release coach capacity** (slot becomes bookable again if still inside weekly `InstructorAvailability`):

```text
CANCELLED
REFUNDED                    (terminal commerce state on booking — session still CANCELLED)
LATE_CANCELLED              (§12.2 — member forfeited; group class may still run with coach)
NO_SHOW                     (optional — if session ended without freeing coach for rebook, facility policy)
```

Overlap uses the same interval logic as §10.1.

**Court / `maxResources`** (§10.1, §10.2) are validated **separately** — coach free does not imply a court is free.

**On cancellation:** Setting a `CLASS` session to a **release** status **refunds coach availability** for that aligned slot — the coach can be matched again for another package or session (§4.8.6, §11.6). **`InstructorAvailability` rows are not deleted**; only the **booking hold** on that slot is removed.

#### 4.8.4 Coach match algorithm (`coachCoversPackage`)

**Assumption:** Coaches enter **weekly availability** (`InstructorAvailability`, §4.8.1) **before** an admin creates or publishes a class package. Matching reads those rows; assigning a coach to a package **does not** add or remove `InstructorAvailability` windows — it **reserves bookable capacity** via **`PackageCoachHold`** (§4.8.6).

Used when:

- Admin assigns a `coachId` to a class package at create or publish time
- Member or instructor match/claim locks a coach
- Re-validation immediately before payment

```text
function coachCoversPackage(instructorId, classInstances[], packageId = null):
  for each instance in classInstances:
    if NOT coachCoversSlot(instructorId, instance.start, instance.end, packageId):  // §4.8.3
      return false
  return true
```

**Admin workflow:** When creating or publishing a package, the admin selects a coach from instructors for whom `coachCoversPackage` is **true** for all generated `ClassInstance` rows. If the chosen coach does not cover every instance, reject with `INSTRUCTOR_UNAVAILABLE` (or show only qualifying coaches in the picker). On success, **create or refresh `PackageCoachHold` rows** (§4.8.6).

**Listing available coaches (recommended):** `GET /admin/class-packages/{packageId}/available-coaches` (or equivalent admin UI) returns instructors where `coachCoversPackage(instructorId, instances, packageId)` is true, plus display fields (`coachName`, bio).

#### 4.8.5 When matching runs (group and private)

| Event | Coach check | Update coach capacity |
|--------|-------------|------------------------|
| Admin saves/publishes package with `coachId` set | `coachCoversPackage` for all generated instances | **Create** `PackageCoachHold` per future instance (§4.8.6) |
| Admin changes or clears `coachId` | `coachCoversPackage` for new coach (if set) | **Release** holds for old coach; **create** holds for new coach |
| Admin saves package with **no** `coachId` | Optional at publish; required before first paid registration or at claim | No package holds until coach locked |
| Member/instructor match or claim | `coachCoversPackage` | Set `package.coachId`; **create** holds |
| Member checkout / payment confirm | Re-run `coachCoversPackage` for locked `coachId` | Holds already in place; fail if another booking took the slot |
| Materialize `CLASS` sessions | `coachId` must be set; fail with `NO_INSTRUCTOR` if missing | Holds remain; `CLASS` sessions also consume capacity (§4.8.3) |
| Package deactivated / unpublished | — | **Release** all holds for that package |

Group packages use the **same** `InstructorAvailability`, `coachCoversPackage`, and **`PackageCoachHold`** rules when assigning a coach to a fixed Wed 4pm series or when listing coaches who can teach a camp week.

#### 4.8.6 Package coach assignment — update bookable availability

When a **`coachId` is assigned** to a `ClassPackage` (admin at publish, instructor **claim**, or member **match** locking `package.coachId`), the system **updates effective coach availability** by creating one hold per **future** `ClassInstance`:

```text
PackageCoachHold
  tenantId
  classPackageId
  classInstanceId           (FK — one hold per instance in the package)
  instructorId              (coachId on the package at assignment time)
  slotStartDateTime         (from ClassInstance — facility local time, slot-aligned)
  slotEndDateTime           (slotStart + tenant.sessionSlotMinutes)
```

**Create holds (required)** after `coachCoversPackage` succeeds:

```text
onCoachAssigned(package, instructorId):
  for each ClassInstance in package where instance.start >= now:
    upsert PackageCoachHold(package, instance, instructorId)
  refresh coach “available slots” views for instructorId (admin + instructor UI)
```

**Semantics:**

- Holds **block** `coachCoversSlot` for **other** packages and match flows the same way active `CLASS` sessions do (§4.8.3).
- **`InstructorAvailability` weekly rows are unchanged** — the coach’s recurring schedule is the same; only **bookable** slots are reduced.
- Admin and instructor UIs should show assigned package instances as **unavailable for new matching** (same visual treatment as an existing `CLASS` session on that slot).

**Release holds (required)** when:

```text
- package.coachId is cleared or changed (release all holds for the package before re-assigning)
- package is deactivated or removed from catalog
- all ClassInstances for the package are in the past (optional nightly cleanup)
- system cancels the whole package (§11.4, §11.5) — release holds for every instance
```

**On `CLASS` session cancellation:** Setting a session to a **release** status frees that slot for matching if no other active session or **other package’s** hold blocks it (§11.6). Package-level holds for the **same** instance remain until the package’s coach assignment is released or the instance is removed.

**Idempotency:** Re-publishing the same `coachId` without schedule changes should not duplicate holds (`classInstanceId` is unique per hold).

---

## 5. Class Package Functionality

All structured class products (weekly group lessons, private series, summer camp) use **`ClassPackage`**. Behavior differs by **`schedulePolicy`**, not by a separate camp entity.

### 5.1 Admin Creates a Class Package

Admin enters:

```text
Package name
Description
Class session type (GROUP_CLASS | PRIVATE_CLASS)
Schedule policy (WEEKLY_RECURRING | DAILY_CAMP)
Program type (STANDARD | SUMMER_CAMP) — use SUMMER_CAMP for July/August camp catalog
Price
Start date
Start time
End time
Coach (instructor) — see §4.8 and §5.6 (fixed coach, member-selected, or open for claim)
Coach name (display)
Coach information (bio, credentials, photo)
Group enrollment: min 2, max 4 members (system-enforced for GROUP_CLASS)
Late registration setting
Proration setting

If schedulePolicy = WEEKLY_RECURRING:
  Day of week (e.g. Wednesday)
  Session count (e.g. 4) OR end date
  End date (optional)

If schedulePolicy = DAILY_CAMP:
  Duration weeks (1 | 2)
  Camp weekdays (default Mon–Fri)
  End date (optional — derived from startDate + durationWeeks if omitted)
```

**Example A — weekly group lesson:**

```text
Beginner Tennis 4-Class Package
schedulePolicy: WEEKLY_RECURRING
programType: STANDARD
GROUP_CLASS
$200
Start: May 1
Every Wednesday, 7:00 PM - 8:00 PM
sessionCount: 4  → instances May 1, 8, 15, 22
```

**Example B — 1-week half-day summer camp:**

```text
Junior Tennis Camp — Week of Jul 7
schedulePolicy: DAILY_CAMP
programType: SUMMER_CAMP
GROUP_CLASS
$450
Start: Jul 7 (Monday)
durationWeeks: 1
campWeekdays: Mon–Fri
Daily window: 9:00 AM - 12:00 PM  (half-day)
sessionSlotMinutes: 30 (tenant) → 6 instances/day × 5 days = 30 ClassInstances
sessionSlotMinutes: 60 (tenant) → 3 instances/day × 5 days = 15 ClassInstances
```

**Example C — 2-week full-day summer camp:**

```text
Junior Tennis Camp — Two Weeks Jul 7–18
schedulePolicy: DAILY_CAMP
programType: SUMMER_CAMP
durationWeeks: 2
Daily: 9:00 AM - 4:00 PM  (full-day)
sessionSlotMinutes: 30 → 14 instances/day × 10 weekdays = 140 ClassInstances
sessionSlotMinutes: 60 → 7 instances/day × 10 weekdays = 70 ClassInstances
```

The system stores the package, then **generates `ClassInstance` rows** using §5.2.

---

### 5.2 Schedule Policy and Class Instance Generation

When a package is created or published, the system builds **`ClassInstance`** records from `schedulePolicy`. Admin may regenerate instances if dates change before any registration.

#### 5.2.1 `WEEKLY_RECURRING`

**Rule:** For each week on the configured **`dayOfWeek`**, create one **`ClassInstance` per consecutive session slot** between package `startTime` and `endTime` (each slot length = `tenant.sessionSlotMinutes`). Repeat weekly for **`sessionCount`** weeks.

```text
slotsInLesson = (endTime - startTime) / sessionSlotMinutes   (must be integer)
instance[0..slotsInLesson-1].start = firstWeekday at startTime + (n * sessionSlotMinutes)
nextWeekInstances = same slot times + 7 days
stop when sessionCount weeks reached
```

**Validation:** `startTime` and `endTime` must align to the tenant slot grid; otherwise package publish fails.

**Example:** Wednesday 7:00–8:00 PM, `sessionSlotMinutes = 60`, count 4 weeks → 4 instances (one slot per week).

**Example:** Wednesday 7:00–8:00 PM, `sessionSlotMinutes = 30` → 2 slots per week (7:00–7:30, 7:30–8:00) × 4 weeks = **8** instances.

#### 5.2.2 `DAILY_CAMP`

**Rule:** For each **camp weekday** in the camp window, split the daily **`startTime`–`endTime`** into consecutive **`sessionSlotMinutes`** slots. Create **one `ClassInstance` per slot** (not one long block per day).

```text
campEndDate = startDate + (durationWeeks * 7 days) - 1 day   (or explicit endDate)
slotsPerDay = (endTime - startTime) / sessionSlotMinutes      (must be integer)

for each date D from startDate to campEndDate:
  if D.dayOfWeek in campWeekdays:
    for slotIndex in 0 .. slotsPerDay - 1:
      slotStart = D at startTime + (slotIndex * sessionSlotMinutes)
      slotEnd   = slotStart + sessionSlotMinutes
      create ClassInstance(slotStart, slotEnd)
```

**Examples** (half-day 9:00–12:00, Mon–Fri):

| `sessionSlotMinutes` | Slots / day | 1-week instances | Per member sessions (1 week) |
|----------------------|-------------|------------------|------------------------------|
| 30 | 6 | 30 | 30 `CLASS` sessions |
| 60 | 3 | 15 | 15 `CLASS` sessions |

**Examples** (full-day 9:00–16:00, 2 weeks, weekdays only):

| `sessionSlotMinutes` | Slots / day | Total instances (10 days) |
|----------------------|-------------|---------------------------|
| 30 | 14 | 140 |
| 60 | 7 | 70 |

**July/August:** `programType = SUMMER_CAMP` is used for catalog filters; scheduling logic is still `DAILY_CAMP` with slot splitting.

#### 5.2.3 Shared constraints after generation

For each instance (any policy):

- Each instance duration **equals** `tenant.sessionSlotMinutes`.
- Reject package definitions whose `startTime`/`endTime` are not slot-aligned.
- Enforce per-resource no overlap (§10.1) **per slot**.
- Enforce tenant **`maxResources`** for each slot’s time window (§10.2).
- If **`coachId`** is set on the package, run **`coachCoversPackage`** (§4.8.4) for all generated instances; reject publish with `INSTRUCTOR_UNAVAILABLE` if false; on success, **create `PackageCoachHold`** rows (§4.8.6).
- Store `sessionCount` on the package as the total number of instances generated.

---

### 5.6 Private Class Registration and Coach Matching

Private lessons (`classSessionType = PRIVATE_CLASS`) use the same `ClassPackage` / `ClassInstance` model as group, with **capacity 1** and **immediate confirmation after payment** (no §5.5 min-members job). Coach assignment follows §4.8; this section defines **private-specific** catalog and checkout behavior.

#### 5.6.1 Catalog patterns

| Pattern | `coachId` on package | Member experience |
|---------|----------------------|-------------------|
| **A — Fixed private series** | Set at publish (e.g. “4 lessons with Coach Jane, Wed 4pm”) | Catalog shows coach name/bio; member registers and pays; no coach picker |
| **B — Member chooses coach** | Optional / null at publish | Member calls **match-instructors**, picks one coach, then registers |
| **C — Open until instructor claims** | Null at publish | Instructors notified; first qualifying instructor **claims** package (`coachCoversPackage`); then members register |

Patterns B and C require **`InstructorAvailability`** data before matching or claim succeeds.

**Recommended default:** Pattern **A** for simplicity; Pattern **B** when the facility sells the same time slot with multiple eligible coaches.

#### 5.6.2 Private enrollment limits

```text
minMembersToRun = 1
maxMembers      = 1   (or product-specific household cap)
```

After successful payment:

- `PackageRegistration` status → **confirmed** (no `PENDING_CONFIRMATION` for private).
- Snapshot `coachId`, `coachName`, `coachInformation` on registration (§4.3).
- Create one `CLASS` session per remaining/future `ClassInstance`, each `classSessionSubtype = PRIVATE_CLASS`.

#### 5.6.3 Workflow — Pattern A (fixed coach)

```text
1. Admin creates PRIVATE_CLASS package (WEEKLY_RECURRING typical).
2. Admin sets coachId, schedule, price; system generates ClassInstances (§5.2).
3. On publish: coachCoversPackage(coachId, all instances) — fail if unavailable; create PackageCoachHold per instance (§4.8.6).
4. Member views catalog (coach visible) → register → pay.
5. Before payment capture: re-run coach conflict check for coachId.
6. On PAID: confirm registration, create CLASS sessions, assign courts (§10).
```

#### 5.6.4 Workflow — Pattern B (member selects coach)

```text
1. Admin creates package with fixed slot times but coachId optional.
2. Member opens package → POST match-instructors(packageId).
3. UI lists only instructors where coachCoversPackage = true.
4. Member selects preferredCoachId → register (store on PackageRegistration).
5. Block payment until package.coachId is locked to selected coach
   (or registration.preferredCoachId resolved to assigned coach).
6. Re-validate coachCoversPackage immediately before charge.
7. On PAID: confirm, snapshot coach on registration, create CLASS sessions.
```

**Errors:**

```text
NO_MATCHING_INSTRUCTOR     — match-instructors returns empty list
INSTRUCTOR_UNAVAILABLE     — selected coach fails coachCoversPackage at pay time
NO_INSTRUCTOR              — checkout attempted with no coach locked
```

#### 5.6.5 Workflow — Pattern C (instructor claim)

```text
1. Admin creates package without coachId; notify eligible instructors.
2. Instructor claims package only if coachCoversPackage(self, all instances).
3. System sets package.coachId and creates PackageCoachHold rows (§4.8.6); members may then register (Pattern A from step 4).
```

Same claim rules may be used for **group** packages without a pre-assigned coach.

#### 5.6.6 What private does not do

- **No random coach assignment** (§4.8.4).
- **No** group confirmation job (§5.5) — one paid member is sufficient to run.
- **No** member self-cancel of the whole package after purchase like practice; per-session cancel follows §11.2 (`privateClassCancelNoticeHours`).

#### 5.6.7 Optional: member-chosen series start (`seriesAnchor`)

If the product allows the member to pick the **first session date** (anchor) and the system derives weekly instances (+0, +7, +14, +21 days):

1. Member proposes `seriesAnchor` (slot-aligned).
2. System derives `ClassInstance` times from anchor + package rules.
3. **Then** run `match-instructors` / `coachCoversPackage` on the **derived** instances (not only admin-fixed instances).

Fixed admin-generated instances (§5.2) remain the default in this specification.

---

### 5.3 Member Registers Before Package Starts

When a member registers before the package starts:

1. System loads all class instances in the package.
2. System calculates full price.
3. System creates an order/payment record.
4. System creates a `PackageRegistration` with coach id, name, and information from the package.
5. System creates one `CLASS` session for each class instance (subtype `GROUP_CLASS` or `PRIVATE_CLASS`).
6. Each session stores the assigned coach and court/resource.
7. System assigns each session to a court/resource.
8. For **group** packages, member payment creates a paid registration if **paid count < tenant.groupMaxMembers**; otherwise registration is rejected (`PACKAGE_FULL`). Package status is `PENDING_CONFIRMATION` until the **confirmation job** (see §5.5, lead time = `tenant.groupConfirmationLeadHours`).
9. For **private** packages, system marks registration **confirmed after payment** (§5.6); coach must pass **`coachCoversPackage`** at publish (if fixed coach) and again before payment (§4.8.5).
10. For **group** packages, member is shown: fixed dates, **no self-cancellation or refund** after the class is **system-confirmed**; if minimum is not met at the confirmation checkpoint, the system cancels the class and **refunds** all paid registrants (see §11.1).

Example:

```text
Member A registers for a 4-class package.

System creates:
- 1 PackageRegistration
- 4 CLASS Sessions
```

---

### 5.4 Member Registers After Package Starts

If late registration is allowed, a member can register after some class instances have already passed.

The system should:

1. Load all class instances in the package.
2. Filter out past class instances.
3. Keep only remaining future class instances.
4. Calculate prorated price.
5. Create a `PackageRegistration`.
6. Create `CLASS` sessions only for the remaining class instances.
7. Assign courts/resources.
8. Process payment based on final price.

Example:

```text
Package has 4 classes:
May 1
May 8
May 15
May 22

Member registers after May 8.

Remaining classes:
May 15
May 22

System creates:
- 1 PackageRegistration
- 2 CLASS Sessions
- Final price = 2 / 4 of original price
```

Proration formula:

```text
pricePerClass = package.price / package.totalClassCount

finalPrice = pricePerClass * remainingClassCount
```

---

### 5.5 Group Class System Confirmation (Tenant Lead Time)

For every **group** `ClassPackage` (weekly or `DAILY_CAMP` summer camp), the system automatically **confirms or cancels** the offering based on enrollment **`tenant.groupConfirmationLeadHours` before** the **first `ClassInstance`** (same checkpoint for all schedule policies; default **24 hours**).

```text
leadHours = tenant.groupConfirmationLeadHours   (default 24)
checkpointTime = firstClassInstance.startDateTime - leadHours
```

#### 5.5.1 Confirmation Job (at checkpoint)

A scheduled system job runs at (or immediately after) `checkpointTime`:

1. Count **paid** `PackageRegistration` records for the group package.
2. Load `minMembers = tenant.groupMinMembers`, `maxMembers = tenant.groupMaxMembers` (or snapshot on package).
3. Compare paid count to `minMembers` (and ensure ≤ `maxMembers`).

**If paid count ≥ minMembers** (and ≤ maxMembers):

- Set package status to `CONFIRMED`.
- Set all related group sessions to `CONFIRMED`.
- Notify members and coach that the class **will run**.
- From this point, member self-cancellation remains **disabled** and registrations are **non-refundable** (see §11.1).

**If paid count < minMembers:**

- Set package status to `CANCELLED_INSUFFICIENT`.
- Cancel all related `GROUP_CLASS` sessions for this package (effective **`groupConfirmationLeadHours` before** the first class).
- **Refund** all paid registrants (full amount paid for that package).
- Notify members and coach that the class **will not run** due to insufficient enrollment.

#### 5.5.2 Between Registration and Confirmation Checkpoint

- Members may register and pay while status is `OPEN_FOR_REGISTRATION` or `PENDING_CONFIRMATION`, until **`tenant.groupMaxMembers`** paid (`PACKAGE_FULL`).
- The class is **not guaranteed to run** until the confirmation job succeeds.
- Members **cannot** voluntarily cancel (protects roster count), but they **will be refunded** if the system cancels at the checkpoint for insufficient members.

#### 5.5.3 After Confirmation

- Package and sessions remain `CONFIRMED` through the series.
- Individual members cannot cancel or obtain refunds (fixed dates).
- Late attendance (§12.2) forfeits only that member’s session attendance, not the whole group.

#### 5.5.4 Coach assignment for group packages

Group packages use the same **`InstructorAvailability`**, **`coachCoversPackage`**, and **`PackageCoachHold`** rules (§4.8) when:

- Admin assigns **`coachId`** at publish (validate all camp or weekly instances; **reserve coach slots** on success — §4.8.6).
- Package is **open** until an instructor **claims** it (claim creates holds).
- Admin runs **match-instructors** before assigning a coach to a new series.

Group confirmation (§5.5) is independent of coach matching: a class can be “confirmed to run” on enrollment while the coach was already validated at publish or claim time.

---

## 6. Practice Court Booking Functionality

Members can book a court for practice without joining a class package. **Practice uses two layers:** `PracticeBooking` + `Payment` (commerce), then **`PRACTICE` Session** (calendar).

### 6.0 Design rule

> **`Session`** = reservation on the court schedule (no payment fields).  
> **`PracticeBooking` + `Payment`** = sale of that reservation (price, Stripe, punch card, membership).

### 6.1 Practice booking window (tenant-configured)

Members may book practice courts **only** for slots whose start time falls within the tenant’s advance booking window.

```text
minLead = tenant.practiceBookingMinLeadHours   (default 2)
maxLead = tenant.practiceBookingMaxLeadHours   (default 24)

hoursUntilStart = session.startDateTime - now

Bookable only if:  minLead <= hoursUntilStart <= maxLead
```

| Situation | `hoursUntilStart` | Result |
|-----------|-------------------|--------|
| Too far in advance | > maxLead | Reject `BOOKING_TOO_EARLY` |
| Bookable window | minLead – maxLead | Allow checkout (subject to availability and slot grid) |
| Too soon | < minLead | Reject `BOOKING_TOO_LATE` |

**Rationale:** `practiceBookingMinLeadHours` should align with **`tenant.practiceCancelNoticeHours`** (§11.3) so every bookable slot can still be cancelled under policy.

**UI:** Offer only aligned slots where `startDateTime` falls in `[now + minLead, now + maxLead]` (intersected with facility hours and `tenant.sessionSlotMinutes` grid).

---

### 6.2 Practice booking flow

1. Member selects a court/resource.
2. Member selects **date** and an **aligned session slot** (`tenant.sessionSlotMinutes`) **within the tenant practice booking window**; arbitrary times such as 10:15–10:35 are not offered.
3. System validates **booking window** (§6.1), slot alignment, and availability (§10).
4. System computes **listedPrice** and **chargedAmount** (membership discount, punch card, or direct pay).
5. System creates **`PracticeBooking`** (`PENDING_PAYMENT` or `CONFIRMED` if $0) and **`Payment`** (`PENDING` or `PAID`).
6. **Slot hold (recommended):** While `PENDING_PAYMENT`, the slot is held so it cannot be double-sold during Stripe checkout.
7. **On payment success** (webhook or immediate if punch/membership/$0):
   - Set `Payment` → `PAID`, `PracticeBooking` → `CONFIRMED`.
   - **Create `PRACTICE` Session** (CONFIRMED) with `practiceBookingId`.
   - Set `PracticeBooking.sessionId` to the new session.
8. **On payment failure / timeout:** `PracticeBooking` → `CANCELLED`; release slot hold; no session created.

```text
Checkout     → PracticeBooking + Payment (PENDING)
Paid         → Payment PAID → create PRACTICE Session → link sessionId
Failed       → no Session; booking cancelled
```

### 6.3 Cancellation and refunds (practice)

Member cancels with notice per **`tenant.practiceCancelNoticeHours`** (§11.3):

1. Cancel **`PRACTICE` Session** (status `CANCELLED`).
2. Update **`PracticeBooking`** → `CANCELLED` or `REFUNDED`.
3. Refund or restore entitlement via **`Payment`** / punch card / membership rules — not via Session fields.

### 6.5 Update (reschedule) practice court booking

Members and admins may **change** an existing practice reservation without cancel-and-rebook when policy allows. The system updates **both** the commerce record and the calendar session in one operation.

#### 6.5.1 What can change

On an existing **`PracticeBooking`** with linked **`PRACTICE` Session** (`status = CONFIRMED`):

| Field | Member may change? | Admin may change? |
|--------|-------------------|-------------------|
| Resource (court) | Yes | Yes |
| `startDateTime` / `endDateTime` (aligned slot) | Yes | Yes |
| Payment source (punch → direct, etc.) | **No** — cancel and create new booking | Yes (exception / support) |

New times must satisfy §10.0 (slot grid) and §10.1 / §10.2 (availability, `maxResources`).

#### 6.5.2 Member update eligibility

A member may **update** their own practice booking only if **all** of the following hold:

```text
1. PracticeBooking.status = CONFIRMED
2. Linked PRACTICE Session.status in (BOOKED, CONFIRMED)
3. hoursUntilCurrentStart >= tenant.practiceCancelNoticeHours
   (same notice as cancellation — default 2h; reuse cancel policy for updates)
4. New slot passes practice booking window from NOW (§6.1):
     minLead <= hoursUntilNewStart <= maxLead
5. New slot is available (no resource overlap; tenant capacity OK)
```

If (3) fails → reject `UPDATE_NOTICE_EXPIRED` (offer cancel-only if still within cancel window, or no action).

If (4) fails → reject `BOOKING_TOO_EARLY` / `BOOKING_TOO_LATE` for the **new** start time.

**Rationale:** Reusing **`practiceCancelNoticeHours`** keeps one “change deadline” for members; the **new** slot must still be bookable under the forward-looking window.

#### 6.5.3 Update flow (member)

```text
1. Member opens confirmed practice booking (PracticeBooking + Session).
2. Member selects new resource and/or date and aligned slot (UI offers valid slots only).
3. System validates §6.5.2 and §10.
4. System updates in one transaction:
     PracticeBooking.resourceId, startDateTime, endDateTime
     Session.resourceId, startDateTime, endDateTime
     (practiceBookingId unchanged; same Payment row unless price adjustment required)
5. Release old slot on calendar; occupy new slot.
6. Apply payment rules (§6.5.5).
7. Write audit log (practice_booking_updated).
```

**API (recommended):** `PATCH /practice-bookings/{id}` with body `{ resourceId?, startDateTime }` (end derived from `sessionSlotMinutes`).

#### 6.5.4 Update before payment (`PENDING_PAYMENT`)

If **`PracticeBooking.status = PENDING_PAYMENT`** (Stripe checkout in progress):

- Member may change court/slot **before** payment completes without cancel notice (slot hold moves to new time).
- Re-validate §6.1 and §10; update hold; do **not** create `Session` until paid.
- On pay → create `Session` at the **latest** chosen slot.

#### 6.5.5 Payment and entitlements on update

| `paymentSource` | On reschedule (same list price) | Price difference |
|-----------------|--------------------------------|------------------|
| **PUNCH_CARD** | No credit debit/refund; booking stays confirmed | N/A if same credit value; if new slot has different listed price, facility policy (usually block or treat as same product) |
| **MEMBERSHIP** | No change to membership allowance | Same |
| **DIRECT** (`Payment` PAID) | No new checkout if `chargedAmount` unchanged | If new slot **higher** price: charge difference (new `Payment` line or Stripe adjustment); if **lower**: partial refund on original `Payment` |

- Do **not** create a second `PracticeBooking` for a simple reschedule — keep one commerce row and one `Session`.
- Punch card: **do not** return then re-debit a credit on update (atomic reschedule).

#### 6.5.6 Admin update

Admin may update any confirmed practice booking **without** member notice rules for operational needs (court maintenance, weather delay).

- Same validation for slot grid and conflicts (unless admin override flag with audit).
- Refund/charge adjustments per facility policy.
- Notify member when admin changes their booking.

#### 6.5.7 What update is not

- **Not** used for group or private **class** sessions (fixed package dates; §11.1).
- **Not** a substitute for cancel when inside cancel notice — member must cancel (if allowed) and book again, or contact admin.
- **Not** changing `sessionType` or linking to a class package.

### 6.4 What practice does not use

```text
ClassPackage
ClassInstance
PackageRegistration
```

A practice booking does **not** use a `Class` catalog model — only `PracticeBooking`, `Payment`, and `Session`.

---

## 6.1 Summer Camp (ClassPackage + DAILY_CAMP)

Summer camp is **not** a separate product or database entity. It is a **`ClassPackage`** configured as:

```text
schedulePolicy: DAILY_CAMP
programType: SUMMER_CAMP          (July–August catalog and reporting)
classSessionType: GROUP_CLASS
durationWeeks: 1 | 2
campWeekdays: Mon–Fri (typical)
startTime / endTime: half-day or full-day window (same time every camp day)
```

**Difference from weekly group lessons** is only **how instances are defined**:

| | Weekly group lesson | Summer camp |
|---|---------------------|-------------|
| `schedulePolicy` | `WEEKLY_RECURRING` | `DAILY_CAMP` |
| Occurrence pattern | Same weekday **once per week** | Camp weekdays **every day** in 1–2 week block |
| Typical instance count | 4–8 slots (weekly) | 6–14 **slots per camp day** × number of weekdays × weeks |
| Daily window | One slot or more per week | Half-day or full-day range split into **sessionSlotMinutes** slots |

Everything else reuses the same model:

```text
ClassPackage → ClassInstance (generated) → PackageRegistration → CLASS Session
```

Shared policies: **tenant group min/max**, **tenant confirmation lead time** on first instance, **no member cancel** after confirm, **`maxResources`** per slot, **`sessionSlotMinutes`** slot grid, coach on package/registration.

### 6.1.1 Admin Configures Summer Camp

Admin creates a **`ClassPackage`** with `schedulePolicy = DAILY_CAMP` and `programType = SUMMER_CAMP` (see §5.1 Example B/C). System generates **`ClassInstance` rows per slot per camp weekday** (not one row per whole day).

### 6.1.2 Member Registers for Summer Camp

Same flow as **§5.3** / **§5.4**: one `PackageRegistration`, one `CLASS` session per `ClassInstance` for that member. Proration (if enabled) uses **remaining instance count** like weekly packages.

### 6.1.3 Camp Confirmation (Tenant Checkpoint)

Same as **§5.5**: **`tenant.groupConfirmationLeadHours` before** the first camp day (first `ClassInstance`), confirm if paid count is within tenant **group min/max**; otherwise cancel entire package and refund all registrants.

### 6.1.4 Season Window

Facilities typically publish multiple camp `ClassPackage` rows for **July and August** (different start weeks, half-day AM/PM, full-day). Each row is an independent package with its own instances and confirmation checkpoint per tenant settings.

---

## 7. Punch Card Functionality

### 7.1 Punch Card Product

A punch card is a prepaid credit product.

Example:

```text
10 Practice Court Punch Card
Price: $150
Credits: 10
Valid for 90 days
```

Admin can configure:

```text
name
price
number of credits
validity period
eligible sport/resource type
active/inactive status
```

### 7.2 Member Buys Punch Card

When a member buys a punch card:

1. System creates an order.
2. Payment is processed.
3. System creates a `MemberPunchCard`.
4. The member receives credits.
5. No session is created at purchase time.

A punch card gives booking rights. It does not reserve a court by itself.

### 7.3 Member Books Court Using Punch Card

When a member uses a punch card:

1. Member selects court and aligned slot.
2. System checks availability and punch card (active, not expired, credits remaining).
3. System creates **`PracticeBooking`** (`paymentSource = PUNCH_CARD`, `chargedAmount = 0`, `CONFIRMED`) and records punch card on the booking.
4. System deducts one credit.
5. System creates **`PRACTICE` Session** linked via `practiceBookingId`.

If the member cancels with **≥ tenant.practiceCancelNoticeHours** before start, cancel Session and `PracticeBooking`; restore one credit (not via Session).

### 7.4 Member Books Court with Direct Payment

When a member pays by card at booking time:

1. Steps 1–4 as §6.2 (including §6.1 booking window) → `PracticeBooking` + `Payment` (`PENDING`).
2. Stripe Checkout completes → `Payment` `PAID` → create **`PRACTICE` Session**.
3. Price and transaction id remain on **`Payment`** / **`PracticeBooking`** only.

---

## 8. Membership Functionality

### 8.1 Membership Plan

A membership is an access or discount product for **practice court booking** (not class packages).

The facility supports these **standard practice membership plans**:

| Plan | Billing | Practice benefit |
|---|---|---|
| **Monthly Unlimited Practice Membership** | Recurring monthly | Unlimited practice court bookings at $0 (subject to booking limits and resource rules) |
| **Annual Unlimited Practice Membership** | Recurring yearly (or annual prepay) | Unlimited practice court bookings at $0 for the membership term |

Additional plan types (credits, discount-only, session caps) may be configured by admin but are optional extensions.

Examples (including standard plans):

```text
Monthly Unlimited Practice Membership
Annual Unlimited Practice Membership
8 Sessions Per Month Membership
Member Discount Plan
```

Admin can configure:

```text
name
planKind                    (MONTHLY_UNLIMITED_PRACTICE | ANNUAL_UNLIMITED_PRACTICE | other)
price
billing period              (monthly | annual)
included practice sessions  (N/A for unlimited plans)
unlimited access setting    (true for both standard unlimited plans)
discount percentage
booking limits
active/inactive status
```

### 8.2 Member Buys Membership

When a member buys a membership:

1. System creates an order.
2. Payment is processed.
3. System creates a `MemberMembership`.
4. No session is created at purchase time.

A membership gives the member rights or benefits for future bookings.

### 8.3 Member Books Court Using Membership

When a member books using membership:

1. Member selects court and aligned slot.
2. System checks availability and active membership; computes **listedPrice** / **chargedAmount** on **`PracticeBooking`**.
3. System creates **`PracticeBooking`** (`paymentSource = MEMBERSHIP`) and **`Payment`** if any charge &gt; 0 (else `PAID` at $0).
4. On confirm → create **`PRACTICE` Session**; deduct included session if applicable.
5. Price and membership reference stay on **`PracticeBooking`**, not Session.

If the member cancels with **≥ tenant.practiceCancelNoticeHours** before start, cancel Session and booking; restore allowance via **`PracticeBooking`** rules.

---

## 9. Payment Logic

Commercial data uses **`Order`** (optional), **`Payment`**, and product-specific aggregates — **`PracticeBooking`** for courts, **`PackageRegistration`** for classes. **`Session` never stores payment fields.**

### 9.1 Order

An `Order` represents a commercial transaction (optional wrapper).

Order types may include:

```text
CLASS_PACKAGE
PRACTICE_BOOKING
PUNCH_CARD
MEMBERSHIP
```

### 9.2 Payment

A `Payment` represents the actual payment attempt/result. It links to the **commerce aggregate**, not to `Session`:

```text
practiceBookingId       (practice court — primary link)
packageRegistrationId   (class package — when applicable)
membershipPlanId        (membership purchase)
orderId                 (optional)
```

Payment status may include:

```text
PENDING
PAID
FAILED
REFUNDED
PARTIALLY_REFUNDED
```

### 9.3 Payment by Product Type

| Product Type | Commerce record | When paid | Calendar record created |
|---|---|---|---|
| Fixed class package | `PackageRegistration` + `Payment` | At registration | `CLASS` Sessions (no price on Session) |
| Practice court (direct) | `PracticeBooking` + `Payment` | At checkout; Session **after** `PAID` | `PRACTICE` Session |
| Practice (punch / membership) | `PracticeBooking` (+ `Payment` if charge &gt; 0) | At booking confirm | `PRACTICE` Session |
| Punch card | `Order` + `Payment` | At purchase | `MemberPunchCard` only |
| Membership | `Order` + `Payment` | At purchase | `MemberMembership` only |

Punch cards and memberships do not create sessions when purchased. They are consumed when a **`PracticeBooking`** is confirmed and a **`PRACTICE` Session** is created.

### 9.4 Practice payment flow (summary)

```text
Member books court
  → PracticeBooking (price, paymentSource, status)
  → Payment (amount, Stripe, status) — linked to practiceBookingId
  → on PAID / CONFIRMED
  → PRACTICE Session (schedule only, practiceBookingId)
```

---

## 10. Availability and Conflict Rules

The system must prevent double-booking, enforce tenant-wide capacity, and only allow **fixed session slots**.

### 10.0 Session Slot Validation (All Session Types)

Before creating or moving any session:

1. Load `tenant.sessionSlotMinutes`.
2. Verify `(endDateTime - startDateTime) == sessionSlotMinutes`.
3. Verify `startDateTime` is aligned to the tenant slot grid (from `facilityOpenTime` + N × `sessionSlotMinutes`).
4. If validation fails → reject with `INVALID_TIME_SLOT`.

**Effect:** Practice bookings cannot request 10:15–10:35 when the tenant uses 30-minute windows; the UI offers 10:00–10:30 and 10:30–11:00 only. Class package instance generation uses the same rules. **Practice reschedule** (§6.5) must validate the **new** slot the same way.

### 10.1 Per-Resource Conflict (No Double-Booking)

```text
A resource cannot have overlapping active sessions.
```

Applies to `CLASS` and `PRACTICE` sessions.

Overlap logic:

```text
existing.startDateTime < requested.endDateTime
AND
requested.startDateTime < existing.endDateTime
```

If there is an overlapping active session for the **same resource**, the booking fails or must pick another resource.

### 10.2 Tenant Capacity (maxResources)

Each tenant stores **`maxResources`**: the maximum number of resources that may be in use **at the same time** for that facility.

**Rule:** For any time period, the count of **active sessions** scheduled for the tenant that overlap that period must **not exceed** `tenant.maxResources`.

```text
concurrentSessions(tenant, window) =
  count of active sessions where
    session.tenantId = tenant.id
    AND session overlaps [window.start, window.end)
    AND session.status in (BOOKED, CONFIRMED, IN_PROGRESS, PENDING_CONFIRMATION)

concurrentSessions(tenant, window) <= tenant.maxResources
```

**When enforced:** On create or reschedule of any session (class or practice), after assigning a resource:

1. Check per-resource overlap (§10.1).
2. Count all overlapping active sessions for the tenant in that time window (each session consumes one resource).
3. If count would exceed `maxResources`, reject with `TENANT_CAPACITY_EXCEEDED`.

**Example:**

```text
Tenant maxResources = 4
May 10, 6:00–7:00 PM — 4 sessions already booked on Courts 1–4
New practice booking for 6:00–7:00 PM → rejected (facility at capacity)
```

**Admin:**

- Set and update `maxResources` on the tenant record.
- `Resource` catalog size should align with `maxResources` (facilities should not define more bookable resources than `maxResources` allows concurrently).

Cancelled sessions do not count toward capacity.

Active session statuses that **consume** capacity may include:

```text
BOOKED
CONFIRMED
IN_PROGRESS
PENDING_CONFIRMATION   (group packages awaiting confirmation checkpoint — optional policy; recommend counting reserved courts)
```

### 10.3 Coach availability and conflicts (class packages)

For every `CLASS` session create, reschedule, or coach assignment:

1. Run **`coachCoversPackage`** or per-slot equivalent (§4.8.4) for the assigned `coachId`.
2. If false → reject with `INSTRUCTOR_UNAVAILABLE`.
3. Then run per-resource overlap (§10.1) and tenant capacity (§10.2).

**Order of checks (recommended):** slot grid (§10.0) → coach slot-aligned availability (§4.8.1–§4.8.2) + coach session conflict (§4.8.3) → resource overlap → `maxResources`.

**Re-check at payment** for private and group registrations when coach was selected earlier (another session may have been booked in the meantime).

---

## 11. Cancellation and Refund Logic

Cancellation rules differ by session type. The system enforces **who may cancel**, **notice period**, and **refund eligibility** at booking time and when the member requests cancellation.

### 11.0 Summary

| Session type | Booking window | Member may cancel? | Cancel notice | Refund |
|---|---|---|---|---|
| **Group class** (`GROUP_CLASS`) | Per package catalog | **No** | N/A | **No** — purchase is final |
| **Private class** (`PRIVATE_CLASS`) | Per package catalog | Yes | **≥ tenant.privateClassCancelNoticeHours** | Yes, if notice met |
| **Practice court** (`PRACTICE`) | **tenant practice booking window** | Yes | **≥ tenant.practiceCancelNoticeHours** | Yes, if notice met |
| **Practice court — update** | New slot must be in **practice booking window** from now | Yes (reschedule) | **≥ tenant.practiceCancelNoticeHours** before **current** start | No refund if same price; price diff per §6.5.5 |

```text
practiceBookingWindow (PRACTICE) — from tenant:
  minLeadHours = tenant.practiceBookingMinLeadHours
  maxLeadHours = tenant.practiceBookingMaxLeadHours

cancelNoticeHours — from tenant:
  GROUP_CLASS   → not cancellable by member
  PRIVATE_CLASS → tenant.privateClassCancelNoticeHours
  PRACTICE      → tenant.practiceCancelNoticeHours
```

---

### 11.1 Group Class — No Member Cancellation, No Refund

**Policy:** Once a member purchases a **group class** package or registers for a group series (including summer camp), **individual cancellation is not allowed**.

**Rationale:**

- Group classes depend on a fixed roster between **`tenant.groupMinMembers`** and **`tenant.groupMaxMembers`**. If one member could cancel after purchase, remaining paid count might fall below the minimum and put the **entire class at risk** of not running or being cancelled for everyone else who signed up.
- Class **dates and times are fixed** at purchase. The member commits to the published schedule.

**System behavior:**

- Do not expose “Cancel registration” or “Cancel session” for `GROUP_CLASS` sessions to members.
- After **system confirmation** at the tenant’s confirmation checkpoint (§5.5), registration stays **confirmed**; sessions stay on the calendar; **no member-initiated refunds**.
- **Before** confirmation, the class is not guaranteed; if minimum is not met at the checkpoint, the system **refunds** all paid registrants (`CANCELLED_INSUFFICIENT`).
- **No refunds** after system confirmation for member no-show or late arrival (§12.2).
- Admin may still cancel the whole confirmed package only for operational exceptions (facility closure, weather); handle refunds per facility policy.

**Late arrival (not the same as cancellation):** If a member is more than **`tenant.classLateThresholdMinutes`** late without check-in (§12.2), that session is marked cancelled/forfeited for **that member only**; this does **not** refund other members and does **not** remove the member from the package roster in a way that threatens `minMembersToRun` for future sessions in the same fixed series.

---

### 11.2 Private Class — Cancel (Tenant Notice), Refundable

**Policy:** A member **may cancel** a `PRIVATE_CLASS` session if cancellation is requested **at least `tenant.privateClassCancelNoticeHours`** before the scheduled `startDateTime` (default 24).

**System behavior:**

1. Member requests cancellation.
2. Load `noticeHours = tenant.privateClassCancelNoticeHours`.
3. System computes: `hoursUntilStart = startDateTime - now`.
4. If `hoursUntilStart >= noticeHours`:
   - Set session status to `CANCELLED`.
   - Process **full refund** for that session (or proportional share if priced per session within a package).
   - Release court/resource for the slot.
5. If `hoursUntilStart < noticeHours`:
   - Reject cancellation request (or show “cancellation window closed”).
   - No refund; session remains confirmed.

**Package note:** For multi-session private packages, each future session is evaluated independently against **`privateClassCancelNoticeHours`** when the member cancels a single occurrence (if the product allows per-session cancel).

---

### 11.3 Practice Court — Booking Window and Cancellation

#### Booking window (see §6.1)

A member **may create** a practice booking only if the slot starts within **`[tenant.practiceBookingMinLeadHours, tenant.practiceBookingMaxLeadHours]`** from now. Align **`practiceBookingMinLeadHours`** with **`practiceCancelNoticeHours`** when possible.

#### Cancellation

**Policy:** A member **may cancel** a `PRACTICE` session if cancellation is requested **at least `tenant.practiceCancelNoticeHours`** before the scheduled `startDateTime` (default 2).

**System behavior:**

1. Member requests cancellation.
2. Load `noticeHours = tenant.practiceCancelNoticeHours`.
3. System computes: `hoursUntilStart = startDateTime - now`.
4. If `hoursUntilStart >= noticeHours`:
   - Set **`PRACTICE` Session** status to `CANCELLED`.
   - Set linked **`PracticeBooking`** to `CANCELLED` / `REFUNDED`.
   - Apply refund or entitlement return via **`Payment`** / **`PracticeBooking.paymentSource`**:
     - **Direct payment** → refund `Payment` (full amount).
     - **Punch card** → return one credit (on booking record).
     - **Membership** → restore included session allowance if applicable.
   - Release court/resource.
5. If `hoursUntilStart < noticeHours`:
   - Reject cancellation request.
   - No refund and no credit return; session remains confirmed.

#### Update (reschedule)

**Policy:** A member **may update** (change court, date, or slot) a confirmed practice booking if requested **at least `tenant.practiceCancelNoticeHours`** before the **current** scheduled `startDateTime`, and the **new** slot is bookable under §6.1 from **now**. Full rules: **§6.5**.

**System behavior:**

1. Member (or admin) requests update with new `resourceId` and/or `startDateTime`.
2. Load `noticeHours = tenant.practiceCancelNoticeHours`.
3. If `hoursUntilCurrentStart < noticeHours` → reject `UPDATE_NOTICE_EXPIRED` (member); admin may override (§6.5.6).
4. Validate new slot: booking window (§6.1), slot grid (§10.0), resource overlap (§10.1), `maxResources` (§10.2).
5. Update **`PracticeBooking`** and linked **`PRACTICE` Session`** in one transaction; keep `practiceBookingId` link.
6. Apply payment/entitlement rules (§6.5.5); audit the change.

**Alternative (not recommended):** Cancel + new booking — only when update is not allowed or member prefers a fresh checkout.

---

### 11.4 System Cancellation — Group Minimum Not Met (Tenant Checkpoint)

This is **automated**, not member-initiated (see §5.5):

- At **`tenant.groupConfirmationLeadHours` before** the first group class session, if `paidCount < tenant.groupMinMembers`, the system cancels the package and all related sessions.
- All paid registrants receive a **full refund**.
- Members are notified that the class will not run.

### 11.5 Admin Cancellation (Override)

Admin may cancel any session or whole group package for operational reasons (facility closure, instructor unavailable).

- **Group package cancelled by admin after system confirmation:** refund policy is facility-defined (exceptional cases only).
- **Group package already started:** admin does not refund individual members for mid-series opt-out.
- Admin override should be audited; member self-service rules in §11.1–§11.3 still apply for member-initiated actions.

### 11.6 Coach availability release (class packages and sessions)

When coach capacity must be freed for matching again:

| Event | Release behavior |
|--------|------------------|
| `CLASS` session → `CANCELLED` / `REFUNDED` / `LATE_CANCELLED` / `NO_SHOW` (per policy) | That slot’s **session hold** ends; coach can match if weekly availability still covers the slot and no other active session or **`PackageCoachHold`** blocks it |
| Whole package cancelled (§11.4, §11.5) | Cancel related sessions **and** **delete all `PackageCoachHold`** rows for the package |
| Admin clears or changes `package.coachId` | **Delete** holds for the package; if new coach set, re-run `coachCoversPackage` and create new holds |
| Package deactivated | **Delete** all package holds |

**`InstructorAvailability` rows are never deleted** for these events — only **booking holds** (`PackageCoachHold` and active `CLASS` sessions) change.

---

## 12. Attendance Logic

Attendance applies mainly to class sessions, but can also apply to practice sessions if the facility wants check-in tracking.

Attendance statuses:

```text
NOT_MARKED
ATTENDED
ABSENT
NO_SHOW
LATE_CANCELLED            (auto-cancelled for late arrival — see §12.2)
```

Instructor or admin can mark:

```text
attended
absent
no-show
```

Attendance is tracked at the `Session` level because each session belongs to a specific member.

### 12.1 Minimum Members and Group Confirmation (Tenant Settings)

For **Group Class Session** packages (including summer camp), use **tenant** values (package may snapshot at publish):

```text
minMembersToRun           = tenant.groupMinMembers
maxMembers                = tenant.groupMaxMembers
confirmationLeadTimeHours = tenant.groupConfirmationLeadHours
```

**Enrollment cap:** Registration closes when **`tenant.groupMaxMembers`** paid members have joined (`PACKAGE_FULL`).

**System confirmation (required):**

- **`groupConfirmationLeadHours` before** the first class instance `startDateTime`, the system counts paid registrations.
- If **`groupMinMembers ≤ paidCount ≤ groupMaxMembers`** → package and sessions become **`CONFIRMED`**; class will run.
- If **`paidCount < groupMinMembers`** → package is **`CANCELLED_INSUFFICIENT`**; all paid members are **refunded**; class does not run.

**Why a configurable lead time:**

- Gives members and staff advance notice whether the class is on or off (default 24 hours).
- Aligns with the rule that members cannot self-cancel (roster is stable leading up to the checkpoint).

**Private Class Session** packages do not use this group checkpoint (effective minimum = 1 paid member; confirmed on payment).

This is separate from per-member attendance marking at class time; it governs whether the **group class runs at all**.

### 12.2 Late Class Attendance Policy

For all **CLASS** sessions (group and private):

```text
lateThreshold = tenant.classLateThresholdMinutes   (default 20)
deadline = startDateTime + lateThreshold
```

If the member has not checked in (or been marked attended) by **`deadline`**:

1. The system (or instructor action triggered by policy) sets the session status to `CANCELLED`.
2. Attendance status is set to `LATE_CANCELLED` (or `NO_SHOW` if the facility maps them equivalently).
3. The member forfeits that session slot for the day; **no refund** (group and private: same as no-show; group registration itself remains non-refundable).
4. The court/resource may be released for other use after cancellation.

Optional automation:

- Scheduled job compares `now` to `deadline` for active `CLASS` sessions with no check-in.
- Instructor may still manually mark `attended` only **before** `deadline`; after that, only admin override can reverse cancellation.

Check-in may be recorded via member app, front desk, or instructor marking `attended` with timestamp stored in `checkInTime`.

---

## 13. Functional Requirements

### 13.1 Class Package Requirements

The system shall allow admin to:

- View and edit **tenant policy settings** (§4.0.1): scheduling, group size, confirmation lead time, practice book/cancel windows, private cancel notice, class late threshold.
- Create class packages as **group** or **private** (`classSessionType`).
- Set **schedule policy** (`WEEKLY_RECURRING` or `DAILY_CAMP`) and **program type** (`STANDARD` or `SUMMER_CAMP`).
- Define package price, start date, daily **start time / end time** (half-day or full-day by window length).
- For **weekly** packages: day of week and session count (or end date).
- For **camp** packages: `durationWeeks` (1 or 2), camp weekdays (default Mon–Fri), `programType = SUMMER_CAMP` for July/August offerings.
- **Generate class instances** from schedule policy (§5.2).
- Maintain **instructor weekly availability** in **fixed slot format** (`InstructorAvailability`, §4.8 — aligned to `sessionSlotMinutes`).
- Assign **coach** (instructor) and publish **coach name** and **coach information** on the package; validate **`coachCoversPackage`** when `coachId` is set (§4.8.4–§4.8.5).
- Run **match-instructors** for packages without a fixed coach (group or private).
- Configure and rely on **tenant policy settings** (§4.0.1); system **group confirmation** job uses `tenant.groupConfirmationLeadHours`.
- Enable or disable late registration and prorated pricing (by remaining instance count).
- Assign instructor/coach to class instances.
- Activate/deactivate class packages.

The system shall allow member to:

- View active class packages with coach name and information.
- Register for a class package (group: while fewer than `tenant.groupMaxMembers` paid spots; private: **max 1** paid member, §5.6).
- For **private** packages with no fixed coach, call **match-instructors** and select a coach before payment (§5.6.4).
- See coach name and information on the registration confirmation.
- Register late if allowed.
- Pay full or prorated price.
- View generated class sessions (group or private) with coach details.
- Cancel **private class** sessions per **`tenant.privateClassCancelNoticeHours`** and receive refund when eligible.
- **Not** self-cancel **group class** registrations after **system confirmation** (fixed dates, non-refundable); receive refund only if system cancels for insufficient members at the confirmation checkpoint.

---

### 13.2 Practice Court Booking Requirements

The system shall allow member to:

- Select court/resource, date, and **aligned session slot** only within the **tenant practice booking window**.
- Book court for practice (no arbitrary start/end times; reject slots outside the window).
- Pay directly, use punch card, or use membership.
- View upcoming **practice sessions** (calendar); view receipts/history via **practice booking** / payment records.
- Cancel practice per **`tenant.practiceCancelNoticeHours`**; receive refund or credit return when eligible.
- **Update (reschedule)** confirmed practice bookings per §6.5 (same notice as cancel before current start; new slot within practice booking window).

The system shall:

- Enforce **`tenant.practiceBookingMinLeadHours`** and **`tenant.practiceBookingMaxLeadHours`** (`BOOKING_TOO_LATE` / `BOOKING_TOO_EARLY`) on **create and update**.
- Reject non-aligned booking times (`INVALID_TIME_SLOT`).
- Prevent resource conflicts and enforce tenant **maxResources** per slot.
- Create **`PracticeBooking`** (+ **`Payment`** when needed) at checkout.
- Create **`PRACTICE` Session** only when booking is **confirmed** (payment `PAID` or entitlement applied).
- **Not** store price, payment status, or Stripe ids on **`Session`**.
- Link `Session.practiceBookingId` to the commerce record.
- Support **`PATCH` / update** of `PracticeBooking` and linked `PRACTICE` `Session` (resource, start, end) per §6.5.

The system shall allow admin to:

- Update any member’s practice booking (override notice rules with audit).

---

### 13.3 Punch Card Requirements

The system shall allow admin to:

- Create punch card products.
- Define price.
- Define total credits.
- Define validity period.
- Define eligible resource/sport type.
- Activate/deactivate punch card products.

The system shall allow member to:

- Buy punch card.
- View remaining credits.
- Use punch card to book practice sessions.
- Receive returned credit only when a practice booking is cancelled per **`tenant.practiceCancelNoticeHours`**.

---

### 13.4 Membership Requirements

The system shall allow admin to:

- Create membership plans, including **Monthly Unlimited Practice Membership** and **Annual Unlimited Practice Membership**.
- Define billing period (monthly or annual).
- Define included sessions (not used for unlimited practice plans).
- Define unlimited practice access for standard unlimited plans.
- Define discount percentage (for non-unlimited plans).
- Define booking limits.
- Activate/deactivate membership plans.

The system shall allow member to:

- Buy monthly or annual unlimited practice membership.
- Use membership benefits for practice court booking only.
- View membership status.
- View remaining included sessions if applicable (non-unlimited plans).

---

## 14. Main Business Rules

### 14.1 Class Package Rules

- A class package is either **group** (`GROUP_CLASS`) or **private** (`PRIVATE_CLASS`).
- A class package displays and stores **coach name** and **coach information**.
- A class package contains multiple class instances.
- A class instance has a concrete start and end time.
- A member registration copies coach fields onto `PackageRegistration` and onto each created session.
- A member registration creates sessions for eligible class instances.
- Late registration creates sessions only for remaining future class instances.
- Prorated price is based on remaining class count.
- A session represents a member’s assigned court/resource slot.
- **Group** packages use **tenant group min/max**; **confirmed** at **`groupConfirmationLeadHours`** before the first session when paid count **≥ groupMinMembers**; otherwise cancelled with full refunds.
- **Summer camp** is a `ClassPackage` with `schedulePolicy = DAILY_CAMP` and `programType = SUMMER_CAMP` (July/August); same registration, session, and group rules as weekly packages.
- **Weekly** group packages use `schedulePolicy = WEEKLY_RECURRING`.
- **Group class cancellation:** members **cannot** cancel after purchase; dates are fixed; **no refunds**.
- **Private class cancellation:** member may cancel per **`tenant.privateClassCancelNoticeHours`**; **refund** if notice met.
- **Coach schedule:** facilities maintain **`InstructorAvailability`** in **fixed slot format** (§4.8.1, same grid as §4.6); matching uses slot-aligned windows **plus** existing `CLASS` session conflicts (§4.8.2–§4.8.4).
- **Private coach matching:** fixed coach on package, member-selected coach via match-instructors, or instructor claim — **never** random assignment (§5.6).
- **Group coach matching:** same `coachCoversPackage` rules when assigning or claiming a coach (§4.8, §5.5.4).
- **Late attendance:** if a member is more than **`tenant.classLateThresholdMinutes`** late without check-in, that session is forfeited; **no refund**.

### 14.2 Practice Booking Rules

- A member can book a court without a class package.
- Every practice checkout creates a **`PracticeBooking`**; money flows through **`Payment`** → `practiceBookingId`.
- A confirmed practice booking creates one **`PRACTICE` Session** (operational); **Session has no price or payment fields**.
- **`PRACTICE` Session** must include member, resource, slot-aligned start/end, and `practiceBookingId`.
- Payment source (direct, punch card, membership) is stored on **`PracticeBooking`**, not Session.
- **Booking window:** member may book only within **`[practiceBookingMinLeadHours, practiceBookingMaxLeadHours]`** from tenant settings.
- Member may cancel per **`practiceCancelNoticeHours`**: cancel Session, update PracticeBooking, refund via Payment or restore entitlement.
- Member may **reschedule** confirmed practice per §6.5 (same notice before current start; new slot in booking window); punch credit is **not** debited again on reschedule.

### 14.3 Punch Card Rules

- Buying a punch card does not create sessions.
- Punch card credits are consumed when practice sessions are booked.
- Punch card must be active, valid, and have remaining credits.
- Cancelled **practice** sessions return one credit only if cancelled per **`tenant.practiceCancelNoticeHours`**.
- Rescheduled practice bookings (§6.5) do **not** return and re-consume a credit.

### 14.4 Membership Rules

- Buying membership does not create sessions.
- Membership applies to **practice court booking** only, not class package purchase.
- **Monthly Unlimited Practice Membership** grants unlimited practice bookings for the monthly billing period.
- **Annual Unlimited Practice Membership** grants unlimited practice bookings for the annual billing period.
- Membership validates access or discount during practice booking.
- Membership may provide free, discounted, limited, or unlimited practice sessions.
- Cancelled **practice** sessions restore included-session allowance only if cancelled per **`tenant.practiceCancelNoticeHours`**.

### 14.5 Resource Rules

- A resource cannot be assigned to overlapping active sessions.
- Resource conflict checking applies to both class and practice sessions.

### 14.6 Tenant Capacity Rules

- Each tenant stores **`maxResources`** (maximum concurrent resources in use).
- For any overlapping time window, active sessions for that tenant must be **≤ maxResources**.
- Session create/reschedule fails with `TENANT_CAPACITY_EXCEEDED` when the facility is at capacity, even if a specific court appears free in isolation (scheduling must respect facility-wide limit).
- Admin maintains `maxResources` on the tenant record; it should match physical court count.

### 14.7 Group Lesson Size Rules

- All **group** packages (weekly and camp) use **`tenant.groupMinMembers`** and **`tenant.groupMaxMembers`** (defaults 2 and 4).
- At the confirmation checkpoint, group class runs only if **paid count ≥ groupMinMembers**; cancelled with refunds if below minimum.
- Registration is not accepted when **paid count = groupMaxMembers** (`PACKAGE_FULL`).

### 14.8 Schedule Policy Rules

- **`WEEKLY_RECURRING`:** instances on the same **day of week**, one **slot** (or consecutive slots) per week, from `startDate` for `sessionCount` weeks; times aligned to `sessionSlotMinutes`.
- **`DAILY_CAMP`:** for each camp weekday, **split** daily `startTime`–`endTime` into slot-sized instances; used for summer camp when `programType = SUMMER_CAMP`.
- **`sessionCount`** is derived from generated instances (includes all camp slots across all days).
- Half-day vs full-day is the **daily window** on the package; instance count per day = window ÷ `sessionSlotMinutes`.
- Camp and weekly packages share: `ClassInstance`, `PackageRegistration`, `CLASS` sessions, coach fields, tenant-driven confirmation, and cancellation policies.

### 14.9 Session Slot Rules

- Every tenant configures **`sessionSlotMinutes`** (e.g. 30 or 60).
- Every `Session` and `ClassInstance` spans exactly one slot (or explicitly defined consecutive slots that are each slot-aligned).
- Bookings must not use arbitrary intervals (e.g. 10:15–10:35); reject with `INVALID_TIME_SLOT`.
- Summer camp half-day at 30 minutes → about **6 sessions per member per day**; at 60 minutes → about **3–5 per day** depending on daily window length.
- Conflict and `maxResources` checks run **per slot**, simplifying scheduling.

### 14.10 PracticeBooking and Session Separation Rules

- **`Session`** (including `PRACTICE`) is for **scheduling, attendance, and conflicts** only.
- **`PracticeBooking`** is for **checkout, price, and entitlement** (punch card / membership).
- **`Payment`** is for **Stripe / refund state**; links to `practiceBookingId`, not `sessionId`.
- Create **`PRACTICE` Session** when `PracticeBooking` is confirmed (`Payment` `PAID` or $0 entitlement).
- Do **not** use a `Class` catalog model for practice courts.

### 14.11 Practice Booking Window Rules

- **`practiceBookingMinLeadHours`** and **`practiceBookingMaxLeadHours`** are **tenant-configurable** (defaults 2 and 24).
- Align **`practiceBookingMinLeadHours`** with **`practiceCancelNoticeHours`** when possible.
- Catalog/UI shows only slots in `[now + minLead, now + maxLead]` from tenant settings.
- **Updates:** the **new** slot must fall in the same window measured from **now** (§6.5.2).

### 14.12 Practice booking update rules

- Member may **reschedule** a confirmed `PracticeBooking` + `PRACTICE` Session if **≥ `practiceCancelNoticeHours`** before the **current** start.
- Update changes **resource** and/or **slot-aligned** start/end on **both** `PracticeBooking` and `Session`; keep one `practiceBookingId`.
- Re-validate slot grid, conflicts, and `maxResources` on every update.
- **`PENDING_PAYMENT`:** member may change slot before pay without cancel notice (§6.5.4).
- Punch card / membership: no double debit on reschedule; direct pay: price difference handling per §6.5.5.
- Admin may reschedule without member notice rules (audited).
- Class sessions (group/private) are **not** member-updatable via this flow.

### 14.13 Tenant Policy Enforcement

- All thresholds in §4.0.1 are read from the **tenant** record at runtime (booking, cancel, confirmation jobs, late forfeiture).
- Do not hardcode policy constants in application code; use tenant defaults only in seed data and documentation examples.
- Admin UI must allow editing tenant policy fields; changes apply to **new** bookings and confirmations (existing sessions keep prior rules unless migrated).

### 14.14 Coach availability and matching rules

- Each instructor has zero or more **`InstructorAvailability`** rows (recurring weekly `dayOfWeek` + **slot-aligned** `startTime`/`endTime` per §4.6 / §4.8.1).
- Admin enters coach hours on the **same slot grid** as practice and class booking (multiples of `sessionSlotMinutes` from `facilityOpenTime`).
- **`coachCoversPackage`** requires every `ClassInstance` (one slot each) to pass **`coachCoversSlot`** (§4.8.2–§4.8.3): weekly availability, no conflicting `CLASS` session, and no conflicting **`PackageCoachHold`** from another package.
- When a coach is **assigned** to a package, the system **creates `PackageCoachHold`** rows for future instances (§4.8.6) so other packages cannot claim the same slots.
- Validate coach at **package publish** (when `coachId` set), at **match/claim**, and **immediately before payment**; on successful assign, **update bookable coach availability** via holds (weekly `InstructorAvailability` rows stay unchanged).
- **Private** packages confirm after one paid member; coach must be locked before payment (§5.6).
- **Group** packages use the same coach validation; enrollment confirmation (§5.5) is separate from coach matching.
- System **must not** assign coaches randomly without member visibility before charge.

---

## 15. Recommended Data Model

### Core Entities

```text
Tenant                        (+ all fields in §4.0.1: scheduling, group min/max, confirmation lead, practice book/cancel windows, private cancel, late threshold)
Member
Instructor                    (coach — User with role INSTRUCTOR)
InstructorAvailability        (§4.8 — weekly dayOfWeek + slot-aligned startTime/endTime per instructor; same grid as §4.6)
PackageCoachHold              (§4.8.6 — one row per future ClassInstance when package.coachId is set; blocks matching for other packages)
Resource
ClassPackage                  (+ schedulePolicy, programType, coachId optional, coach display fields; min/max/confirmationLead snapshotted from tenant at publish, packageRunStatus)
ClassInstance                 (generated from schedulePolicy)
PackageRegistration           (+ coachId, coachName, coachInformation, preferredCoachId optional; commercial data for CLASS)
PracticeBooking               (+ listedPrice, chargedAmount, paymentSource, sessionId; no payment fields on Session)
Session                       (+ classSessionSubtype, practiceBookingId, coach fields; NO price or payment columns)
PunchCardProduct
MemberPunchCard
MembershipPlan                (+ planKind: MONTHLY_UNLIMITED_PRACTICE | ANNUAL_UNLIMITED_PRACTICE)
MemberMembership
Order
Payment
```

### Core Relationships

```text
Tenant 1 → many Members
Tenant.maxResources caps concurrent sessions facility-wide
Tenant 1 → many Resources
Tenant 1 → many Instructors
Instructor 1 → many InstructorAvailability
Tenant 1 → many ClassPackages

ClassPackage 1 → many ClassInstances
ClassPackage 1 → many PackageRegistrations

Member 1 → many PackageRegistrations
Member 1 → many Sessions
Member 1 → many PracticeBookings
Member 1 → many MemberPunchCards
Member 1 → many MemberMemberships

ClassInstance 1 → many Sessions (CLASS)
PackageRegistration 1 → many Sessions (CLASS)

PracticeBooking 1 → 0..1 Session (PRACTICE created on confirm)
PracticeBooking 1 → many Payments
Payment N → 1 PracticeBooking (or PackageRegistration / Order)

Resource 1 → many Sessions

PunchCardProduct 1 → many MemberPunchCards
MemberPunchCard referenced on PracticeBooking (not on Session)

MembershipPlan 1 → many MemberMemberships
MemberMembership referenced on PracticeBooking when used

Order 1 → many Payments
```

---

## 16. Example End-to-End Scenarios

### Scenario A: Full Class Package Registration

1. Admin creates Beginner Tennis 4-Class Package.
2. Admin creates four class instances.
3. Member A registers before the package starts.
4. Member A pays $200.
5. System creates one `PackageRegistration`.
6. System creates four `CLASS` sessions.
7. Each session assigns Member A to Court 1.

Result:

```text
Member A has 4 upcoming class sessions.
```

---

### Scenario B: Late Class Package Registration

1. Package has four class instances.
2. Two classes have already passed.
3. Member B registers late.
4. System identifies two remaining class instances.
5. System calculates final price as $100.
6. Member B pays $100.
7. System creates one `PackageRegistration`.
8. System creates two `CLASS` sessions.

Result:

```text
Member B has 2 upcoming class sessions and pays only for 2.
```

---

### Scenario C: Practice Booking with Punch Card

1. Member C buys a 10-credit punch card (`Payment` at purchase; no Session).
2. Member C books Court 3, May 10, 6:00–6:30 PM (30-min slot).
3. System creates **`PracticeBooking`** (`PUNCH_CARD`, $0 charged, `CONFIRMED`) and deducts one credit.
4. System creates **`PRACTICE` Session** with `practiceBookingId`; Session has no price fields.

Result:

```text
Calendar: 1 PRACTICE Session.
Commerce: 1 PracticeBooking (punch card ref); Payment not required if $0.
```

---

### Scenario D: Practice Booking with Monthly Unlimited Membership

1. Member D has active unlimited membership.
2. Member D books Court 2, May 12, 5:00–6:00 PM.
3. System creates **`PracticeBooking`** (`MEMBERSHIP`, listedPrice may be court list price, chargedAmount $0).
4. System creates **`PRACTICE` Session** linked to booking.

Result:

```text
Session is schedule-only; membership benefit recorded on PracticeBooking.
```

---

### Scenario I: Practice Booking Outside Window

1. Tenant: `practiceBookingMinLeadHours = 2`, `practiceBookingMaxLeadHours = 24` (example defaults).
2. Now is Monday 10:00 AM.
3. Member tries to book Monday 12:00 PM (2h away) → **allowed** (at minimum lead).
4. Member tries to book Tuesday 9:00 AM (23h away) → **allowed**.
5. Member tries to book Monday 11:30 AM (1.5h away) → **`BOOKING_TOO_LATE`**.
6. Member tries to book Tuesday 11:00 AM (25h away) → **`BOOKING_TOO_EARLY`**.

Result:

```text
Practice window and cancel notice come from tenant settings (not hardcoded).
```

---

### Scenario D2: Practice Booking with Stripe (Session After Pay)

1. Member E books Court 1, slot start **8 hours** from now (inside tenant practice booking window); listed $40.
2. System creates **`PracticeBooking`** (`PENDING_PAYMENT`) + **`Payment`** (`PENDING`); slot held.
3. Member completes Stripe Checkout → webhook sets **`Payment`** `PAID`.
4. System creates **`PRACTICE` Session**; sets `PracticeBooking.sessionId`.

Result:

```text
No Session until paid; price lives only on PracticeBooking + Payment.
```

---

### Scenario E: Group Package Confirmed at Checkpoint (Tenant Settings)

1. Tenant: `groupMinMembers = 2`, `groupMaxMembers = 4`, `groupConfirmationLeadHours = 24`.
2. Admin creates a **GROUP_CLASS** package; first class Wednesday 7:00 PM.
3. Three members register and pay by Monday (status `PENDING_CONFIRMATION`).
4. At checkpoint (`groupConfirmationLeadHours` before first class), job runs: paid count = 3 ≥ `groupMinMembers`.
5. System sets package to `CONFIRMED`; notifies members and Coach Jane Smith.

Result:

```text
Class will run with 3 members; no further registrations beyond groupMaxMembers.
```

---

### Scenario E2: Group Package Cancelled at Checkpoint (Below Min)

1. Same tenant/package; only **one** member has paid by checkpoint time.
2. Confirmation job runs: paid count = 1 < `groupMinMembers`.
3. System sets package to `CANCELLED_INSUFFICIENT`; cancels all group sessions; refunds the member.

Result:

```text
Class cancelled one day before start; full refund issued; members notified.
```

---

### Scenario E3: Group Package Full (Max 4)

1. Four members have paid for the same **GROUP_CLASS** package.
2. A fifth member attempts to register.
3. System rejects with `PACKAGE_FULL`.

Result:

```text
Group lesson capped at 4 members.
```

---

### Scenario H: Tenant maxResources Capacity

1. Tenant `maxResources = 3`. Courts 1–3 each have a session 6:00–7:00 PM.
2. Member requests Court 1 for 6:00–7:00 PM (court appears free due to data error) or any fourth concurrent booking.
3. System counts 3 overlapping tenant sessions and rejects: `TENANT_CAPACITY_EXCEEDED`.

Result:

```text
No more than 3 concurrent sessions for this facility in that time window.
```

---

### Scenario F0: Private Class — Member Selects Coach (Match)

1. Admin creates “4 Private Lessons — Wed 4pm” with `coachId` null; generates four `ClassInstance` rows.
2. Admin has set `InstructorAvailability` for Coach Jane (Wed 15:00–18:00) and Coach John (Wed 16:00–20:00).
3. Member calls `match-instructors` → system returns Jane and John (both `coachCoversPackage`).
4. Member selects Jane → registers with `preferredCoachId` → system sets `package.coachId = Jane`.
5. Before Stripe charge, system re-runs `coachCoversPackage(Jane)` → success.
6. Member pays → registration confirmed → four `PRIVATE_CLASS` sessions created with coach snapshot Jane.

Result:

```text
Member has 4 private sessions; coach Jane on registration and all sessions.
```

---

### Scenario F: Private Class Cancellation (24 Hours)

1. Member B has a `PRIVATE_CLASS` session scheduled Saturday at 4:00 PM.
2. On Friday at 3:00 PM (25 hours before), Member B cancels.
3. System refunds the session and releases the court.

Result:

```text
Cancellation allowed; refund issued.
```

If Member B tried to cancel Friday at 5:00 PM (23 hours before), the system would **reject** cancellation and keep the booking non-refundable.

---

### Scenario F2: Late Class Attendance (Tenant Threshold)

1. Tenant: `classLateThresholdMinutes = 20`.
2. Member B has a `PRIVATE_CLASS` session at 4:00 PM.
3. Member B does not check in by 4:20 PM.
3. System marks the session forfeited (`LATE_CANCELLED`); **no refund**.

Result:

```text
Session forfeited for that member; no refund (distinct from voluntary cancel with 24h notice).
```

---

### Scenario F3: Group Class — No Member Cancellation After System Confirm

1. Member C purchases a 4-week **GROUP_CLASS** package; at the tenant confirmation checkpoint the package is `CONFIRMED`.
2. Member C asks to skip week 2 and get a refund.
3. System denies member cancellation; registration stays confirmed.

Result:

```text
No refund after system confirmation; protecting roster for other members.
```

---

### Scenario F3b: Practice Court — Member Reschedules (Update)

1. Tenant: `practiceCancelNoticeHours = 2`, `practiceBookingMinLeadHours = 2`, `practiceBookingMaxLeadHours = 24`.
2. Member F has confirmed practice: Court 2, **today 6:00 PM** (paid with punch card).
3. At **today 2:00 PM** (4h before start), member opens **Update** and picks Court 3, **today 7:00 PM** (still within 2–24h window from now).
4. System validates notice (4h ≥ 2h), new slot available, updates **`PracticeBooking`** and **`PRACTICE` Session`**; no second punch debit.
5. Court 2 slot released; Court 3 slot occupied.

Result:

```text
Same PracticeBooking id; new court and time on booking and session.
```

6. At **today 5:30 PM** (30 min before original 6 PM), member tries to move again → **`UPDATE_NOTICE_EXPIRED`** (< 2h notice).

---

### Scenario F4: Practice Court Cancellation (2 Hours)

1. Member D books Court 3 for today at 6:00 PM.
2. At 1:00 PM (5 hours before), Member D cancels.
3. System refunds payment (or returns punch credit).

Result:

```text
Cancellation allowed; refund or credit restored.
```

If Member D cancelled at 4:30 PM (1.5 hours before), cancellation would be **denied**.

---

### Scenario G: Summer Camp — Half-Day, 30-Minute Slots

1. Tenant `sessionSlotMinutes = 30`.
2. Admin creates **Junior Tennis Camp — Week of Jul 7** (`DAILY_CAMP`, Mon–Fri, daily 9:00 AM–12:00 PM).
3. System generates **30** `ClassInstance` rows (6 slots × 5 days).
4. Member registers; receives **30** `CLASS` sessions (6 per camp day).
5. Practice member cannot book 10:15–10:35; may only book 10:00–10:30 or 10:30–11:00 on an open court.

Result:

```text
Camp = many slot-sized instances; practice uses same slot grid.
```

---

### Scenario G2: Summer Camp — Half-Day, 60-Minute Slots

1. Tenant `sessionSlotMinutes = 60`, same camp window 9:00 AM–12:00 PM.
2. System generates **15** instances (3 slots × 5 days) per package.
3. Each member has **3 sessions per camp day** (~4–5 if daily window extended to 12:30 PM).

Result:

```text
Larger slot window → fewer instances per day; still slot-aligned only.
```

---

### Scenario G3: Invalid Practice Time

1. Tenant `sessionSlotMinutes = 30`.
2. Member requests Court 2, May 10, **10:15 AM – 10:35 AM**.
3. System rejects: `INVALID_TIME_SLOT`.

Result:

```text
Must book 10:00–10:30 or 10:30–11:00 instead.
```

---

## 17. Stakeholder Summary

Court Reserve is designed to support both structured learning programs and flexible court usage.

The system separates:

```text
What the facility sells
When the activity happens
What the member registered for or owns
Which court/time is actually reserved
How the reservation is paid for
```

The core design is:

```text
ClassPackage (schedulePolicy: WEEKLY_RECURRING | DAILY_CAMP, programType, coach info)
  → ClassInstance (generated per policy)
  → PackageRegistration (coach snapshot)
  → CLASS Session (GROUP_CLASS | PRIVATE_CLASS)
```

Weekly lessons and summer camp share this path; camp differs only in **daily** instance generation, and:

```text
Member → PracticeBooking + Payment → PRACTICE Session (on confirm; Session has no price)
```

Punch cards and memberships are not bookings. They are entitlements applied at **`PracticeBooking`** time, then a **`PRACTICE` Session** is created.

Standard practice memberships:

```text
Monthly Unlimited Practice Membership
Annual Unlimited Practice Membership
```

This design gives the business flexibility to support:

```text
Group and private class packages
Coach information on packages and registrations
Practice: PracticeBooking + Payment → PRACTICE Session (no price on Session)
Tenant sessionSlotMinutes (30 or 60): all sessions slot-aligned; no arbitrary booking times
Summer camp: ClassPackage + DAILY_CAMP; daily window split into slots (e.g. 6/day half-day @ 30 min)
Weekly group: ClassPackage + WEEKLY_RECURRING
Group class: tenant group min/max; system confirms per tenant.groupConfirmationLeadHours
Tenant maxResources: concurrent sessions cannot exceed facility cap
Late registration and prorated pricing
20-minute late class forfeiture (no refund)
Group class: no member cancel, no refund, fixed dates
Private class: cancel per tenant.privateClassCancelNoticeHours (default 24h)
Practice court: book/cancel windows from tenant policy (defaults 2–24h book, 2h cancel)
Monthly and annual unlimited practice memberships
Standalone practice court booking
Punch card credits
Direct payments
Attendance tracking
Court conflict prevention
```
