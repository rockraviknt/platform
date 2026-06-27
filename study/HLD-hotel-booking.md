<img width="1536" height="1024" alt="d72b7c40-9232-484f-b729-d7e1d767520e" src="https://github.com/user-attachments/assets/33595703-8599-4ce9-9b80-5b140ed50c55" />

Elasticsearch is usually **not updated directly by user requests** and is usually **not the source of truth**. It acts as a **search index** optimized for fast queries.

> Elasticsearch should be treated as a searchable index rather than the source of truth. I would update source systems first, publish events through Kafka, and use an indexing service to update Elasticsearch asynchronously. For high-frequency pricing and inventory updates, I would avoid constant reindexing and keep volatile data in Redis while Elasticsearch stores relatively stable searchable metadata.

> Indexes make database queries faster, but they still involve disk access and database resources. Redis keeps frequently accessed data in memory, reducing database load and improving latency. The cache is particularly valuable when read traffic is much larger than write traffic. 

Typical flow:

```text id="v9s9gd"
Supplier
    ↓
Adapter Layer
    ↓
Kafka
    ↓
Stream Processing
    ↓
Primary DB + Cache update
    ↓
   CDC(Events)
    ↓
  Queue
    ↓
Indexing Service
    ↓
Transform document
    ↓
Update Elasticsearch
```

Document in Elasticsearch:

```json id="4m09yu"
{
   "hotelId":"H123",
   "city":"Delhi",
   "price":5200,
   "rating":4.5,
   "availableRooms":8,
   "amenities":["wifi","pool"]
}
```

**What if price updates arrive every second?**

If Elasticsearch receives:

```text id="mk2z5f"
100k updates/sec
```

it can struggle.

Common approaches:

1. Batch updates

```text id="g7m74h"
Collect updates for 1–2 seconds
↓
Bulk API
↓
Elasticsearch
```

2. Index only searchable fields

Store:

```text id="d8j8zh"
hotelId
city
rating
amenities
price range
```

Avoid indexing frequently changing fields unnecessarily.

3. Keep ultra-dynamic data in Redis

Instead of:

```text id="ydu5vv"
Elasticsearch:
price=₹5234
inventory=7
```

Search flow:

```text id="9lmg75"
Search ES
↓
Get hotel IDs
↓
Fetch latest prices from Redis
↓
Merge results
```

This is very common because inventory and prices can change much faster than hotel metadata.

----

External suppliers usually do **not** get direct access to internal Kafka clusters of travel platforms. Giving hundreds of external companies direct access to internal infrastructure would create major security and operational risks.

> I would never expose internal Kafka directly to suppliers. Suppliers would communicate through an API Gateway or ingestion layer secured with OAuth or mTLS. Adapter services would authenticate, authorize, validate, normalize, and then publish internally to Kafka. Kafka stays isolated inside private networks and is accessible only to internal services.

Typical design:

```text
Supplier
    ↓
Partner API Gateway
    ↓
Authentication + Authorization
    ↓
Supplier Adapter Service
    ↓
Validation / Normalization
    ↓
Internal Kafka
    ↓
Pricing & Inventory consumers
```

So suppliers push data to a controlled entry point, not directly to Kafka.

Common ways suppliers send updates:

**1. REST APIs (very common)**

Supplier calls:

```http
POST /inventory/update
Authorization: Bearer token

{
   "hotelId":"H123",
   "roomType":"Deluxe",
   "availableRooms":8,
   "timestamp":"2026-06-27T10:00:00Z"
}
```

Flow:

```text
Supplier
    ↓
HTTPS API
    ↓
Adapter
    ↓
Kafka publish
```

---

**2. Webhooks**

Supplier registers callback configuration:

```text
Inventory changed
↓
Supplier pushes update
↓
Webhook endpoint
```

---

**3. Batch file transfer**

Large suppliers sometimes send:

```text
inventory.csv
pricing.xml
```

via:

* SFTP
* scheduled uploads
* cloud object storage

Flow:

```text
Supplier
    ↓
SFTP
    ↓
File processor
    ↓
Kafka
```

---

**4. Partner-specific message queues**

Very large partners may use:

```text
Supplier Queue
↓
Connector Service
↓
Kafka
```

---

Now the security part.

> How do we ensure suppliers cannot access Booking internal Kafka?

Typically multiple security layers are used.

**Network isolation**

```text
Internet
    ↓
API Gateway
    ↓
Private VPC
    ↓
Internal Kafka
```

Kafka lives in private networks:

```text
No public IP
No internet exposure
```

---

**Authentication**

Supplier gets credentials:

Examples:

* OAuth2 client credentials
* API key
* JWT token
* mTLS certificates

Request:

```http
Authorization: Bearer ey...
```

---

**Authorization**

Each supplier only gets access to its own resources.

Enforced via:

* RBAC
* scopes
* access policies

---

**mTLS (Mutual TLS)**

Both sides validate certificates:

```text
Supplier cert
      ↔
API Gateway cert
```

Benefits:

* verify supplier identity
* prevent impersonation

---

**Rate limiting**

Prevent abuse:

```text
Supplier A:
100 requests/sec
```

If exceeded:

```text
HTTP 429
```

---

**Validation layer**

Never trust supplier payloads.

Check:

* schema validation
* mandatory fields
* timestamps
* duplicate requests
* malicious content

Example:

```text
Price = -5000
```

Reject immediately.

---

# If Elasticsearch is already optimized for search, why do we still need Redis?

The answer is that **Elasticsearch and Redis solve different problems**.

Think of them like this:

```text id="h0g95d"
Elasticsearch → "Find me WHICH hotels"

Redis → "Give me CURRENT values quickly"
```

Example user query:

```text id="vwgz9s"
Delhi
Price < ₹7000
Pool
Near airport
Rating > 4
```

### Step 1: Elasticsearch

Elasticsearch is good at:

* filtering
* ranking
* full-text search
* geo search
* sorting

Returns:

```text id="f2s8yg"
Hotel123
Hotel456
Hotel789
...
```

Maybe:

```text id="xq1t4v"
Top 100 hotel IDs
```

---

### Step 2: Redis

Now fetch highly dynamic fields:

```text id="v3dqt5"
Hotel123
Price=₹5200
Inventory=8

Hotel456
Price=₹4800
Inventory=2
```

because these values change frequently.

Flow:

```text id="1egj54"
User Search
      ↓
Elasticsearch
      ↓
Get hotel IDs
      ↓
Redis
      ↓
Get latest prices/inventory
      ↓
Merge response
```

---

Why not keep prices and inventory only in Elasticsearch?

Because inventory and prices change too frequently.

Imagine:

```text id="36k6gz"
100 suppliers
1M inventory changes/hour
```

If every change updates Elasticsearch:

```text id="szyt8f"
Supplier update
↓
Reindex Elasticsearch document
↓
Refresh index
```

Problems:

* heavy indexing load
* expensive updates
* index refresh overhead
* slower search performance

Elasticsearch is optimized more for:

```text id="o7frw2"
Read many
Write relatively less
```

not:

```text id="r9c2pv"
Millions of tiny updates/sec
```

Redis handles that much better:

```text id="zpw0f5"
SET price:H123:2026-07-15 5200
```

Very lightweight.

---

Another reason:

Suppose inventory changes:

```text id="9vph9d"
Hotel123 inventory:
8 → 7 → 6 → 5
```

Redis update:

```text id="wgnb96"
SET inventory:H123 5
```

Very cheap.

ES update:

```text id="r4jyxk"
Update hotel document
Re-index internal structures
Refresh shard
```

More expensive.

---

So at Booking scale:

```text id="0k0kj7"
Elasticsearch:
- hotel metadata
- amenities
- location
- ratings
- searchable attributes

Redis:
- latest price
- inventory
- temporary reservation state

PostgreSQL:
- source of truth
```


