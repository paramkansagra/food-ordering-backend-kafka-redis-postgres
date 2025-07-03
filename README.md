# High level Interaction Flow

| Step | Producer → Consumer | Transport | Canonical | Payload |
| --- | --- | --- | --- | --- |
| 1 | Order Service → Order Processor | HTTP POST /orders | order_placed JSON |
| 2 | Order Processor → Restaurant Service | Kafka topic order_placed | same JSON |
| 3 | Restaurant Service → Order Processor |	Kafka topic order_accepted |	acceptance JSON |
| 4 | Order Processor → Rider Service |	Kafka topic rider_request | 	rider-request JSON |
| 5	| Rider Service → Order Processor |	Kafka topic rider_assigned |	assignment JSON |
| 6 | Order Processor → Notification Service |	Kafka topic order_events | any status JSON |
| 7	| Order Query Service ← Redis / Postgres |	direct read |	materialized view JSON |

# API Gateway and sample responses

## 1.  Order Service

<details>

### 1.1 Create Order – POST /v1/orders

HTTP Request
```text
POST /v1/orders
Content-Type: application/json
Accept: application/json
Idempotency-Key: 0fb1e1b9-e7dc-4aec-b2fa-3794b341fe7a
Authorization: Bearer <JWT>
```

Body
```json
{
  "customerId": "CUST-123",
  "restaurantId": "REST-456",
  "items": [
    { "itemId": "PIZZA-001", "quantity": 1 },
    { "itemId": "DRINK-003", "quantity": 2 }
  ],
  "deliveryAddress": {
    "street": "2510 Lagoon Way",
    "city": "Anytown",
    "state": "CA",
    "zipCode": "12345"
  },
  "specialInstructions": "Extra cheese"
}
```

Successful HTTP Response
```text
HTTP/1.1 201 Created
Location: /v1/orders/ORD-1001
Content-Type: application/json
```

Body
```json
{
  "orderId": "ORD-1001",
  "status": "PLACED",
  "createdAt": "2025-07-02T17:00:01Z"
}
```

Behaviour
1. Synchronous validation & 201 response.
2. Asynchronously publishes `order_placed` event to Kafka.
3. No DB writes here—Order Processor owns persistence.

</details>

## 2. Restaurant Service

<details>

### 2.1 Accept/Reject Order – POST /v1/restaurants/{restaurantId}/orders/{orderId}

Request body (partial update):
```json
{
  "action": "ACCEPT",
  "estimatedReadyTime": "2025-07-02T17:20:00Z"
}
```

Responses
| Status | Body (excerpt) |
| --- | --- |
| 200 OK |	```{ "status": "ACCEPTED", "acceptedAt": "...", "estimatedReadyTime": "…" }``` |
| 409 Conflict |	```{ "errors": [{ "status": "409", "code": "ALREADY_ACCEPTED" }] }``` |

__On success the service emits an `order_accepted` and `rider_request` event to Kafka__ 

</details>

## 3. Rider Service

<details>

### 3.1 Assign Rider – Kafka driven

Consumer of `rider_request`. After internal matching it produces:

```json
{
  "eventType": "rider_assigned",
  "orderId": "ORD-1001",
  "riderId": "RIDER-789",
  "assignedAt": "2025-07-02T17:10:00Z",
  "pickupLocation": {
    "restaurantId": "REST-456",
    "address": "2510 Lagoon Way, Anytown CA"
  }
}
```

### 3.2 Rider Tracking – PATCH /v1/riders/{riderId}/orders/{orderId}

Body when the rider picks up the order:

```json
{ "status": "PICKED_UP", "timestamp": "2025-07-02T17:20:10Z" }
```

Returns __204__ No Content on success.

</details>

## 4. Order Processor Service

_Responsibilities_

1. Consume every domain event (`order_placed`, `order_accepted`, `rider_assigned`, …).
2. Run business rules, update aggregates in __Postgres__ (source of truth) and __Redis__ (read model).
3. Re-emit a normalized event to topic `order_events` for downstream services.

## 5. Notification Service

<details>

Consumes `order_events` and calls an external provider (SMS, push). Canonical outgoing webhook:

```json
{
  "to": "+1-555-0100",
  "template": "ORDER_ACCEPTED",
  "variables": {
    "orderId": "ORD-1001",
    "eta": "17:35"
  }
}
```

If the provider responds __202__ Accepted, a `notification_sent` event is logged.

</details>

## 6. Order Query Service

<details>

`GET /v1/orders/{orderId}`

Response (from Redis, falls back to Postgres):

```json
{
  "orderId": "ORD-1001",
  "customerId": "CUST-123",
  "restaurantId": "REST-456",
  "status": "RIDER_ASSIGNED",
  "timeline": [
    { "status": "PLACED", "at": "17:00:01Z" },
    { "status": "ACCEPTED", "at": "17:05:00Z" },
    { "status": "RIDER_ASSIGNED", "at": "17:10:00Z" }
  ],
  "eta": "17:35:00Z"
}
```

Error Shape (global)

```json
{
  "errors": [
    {
      "status": "400",
      "code": "INVALID_ADDRESS",
      "detail": "Zip code is missing or malformed"
    }
  ]
}
```

</details>

# Redis Key Design
| Key pattern | Value |	TTL strategy |
| --- | --- | --- |
| order:{orderId} |	Full JSON status snapshot |	None (evicted by LRU) |
| customer:{custId}:last_orders |	[orderId…] |	24 h |

# Postgres Schema

```sql
CREATE TABLE orders (
  order_id      VARCHAR PRIMARY KEY,
  customer_id   VARCHAR,
  restaurant_id VARCHAR,
  status        VARCHAR,
  total_amount  NUMERIC,
  created_at    TIMESTAMPTZ,
  updated_at    TIMESTAMPTZ
);
```

More will come as we progress ahead 😃

# TODO
- [x] Create Docker files for kafka, redis and postgres
- [ ] Create table structures for postgres, how topics would look like for kafka and structure for redis key value pairs
- [ ] `Order Service` API created
- [ ] `Order Processor Service` API Created
- [ ] `Order Query Service` API created
- [ ] `Restaurant Service` API created
- [ ] `Rider Service` API created
- [ ] `Notification Service` API Created
- [ ] Examples for each service