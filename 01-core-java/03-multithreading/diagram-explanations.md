# Multithreading — Diagram Explanations

---

## Thread Lifecycle State Machine

```
            start()
   NEW ──────────────► RUNNABLE
                          │
               ┌──────────┴─────────────────────────────┐
               │                                         │
          synchronized              wait() / join() / park()
          (lock held)                                    │
               │                                         ▼
           BLOCKED ◄────────────────────────────────── WAITING
               │    lock acquired     notify()/join      │
               │                      completes          │
               └──────────────────────────────────────►RUNNABLE
                                                         │
                                          sleep(n)/wait(n)/park(n)
                                                         │
                                                         ▼
                                                  TIMED_WAITING
                                                         │
                                              timeout / notify
                                                         │
                                                    RUNNABLE
                                                         │
                                                  run() exits
                                                         │
                                                   TERMINATED
```

---

## ThreadPoolExecutor Work Queue Flow

```
   Task Submitted
        │
        ▼
   Core threads < max?
        │ Yes                    │ No
        ▼                        ▼
   Assign to new          Queue has space?
   core thread               │ Yes         │ No
        │                    ▼             ▼
        │             Add to queue    Max threads?
        │                    │        │ Yes       │ No
        │                    │        ▼            ▼
        │                    │  Reject policy  Create new
        │                    │  (Abort/Caller  thread up
        │                    │  /Discard)      to max
        │                    │                    │
        └────────────────────┴────────────────────┘
                             │
                             ▼
                        Execute Task
```

---

## Lock vs Monitor (synchronized)

```
Object Monitor:
┌──────────────────────────────────────────────────────────┐
│                    Object Monitor                         │
│                                                          │
│  ┌──────────────────────────────────────────────────┐    │
│  │  Entry Set (BLOCKED threads waiting for lock)    │    │
│  │  Thread A, Thread C, Thread E...                 │    │
│  └──────────────────────────────────┬───────────────┘    │
│                                     │ lock released       │
│  ┌──────────────────────────────────▼───────────────┐    │
│  │  Owner (thread currently holding lock)            │    │
│  │  Thread B — executing synchronized block          │    │
│  └──────────────────────────────────┬───────────────┘    │
│                                     │ calls wait()       │
│  ┌──────────────────────────────────▼───────────────┐    │
│  │  Wait Set (WAITING threads after calling wait())  │    │
│  │  Thread D, Thread F...                            │    │
│  │  (moved to Entry Set on notify/notifyAll)         │    │
│  └──────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## Happens-Before Visualization

```
Thread A:                      Thread B:
  x = 1;                         ...
  synchronized(lock) {            synchronized(lock) {
    y = 2;       ─── HB ──────►     int readY = y;  // sees 2
  }                               }
                                   int readX = x;  // sees 1
                                   // because HB is transitive

Volatile example:
  x = 1;                         ...
  volatile_flag = true; ─── HB ─► if (volatile_flag) {
                                     int readX = x;  // guaranteed to see 1
                                   }
```
