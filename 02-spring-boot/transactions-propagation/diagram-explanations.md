# Spring Transactions — Diagram Explanations

> ASCII diagrams for transaction propagation, isolation levels, proxy mechanics, and JPA integration.

---

## Diagram 1: All Propagation Behaviors Side-by-Side

```
Propagation   Caller has TX?   Callee behavior
──────────────────────────────────────────────────────────────────────

REQUIRED       YES             ┌─────── Outer TX ────────────────┐
(default)                      │  callerMethod()                 │
                               │       ↓                         │
                               │  calleeMethod() ← joins same TX │
                               └─────────────────────────────────┘

               NO              ┌─ New TX ─┐
                               │callee()  │
                               └──────────┘

REQUIRES_NEW   YES             ┌─ Outer TX (SUSPENDED) ─────────┐
                               │  callerMethod()                │
                               │  ┌─ New TX ─────────────────┐  │
                               │  │  calleeMethod()           │  │
                               │  │  (commits independently)  │  │
                               │  └───────────────────────────┘  │
                               │  (outer TX resumes)            │
                               └────────────────────────────────┘

NESTED         YES             ┌─ Outer TX ─────────────────────┐
                               │  callerMethod()                │
                               │  ── SAVEPOINT ─────────────┐   │
                               │  │  calleeMethod()          │   │
                               │  │  (rollback → savepoint)  │   │
                               │  └──────────────────────────┘   │
                               │  (outer TX continues)          │
                               └────────────────────────────────┘

SUPPORTS       YES             Joins existing TX (same as REQUIRED when TX exists)
               NO              Runs without TX

NOT_SUPPORTED  YES             ┌─ Outer TX (SUSPENDED) ─┐  ┌─ No TX ─┐
                               │                        │  │ callee  │
                               └────────────────────────┘  └─────────┘
               NO              Runs without TX

MANDATORY      YES             Joins existing TX
               NO              IllegalTransactionStateException ✗

NEVER          YES             IllegalTransactionStateException ✗
               NO              Runs without TX
```

---

## Diagram 2: Self-Invocation — The Proxy Bypass Problem

```
CORRECT (external call via proxy):

  OrderController
       │
       ▼
  [OrderService CGLIB Proxy]
       │
       │  placeOrder() — goes through proxy
       ▼
  [Real OrderService]
  placeOrder() {
       │
       │ self.sendConfirmation()  ← goes through PROXY again ✓
       ▼
  [OrderService CGLIB Proxy]
       │
       │ Starts REQUIRES_NEW transaction ✓
       ▼
  [Real OrderService]
  sendConfirmation() ← has its own transaction ✓


WRONG (self-invocation, bypasses proxy):

  OrderController
       │
       ▼
  [OrderService CGLIB Proxy]
       │
       │  placeOrder() — goes through proxy
       ▼
  [Real OrderService]
  placeOrder() {
       │
       │ this.sendConfirmation()  ← direct call, NO PROXY ✗
       ▼
  [Real OrderService]
  sendConfirmation() ← @Transactional(REQUIRES_NEW) IGNORED ✗
                        runs in same TX as placeOrder()
```

---

## Diagram 3: REQUIRES_NEW and Database Connection Usage

```
Connection Pool (10 connections):
[C1][C2][C3][C4][C5][C6][C7][C8][C9][C10]

Normal flow (REQUIRED — joins outer TX):
  Thread 1: processOrders() → uses C1
     └── saveItems() (REQUIRED) → uses C1 (same connection!) ✓
  Thread 2: processOrders() → uses C2
     └── saveItems() (REQUIRED) → uses C2 (same connection!) ✓
  ... 10 threads → 10 connections used ✓

Dangerous flow (REQUIRES_NEW — needs 2nd connection):
  Thread 1: processOrders() → holds C1
     └── saveItems() (REQUIRES_NEW) → needs C2! → uses C2
  Thread 2: processOrders() → holds C3
     └── saveItems() (REQUIRES_NEW) → needs C4! → uses C4
  ...
  Thread 5: processOrders() → holds C9
     └── saveItems() (REQUIRES_NEW) → needs C10! → uses C10

  Thread 6: processOrders() → holds C5 (from somewhere)
  Wait... all 10 connections taken!
     └── saveItems() (REQUIRES_NEW) → waiting for available connection...
         ↓
  ALL THREADS WAITING FOR CONNECTIONS → DEADLOCK ✗
```

---

## Diagram 4: Transaction Isolation Levels — What Each Protects Against

```
  Isolation Level    │ Dirty  │ Non-Repeatable │ Phantom
                     │ Reads  │ Reads          │ Reads
  ───────────────────┼────────┼────────────────┼────────
  READ UNCOMMITTED   │   ✗    │       ✗        │   ✗
  (lowest)           │allowed │    allowed      │ allowed
                     │        │                │
  READ COMMITTED     │   ✓    │       ✗        │   ✗
  (PostgreSQL,Oracle)│blocked │    allowed      │ allowed
                     │        │                │
  REPEATABLE READ    │   ✓    │       ✓        │   ✗
  (MySQL default)    │blocked │    blocked      │ allowed
                     │        │                │
  SERIALIZABLE       │   ✓    │       ✓        │   ✓
  (highest)          │blocked │    blocked      │ blocked
  ───────────────────┴────────┴────────────────┴────────

Concurrency Problem Illustrated:

  DIRTY READ:
  T1: UPDATE balance = 50 (not committed)
  T2: READ balance → 50  ← reads uncommitted data!
  T1: ROLLBACK
  T2: Used wrong value 50 (actual was 100)

  NON-REPEATABLE READ:
  T1: READ price → 100
  T2: UPDATE price = 200; COMMIT
  T1: READ price → 200  ← same row, different value!

  PHANTOM READ:
  T1: SELECT COUNT(*) WHERE status='active' → 10
  T2: INSERT new active record; COMMIT
  T1: SELECT COUNT(*) WHERE status='active' → 11  ← extra row!
```

---

## Diagram 5: JPA Transaction and EntityManager Lifecycle

```
  @Transactional method begins
          │
          ▼
  ┌──────────────────────────────────────────────────────────┐
  │  Persistence Context (EntityManager) created             │
  │                                                          │
  │  ┌─────────────────────────────────────────────────────┐ │
  │  │  First-Level Cache (L1 Cache)                        │ │
  │  │                                                      │ │
  │  │  em.find(User, 1L) ──► SQL: SELECT * FROM users     │ │
  │  │                        WHERE id=1                    │ │
  │  │  User{id=1} stored in cache                          │ │
  │  │                                                      │ │
  │  │  em.find(User, 1L) ──► Returns from cache (no SQL!) │ │
  │  │                                                      │ │
  │  │  user.setName("Bob") ──► tracked as DIRTY           │ │
  │  │                                                      │ │
  │  └─────────────────────────────────────────────────────┘ │
  │                          │                               │
  │  em.flush() ─────────────┘                               │
  │  → Dirty checking: compare state to snapshot             │
  │  → Generate SQL: UPDATE users SET name='Bob' WHERE id=1  │
  │  → Execute SQL (still in TX)                             │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
          │
          ▼
  TX commit → DB commit
          │
          ▼
  EntityManager closes → L1 cache cleared
  → user entity becomes DETACHED
  → Accessing lazy-loaded collections → LazyInitializationException ✗
```

---

## Diagram 6: Outbox Pattern for Reliable Event Publishing

```
  PROBLEM (without Outbox):

  @Transactional
  placeOrder() {
      DB: INSERT order          ┐  in same TX
      MQ: publish OrderCreated  ┘  DIFFERENT SYSTEM!
  }

  Failure scenarios:
  ● DB commit succeeds + MQ publish fails → order exists, no event ✗
  ● DB commit fails   + MQ publish succeeds → event without order ✗
  ● No atomicity between DB and MQ ✗

  SOLUTION (Outbox Pattern):

  ┌────────────────────────────────────────────────────────────┐
  │  @Transactional                                            │
  │  placeOrder() {                                            │
  │      DB: INSERT orders        ─────────────────────────┐  │
  │      DB: INSERT outbox_events (type, payload, pending)  ┘  │
  │  }  ← Atomic: both or neither ✓                            │
  └────────────────────────────────────────────────────────────┘
                                │
                                ▼
  ┌────────────────────────────────────────────────────────────┐
  │  Outbox Relay (separate process/thread)                    │
  │                                                            │
  │  Poll outbox_events WHERE published = false                │
  │       │                                                    │
  │       ▼                                                    │
  │  Publish to Message Queue ──────────────────────────────►  │
  │       │              success                               │
  │       ▼                                                    │
  │  UPDATE outbox_events SET published = true                 │
  │                                                            │
  │  [On MQ failure: retry with backoff, no data loss]         │
  └────────────────────────────────────────────────────────────┘

  Guarantees: At-least-once delivery
  Message Queue: Idempotent consumers handle duplicate delivery
```

---

## Diagram 7: Optimistic vs Pessimistic Locking

```
  OPTIMISTIC LOCKING (@Version):

  T1: SELECT * FROM products WHERE id=1 → {qty=100, version=5}
  T2: SELECT * FROM products WHERE id=1 → {qty=100, version=5}

  T1: UPDATE products SET qty=90, version=6 WHERE id=1 AND version=5
      → Rows updated: 1 ✓  (version matched, commit)

  T2: UPDATE products SET qty=95, version=6 WHERE id=1 AND version=5
      → Rows updated: 0 ✗  (version=5 no longer exists!)
      → OptimisticLockException thrown
      → Application retries T2

  Good for: Low contention, many reads, occasional concurrent writes
  Bad for:  High contention (many retries under load)

  PESSIMISTIC LOCKING (SELECT FOR UPDATE):

  T1: SELECT * FROM products WHERE id=1 FOR UPDATE
      → Row LOCKED ✓
  T2: SELECT * FROM products WHERE id=1 FOR UPDATE
      → WAITS for T1 to release lock...

  T1: UPDATE products SET qty=90 WHERE id=1
  T1: COMMIT → Lock released
      → T2 can now proceed

  T2: (now gets lock) reads qty=90 (T1's committed value)
  T2: UPDATE products SET qty=85 WHERE id=1
  T2: COMMIT

  Good for: High contention, financial operations
  Bad for:  Lock contention, deadlock risk, scalability
```
