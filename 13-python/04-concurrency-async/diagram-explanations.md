# Chapter 04 — Concurrency & Async: Diagram Explanations

---

## Diagram 1: GIL — Thread Execution Timeline

```
Two Python threads — CPU-bound task:

Thread 1:  ██████GIL██████ · · · wait · · · ██████GIL██████ · · ·
Thread 2:  · · · wait · · · ██████GIL██████ · · · wait · · · █████
Timeline:  ───────────────────────────────────────────────────────►
           0ms       5ms      10ms      15ms      20ms

GIL switches every ~5ms (sys.getswitchinterval())
→ True parallel execution is NOT achieved for CPU work
→ Only ONE thread runs Python bytecode at any instant

Two Python threads — I/O-bound task:

Thread 1:  ████ ·~I/O~· · · · · · · ████
            (GIL)  (GIL RELEASED)   (resumes, re-acquires GIL)
Thread 2:          ████ ·~I/O~· · · · · · · ████
                   (GIL released by T1, T2 gets it, then releases for I/O)

→ GIL is RELEASED during I/O → threads DO run "concurrently" for I/O
→ Total wall time ≈ max(individual_io_times), not sum()
```

---

## Diagram 2: asyncio Event Loop — Coroutine Scheduling

```
                        ┌──────────────────────────────────────────┐
                        │         EVENT LOOP                        │
                        │                                           │
  asyncio.run(main())   │   Ready Queue                             │
         │              │   ┌─────────────────────────────────┐    │
         ▼              │   │  coro_A (ready to run)           │    │
   ┌──────────┐         │   │  coro_B (ready to run)           │    │
   │  main()  │◄────────┤   │  coro_C (just resumed from I/O) │    │
   └──────────┘         │   └─────────────────────────────────┘    │
                        │                                           │
                        │   I/O Waiting (OS selector/epoll)         │
                        │   ┌─────────────────────────────────┐    │
                        │   │  coro_D: socket fd=5 readable?   │    │
                        │   │  coro_E: socket fd=8 writable?   │    │
                        │   └─────────────────────────────────┘    │
                        │                                           │
                        │   Sleeping                                │
                        │   ┌─────────────────────────────────┐    │
                        │   │  coro_F: wake at t+1.5s          │    │
                        │   └─────────────────────────────────┘    │
                        └──────────────────────────────────────────┘

Execution cycle (each "tick"):
  1. Pick next item from Ready Queue
  2. Run it until it hits `await` or returns
  3. If awaiting I/O: register with OS selector, move to I/O Waiting
  4. If awaiting sleep: move to Sleeping with wake time
  5. Check OS selector for completed I/O → move ready coroutines to Ready Queue
  6. Check Sleeping for expired timers → move to Ready Queue
  7. Repeat
```

---

## Diagram 3: Thread vs Process vs Async — Resource Model

```
THREADING:
┌──────────────────────────────────────────────────────────────────┐
│  Process (one Python interpreter, ONE GIL)                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │    Thread 1      │  │    Thread 2      │  │   Thread 3     │  │
│  │  own stack       │  │  own stack       │  │  own stack     │  │
│  │  shared heap ────┼──┼── shared heap ───┼──┼── shared heap  │  │
│  └──────────────────┘  └──────────────────┘  └────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
  GIL: only 1 thread runs bytecode at a time
  Memory: shared → fast, but needs locking

MULTIPROCESSING:
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│  Process 1       │  │  Process 2       │  │  Process 3       │
│  own GIL         │  │  own GIL         │  │  own GIL         │
│  own memory      │  │  own memory      │  │  own memory      │
│  own interpreter │  │  own interpreter │  │  own interpreter │
└────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                    IPC (Queue, Pipe, Manager)
                    — data must be serialised (pickle)

ASYNCIO:
┌──────────────────────────────────────────────────────────────────┐
│  Process (1 thread, 1 GIL, 1 interpreter)                        │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Event Loop                                               │   │
│  │  coroutine_A  coroutine_B  coroutine_C  ...coroutine_N   │   │
│  │  (all share the same thread and memory)                   │   │
│  │  (only 1 runs at a time; switch on await)                 │   │
│  └──────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
  No GIL contention; no IPC overhead; scales to 10,000s connections
  BUT: 1 blocking call freezes everything
```

---

## Diagram 4: concurrent.futures — Executor Abstraction

```
concurrent.futures API:

  executor.submit(fn, *args) → Future
  executor.map(fn, iterable) → iterator of results

  Future object:
  ┌─────────────────────────────────────────────────────────────┐
  │  .result()     — block until done, return result or raise   │
  │  .exception()  — return exception if raised, else None      │
  │  .done()       — True if completed (including exceptions)   │
  │  .cancel()     — attempt to cancel if not started           │
  │  .add_done_callback(fn)  — call fn when done                │
  └─────────────────────────────────────────────────────────────┘

ThreadPoolExecutor:
  ┌──────────────────────────────────────────────────────────────┐
  │  Main Thread                                                  │
  │  submit(fn, args) ──────────────────────────────────────┐   │
  │                                                          ▼   │
  │  Thread Pool  [ worker1 | worker2 | worker3 | worker4 ] │   │
  │                    │         │         │                 │   │
  │                  fn(args)  fn(args)  fn(args)            │   │
  │                    └─────────┴─────────┘──────────────► │   │
  │                                                Future   │   │
  └──────────────────────────────────────────────────────────┘

ProcessPoolExecutor (same API, different workers):
  Main Process → [Worker Process 1 | Worker Process 2 | ...]
  Data flows via pickle (serialisation over pipes)

as_completed() pattern (results as they finish, not in order):

  futures = {executor.submit(fn, x): x for x in items}
  for future in concurrent.futures.as_completed(futures):
      original_arg = futures[future]
      result = future.result()
```

---

## Diagram 5: Producer-Consumer with asyncio.Queue

```
                 ┌─────────────────────────────────────┐
   Producer 1 ──►│                                     │
   Producer 2 ──►│   asyncio.Queue (maxsize=10)         │──► Consumer 1
   Producer 3 ──►│   [item1, item2, item3, item4, ...]  │──► Consumer 2
                 │                                     │
                 └─────────────────────────────────────┘
                         Backpressure: if full, await queue.put() blocks
                         until a consumer calls queue.get()

Queue state machine:
  queue.put(item)     → if full:  await (backpressure)
                     → else:     add item to queue
  queue.get()         → if empty: await (suspend consumer)
                     → else:     remove and return first item
  queue.task_done()   → decrements count of outstanding tasks
  await queue.join()  → waits until task_done() called for every put()

Lifecycle:
  1. Producers call put()   →  queue size increases
  2. Consumers call get()   →  queue size decreases
  3. Consumers call task_done() after processing
  4. queue.join() returns when all tasks done
  5. Cancel consumers (they poll with timeout)
  6. Program exits cleanly
```

---

## Diagram 6: Async/Await — What Happens at the Bytecode Level

```
Python source:                    Equivalent manual coroutine mechanism:

async def fetch(url):             def fetch(url):
    data = await get(url)    →       gen = get(url)       # create sub-generator
    return process(data)             try:
                                         yielded = next(gen)  # prime it
                                         while True:
                                             result = yield yielded  # pass upward
                                             try:
                                                 yielded = gen.send(result)
                                             except StopIteration as e:
                                                 data = e.value
                                                 break
                                     except StopIteration:
                                         pass
                                     return process(data)

`await expr` is equivalent to `yield from expr` for awaitable objects.

The event loop drives the outermost coroutine:
  1. event loop calls coro.send(None)  → coro runs until yield
  2. yielded value = an asyncio Future
  3. event loop registers callback on Future
  4. when Future resolves, event loop calls coro.send(result)
  5. coro resumes after the await point with result
  6. repeat until StopIteration (coroutine returns)

await points are COOPERATIVE YIELD POINTS — the coroutine gives
up control voluntarily. No preemption happens between yield points.
CPU-bound code between await points cannot be interrupted.
```
