For synchronous APIs, circuit breakers, retries, and backoff are useful tools, but they are not "always apply everywhere" patterns. In synchronous APIs, resilience patterns improve availability but can absolutely increase latency if applied blindly.

Yes. That is exactly the tradeoff interviewers expect you to notice. In synchronous APIs, resilience patterns improve availability but can absolutely increase latency if applied blindly.

Example:

```text
Client → Order Service → Payment Service

Payment timeout = 2s
Retries = 3
Backoff = 100ms, 200ms, 400ms
```

Worst case:

```text
First attempt: 2s
Retry1: 100ms + 2s
Retry2: 200ms + 2s
Retry3: 400ms + 2s

Total ≈ 8.7s
```

Your user sees an 8–9 second response. That is usually unacceptable.

This is why experienced systems don't just "add retries"; they use a **latency budget**.

Example:

Request SLO:

```text
95% requests < 500ms
```

Budget allocation:

```text
Gateway: 50ms
Order Service: 50ms
Payment Service: 300ms
Buffer: 100ms
```

Then configure:

```text
Payment timeout = 150ms
Retries = 1
Backoff = 50ms
```

Maximum:

```text
150 + 50 + 150 = 350ms
```

Still inside the budget.

A few practical rules:

* For user-facing synchronous APIs:

  * very short timeouts
  * few retries (often 0–2)
  * aggressive circuit breakers

* For background jobs:

  * longer retries
  * larger backoff windows
  * multiple attempts acceptable

* Avoid retries at multiple layers

Bad:

```text
Gateway retries 3x
↓
Order retries 3x
↓
Payment retries 3x
```

Total calls:

```text
3 × 3 × 3 = 27 requests
```

This creates retry amplification and can take down a struggling service.

Better:

```text
Gateway → Order → Payment

Retry only at one layer
```

> Circuit breakers, retries, and backoff improve reliability, but in synchronous APIs they consume latency budget. I would use short timeouts, limited retries, and apply retries only for transient failures. If meeting response SLAs becomes difficult, I would move the workflow to asynchronous processing rather than stacking resilience mechanisms.


```text
Client → API Gateway → Order Service → Payment Service → Inventory Service
```

Typical things used in synchronous APIs:

* Timeouts
* Retries
* Exponential backoff
* Circuit breakers
* Bulkheads (resource isolation)
* Rate limiting
* Caching
* Idempotency
* Fallbacks
* Request correlation IDs
* Distributed tracing
* Load balancing

**Retries**
Use retries for **transient failures**:

Examples:

* Network timeout
* Temporary service unavailable
* Connection reset
* 503 response
* Short database failover

Good:

```text
Payment service timeout
→ retry after 100ms
→ retry after 200ms
→ retry after 400ms
```

Bad:

```text
POST /createOrder
→ retry blindly
```

Why bad? Because you may create duplicate orders.

For write APIs, retries usually need **idempotency keys**.

Example:

```text
POST /payments
Idempotency-Key: abc123
```

---

**Backoff**

Retries without backoff can create retry storms.

Bad:

```text
10000 requests fail
→ all retry immediately
→ downstream dies
```

Even better:

```text
Exponential backoff + jitter
```

Jitter adds randomness:

```text
Retry = random(0–400ms)
```

This avoids synchronized retries.

---

**Circuit Breaker**

Circuit breakers are useful when a dependency is repeatedly failing.

Flow:

```text
Closed
↓
Many failures
↓
Open
↓
Reject requests immediately
↓
Half-open
↓
Try a few requests
↓
Recover or reopen
```

Good use cases:

* Calling payment providers
* External APIs
* Slow microservices
* Third-party services


**Typical interview answer for synchronous microservices**

```text
Client
   ↓
API Gateway
   ↓
Order Service
   ↓
Circuit breaker
   ↓
Retry + exponential backoff + jitter
   ↓
Payment Service
```

Then add:

```text
Timeout = 2 sec
Retries = 3
Backoff = exponential
Idempotency key for writes
```

---

A common trap interviewers set:

*"Should I always retry?"*

Strong answer:

> No. Retries should be used only for transient failures. For write operations, retries require idempotency to avoid duplicate side effects. Excessive retries can amplify failures and cause cascading issues.
Retries can amplify failures. I would retry only transient failures, add exponential backoff with jitter, and ensure retries happen at a single layer to avoid retry storms.
