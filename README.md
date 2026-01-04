# Long-Distance Delivery Handoff Service

A backend service for managing long-distance deliveries that may be handled by multiple riders sequentially. The system ensures correct handoff of orders between riders while maintaining data consistency under concurrent requests.

## Technology Stack

| Component | Technology          | Purpose                               |
| --------- | ------------------- | ------------------------------------- |
| Framework | NestJS (TypeScript) | REST API backend                      |
| Database  | PostgreSQL          | Data persistence                      |
| Locking   | Redis               | Distributed locks + idempotency cache |

## Assumptions

1. **Sequential Handoffs** - Orders can be handled by multiple riders, one after another. Each rider completes a "leg" of the delivery.

2. **Single Active Rider** - Only one rider can work on an order at any given time.

3. **Trusted Riders** - No validation that the rider finishing is the same rider who started. The client is trusted.

4. **No Authentication** - No auth layer; rider IDs are passed directly in requests.

5. **No Routing** - No maps, distance calculations, or routing logic.

6. **Client-Provided Idempotency** - Clients must provide idempotency keys for duplicate request handling.

## Architecture

### State Machine

Orders follow a strict state machine to ensure valid transitions:

```
┌─────────┐      ┌─────────────┐      ┌──────────────────┐      ┌───────────┐
│ CREATED │ ──▶  │ IN_PROGRESS │ ──▶  │ AWAITING_HANDOFF │ ──▶  │ DELIVERED │
└─────────┘      └─────────────┘      └──────────────────┘      └───────────┘
                        ▲                      │
                        │                      │
                        └──────────────────────┘
                          (next rider starts)
```

**Valid Transitions:**

- `CREATED` → `IN_PROGRESS`: First rider starts the order
- `IN_PROGRESS` → `AWAITING_HANDOFF`: Current rider finishes their leg
- `AWAITING_HANDOFF` → `IN_PROGRESS`: Next rider picks up the order
- `IN_PROGRESS` → `DELIVERED`: Final rider completes delivery (with `isFinalDelivery` flag)

### Concurrency Strategy

**Redis Distributed Lock:**

- Single lock per order: `order:lock:{orderId}`
- TTL: 30 seconds (auto-expire to prevent deadlocks)
- All state-changing operations must acquire the lock first

**Request Flow:**

```
Client Request
      │
      ▼
Check Idempotency Key ──▶ [Cache Hit] ──▶ Return Cached Response
      │
      ▼ [Cache Miss]
Acquire Redis Lock (with retry)
      │
      ▼
Validate State Transition
      │
      ▼
Update Database (within transaction)
      │
      ▼
Cache Response (idempotency)
      │
      ▼
Release Lock
      │
      ▼
Return Response
```

### Idempotency

- Clients provide `Idempotency-Key` header on mutating requests
- Responses cached in Redis with 24-hour TTL
- Duplicate requests return the cached response immediately

## Data Model

### orders

| Column           | Type            | Description                                       |
| ---------------- | --------------- | ------------------------------------------------- |
| id               | UUID            | Primary key                                       |
| status           | ENUM            | CREATED, IN_PROGRESS, AWAITING_HANDOFF, DELIVERED |
| current_rider_id | UUID (nullable) | ID of the rider currently working on the order    |
| created_at       | TIMESTAMP       | Order creation time                               |
| updated_at       | TIMESTAMP       | Last modification time                            |

### order_legs

Tracks each rider's segment of the delivery for audit and debugging.

| Column      | Type                 | Description                      |
| ----------- | -------------------- | -------------------------------- |
| id          | UUID                 | Primary key                      |
| order_id    | UUID                 | Foreign key to orders            |
| rider_id    | UUID                 | Rider handling this leg          |
| leg_number  | INT                  | Sequence number (1, 2, 3...)     |
| status      | ENUM                 | IN_PROGRESS, COMPLETED           |
| started_at  | TIMESTAMP            | When the rider started this leg  |
| finished_at | TIMESTAMP (nullable) | When the rider finished this leg |

## API Endpoints

### Create Order

```
POST /orders
```

**Response:**

```json
{
  "id": "uuid",
  "status": "CREATED",
  "currentRiderId": null,
  "createdAt": "2026-01-04T10:00:00Z"
}
```

### Start Order (Rider Begins Work)

```
POST /orders/:id/start
Headers:
  Idempotency-Key: <unique-key>
Body:
{
  "riderId": "uuid"
}
```

**Response:**

```json
{
  "id": "uuid",
  "status": "IN_PROGRESS",
  "currentRiderId": "rider-uuid",
  "legNumber": 1
}
```

### Finish Order (Rider Completes Leg)

```
POST /orders/:id/finish
Headers:
  Idempotency-Key: <unique-key>
Body:
{
  "riderId": "uuid",
  "isFinalDelivery": false
}
```

**Response:**

```json
{
  "id": "uuid",
  "status": "AWAITING_HANDOFF",
  "currentRiderId": null,
  "legNumber": 1,
  "legStatus": "COMPLETED"
}
```

Set `isFinalDelivery: true` to mark the order as `DELIVERED`.

### Get Order (Optional)

```
GET /orders/:id
```

**Response:**

```json
{
  "id": "uuid",
  "status": "IN_PROGRESS",
  "currentRiderId": "rider-uuid",
  "createdAt": "2026-01-04T10:00:00Z",
  "updatedAt": "2026-01-04T10:05:00Z",
  "legs": [
    {
      "legNumber": 1,
      "riderId": "rider-1-uuid",
      "status": "COMPLETED",
      "startedAt": "2026-01-04T10:01:00Z",
      "finishedAt": "2026-01-04T10:03:00Z"
    },
    {
      "legNumber": 2,
      "riderId": "rider-2-uuid",
      "status": "IN_PROGRESS",
      "startedAt": "2026-01-04T10:05:00Z",
      "finishedAt": null
    }
  ]
}
```

## Edge Case Handling

| Scenario                       | Behavior                                                               |
| ------------------------------ | ---------------------------------------------------------------------- |
| **Duplicate request**          | Return cached response via idempotency key                             |
| **Out-of-order finish**        | Reject with 400 - cannot finish order not in IN_PROGRESS               |
| **Concurrent rider starts**    | First request acquires lock; second waits, then fails state validation |
| **Start on delivered order**   | Reject with 400 - order already delivered                              |
| **Start on in-progress order** | Reject with 400 - another rider already working                        |
| **Lock acquisition timeout**   | Reject with 409 - resource busy, retry later                           |
| **Lock holder crashes**        | Lock auto-expires after 30s TTL                                        |

## Error Responses

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Invalid state transition: cannot start order in IN_PROGRESS state"
}
```

```json
{
  "statusCode": 409,
  "error": "Conflict",
  "message": "Unable to acquire lock. Resource is busy."
}
```

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Order not found"
}
```
