<img width="1536" height="1024" alt="99a8266a-22d7-4042-ad31-6ecebefa4eff" src="https://github.com/user-attachments/assets/c79fe180-3f85-4a65-ae26-dcdd2583d8a7" />


In a real Booking-like system, suppliers do not simply call:

```http
GET /supplier/bookings?supplierId=SUP_456
```

and trust the `supplierId` from the request because that is insecure.

A malicious supplier could do:

```http
GET /supplier/bookings?supplierId=SUP_999
```

and attempt to access another supplier's bookings.

Instead, identity comes from the **authentication token**, not from the request payload.

Flow:

```text
Supplier System
      ↓
API Gateway
      ↓
Authentication
      ↓
Authorization
      ↓
Supplier Query API
      ↓
Read Model/Search DB
      ↓
Return bookings
```

Step by step:

### 1. Supplier authenticates

Supplier first requests an access token.

```http
POST /oauth/token
```

Request:

```json
{
   "clientId":"SUP_456",
   "clientSecret":"xxxx"
}
```

Authentication service validates credentials and issues JWT:

```json
{
   "accessToken":"eyJhbGc..."
}
```

JWT payload might contain:

```json
{
   "supplierId":"SUP_456",
   "roles":["BOOKING_READ"],
   "exp":1780000000
}
```

---

### 2. Supplier requests bookings

Supplier sends:

```http
GET /supplier/bookings?updatedAfter=2026-06-26T10:00:00Z&limit=100

Authorization: Bearer eyJhbGc...
```

Notice:

No supplierId in URL.

Because supplier identity already exists in JWT.

---

### 3. API Gateway validates request

Gateway checks:

```text
API Gateway
   ├── Verify JWT signature
   ├── Token expiry
   ├── Rate limit
   ├── IP whitelist
   └── Request logging
```

If invalid:

```http
401 Unauthorized
```

---

### 4. Authorization layer extracts supplier identity

Gateway or auth middleware extracts:

```json
{
   "supplierId":"SUP_456",
   "roles":["BOOKING_READ"]
}
```

Context becomes:

```text
requestContext.supplierId=SUP_456
```

Now downstream services never trust user input.

---

### 5. Query service automatically injects supplier scope

Instead of:

```sql
SELECT *
FROM supplier_read_model
WHERE updated_at>'10:00'
```

System internally executes:

```sql
SELECT *
FROM supplier_read_model
WHERE supplier_id='SUP_456'
AND updated_at>'10:00'
LIMIT 100
```

Supplier cannot alter this.

---

### 6. CQRS read model serves data

Remember earlier architecture:

```text
Booking DB
     ↓
CDC/Kafka
     ↓
Projection Service
     ↓
Supplier Read Model
```

Read model may look like:

```text
supplier_read_model

booking_id
supplier_id
hotel_id
status
checkin_date
updated_at
guest_name
```

Indexes:

```sql
INDEX(supplier_id, updated_at)
```

Now query becomes fast:

```text
Find SUP_456
      ↓
Apply timestamp filter
      ↓
Return first 100 rows
```

No full table scan across millions of bookings.

---

Complete request flow:

```text
Supplier System
      ↓
Bearer Token
      ↓
API Gateway
      ↓
JWT Validation
      ↓
Extract supplierId=SUP_456
      ↓
Supplier Query Service
      ↓
Inject supplier scope
      ↓
Read Model / Elasticsearch
      ↓
Return booking summaries
```

Response:

```json
{
   "bookings":[
      {
         "bookingId":"B1001",
         "hotelId":"H123",
         "status":"CONFIRMED"
      },
      {
         "bookingId":"B1002",
         "hotelId":"H234",
         "status":"CANCELLED"
      }
   ],
   "nextCursor":"ABC123"
}
```

This is also a common interview follow-up:

> Why not pass supplierId in API request?

Expected answer:

> Because request parameters are user-controlled and can be tampered with. Supplier identity should come from authenticated claims (JWT/session context), and authorization should enforce row-level access automatically.


>Read replicas are useful to scale straightforward reads and reduce load on the primary database. However, for supplier queries involving filtering, search, dashboards, or millions of records, replicas alone become insufficient. I would combine replicas with CQRS and specialized read models so transactional workloads remain isolated from analytical or search-heavy workloads.

