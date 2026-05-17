# UTennis
# Functional Specification: Court Reserve Booking Platform

## 1. Product Overview

**Court Reserve** is a configurable booking platform for sports facilities that manage court reservations, fixed class packages, memberships, and punch cards.

The product supports two major booking scenarios:

1. **Class package registration**  
   Members register for a fixed class package, such as a 4-class tennis course. The package already has fixed class dates and times.

2. **Practice court booking**  
   Members book a court for personal practice. This can be paid directly, covered by a punch card, or covered/discounted by a membership.

The system uses one shared concept called **Session**.

A `Session` represents a reserved time slot for a member on a court/resource.

A session can be:

```text
CLASS session
- Created from a class package registration

PRACTICE session
- Created from a standalone court booking
```

---

## 2. Business Goals

The platform should help a sports facility:

- Sell fixed class packages.
- Allow members to register before or after a package starts.
- Support prorated pricing for late registration.
- Assign members to courts/resources for each class.
- Allow members to book practice courts outside of classes.
- Support payment by one-time payment, punch card, or membership.
- Prevent court/resource double-booking.
- Track attendance, cancellations, payment status, and session history.
- Support multiple tenants/facilities in the future.

---

## 3. User Roles

### 3.1 Admin / Facility Manager

The admin manages business operations.

Admin can:

- Create and manage class packages.
- Create concrete class instances inside a package.
- Manage instructors.
- Manage courts/resources.
- Register members for packages.
- View package registrations.
- View class attendance.
- View court bookings.
- Create punch card products.
- Create membership plans.
- Manage payment and booking status.
- Cancel or update sessions.

### 3.2 Member / Customer

The member uses the platform to book and manage activities.

Member can:

- View available class packages.
- Register for a class package.
- Register late for an active package if allowed.
- Book practice courts.
- Use punch card credits.
- Use membership benefits.
- View upcoming sessions.
- Cancel eligible sessions.
- View booking/payment history.

### 3.3 Instructor

The instructor teaches classes.

Instructor can:

- View assigned class instances.
- View members attending each class.
- View assigned courts/resources.
- Mark attendance.
- Mark no-shows.

---

## 4. Core Product Concepts

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
endDate
dayOfWeek
startTime
endTime
totalClassCount
price
allowLateRegistration
prorateLateRegistration
```

The package itself does not represent one member’s registration. It is the product definition.

---

### 4.2 ClassInstance

A `ClassInstance` is one concrete class occurrence inside a package.

Example:

```text
ClassInstance 1: May 1, 7:00 PM - 8:00 PM
ClassInstance 2: May 8, 7:00 PM - 8:00 PM
ClassInstance 3: May 15, 7:00 PM - 8:00 PM
ClassInstance 4: May 22, 7:00 PM - 8:00 PM
```

Relationship:

```text
One ClassPackage has many ClassInstances.
```

The `ClassInstance` stores the real date and time of each individual class.

---

### 4.3 PackageRegistration

A `PackageRegistration` represents one member’s registration for one class package.

It tracks:

```text
member
package
registration date
original class count
registered class count
skipped class count
original price
final price
payment status
registration status
```

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

A `Session` is the actual reserved slot.

It contains:

```text
member
resource / court
startDateTime
endDateTime
sessionType
status
attendanceStatus
```

There are two session types:

```text
CLASS
PRACTICE
```

#### CLASS Session

A `CLASS` session is created from a package registration.

Example:

```text
Member A
Beginner Tennis ClassInstance 1
Court 1
May 1, 7:00 PM - 8:00 PM
```

For a 4-class package, one member registration creates multiple `CLASS` sessions.

#### PRACTICE Session

A `PRACTICE` session is created when a member books a court for practice.

Example:

```text
Member A
Court 3
May 10, 6:00 PM - 7:00 PM
```

This has no relationship to a class package or class instance.

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

## 5. Fixed Class Package Functionality

### 5.1 Admin Creates a Class Package

Admin enters:

```text
Package name
Description
Price
Total class count
Start date
End date
Day of week
Start time
End time
Instructor
Late registration setting
Proration setting
```

Example:

```text
Beginner Tennis 4-Class Package
$200
May 1 - May 22
Every Wednesday
7:00 PM - 8:00 PM
Late registration allowed
Proration enabled
```

The system then stores the package-level schedule.

---

### 5.2 Admin Creates Class Instances

For each package, the system or admin creates concrete class instances.

Example:

```text
May 1, 7:00 PM - 8:00 PM
May 8, 7:00 PM - 8:00 PM
May 15, 7:00 PM - 8:00 PM
May 22, 7:00 PM - 8:00 PM
```

Each class instance belongs to the package.

---

### 5.3 Member Registers Before Package Starts

When a member registers before the package starts:

1. System loads all class instances in the package.
2. System calculates full price.
3. System creates an order/payment record.
4. System creates a `PackageRegistration`.
5. System creates one `CLASS` session for each class instance.
6. System assigns each session to a court/resource.
7. System marks the registration as confirmed after payment.

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

## 6. Practice Court Booking Functionality

Members can book a court for practice without joining a class package.

Practice booking flow:

1. Member selects a court/resource.
2. Member selects date and time.
3. System checks availability.
4. System determines payment method:
   - one-time payment
   - punch card
   - membership
5. System creates a `PRACTICE` session.
6. System links the session to payment, punch card, or membership where applicable.

A practice session does not require:

```text
ClassPackage
ClassInstance
PackageRegistration
```

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

1. Member selects court and time.
2. System checks court availability.
3. System checks punch card is active.
4. System checks punch card is not expired.
5. System checks remaining credits.
6. System creates a `PRACTICE` session.
7. System deducts one credit.
8. Session is linked to the punch card.

If the session is cancelled and the policy allows credit return, the system restores the credit.

---

## 8. Membership Functionality

### 8.1 Membership Plan

A membership is an access or discount product.

Examples:

```text
Monthly Practice Membership
Unlimited Practice Membership
8 Sessions Per Month Membership
Member Discount Plan
```

Admin can configure:

```text
price
billing period
included practice sessions
unlimited access setting
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

1. Member selects court and time.
2. System checks court availability.
3. System checks active membership.
4. System applies membership rules.
5. System determines final price:
   - free if covered by membership
   - discounted if membership gives a discount
   - regular price if not covered
6. System creates a `PRACTICE` session.
7. If the plan has limited included sessions, the system deducts one included session.

If the session is cancelled and the policy allows benefit return, the system restores the included session allowance.

---

## 9. Payment Logic

The product supports payment through `Order` and `Payment`.

### 9.1 Order

An `Order` represents the commercial transaction.

Order types may include:

```text
CLASS_PACKAGE
PRACTICE_BOOKING
PUNCH_CARD
MEMBERSHIP
```

### 9.2 Payment

A `Payment` represents the actual payment attempt/result.

Payment status may include:

```text
PENDING
PAID
FAILED
REFUNDED
PARTIALLY_REFUNDED
```

### 9.3 Payment by Product Type

| Product Type | When Payment Happens | What Is Created |
|---|---|---|
| Fixed Class Package | At registration | PackageRegistration + CLASS Sessions |
| One-Time Practice Booking | At court booking | PRACTICE Session |
| Punch Card | At punch card purchase | MemberPunchCard |
| Membership | At membership purchase or renewal | MemberMembership |

Punch cards and memberships do not create sessions when purchased. They are used later to create practice sessions.

---

## 10. Availability and Conflict Rules

The system must prevent double-booking.

Universal conflict rule:

```text
A resource cannot have overlapping active sessions.
```

The rule applies to:

```text
CLASS sessions
PRACTICE sessions
```

Overlap logic:

```text
existing.startDateTime < requested.endDateTime
AND
requested.startDateTime < existing.endDateTime
```

If there is an overlapping active session for the same resource, the booking should fail or require a different resource.

Active session statuses may include:

```text
BOOKED
CONFIRMED
IN_PROGRESS
```

Cancelled sessions should not block availability.

---

## 11. Cancellation Logic

### 11.1 Class Package Session Cancellation

If a `CLASS` session is cancelled:

- The session status becomes `CANCELLED`.
- Attendance should not be counted.
- Refund or credit behavior depends on facility policy.
- If the whole package registration is cancelled, all related future sessions may be cancelled.

### 11.2 Practice Session Cancellation

If a `PRACTICE` session is cancelled, the system checks how it was paid for.

#### If paid directly

- Cancel session.
- Check refund policy.
- Create refund if eligible.

#### If paid by punch card

- Cancel session.
- If policy allows, return one punch card credit.

#### If covered by membership

- Cancel session.
- If policy allows, restore included session allowance.

---

## 12. Attendance Logic

Attendance applies mainly to class sessions, but can also apply to practice sessions if the facility wants check-in tracking.

Attendance statuses:

```text
NOT_MARKED
ATTENDED
ABSENT
NO_SHOW
```

Instructor or admin can mark:

```text
attended
absent
no-show
```

Attendance is tracked at the `Session` level because each session belongs to a specific member.

---

## 13. Functional Requirements

### 13.1 Class Package Requirements

The system shall allow admin to:

- Create class packages.
- Define package price.
- Define total class count.
- Define package start/end date.
- Define day of week and class time.
- Enable or disable late registration.
- Enable or disable prorated pricing.
- Create class instances under a package.
- Assign instructor to class instances.
- Activate/deactivate class packages.

The system shall allow member to:

- View active class packages.
- Register for a class package.
- Register late if allowed.
- Pay full or prorated price.
- View generated class sessions.

---

### 13.2 Practice Court Booking Requirements

The system shall allow member to:

- Select court/resource.
- Select date and time.
- Book court for practice.
- Pay directly, use punch card, or use membership.
- View upcoming practice sessions.
- Cancel eligible sessions.

The system shall:

- Prevent resource conflicts.
- Create a `PRACTICE` session after successful booking.
- Track payment/entitlement source.

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
- Receive returned credit if cancellation policy allows.

---

### 13.4 Membership Requirements

The system shall allow admin to:

- Create membership plans.
- Define billing period.
- Define included sessions.
- Define unlimited access option.
- Define discount percentage.
- Define booking limits.
- Activate/deactivate membership plans.

The system shall allow member to:

- Buy membership.
- Use membership benefits for practice court booking.
- View membership status.
- View remaining included sessions if applicable.

---

## 14. Main Business Rules

### 14.1 Class Package Rules

- A class package contains multiple class instances.
- A class instance has a concrete start and end time.
- A member registration creates sessions for eligible class instances.
- Late registration creates sessions only for remaining future class instances.
- Prorated price is based on remaining class count.
- A session represents a member’s assigned court/resource slot.

### 14.2 Practice Booking Rules

- A member can book a court without a class package.
- Practice booking creates a `PRACTICE` session.
- Practice session must include member, resource, start time, and end time.
- Practice session can be paid by direct payment, punch card, or membership.

### 14.3 Punch Card Rules

- Buying a punch card does not create sessions.
- Punch card credits are consumed when practice sessions are booked.
- Punch card must be active, valid, and have remaining credits.
- Cancelled sessions may return credits depending on policy.

### 14.4 Membership Rules

- Buying membership does not create sessions.
- Membership validates access or discount during practice booking.
- Membership may provide free, discounted, limited, or unlimited sessions.
- Cancelled sessions may restore membership allowance depending on policy.

### 14.5 Resource Rules

- A resource cannot be assigned to overlapping active sessions.
- Resource conflict checking applies to both class and practice sessions.

---

## 15. Recommended Data Model

### Core Entities

```text
Tenant
Member
Instructor
Resource
ClassPackage
ClassInstance
PackageRegistration
Session
PunchCardProduct
MemberPunchCard
MembershipPlan
MemberMembership
Order
Payment
```

### Core Relationships

```text
Tenant 1 → many Members
Tenant 1 → many Resources
Tenant 1 → many Instructors
Tenant 1 → many ClassPackages

ClassPackage 1 → many ClassInstances
ClassPackage 1 → many PackageRegistrations

Member 1 → many PackageRegistrations
Member 1 → many Sessions
Member 1 → many MemberPunchCards
Member 1 → many MemberMemberships

ClassInstance 1 → many Sessions
PackageRegistration 1 → many Sessions

Resource 1 → many Sessions

PunchCardProduct 1 → many MemberPunchCards
MemberPunchCard 1 → many PRACTICE Sessions

MembershipPlan 1 → many MemberMemberships
MemberMembership 1 → many PRACTICE Sessions

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

1. Member C buys a 10-credit punch card.
2. No session is created at purchase time.
3. Member C later books Court 3 for May 10, 6–7 PM.
4. System checks availability.
5. System checks punch card has credits.
6. System creates one `PRACTICE` session.
7. System deducts one credit.

Result:

```text
Member C has one practice session.
Punch card remaining credits decrease by 1.
```

---

### Scenario D: Practice Booking with Membership

1. Member D buys monthly membership with 8 included sessions.
2. No session is created at purchase time.
3. Member D books Court 2 for May 12, 5–6 PM.
4. System checks active membership.
5. System creates one `PRACTICE` session.
6. System deducts one included session.

Result:

```text
Member D has one practice session.
Membership remaining included sessions decrease by 1.
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
ClassPackage → ClassInstance → CLASS Session
```

for fixed class packages, and:

```text
Member → PRACTICE Session
```

for standalone court bookings.

Punch cards and memberships are not bookings. They are member-owned entitlements that can be used later to create practice sessions.

This design gives the business flexibility to support:

```text
Fixed class packages
Late registration
Prorated pricing
One member per court during class
Standalone practice court booking
Punch card credits
Membership benefits
Direct payments
Attendance tracking
Cancellation handling
Court conflict prevention
```
