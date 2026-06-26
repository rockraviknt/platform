This is a very common Lead-level platform question. Interviewers are not checking whether you can recite definitions of unit/integration/E2E tests. They want to see whether you can connect **test type → component → failure mode → cost/benefit**.

A strong answer follows this pattern:

> “I think about testing as layers. Each layer validates different risks. I avoid overusing expensive tests and ensure failure modes are covered at the right level.”

Let's use an Agoda-style **Booking Service** example.

Architecture:

```text
Client
   |
API Gateway
   |
Booking Service
   |
---------------------------------
|               |               |
Redis         Database        Kafka
                                 |
                         Notification Service
```

Now map tests properly.

# 1. Unit tests

Purpose:

* Verify isolated business logic
* Mock dependencies
* Fast and cheap

Components tested:

* Booking validation logic
* Pricing calculation
* Retry logic
* Utility classes

Example:

```java
public BookingStatus createBooking(User u){

    if(u.isBlocked())
        throw new ValidationException();

    return SUCCESS;
}
```

Unit test:

```java
@Test
void shouldRejectBlockedUser()
```

Failure modes covered:

✓ Invalid inputs

✓ Boundary conditions

✓ Business-rule bugs

✓ Null handling

✓ Retry count logic

Not covered:

✗ DB failures

✗ Kafka failures

✗ Network problems

✗ Serialization issues

Interview statement:

> Unit tests give high confidence at low cost but cannot validate infrastructure interactions.

---

# 2. Integration tests

Purpose:

Verify interactions between components.

Components tested:

```text
Booking Service
      |
Database
```

or

```text
Booking Service
      |
Kafka
```

Usually:

* real DB/TestContainers
* embedded Kafka
* mocked external systems

Example:

```java
@Test
void shouldPersistBooking()
```

Verify:

```text
API call
    ↓
Repository call
    ↓
DB row created
```

Failure modes covered:

✓ Incorrect SQL

✓ Transaction failures

✓ Serialization/deserialization

✓ Connection issues

✓ Schema mismatch

✓ Repository bugs

Not covered:

✗ Full user flow

✗ Browser/mobile behavior

---

Example Agoda-style question:

**How would you test Kafka publishing?**

Good answer:

> I'd use integration testing with TestContainers Kafka instance and verify that events are published with expected payloads.

---

# 3. System tests

Purpose:

Verify complete application behavior.

Components:

```text
Booking Service
    |
Redis
Database
Kafka
```

All internal dependencies active.

Test:

```text
Create booking
    ↓
Booking saved
    ↓
Cache updated
    ↓
Event published
```

Failure modes covered:

✓ Configuration errors

✓ Service interactions

✓ Cache issues

✓ Resource exhaustion

✓ Timeout configuration

✓ Connection pooling problems

Example:

Suppose Redis is down.

Expected behavior:

```text
Booking still succeeds
↓
Fallback to database
```

Interview statement:

> System tests verify that internal components work together correctly.

---

# 4. End-to-End (E2E) tests

Purpose:

Simulate actual user journey.

Components:

```text
Client
 ↓
Gateway
 ↓
Booking Service
 ↓
DB
 ↓
Kafka
 ↓
Notification Service
```

Scenario:

```text
User searches hotel
      ↓
User books room
      ↓
Payment succeeds
      ↓
Notification received
```

Failure modes covered:

✓ User workflow failures

✓ Authentication issues

✓ Deployment issues

✓ API contract mismatches

✓ Missing configurations

✓ Cross-service failures

---

Example:

Test:

```text
User books hotel
```

Expected:

```text
Booking status = CONFIRMED

Notification delivered

Inventory reduced
```

---

# Failure-mode mapping (important to memorize)

| Failure mode               | Unit | Integration | System | E2E |
| -------------------------- | ---: | ----------: | -----: | --: |
| Validation logic bug       |    ✓ |             |        |     |
| SQL query issue            |      |           ✓ |      ✓ |   ✓ |
| Kafka serialization        |      |           ✓ |      ✓ |   ✓ |
| Redis unavailable          |      |             |      ✓ |   ✓ |
| Wrong service config       |      |             |      ✓ |   ✓ |
| Authentication issue       |      |             |        |   ✓ |
| Entire booking flow broken |      |             |        |   ✓ |
| Performance bottleneck     |      |             |      ✓ |   ✓ |

---

# Cost vs benefit (Lead-level discussion)

Interviewers love this follow-up:

> Why not do everything with E2E tests?

Good answer:

> E2E tests provide broad confidence but are expensive, slow, and brittle. I prefer many unit tests, fewer integration tests, limited system tests, and only critical-path E2E tests.

Typical ratio:

```text
          E2E
         /    \
    System
    /      \
Integration
 /          \
Unit
```

Example distribution:

* Unit → 70–80%
* Integration → 15–20%
* System → 5–10%
* E2E → critical flows only

---

If Agoda asks:

**"How would you test a booking platform?"**

You can answer in one concise flow:

> I’d test in layers. Unit tests validate booking logic and pricing rules. Integration tests verify DB and Kafka interactions. System tests ensure Redis, database, and service interactions work together including failure scenarios like cache outages. E2E tests validate the full user journey such as search → booking → payment → notification. I map each layer to specific failure modes and keep the majority of tests at the unit level for speed and maintainability.

That answer sounds like a Java Lead rather than someone reciting textbook definitions.
