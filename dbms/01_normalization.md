# Event Management Database Normalization (CP1)

## 1. Introduction and scope

This report analyzes the PostgreSQL schema for the Event Management application. It covers six relations defined by the Prisma model and committed SQL migrations: `User`, `RefreshToken`, `Event`, `Booking`, `Payment`, and `ProcessedWebhook`.

The walkthrough separates **database guarantees**, **rules followed by the current application**, and **hypothetical examples** used to illustrate 1NF, 2NF, and 3NF. The examples are not earlier versions of the schema. The analysis describes the repository's migrations and server code, not a separately inspected deployment.

## 2. Actual relations, keys, and functional dependencies

A functional dependency (FD) `X → Y` means any two rows that agree on `X` must also agree on `Y`. A candidate key is a minimal set of attributes that uniquely identifies every row. Here, `R − {x}` means all columns of relation `R` except `x`. Prisma relation fields such as `User.bookings` are navigation fields and do not appear as columns below.

### 2.1 `User`

**Relation:** `User(id, name, email, passwordHash, role, createdAt, updatedAt)`

- Primary key: `{id}`. Alternate candidate key: `{email}` (unique and required).
- FDs:
  - `id → name, email, passwordHash, role, createdAt, updatedAt`
  - `email → id, name, passwordHash, role, createdAt, updatedAt`
- No foreign keys. The database enum permits `ATTENDEE`, `ORGANIZER`, and `ADMIN`.

### 2.2 `RefreshToken`

**Relation:** `RefreshToken(id, userId, familyId, tokenHash, expiresAt, usedAt, revokedAt, createdAt)`

- Primary key: `{id}`. Alternate candidate key: `{tokenHash}` (unique and required).
- FDs:
  - `id → userId, familyId, tokenHash, expiresAt, usedAt, revokedAt, createdAt`
  - `tokenHash → id, userId, familyId, expiresAt, usedAt, revokedAt, createdAt`
- Foreign key: `userId` references `User.id` (deleting the user cascades to their refresh tokens).
- `familyId` is indexed and can repeat across token rotations. The application gives a new family to a new session and retains its owner on rotation. The resulting ownership rule is analyzed in §5.3.

### 2.3 `Event`

**Relation:** `Event(id, organizerId, title, description, category, coverKey, venue, address, city, timezone, startAt, endAt, registrationClosesAt, capacity, priceMinor, currency, status, cancelledAt, cancelReason, hiddenAt, moderationReason, moderatedAt, moderatedById, createdAt, updatedAt)`

- Primary and only declared candidate key: `{id}`.
- FD: `id → Event − {id}`.
- Foreign keys: `organizerId` references `User.id` (delete restricted); nullable `moderatedById` references `User.id` (set to null on deletion).
- The database enforces positive capacity, non-negative price, `endAt > startAt`, and `registrationClosesAt <= startAt`. These inequalities constrain valid values without fixing an exact value from the other column. The event status enum permits `DRAFT`, `PUBLISHED`, and `CANCELLED`.
- `category` is a text column. There is no separate category or venue table, and the schema does not require a venue name to identify a single address.

### 2.4 `Booking`

**Relation:** `Booking(id, userId, eventId, status, amountMinor, currency, eventSnapshot, holdExpiresAt, ticketCode, checkedInAt, cancelledAt, createdAt, updatedAt)`

- Primary and only declared candidate key for the complete table: `{id}`.
- FD: `id → userId, eventId, status, amountMinor, currency, eventSnapshot, holdExpiresAt, ticketCode, checkedInAt, cancelledAt, createdAt, updatedAt`.
- Foreign keys: `userId` references `User.id`; `eventId` references `Event.id` (both restrict deletion).
- `ticketCode` is unique when present, but nullable, so it is not a candidate key for every booking.
- A partial unique index makes `(userId, eventId)` unique among rows whose status is `PENDING_PAYMENT` or `CONFIRMED`. It permits multiple historical rows for the same pair outside that subset. The index uses status alone; a pending booking remains covered by it until its status changes, even after its hold time passes.
- `eventSnapshot` is a required JSONB value. The booking service records the title, start and end times, venue, address, city, timezone, price, and currency at booking creation. These are historical values; an event ID identifies the current event, which can have different values later.

### 2.5 `Payment`

**Relation:** `Payment(id, bookingId, provider, providerCheckoutId, providerPaymentId, status, amountMinor, currency, refundReference, lastErrorCode, createdAt, updatedAt)`

- Primary key: `{id}`. Alternate candidate key: `{bookingId}` (unique and required).
- FDs:
  - `id → Payment − {id}`
  - `bookingId → Payment − {bookingId}`
- Foreign key: `bookingId` references `Booking.id` (delete restricted). Each payment belongs to one booking; a booking has at most one payment. The free booking path creates no payment row.
- `providerCheckoutId` and `providerPaymentId` are each globally unique when present. Both are nullable and neither is a candidate key for the complete table.
- At paid booking creation, the application copies the booking amount and currency into the payment. The database does not enforce cross-table equality between those copies.

### 2.6 `ProcessedWebhook`

**Relation:** `ProcessedWebhook(provider, providerEventId, processedAt)`

- Composite primary and only declared candidate key: `{provider, providerEventId}`.
- FD: `(provider, providerEventId) → processedAt`.
- No foreign keys. The key lets the application record a provider event once. The SQL does not require provider event IDs to be globally unique across providers. The current webhook handler writes `razorpay`, while the table itself accepts other provider names.

## 3. First Normal Form (1NF)

For this analysis, 1NF means each attribute holds a single value at the chosen level of analysis and repeating business records have their own rows. A key identifies each row.

### 3.1 Actual relations

Users, events, bookings, payments, refresh tokens, and processed webhooks occupy separate rows. An event does not contain a list of its attendees, and a user does not contain a list of refresh tokens in one column.

At the top-level PostgreSQL column boundary, `Booking.eventSnapshot` is treated as one historical document value. The application writes a fixed object of named fields, not a repeating list of attendees or events. The SQL requires a non-null JSONB column but does not enforce the object's fields or structure. Under a stricter classroom interpretation in which addressable JSON members must be represented as separate relational attributes, this booking snapshot would need to be flattened before claiming 1NF for the full design. Section 5.3 explains what that would reveal.

### 3.2 Illustrative 1NF decomposition

Suppose a draft design kept a list of attendees in each event row:

| eventId | title           | attendeeIds |
| ------- | --------------- | ----------- |
| E101    | Design Workshop | `U21, U35`  |

The list stores multiple attendee references in one cell. Moving them to booking rows gives:

| eventId (PK) | title           |
| ------------ | --------------- |
| E101         | Design Workshop |

| bookingId (PK) | eventId (FK) | userId (FK) |
| -------------- | ------------ | ----------- |
| B501           | E101         | U21         |
| B502           | E101         | U35         |

Each booking refers to one event and one user. A booking ID also allows the same user and event to have distinct historical bookings. The IDs here are illustrative labels; the real tables use UUIDs.

## 4. Second Normal Form (2NF)

A relation is in 2NF if it is in 1NF and no non-prime attribute depends on a proper subset of a candidate key. A non-prime attribute belongs to no candidate key.

### 4.1 Actual relations

- `User`, `RefreshToken`, `Event`, `Booking`, and `Payment` have only single-column candidate keys under the declared whole-table constraints. There is no composite key to test for a partial dependency. The partial index on `Booking` does not change the key of the full table.
- `ProcessedWebhook` has the composite key `(provider, providerEventId)`. A provider handles multiple events, and one provider's event ID need not be unique across providers. The processing time is a fact about a particular recorded provider event; neither component alone determines it under the declared design.

Under the column-level treatment of JSONB in §3.1, all six relations pass the 2NF test for the stated schema-wide dependencies.

### 4.2 Illustrative 2NF decomposition

Consider a hypothetical current-attendance table with one row per user and event, and with current names copied in:

`CurrentAttendance(eventId, userId, eventTitle, attendeeName, bookedAt)`

- `(eventId, userId) → bookedAt`
- `eventId → eventTitle`
- `userId → attendeeName`

The last two dependencies use only part of the hypothetical key, so the table is not in 2NF. Separate the data into `Event(eventId, eventTitle)`, `User(userId, attendeeName)`, and `CurrentAttendance(eventId, userId, bookedAt)`, with the remaining pair referencing the two entity tables. The actual schema uses `Booking.id` for booking history and stores a historical event title in its snapshot. That title has different semantics from the copied **current** title in this example.

## 5. Third Normal Form (3NF)

A relation is in 3NF if, for each non-trivial FD `X → A`, `X` is a superkey or `A` is prime. A superkey uniquely identifies a row; unlike a candidate key, it need not be minimal.

### 5.1 Actual separation of entity facts

`Event` refers to its organizer and optional moderator by user ID rather than repeating their names, emails, or roles. `Booking` references the attendee and event, and `Payment` references the booking without including its attendee or event IDs. Keeping these descriptions in the relations that own them avoids dependencies through non-key foreign keys. The booking snapshot is an exception in purpose: it preserves booking-time values rather than acting as the source of current event details.

### 5.2 Illustrative 3NF decomposition

Suppose the current organizer's contact details were repeated on every event:

`EventWithOrganizer(eventId, organizerId, organizerName, organizerEmail, title)`

The event ID determines the entire row, but the organizer ID also determines the organizer's name and email. The chain `eventId → organizerId → organizerName, organizerEmail` violates 3NF because one organizer can have multiple events and those contact fields are non-prime.

Decompose into `User(organizerId, organizerName, organizerEmail)` and `Event(eventId, organizerId, title)`. The shared organizer ID can be used to join the two relations. The actual design already uses this pattern through `Event.organizerId → User.id`.

### 5.3 Physical schema and application qualifications

**Refresh token family ownership.** The authentication service creates a new family ID for a new session and reuses it with the same user when rotating tokens. If one owner per family is included as a domain rule, `familyId → userId` is a non-trivial FD in `RefreshToken`. A family can have multiple tokens, so `familyId` is not a superkey, and `userId` is non-prime. Under that rule, `RefreshToken` fails 3NF and BCNF. The SQL does not enforce the FD: a family ID is indexed, but there is no family table or ownership constraint. A conceptual decomposition would store ownership once in `TokenFamily(familyId, userId)` and reference the family from each token. The repository does not implement that decomposition.

**Booking snapshot and charge values.** The booking service writes `eventSnapshot.currency` and `Booking.currency` from the same event at creation. The paid path also writes `eventSnapshot.priceMinor` and `Booking.amountMinor` from the event price; the free path requires a zero price and writes an amount of zero. These copies overlap on the current creation paths. They are not equations enforced by a SQL check or generated column. With JSON treated as one object and the analysis limited to declared database rules, no schema-wide non-key FD follows from that overlap. If the snapshot fields are flattened and equality of the copies is adopted as a domain invariant, `snapshotCurrency → currency` and `snapshotPriceMinor → amountMinor` become non-key dependencies and the flattened design would fail 3NF and BCNF unless the duplicate attributes were removed. This is distinct from the historical snapshot itself: `eventId` does not determine a single historical title or price across all bookings.

**Payment amount and currency.** The paid booking service creates the booking and payment in the same transaction and copies the booking's amount and currency onto the payment. Keeping the two records consistent is a cross-table concern; the foreign key does not check copied amounts or currencies. Within `Payment`, `bookingId` is already a candidate key, so `bookingId → amountMinor, currency` does not violate 3NF. This does not eliminate the redundancy between the two tables.

**Moderation timestamps.** The admin route sets `moderatedAt` to the time of a hide or unhide action. A hide also sets `hiddenAt` to that timestamp, while an unhide clears `hiddenAt`. The migration backfills `moderatedAt` from existing non-null `hiddenAt` values. This is a conditional equality in the write path for hidden rows, not an equality constraint across the full table. The columns represent current visibility and the time of the latest moderation action, respectively.

### 5.4 3NF conclusion

With the snapshot treated as a single value, the declared schema-wide FDs identified in §2 satisfy 3NF: each listed determinant is a superkey. The broader domain analysis qualifies this result. Treating family ownership as an invariant gives a 3NF violation in `RefreshToken`; flattening the booking snapshot and treating its overlapping charge fields as invariants would expose additional dependencies.

## 6. Boyce–Codd Normal Form (BCNF, optional extension)

BCNF requires every determinant of a non-trivial FD to be a superkey, even when the dependent attribute is prime.

The declared whole-table FDs listed in §2 meet that test. In particular, `ProcessedWebhook` uses the whole provider and event ID pair, and `Payment.bookingId` is a candidate key. Nullable unique identifiers and `Booking`'s partial unique index do not create full-table candidate keys. Including the application-level `familyId → userId` dependency gives the same BCNF exception as in the 3NF analysis. Cross-table duplication of charge values is a separate concern.

## 7. Summary

| Stage | Actual-schema result                                                                                             | Qualification                                                                                                                                                |
| ----- | ---------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1NF   | Each record has a key; repeating bookings, refresh tokens, and webhook events occupy separate rows.              | `Booking.eventSnapshot` is treated as one JSONB document; a strict relational treatment of its fields would require flattening.                              |
| 2NF   | The only declared composite candidate key belongs to `ProcessedWebhook`; `processedAt` depends on the full pair. | The active booking pair is unique only within the index predicate.                                                                                           |
| 3NF   | All listed schema-wide FDs have superkey determinants.                                                           | The app's one-owner-per-family rule fails 3NF in `RefreshToken`; flattening the snapshot under the stated copy rules would also expose non-key determinants. |
| BCNF  | All listed schema-wide FDs satisfy the superkey test.                                                            | The same added domain dependencies prevent a blanket BCNF claim.                                                                                             |

## 8. Conclusion

The Event Management schema separates accounts, sessions, events, bookings, payments, and webhook processing into six related tables. The booking index allows historical bookings while preventing two rows for the same user and event from simultaneously occupying the indexed statuses. The distinction between schema guarantees and application behavior matters for the normalization result: refresh token family ownership is maintained by the authentication service, the event snapshot preserves historical details and overlaps with booking charge fields, and payment charge fields are copied across tables without a database equality constraint.

## 9. Schema, migration, and application sources

- [Prisma schema](../server/prisma/schema.prisma)
- [User and refresh token migration](../server/prisma/migrations/20261007123151_user_model/migration.sql)
- [Event, booking, payment, and webhook migration](../server/prisma/migrations/20261018185622_events_bookings_payments/migration.sql)
- [Event moderation timestamp migration](../server/prisma/migrations/20261115120000_event_moderation_timestamp/migration.sql)
- Application behavior: [authentication service](../server/src/services/auth.service.js), [booking service](../server/src/services/booking.service.js), [payment service](../server/src/services/payment.service.js), and [admin routes](../server/src/routes/admin.route.js).
