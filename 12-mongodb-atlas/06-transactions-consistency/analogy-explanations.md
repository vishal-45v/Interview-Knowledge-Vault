# Chapter 06 — Transactions & Consistency: Analogy Explanations

---

## ACID Properties — The Vending Machine

A vending machine is a perfect analogy for ACID transactions:

**Atomicity**: When you put in $1.50 and press B3, either you get the chips AND $1.50 is deducted, OR nothing happens. The machine never takes your money without giving you chips (or gives you chips for free). All or nothing.

**Consistency**: The machine always ends in a valid state. It can't give you 3 chips when 1 was pressed. The inventory count can never be negative. Internal rules are always maintained.

**Isolation**: If two people press buttons simultaneously on different machines, each transaction is independent. One person getting chips doesn't interfere with another person's selection (even if both machines share a supplier delivery truck).

**Durability**: Once the machine dispenses chips and the transaction is complete, the event is recorded in the machine's log permanently. Unplugging the machine doesn't un-dispense the chips or undo the inventory deduction.

```
MongoDB single-doc write:
You: update one product's price       → atomic (all field changes succeed or none)
Consistency: price can't be negative  → JSON Schema validator enforces
Isolation: other writes can happen    → document-level locking
Durability: j:true + w:majority       → survives power outage on majority of nodes
```

---

## Multi-Document Transactions — The Bank Teller

Transferring money between two bank accounts requires a teller (transaction coordinator):

1. Teller **starts the transaction**: opens both account ledgers simultaneously
2. Teller **checks** Alice has enough ($1,000 balance, request for $200 transfer)
3. Teller **debits** Alice: crosses out $1,000, writes $800
4. Teller **credits** Bob: crosses out $500, writes $700
5. Teller **commits**: stamps both ledgers, closes them

If the bank loses power between steps 3 and 4 (after debiting Alice but before crediting Bob):
- **Without transactions**: Alice lost $200, Bob never got it → money vanished
- **With transactions**: The teller's ledger book records both changes as "pending." On recovery, the bank sees the incomplete transaction and **rolls it back** — restoring Alice's $1,000 as if nothing happened.

The key insight: you never see Alice's account with $800 unless Bob's account simultaneously shows $700. They change together or not at all.

---

## Snapshot Isolation — The Photograph

Imagine two photographers both photograph a living room simultaneously. After the photos are taken:
- Photographer A starts rearranging the furniture
- Photographer B looks at their own photo — they see the ORIGINAL arrangement

Photographer B's photo (their snapshot) is frozen at the moment it was taken. No matter how much Photographer A rearranges the room, B sees the original state in their photo.

This is exactly how MongoDB snapshot isolation works:
- Transaction T1 takes a "photo" of the database when it starts
- While T1 is running, T2 commits changes to several documents
- T1 still sees its original "photo" — T2's changes are invisible to T1

The important consequence: if T1 also tries to change the same documents T2 already changed, there's a conflict (two photographers can't occupy the same spot). One "photographer" retries with a fresh photo.

---

## Write Concerns — The Newspaper Reporter

A reporter writes an important article and needs to submit it. Three submission options:

**w:0** (fire and forget): Send the email and immediately move on without waiting for confirmation the editor received it. If the email server crashes, the article is lost — but the reporter already moved on to the next story.

**w:1** (primary acknowledged): The reporter waits for the editor-in-chief to say "received." But if the chief editor is hit by a bus before filing it (primary crash before replication), the article may be lost even though the reporter thinks it was received.

**w:majority** (majority acknowledged): The reporter waits for confirmation from the chief editor AND at least one deputy editor. Even if the chief editor is incapacitated, the deputy has the article and can publish it. The article will not be "un-received."

```
w:0        = send email, don't wait        → fastest, risky for important content
w:1        = wait for chief editor's ack   → fast, small rollback risk
w:majority = wait for chief + deputy       → slower, article is safe forever
```

---

## Read Concerns — The Library Book Catalog

**local** (default): Check the nearest catalog terminal. It shows books returned in the last few minutes, but might not show a book returned 30 seconds ago (hasn't synced yet). Fast, possibly slightly stale.

**majority**: Check the main catalog that's been confirmed by all branches. Every book in this catalog has been confirmed delivered by multiple branches. Slightly behind "local" but you know the data is solid — no branch will "un-return" a book.

**linearizable**: Call the head librarian and ask "Is Book X truly available right now?" The head librarian calls all branch managers, confirms consensus, then tells you. Slowest but absolute certainty.

**snapshot** (transactions): Take a photograph of the entire catalog at 2:00 PM. All your searches within this transaction use that 2:00 PM photo — even if books are checked in/out between 2:00 PM and 2:05 PM while you're browsing.

```
local           = nearest terminal    = fastest, might be 1-2 sec stale
majority        = confirmed catalog   = slightly slower, never rolls back
linearizable    = head librarian call = slowest, absolute certainty
snapshot (txn)  = 2:00 PM photograph = all queries consistent with each other
```

---

## Causal Consistency — The WhatsApp Message Thread

You send a message in a WhatsApp group: "Meeting at 3pm." Your friend Bob immediately replies: "Got it!"

Without causal consistency: Bob might see his own reply BEFORE he sees your original message (if he reads from a server that received his reply but not yet your message). His reply appears to be a response to nothing.

With causal consistency: MongoDB tracks the "happened-before" relationship. Bob's client says "I wrote a reply at time T. When I read the group, don't show me messages unless you've caught up to at least time T." This ensures the sequence makes sense — original message → reply → your read shows both in order.

```
Without causal consistency: "Got it!" appears before "Meeting at 3pm"
With causal consistency:    "Meeting at 3pm" → "Got it!" (correct order, always)
```

In MongoDB: after your write (profile update) at logical time T, your session's reads always wait for data up to time T before returning results — even from a secondary node.

---

## Retryable Writes — The Duplicate-Safe Mail Order

You send an order form by mail. The mail carrier loses it on their truck and you receive a "delivery failed" notice. You send the form again.

**Without retryable writes**: The post office processes whatever arrives. If the first form shows up later, you get the same order twice (duplicates). If it never shows up, you have nothing (lost write).

**With retryable writes**: The order form has a unique serial number. The post office checks: "Have I already processed serial number 98765?" If YES → don't process again, just confirm receipt. If NO → process it and record it.

If the mail carrier's truck crashes and you retry, the post office's serial number check prevents duplicate orders. The outcome is always exactly one processed order.

```javascript
// MongoDB's implementation:
// Each write carries: lsid (session ID) + txnNumber (sequence number)
// Server checks: "have I already applied txnNumber 42 for session X?"
// YES → return the original result (idempotent)
// NO  → apply the write
// Network retry = safe, no duplicates
```

---

## The Saga Pattern — The Travel Booking

Booking a trip requires three separate services:
1. Book a flight (airline reservation system)
2. Reserve a hotel (hotel booking system)
3. Rent a car (car rental system)

These are three different companies' systems — you can't run a transaction across all three. But they must be coordinated: if the hotel is fully booked, you don't want the flight reservation or car rental to still be active.

The Saga pattern is like a coordinated travel agency:
- Step 1: Book flight (success)
- Step 2: Reserve hotel → FAILS (no rooms)
- **Compensate**: Cancel the flight reservation (compensating transaction)
- Report failure to the customer

Each step is its own independent transaction. The saga keeps a state log ("flight booked, hotel failed, flight cancelled"). If the cancellation also fails, the log allows a recovery process to retry it.

```
Saga = sequence of local transactions + compensating transactions on failure
vs.
Distributed transaction = single global transaction across all systems
```

Sagas are eventual consistency — there's a moment when the flight is booked but hotel has failed but the flight hasn't been cancelled yet. That's acceptable; what's not acceptable is leaving the flight booked forever after hotel failure.

---

## MVCC (Multi-Version Concurrency Control) — The Git Repository

Git is a perfect analogy for MVCC:

When you checkout a branch (start a transaction), you see the codebase as it was at that commit (your snapshot). Someone else checks out main and starts making changes (another transaction). Their changes don't appear in your branch until you explicitly merge.

If you both edit the same file in the same line (write conflict), Git says "merge conflict — resolve manually." MongoDB does the same: if two transactions write to the same document, one succeeds and the other gets a WriteConflict error and must retry.

```
Git branch    = MongoDB transaction
Git checkout  = start transaction (take snapshot)
Git commit    = commitTransaction
Merge conflict = WriteConflict error → retry transaction
Old commits kept in git history = WiredTiger keeps old document versions
                                    while transactions are active
Git garbage collect = WiredTiger garbage collects old MVCC versions
                      when no active transactions need them
```

---

## Distributed Locks — The Bathroom Key at a Gas Station

Some gas stations have a single bathroom key attached to a large wooden paddle. To use the bathroom, you must get the key from the cashier. While you have the key, no one else can use the bathroom.

When you're done: you return the key (release the lock). If you walk off with the key accidentally: the key has a label with the gas station's address and a phone number — the "lost key" will eventually be returned (TTL index expires the lock).

In MongoDB:
- The lock document = the bathroom key
- `insertOne` with unique index on lock name = taking the key (fails if already taken)
- `deleteOne` with owner check = returning your key (not someone else's)
- `expireAfterSeconds` on the lock document = the return address on the paddle (auto-expires if owner crashes)

```javascript
db.locks.findOneAndUpdate(
  { _id: "bathroom", owner: { $exists: false } },  // key available
  { $set: { owner: myId, expiresAt: new Date(Date.now() + 300_000) } },
  { upsert: true }
)
// Returns null if someone else has the key → wait and try again
```
