# Chapter 03 — Go Concurrency: Diagram Explanations

---

## Diagram 1: GMP Scheduler Model

```
Go Runtime Scheduler — GMP Model
=================================

  G = Goroutine (unit of work, ~2KB stack)
  M = Machine (OS thread, managed by kernel)
  P = Processor (logical CPU, run queue, scheduler state)

  GOMAXPROCS = 2  →  2 Processors, 2 active OS threads

  ┌─────────────────────────────────────────────────────────────┐
  │                      Global Run Queue                       │
  │         G9  G10  G11  G12  ...                              │
  └───────────────────────────┬─────────────────────────────────┘
                              │ (steal when local queue empty)
          ┌────────────────── ┼ ──────────────────┐
          │                   │                   │
          ▼                   │                   ▼
  ┌───────────────┐           │         ┌───────────────┐
  │      P1       │           │         │      P2       │
  │  Local Queue  │           │         │  Local Queue  │
  │  [G1 G2 G3]  │           │         │  [G4 G5 G6]  │
  └───────┬───────┘           │         └───────┬───────┘
          │                   │                 │
          │ runs              │         runs    │
          ▼                   │                 ▼
       ┌────┐                 │              ┌────┐
       │ M1 │◄── syscall ─────┘              │ M2 │
       └────┘  (M detaches,                  └────┘
                new M picks up P1)
          │                                    │
          ▼                                    ▼
   CPU Core 1                           CPU Core 2

Work Stealing:
  When P1's local queue is empty:
  1. Check global queue → steal half
  2. Check other Ps → steal half of their queues
  3. Check network poller → pick up goroutines waiting on I/O

Goroutine lifecycle:
  Created → Runnable (in queue) → Running (on M) → Blocked/Waiting
  Blocked (on channel/mutex/syscall) → back to Runnable when unblocked
```

---

## Diagram 2: Buffered vs Unbuffered Channel

```
UNBUFFERED CHANNEL (make(chan int))
====================================

  Sender goroutine          Receiver goroutine
  ─────────────────         ──────────────────
  ch <- 42                  v := <-ch

  Timeline:
  ─────────
  Sender:   ──send──BLOCKED───────────────────UNBLOCKED──►
                         ↕
  Receiver: ─────────────────receive──────────────────────►
                         │
                  handoff happens HERE (synchronous)

  Both goroutines must be at the channel at the same time.


BUFFERED CHANNEL (make(chan int, 3))
=====================================

  Buffer: [ _ | _ | _ ]   capacity = 3

  After ch <- 1:  [ 1 | _ | _ ]  sender continues immediately
  After ch <- 2:  [ 1 | 2 | _ ]  sender continues immediately
  After ch <- 3:  [ 1 | 2 | 3 ]  sender continues immediately
  ch <- 4:  BLOCKS — buffer full, sender waits for receiver

  v1 := <-ch  →  [ _ | 2 | 3 ]  v1=1, receiver continues
  v2 := <-ch  →  [ _ | _ | 3 ]  v2=2

  Timeline (buffer capacity = 3):
  ─────────────────────────────────────────────────────────
  Sender:   ──1──2──3──4─BLOCKED──────UNBLOCKED──5──6──►
                          buffer full │ receiver drained one
  Buffer:   [0][0][0]→[1][2][3]→[1][2][3]→[2][3]→[2][3][5]►
  Receiver: ────────────────────drain────drain──────────────►
```

---

## Diagram 3: Fan-Out / Fan-In Pattern

```
Fan-Out / Fan-In Pipeline
==========================

         Input
         Channel
           │
           │  1 2 3 4 5 6 7 8 9 10 ...
           │
     ┌─────┴──────┐
     │  Fan-Out   │  (distribute to multiple workers)
     └─────┬──────┘
           │
   ┌───────┼───────┐
   │       │       │
   ▼       ▼       ▼
 ┌────┐  ┌────┐  ┌────┐
 │ W1 │  │ W2 │  │ W3 │   Workers (goroutines) process items
 │    │  │    │  │    │   concurrently from the SAME input channel
 └──┬─┘  └──┬─┘  └──┬─┘
    │        │       │
    │   ┌────┴────┐  │
    └──►│ Fan-In  │◄─┘   (merge results into one output)
        └────┬────┘
             │
             ▼
           Output
           Channel
      1² 4² 9² 16² ...

Code flow:
  input := generateJobs()            // one producer
  w1 := worker(input)                // W1 reads from input
  w2 := worker(input)                // W2 reads from same input
  w3 := worker(input)                // W3 reads from same input
  output := fanIn(w1, w2, w3)       // merge W1, W2, W3 results
  for result := range output { ... }
```

---

## Diagram 4: Worker Pool Architecture

```
Worker Pool Pattern
====================

  Producer goroutine              Collector goroutine
  ──────────────────              ───────────────────
  for _, job := range jobs {      for r := range results {
      jobsCh <- job                   process(r)
  }                               }
  close(jobsCh)

        │ jobsCh (buffered chan Job)         ▲
        │                                   │ resultsCh (buffered chan Result)
        ▼                                   │
  ┌──────────────────────────────────────────────────────┐
  │                   Worker Pool                         │
  │                                                       │
  │   ┌─────────┐  ┌─────────┐  ┌─────────┐             │
  │   │Worker 1 │  │Worker 2 │  │Worker 3 │   N workers  │
  │   │for j := │  │for j := │  │for j := │             │
  │   │range    │  │range    │  │range    │             │
  │   │jobsCh { │  │jobsCh { │  │jobsCh { │             │
  │   │  result │  │  result │  │  result │             │
  │   │  sCh<-  │  │  sCh<-  │  │  sCh<-  │             │
  │   │  proc(j)│  │  proc(j)│  │  proc(j)│             │
  │   │}        │  │}        │  │}        │             │
  │   └─────────┘  └─────────┘  └─────────┘             │
  └──────────────────────────────────────────────────────┘

  When jobsCh is closed, all workers' range loops exit.
  When all workers finish, wg.Wait() unblocks and closes resultsCh.
  When resultsCh is closed, collector's range loop exits.
```

---

## Diagram 5: Context Tree — Parent → Child Cancellation

```
Context Tree — Cancellation Propagation
=========================================

  context.Background()   ← root, never cancelled
         │
         │ context.WithTimeout(bg, 30s)
         ▼
  reqCtx (30s timeout)   ← created per HTTP request
    cancel() auto-fires at 30s OR client disconnects
         │
    ┌────┴─────────────┐
    │                  │
    │ context.WithTimeout(reqCtx, 5s)
    ▼                  ▼
  dbCtx (5s)      apiCtx (5s)    ← child contexts
         │
         │ context.WithTimeout(dbCtx, 1s)
         ▼
  queryCtx (1s)                   ← grandchild

  Cancellation rules:
  ─────────────────────────────────────────
  If reqCtx is cancelled (30s or disconnect):
    → dbCtx is cancelled automatically
      → queryCtx is cancelled automatically
    → apiCtx is cancelled automatically

  If dbCtx times out (5s):
    → queryCtx is cancelled automatically
    → reqCtx is NOT affected (parent not cancelled by child)

  Example:
  ─────────────────────────────────────────
  func handleRequest(ctx context.Context) {
      // reqCtx ← from http request
      dbCtx, cancel := context.WithTimeout(ctx, 5*time.Second)
      defer cancel()

      row := db.QueryRowContext(dbCtx, "SELECT ...") // uses dbCtx
      // If HTTP client disconnects: ctx cancelled → dbCtx cancelled → query aborted
      // If DB is slow > 5s: dbCtx times out → query aborted
  }

ctx.Done() channel:
  ─────────────────────────────────────────
  The context's Done() channel is closed when the context is cancelled.
  Multiple goroutines can all select on ctx.Done() to detect cancellation.

  select {
  case result := <-workCh:   // got a result
  case <-ctx.Done():          // context cancelled — stop working
      return ctx.Err()
  }
```

---

## Diagram 6: select Statement Flow

```
select Statement with Multiple Channels
========================================

Code:
  select {
  case msg := <-inbox:
      handleMessage(msg)
  case outbox <- reply:
      log("sent reply")
  case <-time.After(5 * time.Second):
      log("timeout")
  case <-done:
      return
  }

Execution flow:

  ┌─────────────────────────────────────────────────────┐
  │            select evaluates ALL cases               │
  └──────────────────────┬──────────────────────────────┘
                         │
     ┌───────────────────┼────────────────────┐
     ▼                   ▼                    ▼
  case 1: inbox    case 2: outbox       case 3: time.After
  ready? NO        ready? NO            ready? NO
     │                   │                    │
     └───────────────────┴────────────────────┘
                         │
                         ▼
               ALL blocked: goroutine
               suspends, waits for ANY
               case to become ready
                         │
               (5 seconds pass)
                         │
                         ▼
              time.After fires! Case 3 ready
                         │
                         ▼
              case <-time.After: runs → "timeout"

Multiple cases ready at same time:
  ┌──────────────────────────────────────┐
  │  case 1: inbox  ready? YES           │
  │  case 4: done   ready? YES           │
  │                                      │
  │  Go picks ONE at random              │
  │  (uniform pseudo-random selection)   │
  └──────────────────────────────────────┘

Default case (non-blocking):
  select {
  case v := <-ch:
      // use v
  default:
      // ch has nothing — don't block, run this
  }
  → Immediately runs default if no case is ready
```

---

## Diagram 7: Mutex Lock/Unlock Timeline

```
sync.Mutex — Single Lock Timeline
====================================

  3 goroutines: G1, G2, G3

  Time ────────────────────────────────────────────────►

  G1: ──Lock()──[CRITICAL SECTION]──Unlock()──────────────────
              │                         │
  G2: ─Lock()─BLOCKED─────────────────Lock()──[CRITICAL]──Unlock()─
         (waiting for G1)            │
  G3: ────Lock()─────BLOCKED──────────┼─BLOCKED──────────Lock()──►
         (waiting for G1 or G2)       │     (waiting for G2)


sync.RWMutex — Multiple Readers Timeline
=========================================

  R = reader, W = writer

  Time ─────────────────────────────────────────────────────────►

  R1: ──RLock()──[READ]──RUnlock()─────────────────────────────
  R2: ──RLock()──[READ]───────────RUnlock()────────────────────
  R3: ─────RLock()──[READ]────────────────RUnlock()────────────
       ↑ All three readers hold RLock simultaneously — concurrent reads OK

   W: ──────Lock()─────────────────────BLOCKED──Lock()──[WRITE]──Unlock()──
       (writer waits for all active readers to release RLock)
                                       │
  R4: ──────────────────────────RLock()─BLOCKED──────────RLock()──[READ]──►
       (new readers blocked after writer is waiting — prevents writer starvation)
```
