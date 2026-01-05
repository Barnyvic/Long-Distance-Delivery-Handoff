# Long-Distance Delivery Handoff Service

A backend service for managing long-distance deliveries that may be handled by multiple riders sequentially. The system ensures correct handoff of orders between riders while maintaining data consistency under concurrent requests.

## Problem Statement

In long-distance delivery scenarios, a single rider may not be able to complete the entire journey. Instead, the delivery is broken into segments (or "legs"), where each segment is handled by a different rider. This creates a relay-style delivery system.

**Key challenges this system solves:**

1. **Handoff Coordination** - Ensuring only one rider works on an order at a time, with clean transitions between riders.
2. **Concurrent Access** - Preventing race conditions when multiple riders or requests try to modify the same order simultaneously.
3. **Network Reliability** - Handling duplicate requests from network retries and out-of-order message delivery gracefully.
4. **Audit Trail** - Tracking which rider handled which segment for accountability and debugging.

## Technology Stack

| Component | Technology          | Purpose                               |
| --------- | ------------------- | ------------------------------------- |
| Framework | NestJS (TypeScript) | REST API backend                      |
| Database  | PostgreSQL          | Data persistence                      |
| Locking   | Redis               | Distributed locks + idempotency cache |

## Assumptions

1. **Sequential Handoffs** - Orders can be handled by multiple riders, one after another. Each rider completes a "leg" of the delivery. There is no limit to the number of handoffs an order can have.

2. **Single Active Rider** - Only one rider can work on an order at any given time. A new rider cannot start until the previous rider has finished their leg.

3. **Trusted Riders** - No validation that the rider finishing is the same rider who started. The client application is trusted to send correct rider information. This simplifies the implementation while assuming the mobile app enforces these rules.

4. **No Authentication** - No auth layer; rider IDs are passed directly in requests. In a production system, this would be replaced with JWT tokens or similar authentication mechanisms.

5. **No Routing** - No maps, distance calculations, or routing logic. The system only tracks order state and rider assignments, not physical logistics.

6. **Client-Provided Idempotency** - Clients must provide idempotency keys for duplicate request handling. This is a common pattern in payment and delivery systems where exactly-once semantics are critical.

## Architecture

### State Machine

The order lifecycle is modeled as a finite state machine. This pattern ensures that orders can only transition between valid states, preventing invalid operations like finishing an order that hasn't started.

```
┌─────────┐      ┌─────────────┐      ┌──────────────────┐      ┌───────────┐
│ CREATED │ ──▶  │ IN_PROGRESS │ ──▶  │ AWAITING_HANDOFF │ ──▶  │ DELIVERED │
└─────────┘      └─────────────┘      └──────────────────┘      └───────────┘
                        ▲                      │
                        │                      │
                        └──────────────────────┘
                          (next rider starts)
```

**State Descriptions:**

| State              | Description                                                                                       |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| `CREATED`          | Order has been created but no rider has picked it up yet. This is the initial state.              |
| `IN_PROGRESS`      | A rider is actively working on this order. The `current_rider_id` field identifies who.           |
| `AWAITING_HANDOFF` | The previous rider has finished their leg. The order is waiting for the next rider to pick it up. |
| `DELIVERED`        | The order has been successfully delivered to its final destination. This is a terminal state.     |

**Valid Transitions:**

| From               | To                 | Trigger                        | What Happens                                 |
| ------------------ | ------------------ | ------------------------------ | -------------------------------------------- |
| `CREATED`          | `IN_PROGRESS`      | First rider starts             | Rider assigned, first leg created            |
| `IN_PROGRESS`      | `AWAITING_HANDOFF` | Rider finishes (not final)     | Current leg completed, rider unassigned      |
| `AWAITING_HANDOFF` | `IN_PROGRESS`      | Next rider starts              | New rider assigned, new leg created          |
| `IN_PROGRESS`      | `DELIVERED`        | Rider finishes with final flag | Order completed, current leg marked complete |

**Why a State Machine?**

- **Predictability**: Every possible order state and transition is explicitly defined
- **Validation**: Invalid transitions (e.g., finishing an order in CREATED state) are automatically rejected
- **Debugging**: Easy to trace the order's journey through states
- **Extensibility**: New states (e.g., CANCELLED, RETURNED) can be added with clear transition rules

### Concurrency Strategy

When multiple requests try to modify the same order simultaneously, we need a mechanism to ensure only one request succeeds at a time. This system uses Redis distributed locks.

**Why Redis for Locking?**

1. **Speed**: Redis operations are sub-millisecond, minimizing lock overhead
2. **Distributed**: Works across multiple API server instances
3. **TTL Support**: Built-in key expiration prevents deadlocks if a server crashes while holding a lock
4. **Atomic Operations**: `SET NX EX` command provides atomic lock acquisition

**Lock Implementation Details:**

- **Lock Key Pattern**: `order:lock:{orderId}` - Each order has its own lock
- **TTL**: 30 seconds - If a server crashes while holding a lock, it auto-releases after 30 seconds
- **Retry Strategy**: Failed lock acquisitions retry with exponential backoff (e.g., 50ms, 100ms, 200ms)
- **Max Retries**: 3 attempts before returning a 409 Conflict error

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

Idempotency ensures that making the same request multiple times has the same effect as making it once. This is critical for handling network failures where a client might retry a request that actually succeeded.

**Why Idempotency Matters:**

Consider this scenario without idempotency:

1. Client sends "finish order" request
2. Server processes request, marks order as AWAITING_HANDOFF
3. Network drops the response
4. Client retries the same request
5. Server rejects with "Order not in IN_PROGRESS state" - confusing!

With idempotency:

1. Client sends "finish order" request with `Idempotency-Key: abc123`
2. Server processes request, caches response with key `abc123`
3. Network drops the response
4. Client retries with same `Idempotency-Key: abc123`
5. Server returns cached response - client sees success!

**Implementation Details:**

- **Header**: `Idempotency-Key` (client-generated, typically a UUID)
- **Cache Storage**: Redis with 24-hour TTL
- **Cache Key Pattern**: `idempotency:{key}`
- **Cached Data**: Full response body + status code

## Data Model

### orders

The main table tracking each delivery order's current state.

| Column           | Type            | Description                                       |
| ---------------- | --------------- | ------------------------------------------------- |
| id               | UUID            | Primary key                                       |
| status           | ENUM            | CREATED, IN_PROGRESS, AWAITING_HANDOFF, DELIVERED |
| current_rider_id | UUID (nullable) | ID of the rider currently working on the order    |
| created_at       | TIMESTAMP       | Order creation time                               |
| updated_at       | TIMESTAMP       | Last modification time                            |

### order_legs

Tracks each rider's segment of the delivery. This provides a complete audit trail of who handled the order and when.

| Column      | Type                 | Description                      |
| ----------- | -------------------- | -------------------------------- |
| id          | UUID                 | Primary key                      |
| order_id    | UUID                 | Foreign key to orders            |
| rider_id    | UUID                 | Rider handling this leg          |
| leg_number  | INT                  | Sequence number (1, 2, 3...)     |
| status      | ENUM                 | IN_PROGRESS, COMPLETED           |
| started_at  | TIMESTAMP            | When the rider started this leg  |
| finished_at | TIMESTAMP (nullable) | When the rider finished this leg |

**Why a Separate Legs Table?**

1. **Audit Trail**: Complete history of all riders who handled the order
2. **Analytics**: Calculate average leg duration, rider performance metrics
3. **Debugging**: Trace issues to specific riders and time windows
4. **Normalization**: Keeps the orders table simple; leg details are queried when needed

**Example: Order with 3 Handoffs**

```
Order #123: DELIVERED

Leg 1: Rider A | 10:00 - 10:45 | COMPLETED
Leg 2: Rider B | 11:00 - 12:30 | COMPLETED
Leg 3: Rider C | 13:00 - 14:15 | COMPLETED (final delivery)
```

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

**Business Rules:**

- Order must be in `CREATED` or `AWAITING_HANDOFF` state
- Creates a new leg with incremented `leg_number`
- Sets `current_rider_id` on the order

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

Marks the current leg as complete. Either transitions to `AWAITING_HANDOFF` (for next rider) or `DELIVERED` (final delivery).

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

**Business Rules:**

- Order must be in `IN_PROGRESS` state
- Marks current leg as `COMPLETED` with `finished_at` timestamp
- Sets `current_rider_id` to null
- If `isFinalDelivery: true`, transitions to `DELIVERED` (terminal state)
- If `isFinalDelivery: false`, transitions to `AWAITING_HANDOFF`

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

Retrieves order details including all legs. Useful for tracking and debugging.

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

All errors follow a consistent JSON structure:

**400 Bad Request** - Invalid state transition or validation error:

```json
{
  "statusCode": 400,
  "error": "Bad Request",
  "message": "Invalid state transition: cannot start order in IN_PROGRESS state"
}
```

**404 Not Found** - Order doesn't exist:

```json
{
  "statusCode": 404,
  "error": "Not Found",
  "message": "Order not found"
}
```

**409 Conflict** - Could not acquire lock (resource contention):

```json
{
  "statusCode": 409,
  "error": "Conflict",
  "message": "Unable to acquire lock. Resource is busy."
}
```

## Scalability Considerations

This architecture is designed to scale horizontally:

1. **Stateless API Servers**: No session state; any instance can handle any request
2. **Redis for Coordination**: Distributed locks work across all API instances
3. **Database Indexing**: Indexes on `order_id`, `status`, and `rider_id` for fast queries

