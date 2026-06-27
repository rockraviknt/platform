# 1. Search shows ₹5000, booking returns ₹5500 — what happens?

**Answer:**

Search traffic is huge, so I would use cached/asynchronously updated pricing for search. During booking I would synchronously validate with supplier before confirmation.

Flow:

```text
Search → cached price ₹5000
↓
User clicks Book
↓
Live supplier validation
↓
Supplier returns ₹5500
↓
Show "Price updated"
↓
User confirms
```

Tradeoff:

* Fast search
* Accurate booking
* Eventual consistency accepted during search

---

# 2. Two users book last room simultaneously

**Answer:**

I would avoid overselling using reservation + optimistic locking.

Flow:

```text
Inventory=1 Version=10

UserA → reserve inventory
UserB → reserve inventory
```

DB:

```sql
UPDATE inventory
SET rooms=rooms-1,version=11
WHERE hotelId=123
AND rooms>0
AND version=10
```

One succeeds.

Other receives:

```text
Inventory unavailable
```

---

# 3. Payment succeeds but inventory reservation fails

**Answer:**

Use Saga pattern with compensating actions.

Flow:

```text
Reserve inventory
↓
Process payment
↓
Create booking
```

Failure:

```text
Payment successful
Inventory failed
↓
Trigger refund event
```

Avoid distributed transactions.

---

# 4. Inventory reserved but payment fails

**Answer:**

Release reservation.

```text
Reserve inventory
↓
Payment failed
↓
Compensation:
Release room
```

Could be done asynchronously through event processing.

---

# 5. One supplier is slow

**Answer:**

Apply:

* timeout
* circuit breaker
* bulkhead
* partial response

Example:

```text
Supplier timeout=200ms
```

If timeout:

```text
Ignore supplier
Return remaining results
```

Never block entire search.

---

# 6. How to integrate 100 suppliers with different APIs?

**Answer:**

Use Adapter pattern.

Flow:

```text
Supplier API
↓
Adapter Layer
↓
Common internal model
↓
Pricing engine
```

Example:

```text
Supplier A → XML
Supplier B → JSON

Normalized:

Hotel {
    hotelId
    price
    inventory
}
```

---

# 7. How to avoid cache stampede?

Problem:

```text
Hot hotel cache expires
10000 requests hit DB
```

Solution:

* Redis distributed lock
* stagger TTL
* background refresh
* cache warming

Flow:

```text
Cache miss
↓
One request acquires lock
↓
Refresh cache
↓
Others wait
```

---

# 8. Why Kafka?

Use Kafka for:

* inventory updates
* pricing updates
* booking events
* notifications

Avoid Kafka for:

```text
Synchronous payment authorization
```

because immediate response is needed.

---

# 9. Duplicate booking event arrives twice

**Answer:**

Use idempotency.

```text
BookingEvent:
eventId=ABC123
```

Before processing:

```text
Check processed_events table
```

If exists:

```text
Ignore duplicate
```

---

# 10. Search latency target <300ms

**Answer:**

Architecture:

```text
Client
↓
API Gateway
↓
Search Service
↓
Redis cache
↓
Elasticsearch
↓
Supplier Aggregator
```

Techniques:

* parallel supplier calls
* async I/O
* partial response
* caching
* precomputed indexes
* CDN

---

# 11. Cache-aside vs write-through

For Booking:

**Pricing/Inventory**

Cache-aside preferred:

```text
Read:
Cache miss
↓
DB
↓
Update cache
```

Reason:

* write volume is huge

Write-through can create heavy write load.

---

# 12. How would you shard booking data?

Partition by:

```text
BookingId hash
```

or

```text
Region + BookingId
```

Avoid:

```text
Country only
```

because traffic can become skewed.

---
