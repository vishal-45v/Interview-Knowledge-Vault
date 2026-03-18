# Chapter 02 — Documents & Collections: Diagram Explanations

---

## Diagram 1: CRUD Operations Flow

```
MongoDB CRUD Operations Decision Tree
══════════════════════════════════════════════════════════════════════

CREATE
┌─────────────────────────────────────────────────────┐
│  Single document?  → insertOne({ ... })             │
│  Multiple docs?    → insertMany([...])              │
│  Need _id back?    → result.insertedId / insertedIds│
│  Allow duplicates? → ordered: false on insertMany   │
└─────────────────────────────────────────────────────┘

READ
┌─────────────────────────────────────────────────────┐
│  One document?     → findOne(filter)                │
│  Many documents?   → find(filter)                   │
│  Need count only?  → countDocuments(filter)         │
│  Complex transform? → aggregate([pipeline])         │
│  Reduce fields?    → projection: { field: 1, _id: 0}│
└─────────────────────────────────────────────────────┘

UPDATE
┌─────────────────────────────────────────────────────┐
│  One doc, no return?   → updateOne(filter, {$set})  │
│  Many docs?            → updateMany(filter, {$set}) │
│  Need doc returned?    → findOneAndUpdate(...)      │
│  Replace whole doc?    → replaceOne(filter, newDoc) │
│  Create if not found?  → { upsert: true }           │
└─────────────────────────────────────────────────────┘

DELETE
┌─────────────────────────────────────────────────────┐
│  One doc?              → deleteOne(filter)          │
│  Many docs?            → deleteMany(filter)         │
│  Need deleted doc?     → findOneAndDelete(filter)   │
│  Drop collection?      → db.collection.drop()       │
│  WARNING: empty {}     → deletes ALL documents!     │
└─────────────────────────────────────────────────────┘
```

---

## Diagram 2: Update Operators Visual Guide

```
UPDATE OPERATORS BY CATEGORY
══════════════════════════════════════════════════════════════════════

FIELD OPERATORS
─────────────────────────────────────────────────────
$set     { username: "alice" }  →  sets/creates field
$unset   { tempFlag: "" }       →  removes field entirely
$inc     { counter: 1 }         →  add/subtract (atomic)
$mul     { price: 1.1 }         →  multiply by factor
$rename  { old: "new" }         →  rename field key
$min     { lowest: 5 }          →  set if new value < current
$max     { highest: 95 }        →  set if new value > current
$currentDate { ts: true }       →  server-side current time

BEFORE                          AFTER $set + $inc + $unset
┌───────────────────────┐       ┌───────────────────────┐
│ name: "Alice"         │       │ name: "Alice Updated" │ ← $set
│ loginCount: 5         │  →    │ loginCount: 6         │ ← $inc
│ tempToken: "abc123"   │       │ (tempToken gone)      │ ← $unset
│ email: "a@b.com"      │       │ email: "a@b.com"      │ unchanged
└───────────────────────┘       └───────────────────────┘

ARRAY OPERATORS
─────────────────────────────────────────────────────
$push      adds element (allows duplicates)
$addToSet  adds element ONLY IF not present
$pull      removes ALL matching elements
$pullAll   removes all listed values
$pop       removes first (-1) or last (1) element
$each      modifier for $push/$addToSet (multiple elements)
$slice     modifier: keep only N elements after $push
$sort      modifier: sort array after $push
$position  modifier: insert at specific index

ARRAY STATE TRANSITIONS
─────────────────────────────────────────────────────
Initial: tags = ["js", "python", "js"]

$push: { tags: "go" }
→ ["js", "python", "js", "go"]        (duplicates kept)

$addToSet: { tags: "js" }
→ ["js", "python", "js"]               (no change — "js" exists)

$addToSet: { tags: "rust" }
→ ["js", "python", "js", "rust"]       (added — "rust" was new)

$pull: { tags: "js" }
→ ["python", "rust"]                   (ALL "js" removed)

$pop: { tags: -1 }
→ ["python", "rust", (last element gone if 1)]

$push with $each + $slice:
$push: { tags: { $each: ["go","scala"], $slice: -3 } }
→ ["rust", "go", "scala"]              (kept only last 3)
```

---

## Diagram 3: findOneAndUpdate Return Behavior

```
findOneAndUpdate RETURN DOCUMENT TIMING
══════════════════════════════════════════════════════════════════════

Document before operation: { counter: 10 }

WITHOUT returnDocument option (default = "before"):
                  ┌──────────────────────────────────┐
Application  ───► │ findOneAndUpdate(                │
                  │   filter,                        │
                  │   { $inc: { counter: 1 } }       │
                  │ )                                │
                  └──────────────────────────────────┘
                  ↓ returns
              { counter: 10 }  ← BEFORE the increment!
              (DB now has counter: 11, but you got 10)

WITH returnDocument: "after":
                  ┌──────────────────────────────────┐
Application  ───► │ findOneAndUpdate(                │
                  │   filter,                        │
                  │   { $inc: { counter: 1 } },      │
                  │   { returnDocument: "after" }    │  ← specify this
                  │ )                                │
                  └──────────────────────────────────┘
                  ↓ returns
              { counter: 11 }  ← AFTER the increment!

KEY: default "before" is useful when you need the old value (e.g., audit log)
     "after" is useful when you need to confirm the new state (e.g., inventory check)

NULL return = no document matched the filter
(use this to detect: out of stock, document not found, etc.)
```

---

## Diagram 4: bulkWrite Batching Strategy

```
INDIVIDUAL WRITES vs bulkWrite (10,000 operations)
══════════════════════════════════════════════════════════════════════

INDIVIDUAL WRITES (N round trips):
                         Network latency = ~5ms each
App ──write──► MongoDB   ──ack──► App ──write──► MongoDB ──ack──► App...
    │◄───────5ms────────►│         │◄───────5ms────────►│

Total time: 10,000 × 5ms = 50,000ms = 50 seconds!


bulkWrite in batches of 1000 (10 round trips):
                              One large payload per batch
App ──[1000 ops]──► MongoDB ──ack──► App ──[1000 ops]──► MongoDB ──ack──► App
    │◄────────~10ms──────────►│          │◄────────~10ms──────────►│

Total time: 10 × 10ms = 100ms  ← 500x faster!


BATCH SIZE RECOMMENDATIONS:
────────────────────────────────────────
Document Size    Recommended Batch Size
< 1 KB           1000–5000
1–10 KB          500–1000
10–100 KB        100–500
> 100 KB         50–100

TOO LARGE batch → Network timeout, memory pressure
TOO SMALL batch → Loses efficiency benefit


ORDERED vs UNORDERED bulkWrite:
────────────────────────────────────────
ordered: true (default)         ordered: false
──────────────────────          ──────────────────
Op1 ──► success                Op1 ──► success
Op2 ──► success                Op2 ──► ERROR (logged, continues)
Op3 ──► ERROR ─► STOP          Op3 ──► success
Op4     (never runs)           Op4 ──► success
                               Result: 3 succeeded, 1 error
```

---

## Diagram 5: Upsert Logic Flow

```
UPSERT (updateOne with upsert: true)
══════════════════════════════════════════════════════════════════════

db.users.updateOne(
  { email: "alice@example.com" },          // filter
  { $set: { lastLogin: now },              // update
    $setOnInsert: { createdAt: now } },    // only on INSERT
  { upsert: true }
)

                    ┌─────────────────────────────┐
                    │  Does document matching      │
                    │  filter exist?               │
                    └─────────────────────────────┘
                              │
            ┌─────────────────┴─────────────────┐
            │ YES                               │ NO
            ▼                                   ▼
  ┌──────────────────────┐         ┌──────────────────────────┐
  │  UPDATE existing doc │         │  INSERT new document     │
  │                      │         │                          │
  │  Apply:              │         │  Apply:                  │
  │  $set { lastLogin }  │         │  filter fields (email)   │
  │                      │         │  $set { lastLogin }      │
  │  $setOnInsert: SKIP  │         │  $setOnInsert { createdAt│
  └──────────────────────┘         └──────────────────────────┘
            │                                   │
            ▼                                   ▼
  result.upsertedId = null          result.upsertedId = ObjectId("...")
  result.modifiedCount = 1          result.insertedCount = 1


RESULT INSPECTION:
if (result.upsertedId) {
  // Document was CREATED (insert path)
} else if (result.modifiedCount > 0) {
  // Document was UPDATED (update path)
} else {
  // Document matched but values were already the same (no change)
}
```

---

## Diagram 6: Array Update Operators — Positional Operator

```
POSITIONAL OPERATOR $ IN ARRAY UPDATES
══════════════════════════════════════════════════════════════════════

Document:
{
  _id: 1,
  items: [
    { sku: "A", qty: 5, status: "pending" },   ← index 0
    { sku: "B", qty: 2, status: "pending" },   ← index 1
    { sku: "C", qty: 8, status: "shipped" }    ← index 2
  ]
}

$ — positional: updates FIRST matching element

db.orders.updateOne(
  { _id: 1, "items.sku": "B" },           // filter finds doc AND identifies element
  { $set: { "items.$.status": "shipped" } } // $ = matched element (index 1)
)

Result: items[1].status = "shipped"
        items[0] and items[2] unchanged


$[] — all positional: updates ALL elements

db.orders.updateOne(
  { _id: 1 },
  { $mul: { "items.$[].qty": 2 } }   // double ALL quantities
)

Result: all items get qty doubled


$[identifier] — filtered positional: updates elements matching arrayFilter

db.orders.updateOne(
  { _id: 1 },
  { $set: { "items.$[item].status": "processing" } },
  {
    arrayFilters: [
      { "item.status": "pending" }    // only items where status is "pending"
    ]
  }
)

Result:
items[0].status = "processing"  (was "pending") ✓
items[1].status = "processing"  (was "pending") ✓
items[2].status = "shipped"     (was "shipped") — unchanged ✓
```
