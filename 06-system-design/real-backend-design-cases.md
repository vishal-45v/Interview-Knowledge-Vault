# Real Backend Design Cases

---

## Design an E-Commerce Flash Sale System

**Challenge:** 1M users try to buy the same item at exactly 9:00 AM. Only 1,000 items available.

**Anti-patterns:**
- Direct DB write per request → DB overwhelmed
- Optimistic locking → Too many conflicts, thundering herd on retries

**Solution:**
```
Pre-sale preparation:
  1. Load inventory (1,000) into Redis:
     SET flash:inventory:productId 1000

During sale:
  1. Request arrives → API Gateway → Rate limiter (per-user)
  2. Atomic decrement in Redis:
     DECR flash:inventory:productId → if result >= 0: reserved!
  3. Place reservation in queue (Kafka)
  4. Return: "Order reserved, processing..."

Async processing (consumers):
  5. Consumer picks up reservation from queue
  6. Create order in DB (with DB-level constraint)
  7. Process payment (async)
  8. Notify user via WebSocket/email

If payment fails:
  9. Increment inventory back in Redis
  10. Notify user, allow re-purchase
```

---

## Design a Real-Time Location Tracking System (Uber)

**Scale:** 5M active drivers, 20M riders, location updates every 5 seconds

```
Driver app → Location Service
  WRITE: driver_id → {lat, lng, timestamp, availability}
  Redis GEO: GEOADD active-drivers {lng} {lat} {driver_id}

Rider requests ride:
  1. GEORADIUS active-drivers {rider_lng} {rider_lat} 5km ASC → nearest 10 drivers
  2. For each candidate: calculate ETA (via maps API, cached)
  3. Match rider to nearest available driver
  4. WebSocket push to both rider and driver

Location updates pipeline:
  Driver → WebSocket server → Kafka → Location consumer → Redis update
                                                        → Persistence to Cassandra (for history)
```

---

## Design a Microservice Saga Pattern

**Problem:** Place order requires inventory reservation, payment, and shipping. Must be atomic across services.

```
Choreography-based Saga:

  OrderService:
    1. Create Order (PENDING)
    2. Publish: OrderCreated event

  InventoryService:
    3. Reserve inventory
    4a. Success: Publish InventoryReserved
    4b. Fail: Publish InventoryFailed

  PaymentService (listens for InventoryReserved):
    5. Process payment
    6a. Success: Publish PaymentProcessed
    6b. Fail: Publish PaymentFailed → triggers InventoryReleased

  OrderService (listens for PaymentProcessed):
    7. Update order status to CONFIRMED

  Compensating transactions (rollback saga):
    InventoryFailed → OrderService marks order FAILED
    PaymentFailed → InventoryService releases reservation
                  → OrderService marks order FAILED
```
