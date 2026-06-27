<img width="1536" height="1024" alt="d72b7c40-9232-484f-b729-d7e1d767520e" src="https://github.com/user-attachments/assets/33595703-8599-4ce9-9b80-5b140ed50c55" />

Elasticsearch is usually **not updated directly by user requests** and is usually **not the source of truth**. It acts as a **search index** optimized for fast queries.

> Elasticsearch should be treated as a searchable index rather than the source of truth. I would update source systems first, publish events through Kafka, and use an indexing service to update Elasticsearch asynchronously. For high-frequency pricing and inventory updates, I would avoid constant reindexing and keep volatile data in Redis while Elasticsearch stores relatively stable searchable metadata.

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
