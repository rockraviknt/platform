
---
> Partial responses improve user experience, retries handle transient failures, observability enables diagnosis, and backpressure protects system stability.


# 1. Handling partial responses

Problem:

A request depends on multiple downstream services.

Example:

```text id="jkg50g"
Client
   |
Booking Service
   |
--------------------------------
|              |              |
Payment     Inventory    Recommendation
```

Suppose:

* Payment → success
* Inventory → success
* Recommendation → timeout

Question:

Should entire request fail?

Many candidates say:

> Return error.

Lead-level thinking:

> It depends on whether the dependency is critical.

Classify dependencies:

### Critical

If it fails:

```text id="8st75g"
Payment
Inventory
```

Booking cannot proceed.

---

### Non-critical

If it fails:

```text id="8g8pbm"
Recommendations
Analytics
Personalization
```

Booking can still succeed.

Response:

```json
{
   "bookingStatus":"CONFIRMED",
   "recommendations":[]
}
```

Possible response metadata:

```json
{
   "bookingStatus":"CONFIRMED",
   "warnings":[
      "recommendations unavailable"
   ]
}
```

Interview answer:

> I identify critical versus non-critical dependencies. Critical failures block the workflow, while non-critical failures degrade gracefully and return partial responses.

Expected follow-up:

**How do you prevent long waits?**

Answer:

> I apply downstream timeouts and circuit breakers so one failing dependency doesn't delay the entire request.

---

# 2. Retries with backoff

Problem:

Transient failures happen:

* network hiccups
* temporary overload
* brief service outages

Bad approach:

```java id="q2ojtz"
while(true){
    retry();
}
```

Problems:

* retry storms
* overload amplification
* cascading failures

---

Correct approach:

### Exponential backoff

```text id="cpx4vx"
Retry 1 → 1 sec

Retry 2 → 2 sec

Retry 3 → 4 sec

Retry 4 → 8 sec
```

Even better:

### Exponential backoff + jitter

```text id="ad3vfo"
1.3 sec
2.1 sec
4.8 sec
```

Why jitter?

Without jitter:

```text id="52g2zm"
1000 clients retry together
```

causing another traffic spike.

---

Also mention:

* maximum retry count
* retry only transient errors
* don't retry business errors

Example:

Retry:

✓ 503

✓ timeout

✓ connection reset

Don't retry:

✗ invalid payment card

✗ authentication failure

Strong interview answer:

> I retry only transient failures with exponential backoff and random jitter, while limiting retry count to avoid retry storms.

---

# 3. Observability

This is almost guaranteed in Lead interviews.

Say:

> I think in terms of logs, metrics, and traces.

These are the three pillars.

---

## Logs

Purpose:

Understand individual events.

Example:

Bad:

```java id="4x08t4"
log.error("Error")
```

Good:

```java id="q9m3b1"
log.error(
"Payment failed",
kv("bookingId",bookingId),
kv("userId",userId),
kv("errorCode",code)
)
```

Important:

* structured logs
* correlation IDs
* avoid sensitive data

---

## Metrics

Purpose:

See system health trends.

Typical metrics:

```text id="w7lm98"
Request count

Error rate

Latency

CPU

Memory

Kafka lag
```

Important percentiles:

```text id="j5s4fd"
p50

p95

p99
```

Expected answer:

> Average latency can hide spikes, so I monitor p95 and p99.

---

## Tracing

Purpose:

Follow requests across services.

Example:

```text id="l4nslk"
RequestID=ABC

Gateway
   ↓
Booking Service
   ↓
Payment Service
   ↓
Notification Service
```

If latency becomes:

```text id="9j6s72"
3 sec
```

Tracing shows:

```text id="a5x8qo"
Payment Service:
2.7 sec
```

Tools commonly mentioned:

* OpenTelemetry
* Jaeger
* Zipkin
* Prometheus
* Grafana

Interview answer:

> Logs help explain events, metrics reveal trends, and tracing identifies where latency occurs across services.

---

# 4. Backpressure controls

This is a favorite distributed systems topic.

Problem:

Producer generates traffic faster than consumer can process.

Example:

```text id="b21n4o"
Producer

1000 req/sec

↓

Consumer

100 req/sec
```

Without control:

```text id="ztwn1k"
Queue grows

Memory increases

Latency increases

Eventually service crashes
```

Solutions:

---

### Rate limiting

Restrict incoming traffic.

Examples:

```text id="eqxwte"
100 requests/user/minute
```

Algorithms:

* Token bucket
* Sliding window

---

### Bounded queues

Bad:

```text id="ewqz9u"
Infinite queue
```

Better:

```text id="6l4mtc"
Queue size = 5000
```

Reject or shed traffic after threshold.

---

### Autoscaling

Increase consumers:

```text id="c0xbmk"
Consumers

2 → 10
```

---

### Circuit breakers

When dependency overloads:

```text id="r6o2uv"
Service A
     X
Service B
```

Stop sending requests temporarily.

---

### Kafka lag monitoring

Monitor:

```text id="zl5qv9"
Consumer offset lag
```

High lag:

* add consumers
* increase partitions
* reduce processing time

---

**Your notification service suddenly receives 10x traffic. What do you do?**

Strong answer:

> First I'd observe metrics such as request rate, CPU usage, queue depth, and Kafka lag. I would enable rate limiting and bounded queues to protect the system. I'd autoscale consumers if processing is parallelizable and use backpressure mechanisms so producers slow down rather than overwhelming consumers. If downstream dependencies become unhealthy, circuit breakers and retries with exponential backoff help prevent cascading failures.
