## How pricing team connect with external partners for getting price details 

Pricing systems usually integrate with many external hotel suppliers, airlines, channel managers, and partners.

A common flow looks like this:

```text id="7y5g5s"
User searches:
Delhi → Bangkok (July 10–15)

        ↓
Search API

        ↓
Pricing Aggregator Service
   ↙      ↓        ↘
Supplier A Supplier B Supplier C
(Hotelbeds) (Direct Hotel) (Channel Manager)

        ↓
Normalize responses

        ↓
Pricing Engine

        ↓
Apply business rules
- discounts
- promotions
- taxes
- currency conversion
- ranking

        ↓
Return final results
```

External partners usually connect through:

**Synchronous APIs**

```text id="5gkkq1"
GET /availability
GET /pricing
POST /reservation
```

Used when:

* live inventory needed
* real-time prices needed

Problem:

* slow partner response
* partner outages
* latency accumulation

Typical protection:

```text id="11c4jb"
Partner API call
   ↓
Timeout
   ↓
Circuit breaker
   ↓
Retry (small)
   ↓
Fallback
```

---

**Asynchronous feeds**

Partners often don't send every price request live.

Instead:

```text id="g5v7zq"
Partner
   ↓
Kafka/Event stream/SFTP/API feed
   ↓
Price ingestion service
   ↓
Store in cache/database
```

Examples:

* hotel inventory updates
* room prices
* availability changes
* promotions

Then user search hits cached/internal data instead of 50 partner APIs.

---

For booking-scale systems, a hybrid model is common:

```text id="s9g6sn"
Search request
     ↓
Read cached inventory/pricing
     ↓
If needed:
Fetch live updates from partners
     ↓
Merge results
```

Reason:

If company queried 100 external partners synchronously for every search:

```text id="x04by3"
Partner A = 100ms
Partner B = 200ms
Partner C = timeout
Partner D = 500ms
...
```

Search latency becomes very high.

Instead they often:

* pre-ingest pricing feeds
* cache aggressively
* refresh asynchronously
* fetch live only when required

---------
For pricing systems specifically, I would not choose purely synchronous or purely asynchronous. I would choose a **hybrid approach**.

**Synchronous is better when freshness is critical**

Example:

```text id="gcafq6"
User clicks "Book now"
↓
Verify latest room availability
↓
Verify latest price
↓
Reserve inventory
```

Reason:

* You need the latest data
* Cached data may be stale
* Wrong price can cause booking failures
---

**Asynchronous is better when scale is critical** PULL or PUSH model

Example:

```text id="7b5hr5"
Partner sends:
- Room prices
- Promotions
- Availability changes
↓
Ingestion service
↓
Cache/DB
```

Reason:

* User searches become very fast
* Less load on partners
* Better availability

Downside:

* Data may be slightly stale
* Eventual consistency issues

---

For Booking search flow, I would answer:

```text id="ek3icq"
Search request
     ↓
Read cached pricing
     ↓
Show results quickly

Booking request
     ↓
Real-time validation with partner
     ↓
Confirm booking
```

This gives:

* Fast search experience
* Fresh booking data
* Lower partner dependency
* Better scalability

*"Should pricing retrieval be synchronous or asynchronous?"*
> I would use asynchronous ingestion for search because search traffic is huge and users expect low latency. During booking I would use synchronous validation with the partner to ensure price and availability are current. That balances freshness, scalability, and user experience.

--------

Bad design:

```text id="r4j6sm"
Supplier1 → 100ms
Supplier2 → 150ms
Supplier3 → 200ms
...
Supplier100 → 120ms
```

Total latency:

```text id="1h6w6t"
100 + 150 + 200 + ...
≈ several seconds
```

No travel site can wait that long.

Instead, requests are typically sent **in parallel**.

```text id="k7cpl4"
                 → Supplier1
               ↗
Search Service → Supplier2
               ↘
                 → Supplier3
                    ...
                 → Supplier100
```

Latency now becomes approximately:

```text id="n88e2g"
Max(response time of suppliers)
```

Instead of:

```text id="5itv0o"
Sum(response time of suppliers)
```

But another issue appears: **100 parallel requests can create problems.**

Problems:

* Thread explosion
* Connection pool exhaustion
* CPU overhead
* Slow suppliers blocking resources
* Cascading failures

So large systems usually use:

**Async non-blocking calls**

Java examples:

```java
CompletableFuture
WebClient
Reactive streams
Netty
```

Instead of:

```java
RestTemplate (blocking)
```

Then add protections:

```text id="r1w80o"
Supplier call
    ↓
Timeout: 200ms
    ↓
Circuit breaker
    ↓
Bulkhead
    ↓
Limited retries
```

Also, they often don't wait for all suppliers.

Example:

```text id="rz8rpd"
Need top 20 hotel results
```

Approach:

```text id="s5jh0e"
Send requests to 100 suppliers
↓
Wait only 300ms
↓
Collect available responses
↓
Return results
↓
Late responses ignored
```

This is called a **partial response strategy**.

So users may see:

```json id="x6z5iq"
{
  "hotels":[...20 results...],
  "partialResults": true
}
```

rather than waiting 5 seconds.

> I would not make sequential calls. I would send requests in parallel using asynchronous non-blocking clients, enforce timeouts and circuit breakers per supplier, and return partial results after a response deadline instead of waiting for every supplier. This keeps search latency predictable while avoiding resource exhaustion.

----------

That is a strong follow-up. The remaining results usually are **not discarded blindly**. What happens depends on the business requirement.

For a travel search system, a common approach is:

```text id="prg2z6"
User search
↓
Send requests to 100 suppliers
↓
Wait for 300ms SLA
↓
Receive 80 supplier responses
↓
Return results to user immediately
```

The remaining 20 suppliers may arrive later.

Options:

**1. Update results asynchronously (common)**

```text id="m6jlwm"
Late supplier response
↓
Update cache/search store
↓
Push refreshed results
```

Examples:

* User sees "Loading more hotels..."
* UI refreshes automatically
* Infinite scroll adds more results

Very common because search experience remains fast.

---

**2. Ignore late responses**

```text id="m9rxlr"
Response after timeout
↓
Drop response
```

Used when:

* freshness window already expired
* search request already completed

No point keeping results that arrive after several seconds.

---

**3. Store for future searches**

```text id="xb6aqx"
Late response
↓
Normalize data
↓
Cache
```

Next user:

```text id="4tzh4k"
Delhi → Bangkok
```

May get faster results from cache.

---

"Asynchronous" does **not** mean you suddenly stop making REST calls. You still make REST calls to suppliers, but instead of blocking one thread per request, you use **non-blocking asynchronous I/O** and fire requests concurrently.

Typical flow:

```text id="7r3z6k"
Search Request
      ↓
Supplier Aggregator
      ↓
Create 100 async requests
      ↓
Send all requests concurrently
      ↓
Collect responses as they arrive
      ↓
Merge + rank results
      ↓
Return response
```

In Java, conceptually:

```java id="v0sx4o"
List<CompletableFuture<Response>> futures =
    suppliers.stream()
             .map(s -> asyncClient.getPrice(s))
             .toList();

CompletableFuture.allOf(
    futures.toArray(new CompletableFuture[0])
);
```

Problems:

* thread exhaustion
* context switching overhead
* memory usage

Better:

```text id="fzb2ej"
100 requests
→ event loop (few threads)
→ callbacks when responses arrive
```

For example:

```text id="snl0jd"
Thread1
    ↓
Send request to Supplier1
Send request to Supplier2
Send request to Supplier3
...
Send request to Supplier100

(No waiting)

Responses arrive later:
Supplier2 finished
Supplier17 finished
Supplier5 finished
```

This is why frameworks like:

* Java `WebClient`
* `CompletableFuture`
* `Netty`
* Reactive streams

are often used for aggregator services.

But another important issue appears:

Even with async, sending 100 requests simultaneously can overwhelm:

* your connection pool
* supplier APIs
* your CPU

So usually you limit concurrency:

```text id="8m18xq"
100 suppliers total

Process:
Batch1 → 20 concurrent
Batch2 → next 20
Batch3 → next 20
...
```

Or use a semaphore:

```text id="x1lnsh"
Max concurrent supplier calls = 20
```

This prevents resource exhaustion.

> I would still make REST calls, but use non-blocking asynchronous clients to issue requests concurrently. I would avoid creating one thread per supplier and instead rely on event-driven I/O. I would also limit concurrent requests using bulkheads or concurrency limits, apply timeouts and circuit breakers, and aggregate responses within a predefined latency budget.

----

## How to handle real time Inventory updates
> I would prefer delta updates for real-time inventory because full replacement is expensive at scale. I would include versioning or sequence numbers to avoid stale updates overwriting newer data. I would also run periodic full synchronization jobs to reconcile any missed events and maintain correctness.

Let's examine the options.

**1. Full overwrite (replace entire hotel data)**

```text id="vow5tw"
Existing cache:
Hotel123
Rooms = 10
Price = ₹5000

New supplier payload:
Hotel123
Rooms = 8
Price = ₹5200

Action:
Replace entire object
```

Advantages:

* Simple & easy implementation
* Easy consistency

Problems:

* Large payloads
* High network cost
* Frequent cache churn
* Risk of temporarily removing valid data
* Not suitable for frequent updates

Bad for:

```text id="l9a0l7"
Millions of hotels
Thousands of updates/sec
```

---

**2. Delete existing + reload**

```text id="3czzbo"
Delete Hotel123
↓
Load fresh Hotel123
```

Usually not preferred.

Problems:

```text id="3ezf9q"
t1: delete cache
t2: user reads cache
t3: cache miss
t4: reload arrives
```

Users may see missing inventory.

---

**3. Delta updates (commonly preferred)**

Supplier sends only changes:

```text id="r5h2w0"
Existing cache:

Hotel123
RoomA = 5
RoomB = 10
RoomC = 2

Supplier event:

RoomB = 7
```

Update:

```text id="xmuw03"
Hotel123
RoomA = 5
RoomB = 7
RoomC = 2
```

Advantages:

* Small payloads
* Less network traffic
* Faster updates
* Lower cache write load(**important**)

This is common at large scale.

---

But delta updates introduce another problem:

**Out-of-order events** unorder events

Example:

```text id="89by4h"
Event1:
Inventory=8
Timestamp=10:01

Event2:
Inventory=6
Timestamp=10:02
```

Suppose network delay occurs:

```text id="bplmcv"
Receive Event2 first
Receive Event1 later
```

Without protection:

```text id="f7ht5s"
Inventory becomes 8
```

Wrong result.

Usually systems add:

```text id="vf2t3j"
version
timestamp
sequence number
```

Example:

```text id="lnmiv0"
Hotel123
Inventory=6
Version=25
```

Incoming:

```text id="8hmh5e"
Inventory=8
Version=24
```

Reject because:

```text id="m8jdtw"
24 < 25
```

---

What many large systems do:

```text id="l3n3v9"
Daily/weekly:
Full refresh

Real-time:
Delta updates
```

Flow:

```text id="vgr5s3"
Supplier
    ↓
Inventory event
    ↓
Normalize event
    ↓
Version check
    ↓
Apply delta update
    ↓
Update cache
```

Then periodically:

```text id="y8qsl0"
Full reconciliation job
```

to fix missed events.
