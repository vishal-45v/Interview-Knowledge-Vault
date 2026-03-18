# Chapter 05 — Concurrency & Threading: Diagram Explanations

---

## Diagram 1 — GVL: CPU-bound vs I/O-bound Thread Behavior

```
  CPU-BOUND (no benefit from threads in MRI):

  Time →  0    1    2    3    4    5    6s
  T1:  [RUBY][    ][RUBY][    ][RUBY][    ]
  T2:  [    ][RUBY][    ][RUBY][    ][RUBY]
           ↑ GVL switch
  Result: each thread gets 50% CPU → total = 1x speed (no gain)

  ─────────────────────────────────────────────────────────────

  I/O-BOUND (threads DO help — GVL released during I/O):

  Time →  0    1    2    3    4    5    6s
  T1:  [SEND][WAITING for network...  ][RECV]
  T2:      [SEND][WAITING for network...  ][RECV]
  T3:          [SEND][WAITING for network...  ][RECV]

  During WAITING: GVL is released → other threads can run Ruby code
  Result: 3 concurrent requests finish in ~1s instead of ~3s
```

---

## Diagram 2 — Thread Lifecycle

```
  Thread.new { ... }
       │
       ▼
  ┌─────────┐    GVL available
  │ RUNNABLE│◄──────────────────┐
  └────┬────┘                   │
       │ GVL acquired           │
       ▼                        │ GVL released
  ┌─────────┐                   │ (by OS scheduler)
  │ RUNNING │──────────────────►┘
  └────┬────┘
       │ I/O or sleep or mutex wait
       ▼
  ┌─────────┐
  │ WAITING │ (GVL released here!)
  └────┬────┘
       │ I/O complete or mutex available
       ▼
  ┌─────────┐
  │ RUNNABLE│
  └────┬────┘
       │ block finishes
       ▼
  ┌─────────┐
  │  DEAD   │
  └─────────┘
```

---

## Diagram 3 — Mutex: Critical Section Protection

```
  WITHOUT MUTEX (race condition):
  counter = 0

  Thread 1: reads counter (=0)
                              Thread 2: reads counter (=0)
  Thread 1: computes 0+1=1
  Thread 1: writes counter = 1
                              Thread 2: computes 0+1=1  (stale read!)
                              Thread 2: writes counter = 1

  Expected: 2,  Actual: 1  ← RACE CONDITION

  ─────────────────────────────────────────────────────────────

  WITH MUTEX (protected):
  mutex = Mutex.new

  Thread 1: mutex.lock  ← ACQUIRES LOCK
  Thread 1: reads counter (=0)
  Thread 1: computes 0+1=1
  Thread 1: writes counter = 1
  Thread 1: mutex.unlock  ← RELEASES LOCK
                              Thread 2: mutex.lock  ← ACQUIRES LOCK
                              Thread 2: reads counter (=1)  ← sees updated value
                              Thread 2: computes 1+1=2
                              Thread 2: writes counter = 2
                              Thread 2: mutex.unlock

  Expected: 2,  Actual: 2  ← CORRECT
```

---

## Diagram 4 — Fiber: Cooperative Execution

```
  MAIN       FIBER
  ────────   ──────────────────────────
  resume →   puts "step 1"
             Fiber.yield ──────────────→ PAUSED
  ← "val1"  (main gets "val1")
  ...
  resume →   (fiber resumes from yield)
             puts "step 2"
             Fiber.yield ──────────────→ PAUSED
  ← "val2"  (main gets "val2")
  ...
  resume →   puts "step 3"
             return "done"  ────────────→ DEAD
  ← "done"  (main gets final value)

  Key: SINGLE THREAD — main and fiber take turns
       Fiber controls when it yields
       No OS thread switching — all in one Ruby thread
```

---

## Diagram 5 — Thread vs Fiber vs Ractor Comparison

```
  ┌────────────────┬──────────────┬──────────────┬──────────────┐
  │                │   Thread     │   Fiber      │   Ractor     │
  ├────────────────┼──────────────┼──────────────┼──────────────┤
  │ Scheduling     │ OS (preempt) │ Manual (coop)│ OS (preempt) │
  │ Parallelism    │ No (GVL)     │ No           │ YES          │
  │ Shared Memory  │ Yes          │ Yes          │ No           │
  │ Creation Cost  │ Medium       │ Very Low     │ Medium       │
  │ Stacks         │ ~8MB each    │ ~4KB each    │ Own heap     │
  │ Good for       │ I/O-bound    │ Generators   │ CPU-bound    │
  │ Ruby version   │ All          │ 1.9+         │ 3.0+         │
  └────────────────┴──────────────┴──────────────┴──────────────┘
```

---

## Diagram 6 — Deadlock: Circular Wait

```
  mutex_a = Mutex.new
  mutex_b = Mutex.new

  THREAD 1:                     THREAD 2:
  lock mutex_a ✓ ────────────── lock mutex_b ✓
  waiting for mutex_b... ←────→ ...waiting for mutex_a
  ↑                                          ↑
  BLOCKED                               BLOCKED
  (mutex_b held by T2)        (mutex_a held by T1)

  Result: Neither thread can proceed → DEADLOCK

  FIX: Always acquire locks in the same order:
  THREAD 1:          THREAD 2:
  lock mutex_a ────► lock mutex_a (waits if T1 holds it)
  lock mutex_b       lock mutex_b
  → no circular wait possible
```

---

## Diagram 7 — Queue: Producer-Consumer Pattern

```
  PRODUCER THREAD               QUEUE              CONSUMER THREADS
  ─────────────────     ─────────────────────     ────────────────────
  generate job  ──────► [job1|job2|job3|...] ◄── pop & process (T1)
  generate job  ──────►                     ◄── pop & process (T2)
  generate job  ──────►                     ◄── pop & process (T3)
  :done         ──────► [:done|:done|:done]     (stop signals)

  SizedQueue adds BACKPRESSURE:
  Producer blocks when queue is full:
  ──────►[●●●●●●●●●●] ← full! producer WAITS
          Consumers pop → space opens → producer continues
```

---

## Diagram 8 — Ractor: Isolated Message Passing

```
  MAIN RACTOR:              WORKER RACTOR:
  ┌──────────────┐          ┌──────────────────┐
  │  data = [1,2,3]         │  Own heap         │
  │  Ractor.new(data) ─────►│  receives data    │
  │                │  MOVE  │  (moved, not copy)│
  │  data is now   │        │                  │
  │  INACCESSIBLE  │        │  result = data.sum│
  └──────┬─────────┘        │  Ractor outputs 6 │
         │                  └────────┬─────────┘
         │ .take ◄────────────────────┘
         │ returns 6
         ▼
  puts result  # => 6

  FROZEN objects CAN be shared (not moved):
  MAIN:                     RACTOR:
  config = {...}.freeze ───►config[:key]  # shared, not moved
                             (main keeps access too)
```

---

## Diagram 9 — Fork vs Thread Memory Model

```
  THREAD MODEL (shared memory):

  ┌─────────────────────────────────────────┐
  │  PROCESS (shared memory space)          │
  │                                         │
  │  @counter = 0   ←─── SHARED            │
  │                                         │
  │  Thread 1 ─────────────────────────┐   │
  │  Thread 2 ─────────────────────────┤ reads/writes @counter
  │  Thread 3 ─────────────────────────┘   │
  └─────────────────────────────────────────┘
  Need: Mutex to protect shared data

  ─────────────────────────────────────────────────────────────

  FORK MODEL (separate memory via Copy-on-Write):

  ┌─────────────────┐  fork  ┌─────────────────┐
  │  PARENT PROCESS │───────►│  CHILD PROCESS   │
  │  @counter = 0   │  CoW   │  @counter = 0    │ (own copy)
  │  DB connection  │        │  DB connection   │ (MUST reconnect!)
  └─────────────────┘        └─────────────────┘
  No shared memory — each process is isolated
  Need: IPC (pipe, socket) to share results
```

---

## Diagram 10 — Fiber Scheduler: Single-Thread Async I/O

```
  SINGLE OS THREAD — Fiber Scheduler

  Time:  0ms    200ms   400ms   600ms   800ms  1000ms
  F1:   [send]──[waiting for I/O]──────────────[recv]
  F2:         [send]──[waiting for I/O]──────────────[recv]
  F3:               [send]──[waiting for I/O]──────────────[recv]

  Scheduler switches to next Fiber when current Fiber awaits I/O
  No OS threads needed — OS handles I/O with epoll/kqueue
  Memory: ~4KB per Fiber (vs ~8MB per Thread)

  Result: 1000 concurrent connections = 4MB RAM (Fibers)
                                 vs ~8GB RAM (Threads)
```
