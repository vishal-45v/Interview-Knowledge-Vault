# Chapter 06 — Transactions & Consistency: Scenario Questions

---

## Scenario 1: Bank Transfer

**Scenario**: Implement a money transfer between two bank accounts. Alice has $1,000 and wants to send $200 to Bob. The transfer must be atomic — either both accounts are updated or neither is.

```javascript
async function transfer(fromAccountId, toAccountId, amount, currency) {
  const session = client.startSession()

  try {
    const result = await session.withTransaction(async () => {
      // Step 1: Debit sender (conditional on sufficient balance)
      const sender = await db.accounts.findOneAndUpdate(
        {
          _id: ObjectId(fromAccountId),
          currency: currency,
          balance: { $gte: NumberDecimal(amount.toString()) },
          status: "active"
        },
        {
          $inc: { balance: NumberDecimal((-amount).toString()) },
          $set: { lastTransactionAt: new Date() }
        },
        {
          session,
          returnDocument: "after",
          writeConcern: { w: "majority" }
        }
      )

      if (!sender) {
        throw new Error("Insufficient funds or account not active")
      }

      // Step 2: Credit receiver
      const receiver = await db.accounts.findOneAndUpdate(
        { _id: ObjectId(toAccountId), status: "active" },
        {
          $inc: { balance: NumberDecimal(amount.toString()) },
          $set: { lastTransactionAt: new Date() }
        },
        { session, returnDocument: "after" }
      )

      if (!receiver) {
        throw new Error("Destination account not found or not active")
      }

      // Step 3: Create immutable audit record
      await db.transfers.insertOne({
        fromAccountId: ObjectId(fromAccountId),
        toAccountId: ObjectId(toAccountId),
        amount: NumberDecimal(amount.toString()),
        currency: currency,
        fromBalance: sender.balance,
        toBalance: receiver.balance,
        status: "completed",
        createdAt: new Date()
      }, { session })

      return {
        success: true,
        transferId: result?.insertedId,
        newBalance: sender.balance
      }
    }, {
      readConcern: { level: "snapshot" },
      writeConcern: { w: "majority", j: true }
    })

    return result
  } finally {
    await session.endSession()
  }
}
```

---

## Scenario 2: E-Commerce Order Placement

**Scenario**: A customer places an order for 3 items. You must:
1. Reserve inventory for each item (check and decrement stock)
2. Create the order document
3. Update the customer's order count
4. Deduct loyalty points if used

All four operations must succeed or none.

```javascript
async function placeOrder(customerId, cartItems, loyaltyPointsUsed, idempotencyKey) {
  // Check for duplicate request
  const existing = await db.orders.findOne({ idempotencyKey })
  if (existing) return { orderId: existing._id, alreadyProcessed: true }

  const session = client.startSession()

  try {
    const order = await session.withTransaction(async () => {
      // Step 1: Reserve inventory for all items atomically
      const reservations = []
      for (const item of cartItems) {
        const inventory = await db.inventory.findOneAndUpdate(
          {
            productId: ObjectId(item.productId),
            available: { $gte: item.quantity }
          },
          { $inc: { available: -item.quantity, reserved: item.quantity } },
          { session, returnDocument: "after" }
        )

        if (!inventory) {
          throw new Error(`Insufficient stock for product ${item.productId}`)
          // transaction aborts → all previous inventory decrements are rolled back
        }
        reservations.push(inventory)
      }

      // Step 2: Deduct loyalty points if applicable
      if (loyaltyPointsUsed > 0) {
        const customer = await db.customers.findOneAndUpdate(
          {
            _id: ObjectId(customerId),
            loyaltyPoints: { $gte: loyaltyPointsUsed }
          },
          { $inc: { loyaltyPoints: -loyaltyPointsUsed, orderCount: 1 } },
          { session }
        )
        if (!customer) throw new Error("Insufficient loyalty points")
      } else {
        await db.customers.updateOne(
          { _id: ObjectId(customerId) },
          { $inc: { orderCount: 1 } },
          { session }
        )
      }

      // Step 3: Create order
      const orderTotal = cartItems.reduce((sum, item) => sum + item.price * item.quantity, 0)
      const newOrder = await db.orders.insertOne({
        customerId: ObjectId(customerId),
        items: cartItems,
        total: NumberDecimal(orderTotal.toFixed(2)),
        loyaltyPointsUsed,
        status: "placed",
        idempotencyKey,
        createdAt: new Date()
      }, { session })

      return { _id: newOrder.insertedId, total: orderTotal }
    })

    return { orderId: order._id, success: true }
  } finally {
    session.endSession()
  }
}
```

---

## Scenario 3: Distributed Event Ticket Sale

**Scenario**: A concert ticket sale system. 1,000 tickets available. The system must:
- Prevent overselling (never sell more than 1,000 tickets)
- Handle 10,000 concurrent purchase requests
- Each purchase is atomic

```javascript
// Non-transactional approach (WRONG — race condition)
async function buyTicketWrong(eventId, userId) {
  const event = await db.events.findOne({ _id: ObjectId(eventId) })
  if (event.ticketsAvailable <= 0) throw new Error("Sold out")
  // RACE CONDITION: another process reads the same value here
  await db.events.updateOne(
    { _id: ObjectId(eventId) },
    { $inc: { ticketsAvailable: -1 } }
  )
  // Two processes both pass the check and both decrement → negative tickets!
}

// Correct approach: atomic findOneAndUpdate (no transaction needed!)
async function buyTicket(eventId, userId, quantity = 1) {
  const session = client.startSession()

  try {
    return await session.withTransaction(async () => {
      // Atomic: check AND decrement in one operation
      const event = await db.events.findOneAndUpdate(
        {
          _id: ObjectId(eventId),
          ticketsAvailable: { $gte: quantity },
          saleStatus: "active"
        },
        { $inc: { ticketsAvailable: -quantity, ticketsSold: quantity } },
        { session, returnDocument: "after" }
      )

      if (!event) throw new Error("No tickets available or sale not active")

      // Create ticket documents
      const tickets = []
      for (let i = 0; i < quantity; i++) {
        tickets.push({
          eventId: ObjectId(eventId),
          userId: ObjectId(userId),
          ticketNumber: `TKT-${eventId}-${event.ticketsSold - quantity + i + 1}`,
          seatSection: "GA",
          purchasedAt: new Date(),
          status: "valid"
        })
      }

      const result = await db.tickets.insertMany(tickets, { session })

      return {
        success: true,
        ticketIds: Object.values(result.insertedIds),
        ticketsRemaining: event.ticketsAvailable
      }
    })
  } finally {
    session.endSession()
  }
}

// Index to prevent duplicate tickets per user per event (optional business rule)
db.tickets.createIndex({ eventId: 1, userId: 1 })
```

---

## Scenario 4: Flash Sale with Limited Quantity

**Scenario**: Flash sale: 100 units of a product at 50% discount. The window is 1 hour. Thousands of users try to purchase simultaneously. How do you handle this without transactions becoming a bottleneck?

```javascript
// Challenge: high contention on a single hot document (the flash sale product)
// Transactions would create a queue of waiting operations → high latency

// Approach 1: Atomic findOneAndUpdate (no transaction needed for single-doc atomicity)
async function claimFlashSaleItem(saleId, userId, quantity) {
  const result = await db.flashSales.findOneAndUpdate(
    {
      _id: ObjectId(saleId),
      remainingQuantity: { $gte: quantity },
      status: "active",
      endsAt: { $gt: new Date() }
    },
    {
      $inc: { remainingQuantity: -quantity, claimedCount: quantity },
      $push: {
        claims: {
          $each: [{ userId: ObjectId(userId), quantity, claimedAt: new Date() }],
          $slice: -1000   // keep only last 1000 claims in array (bounded!)
        }
      }
    },
    { returnDocument: "after", writeConcern: { w: 1 } }  // w:1 for speed (sale window is short)
  )

  if (!result) return { success: false, reason: "sold_out_or_expired" }

  // Create the order asynchronously (non-blocking)
  setImmediate(() => {
    db.orders.insertOne({
      userId: ObjectId(userId),
      saleId: ObjectId(saleId),
      quantity,
      status: "pending_payment",
      createdAt: new Date()
    })
  })

  return { success: true, remaining: result.remainingQuantity }
}

// Approach 2: Queue-based (for very high contention)
// Write claim requests to a queue collection, process serially
await db.saleQueue.insertOne({
  saleId: ObjectId(saleId),
  userId: ObjectId(userId),
  quantity,
  requestedAt: new Date(),
  status: "pending"
})
// Background worker processes queue one at a time → no contention
// Trade-off: eventual processing (not instant confirmation)
```

---

## Scenario 5: Inventory Transfer Between Warehouses

**Scenario**: Transfer 50 units of Product A from Warehouse 1 to Warehouse 2. The transfer must be atomic — you can't have 50 units "disappear" from Warehouse 1 while Warehouse 2 hasn't received them yet.

```javascript
async function transferInventory(productId, fromWarehouse, toWarehouse, quantity, transferId) {
  const session = client.startSession()

  try {
    await session.withTransaction(async () => {
      // Idempotency check
      const existingTransfer = await db.inventoryTransfers.findOne(
        { transferId },
        { session }
      )
      if (existingTransfer && existingTransfer.status === "completed") {
        return existingTransfer
      }

      // Deduct from source
      const source = await db.inventory.findOneAndUpdate(
        {
          productId: ObjectId(productId),
          warehouseId: fromWarehouse,
          available: { $gte: quantity }
        },
        {
          $inc: { available: -quantity, quantity: -quantity },
          $set: { lastUpdated: new Date() }
        },
        { session, returnDocument: "after" }
      )

      if (!source) {
        throw new Error(`Insufficient stock in ${fromWarehouse}`)
      }

      // Add to destination
      await db.inventory.findOneAndUpdate(
        {
          productId: ObjectId(productId),
          warehouseId: toWarehouse
        },
        {
          $inc: { available: quantity, quantity: quantity },
          $set: { lastUpdated: new Date() }
        },
        { session, upsert: true }
      )

      // Record transfer (idempotency key)
      await db.inventoryTransfers.updateOne(
        { transferId },
        {
          $set: {
            productId: ObjectId(productId),
            fromWarehouse,
            toWarehouse,
            quantity,
            status: "completed",
            completedAt: new Date()
          }
        },
        { session, upsert: true }
      )
    })
  } finally {
    session.endSession()
  }
}
```

---

## Scenario 6: User Account Deletion (GDPR Right to Erasure)

**Scenario**: A GDPR deletion request. When a user is deleted, you must atomically:
1. Soft-delete the user account
2. Anonymize their posts (replace name with "Deleted User")
3. Delete their private messages
4. Create an audit record of the deletion

All or nothing — partial deletion is a compliance risk.

```javascript
async function gdprDeleteUser(userId, requesterId, reason) {
  const session = client.startSession()

  try {
    await session.withTransaction(async () => {
      const userObjId = ObjectId(userId)

      // Step 1: Soft delete user
      const user = await db.users.findOneAndUpdate(
        { _id: userObjId, isDeleted: { $ne: true } },
        {
          $set: {
            isDeleted: true,
            deletedAt: new Date(),
            // Overwrite PII fields
            email: `deleted-${userId}@redacted.invalid`,
            firstName: "Deleted",
            lastName: "User",
            phone: null,
            dateOfBirth: null
          }
        },
        { session }
      )

      if (!user) throw new Error("User not found or already deleted")

      // Step 2: Anonymize public posts (keep content, remove attribution)
      await db.posts.updateMany(
        { authorId: userObjId },
        {
          $set: {
            "author.name": "Deleted User",
            "author.avatarUrl": null,
            "author._id": null
          },
          $unset: { authorId: "" }
        },
        { session }
      )

      // Step 3: Delete private messages (truly delete — not just anonymize)
      await db.messages.deleteMany(
        { $or: [{ senderId: userObjId }, { recipientId: userObjId }] },
        { session }
      )

      // Step 4: Create immutable compliance audit record
      await db.gdprAuditLog.insertOne({
        action: "user_deletion",
        subjectUserId: userObjId,
        requestedBy: ObjectId(requesterId),
        reason: reason,
        dataDeleted: ["profile", "messages"],
        dataAnonymized: ["posts"],
        executedAt: new Date(),
        retentionUntil: new Date(Date.now() + 7 * 365 * 24 * 60 * 60 * 1000)  // 7 years
      }, { session })
    }, {
      writeConcern: { w: "majority", j: true }   // must be durable for compliance
    })
  } finally {
    session.endSession()
  }
}
```

---

## Scenario 7: Optimistic Locking Without Transactions

**Scenario**: Multiple editors can edit the same document simultaneously. You want to prevent "last write wins" — if two editors start editing, the second to save should be told there's a conflict. Implement this WITHOUT transactions using optimistic locking.

```javascript
// Document schema with version field
{
  _id: ObjectId("..."),
  title: "MongoDB Guide",
  body: "...",
  version: 7,                   // increment on every save
  lastEditedBy: ObjectId("..."),
  updatedAt: ISODate("...")
}

// Client fetches document (captures current version)
const doc = await db.articles.findOne({ _id: ObjectId(articleId) })
const originalVersion = doc.version

// User edits locally for 5 minutes...

// Client saves — only succeeds if version hasn't changed
async function saveArticle(articleId, editorId, content, expectedVersion) {
  const result = await db.articles.findOneAndUpdate(
    {
      _id: ObjectId(articleId),
      version: expectedVersion          // OPTIMISTIC CHECK: fail if version changed
    },
    {
      $set: {
        title: content.title,
        body: content.body,
        lastEditedBy: ObjectId(editorId),
        updatedAt: new Date()
      },
      $inc: { version: 1 }              // increment version on success
    },
    { returnDocument: "after" }
  )

  if (!result) {
    // Version mismatch — someone else saved in the meantime
    const current = await db.articles.findOne({ _id: ObjectId(articleId) })
    throw new ConflictError(`Document was modified by another editor (current version: ${current.version})`)
  }

  return result
}

// Editor A (version 7) → saves → success (version becomes 8)
// Editor B (version 7) → saves → FAILS (version is now 8, not 7)
// Editor B receives conflict error → must re-fetch and merge changes

// Client-side conflict resolution
async function saveWithConflictResolution(articleId, editorId, content, expectedVersion) {
  try {
    return await saveArticle(articleId, editorId, content, expectedVersion)
  } catch (err) {
    if (err instanceof ConflictError) {
      // Fetch current version and present diff to user
      const current = await db.articles.findOne({ _id: ObjectId(articleId) })
      return { conflict: true, currentContent: current, yourContent: content }
    }
    throw err
  }
}
```

---

## Scenario 8: Saga Pattern for Long-Running Business Processes

**Scenario**: An order fulfillment saga: place order → reserve inventory → charge payment → create shipment. Each step can fail. If payment fails after inventory is reserved, you must release the inventory.

```javascript
// Saga state machine — one document tracks the entire saga
{
  _id: ObjectId("..."),
  orderId: ObjectId("..."),
  type: "order_fulfillment",
  status: "in_progress",         // in_progress | completed | failed | compensating
  currentStep: "reserve_inventory",

  steps: [
    { name: "reserve_inventory", status: "completed", completedAt: ISODate("..."), data: { reservationId: "..." } },
    { name: "charge_payment",    status: "failed",    failedAt: ISODate("..."),    error: "Card declined" },
    { name: "create_shipment",   status: "pending" }
  ],

  compensations: [
    { name: "release_inventory", status: "pending" }
  ],

  createdAt: ISODate("..."),
  updatedAt: ISODate("...")
}

class OrderFulfillmentSaga {
  async execute(orderId) {
    const saga = await db.sagas.insertOne({
      orderId: ObjectId(orderId),
      type: "order_fulfillment",
      status: "in_progress",
      steps: [
        { name: "reserve_inventory", status: "pending" },
        { name: "charge_payment",    status: "pending" },
        { name: "create_shipment",   status: "pending" }
      ],
      compensations: [],
      createdAt: new Date()
    })

    try {
      // Step 1: Reserve inventory
      const reservation = await this.reserveInventory(orderId)
      await this.updateStep(saga.insertedId, "reserve_inventory", "completed", { reservationId: reservation.id })

      // Step 2: Charge payment
      let payment
      try {
        payment = await this.chargePayment(orderId)
        await this.updateStep(saga.insertedId, "charge_payment", "completed", { paymentId: payment.id })
      } catch (paymentErr) {
        // Payment failed → compensate inventory reservation
        await this.updateStep(saga.insertedId, "charge_payment", "failed", { error: paymentErr.message })
        await this.compensate(saga.insertedId, orderId, reservation.id)
        throw paymentErr
      }

      // Step 3: Create shipment
      await this.createShipment(orderId, payment.id)
      await this.updateStep(saga.insertedId, "create_shipment", "completed", {})

      await db.sagas.updateOne({ _id: saga.insertedId }, { $set: { status: "completed" } })

    } catch (err) {
      await db.sagas.updateOne({ _id: saga.insertedId }, {
        $set: { status: "failed", error: err.message }
      })
      throw err
    }
  }

  async compensate(sagaId, orderId, reservationId) {
    await db.sagas.updateOne({ _id: sagaId }, { $set: { status: "compensating" } })
    await this.releaseInventoryReservation(reservationId)
    await db.sagas.updateOne(
      { _id: sagaId },
      { $push: { compensations: { name: "release_inventory", status: "completed", at: new Date() } } }
    )
    await db.sagas.updateOne({ _id: sagaId }, { $set: { status: "failed" } })
  }

  async updateStep(sagaId, stepName, status, data) {
    await db.sagas.updateOne(
      { _id: sagaId, "steps.name": stepName },
      { $set: {
          "steps.$.status": status,
          "steps.$.data": data,
          [`steps.$.${status}At`]: new Date()
      }}
    )
  }
}
```

---

## Scenario 9: Read Your Own Writes Across Multiple Nodes

**Scenario**: A user updates their profile photo. The update is written to the primary. The next request is load-balanced to a secondary. The user sees their old photo. How do you fix this?

```javascript
// Problem: write to primary, read from secondary, secondary hasn't replicated yet

// SOLUTION 1: Causal consistency session
const session = client.startSession({ causalConsistency: true })

// Write to primary with majority concern
await db.users.updateOne(
  { _id: ObjectId(userId) },
  { $set: { avatarUrl: newAvatarUrl } },
  {
    session,
    writeConcern: { w: "majority" }
  }
)

// Subsequent read — even on secondary — will see the write above
const user = await db.users.findOne(
  { _id: ObjectId(userId) },
  {
    session,
    readPreference: "secondary",
    readConcern: { level: "majority" }
  }
)
// user.avatarUrl === newAvatarUrl ✓ (causal consistency guarantees this)

// SOLUTION 2: After critical writes, route to primary for a short window
// In the HTTP response, set a cookie/header: "readFromPrimary: true" for 5 seconds
// Application middleware checks the cookie and routes reads to primary

// SOLUTION 3: Read concern "linearizable" for the specific post-write read
const user = await db.users.findOne(
  { _id: ObjectId(userId) },
  { readConcern: { level: "linearizable" } }   // always returns latest committed value
)
// Slowest option — confirms with majority before returning

// SOLUTION 4: Don't use secondary reads for user-facing, session-specific data
// Use secondary reads only for analytics, reporting, background jobs
// Keep user-facing reads on primary (or use causal sessions)
```

---

## Scenario 10: Change Stream with Exactly-Once Processing

**Scenario**: You have a change stream on the orders collection. When an order is placed, you send a confirmation email. If the service crashes mid-processing, you must not send duplicate emails. Implement exactly-once processing.

```javascript
async function processOrdersExactlyOnce() {
  // Load the last processed resume token from a persistent store
  const state = await db.processorState.findOne({ _id: "order_email_processor" })
  const resumeToken = state?.resumeToken || null

  const changeStream = db.orders.watch(
    [{ $match: { operationType: "insert" } }],
    {
      resumeAfter: resumeToken,
      fullDocument: "updateLookup"
    }
  )

  for await (const change of changeStream) {
    const order = change.fullDocument
    const token = change._id

    // Check if we already processed this event (idempotency check)
    const alreadyProcessed = await db.processedEvents.findOne({
      _id: token._data    // resume token data is unique per event
    })

    if (alreadyProcessed) {
      // Already sent this email — skip
      await db.processorState.updateOne(
        { _id: "order_email_processor" },
        { $set: { resumeToken: token } },
        { upsert: true }
      )
      continue
    }

    // Use a session to atomically: process event AND mark as processed
    const session = client.startSession()
    try {
      await session.withTransaction(async () => {
        // Mark event as processed (prevent duplicate)
        await db.processedEvents.insertOne(
          {
            _id: token._data,
            orderId: order._id,
            processedAt: new Date(),
            expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)  // 7-day TTL
          },
          { session }
        )

        // Update processor state
        await db.processorState.updateOne(
          { _id: "order_email_processor" },
          { $set: { resumeToken: token, lastProcessedAt: new Date() } },
          { session, upsert: true }
        )
      })

      // Send email outside the transaction (side effect)
      // If this fails → retry entire event (email provider should handle duplicates)
      await sendOrderConfirmationEmail(order)

    } catch (err) {
      if (err.code === 11000) {
        // Duplicate key on processedEvents → already processed, safe to skip
        console.log(`Event ${token._data} already processed, skipping`)
      } else {
        throw err
      }
    } finally {
      session.endSession()
    }
  }
}

// TTL index on processedEvents for cleanup
db.processedEvents.createIndex({ expiresAt: 1 }, { expireAfterSeconds: 0 })
db.processedEvents.createIndex({ _id: 1 }, { unique: true })
```
