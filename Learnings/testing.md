# Testing Strategy for a Booking Service

A strong answer follows this pattern:
> “I think about testing as layers. Each layer validates different risks. I avoid overusing expensive tests and ensure failure modes are covered at the right level.”

Let's use an Agoda-style Booking Service example.

## Architecture:
- Client
  
- API Gateway
  
- Booking Service
  
---
|               |               |
|--------------|--------------|
| Redis        | Database     |
| Kafka        | Notification Service |

## Map Tests Properly

### 1. Unit Tests
**Purpose:**
- Verify isolated business logic
- Mock dependencies
- Fast and cheap
**Components Tested:**
- Booking validation logic
- Pricing calculation
- Retry logic
- Utility classes
**Example:**
```java
public BookingStatus createBooking(User u){
    if(u.isBlocked())
        throw new ValidationException();
    return SUCCESS;
}
```
**Unit Test:**
```java
'test shouldRejectBlockedUser()'
defaults to cover failure modes:
x Valid inputs ✓
x Boundary conditions ✓
x Business-rule bugs ✓
x Null handling ✓
x Retry count logic ✓
definitions of not covered:
x DB failures ✗
x Kafka failures ✗
x Network problems ✗
x Serialization issues ✗
discussion: Unit tests give high confidence at low cost but cannot validate infrastructure interactions.
```

### 2. Integration Tests
**Purpose:** Verify interactions between components.
**Components Tested:** 
- Booking Service with Database or Kafka (or both)
**Usually:** 
test with real DB/TestContainers, embedded Kafka, mocked external systems.
**Example:** 
defaults to verify persistence:
def test shouldPersistBooking()
does the following:
detect API call → repository call → DB row created.
failure modes covered:
x Incorrect SQL ✓,
x Transaction failures ✓,
x Serialization/deserialization ✓,
x Connection issues ✓,
x Schema mismatch ✓,
x Repository bugs ✓.
note: Not covering full user flow or browser/mobile behavior.
e.g., How would you test Kafka publishing?
good answer: Use integration testing with TestContainers Kafka instance and verify events are published with expected payloads.

### 3. System Tests
**Purpose:** Verify complete application behavior.
**Components:** 
booking service, Redis, Database, Kafka (all internal dependencies active)
deliverables:
test creating booking → booking saved → cache updated → event published.
failure modes covered:
x Configuration errors ✓,
x Service interactions ✓,
x Cache issues ✓,
x Resource exhaustion ✓,
x Timeout configuration ✓,
x Connection pooling problems ✓.
e.g., Suppose Redis is down; expected behavior: booking still succeeds via fallback to database.
discussion: System tests verify that internal components work together correctly.

### 4. End-to-End (E2E) Tests
**Purpose:** Simulate actual user journey.
**Components:** client, gateway, booking service, DB, Kafka, notification service.
scenario:
hotel search → book room → payment success → notification received.
failure modes covered:
y User workflow failures ✓,
y Authentication issues ✓,
y Deployment issues ✓,
y API contract mismatches ✓,
y Missing configurations ✓,
y Cross-service failures ✓.
e.g., Test user books hotel; expected outcomes include booking status = CONFIRMED, notification delivered, inventory reduced.
distribution of failure mode mapping across layers is summarized in a table below:
| Failure Mode | Unit | Integration | System | E2E |
|----------------|-------|--------------|--------|-----|
| Validation logic bug |✓|||
or SQL query issue ||✓|✓|✓|
or Kafka serialization ||✓|✓|✓|
or Redis unavailable ||||✓|
or Wrong service config ||||✓|
or Authentication issue ||||✓|
or Entire booking flow broken ||||✓|
or Performance bottleneck ||||✓|✓|
table end.
