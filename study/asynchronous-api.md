This is a common HLD follow-up question. If processing becomes asynchronous, you usually **do not keep the client waiting**. You acknowledge the request quickly and provide a way to track completion.

Typical pattern:

```text
Client
   ↓
POST /orders
   ↓
Order Service
   ↓
Save request
   ↓
Publish event → Queue/Kafka
   ↓
Return immediately
```

Immediate response:

```json
HTTP 202 Accepted

{
   "requestId":"req-12345",
   "status":"PROCESSING",
   "message":"Order request accepted"
}
```

Then background workers process it:

```text
Queue
   ↓
Payment Service
   ↓
Inventory Service
   ↓
Update DB status
```

The user later checks status:

```text
GET /orders/req-12345
```

Response:

```json
{
   "requestId":"req-12345",
   "status":"COMPLETED"
}
```

or:

```json
{
   "requestId":"req-12345",
   "status":"FAILED",
   "reason":"Payment declined"
}
```

There are several ways to inform users when processing completes:

1. Polling

```text
Client
→ POST /orders
← 202 Accepted

Client
→ GET /orders/{id}
← COMPLETED
```

Simple, but frequent polling can increase load.

2. WebSocket / Server-Sent Events

```text
Client
→ POST /orders
← requestId

Server pushes:
"Order completed"
```

Useful for live updates such as food delivery or chat.

3. Callback / Webhook

```text
POST /payment

{
   "callbackUrl":"https://client.com/payment-result"
}
```

After processing:

```text
POST https://client.com/payment-result
```

Common in payment systems.

4. Notification

```text
SMS
Email
Push notification
```

Common for travel booking or e-commerce systems.


> once you move to asynchronous processing, you also need to discuss **idempotency**, **duplicate event handling**, and **eventual consistency**, because queues can deliver the same message more than once.
