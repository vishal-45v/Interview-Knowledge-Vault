# Chapter 04 — Concurrency & Async: Structured Answers

---

## Q1: Explain the GIL — what it is, why it exists, and what to do about it

**Core answer:**
The Global Interpreter Lock (GIL) is a mutex that protects CPython's internal state.
Only one thread can hold the GIL (and thus execute Python bytecode) at a time.

**Why it exists:**
CPython's memory management (reference counting) is not thread-safe without the GIL.
Every Python object has a `ob_refcnt` field that is incremented/decremented by every
reference operation. Without serialisation, two threads could simultaneously modify
a refcount, causing memory corruption or premature deallocation.

**What the GIL affects:**
```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(100_000):
        counter += 1   # NOT atomic: LOAD_FAST, INPLACE_ADD, STORE_FAST
                       # GIL can switch between these bytecodes → race condition

threads = [threading.Thread(target=increment) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)   # likely LESS than 1,000,000 due to race conditions
```

**When the GIL is released:**
- During I/O operations (socket, file, sleep)
- During C extension calls that explicitly release it (numpy, lxml, sqlite3)
- Every 5ms (Python 3.2+) for other threads to get a turn

**Strategies around the GIL:**
1. **I/O-bound**: Use `threading` — GIL is released during I/O
2. **CPU-bound**: Use `multiprocessing` — each process has its own GIL
3. **CPU-bound Python**: Use `concurrent.futures.ProcessPoolExecutor`
4. **Cython/C extensions**: Can release GIL manually with `nogil`
5. **Experimental**: Python 3.13+ has an opt-in free-threaded build (PEP 703)

---

## Q2: How does asyncio work — the event loop model

**Core answer:**
asyncio uses a single-threaded event loop that multiplexes I/O using the OS
selector (epoll on Linux, kqueue on macOS, IOCP on Windows). Coroutines are
scheduled on the event loop and run until they hit an `await`, at which point
the event loop picks the next ready coroutine.

```python
import asyncio

async def main():
    # These three tasks are truly concurrent on the event loop:
    task_a = asyncio.create_task(fetch_url("https://api1.example.com"))
    task_b = asyncio.create_task(fetch_url("https://api2.example.com"))
    task_c = asyncio.create_task(fetch_url("https://api3.example.com"))

    # Await all three:
    results = await asyncio.gather(task_a, task_b, task_c)
    return results

async def fetch_url(url: str) -> str:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.text()

asyncio.run(main())
```

**Event loop scheduling:**

```
Coroutine A: ──► await network_io ──────────────────────► resumes ──►
Coroutine B:                    ──► runs ──► await db_io ──────────►
Coroutine C:                                            ──► runs ──►
Timeline:     [A runs][B runs][C runs][A resumes][B resumes][...]
```

**Key asyncio APIs:**
```python
import asyncio

# Create a task (scheduled, runs concurrently):
task = asyncio.create_task(some_coro())

# Gather multiple coroutines/tasks:
results = await asyncio.gather(coro1(), coro2(), coro3())

# Timeout:
try:
    result = await asyncio.wait_for(slow_coro(), timeout=5.0)
except asyncio.TimeoutError:
    print("timed out")

# Run blocking code without freezing event loop:
loop = asyncio.get_running_loop()
result = await loop.run_in_executor(None, blocking_function, arg1, arg2)

# Sleep without blocking event loop:
await asyncio.sleep(1.0)   # yields control; NOT time.sleep()!
```

---

## Q3: When to use threads vs processes vs asyncio

**Core answer:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  TASK TYPE          BEST TOOL           REASON                       │
├─────────────────────────────────────────────────────────────────────┤
│  I/O-bound          asyncio             single-thread, lowest        │
│  (many concurrent   (+ aiohttp, etc.)   overhead, scales to 1000s   │
│   network/disk)                         of concurrent connections    │
│                     OR threading        simpler mental model for     │
│                                         modest concurrency           │
├─────────────────────────────────────────────────────────────────────┤
│  CPU-bound          multiprocessing     bypasses GIL; true parallel  │
│  (computation)      ProcessPoolExecutor execution on all CPU cores   │
├─────────────────────────────────────────────────────────────────────┤
│  Mixed I/O + CPU    asyncio +           async for I/O, executor      │
│                     run_in_executor     for CPU bursts               │
├─────────────────────────────────────────────────────────────────────┤
│  Simple parallel    concurrent.futures  higher-level; automatic      │
│  with result        ThreadPool /        pool management; futures      │
│  collection         ProcessPool         for result gathering         │
└─────────────────────────────────────────────────────────────────────┘
```

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import requests

urls = ["https://httpbin.org/delay/1"] * 20

# I/O-bound — threads (GIL released during socket I/O):
def fetch(url): return requests.get(url).status_code

with ThreadPoolExecutor(max_workers=20) as pool:
    codes = list(pool.map(fetch, urls))  # ~1s instead of 20s

# CPU-bound — processes (bypass GIL):
def crunch(n): return sum(i**2 for i in range(n))

tasks = [10**6] * 8
with ProcessPoolExecutor(max_workers=8) as pool:
    results = list(pool.map(crunch, tasks))  # uses all CPU cores
```

---

## Q4: Implement a producer-consumer pattern with asyncio.Queue

**Core answer:**

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, producer_id: int):
    """Generates work items and puts them on the queue."""
    for i in range(5):
        item = f"task-{producer_id}-{i}"
        await queue.put(item)
        print(f"Producer {producer_id}: put {item}")
        await asyncio.sleep(random.uniform(0.1, 0.3))
    print(f"Producer {producer_id}: done")

async def consumer(queue: asyncio.Queue, consumer_id: int):
    """Processes work items from the queue."""
    while True:
        try:
            item = await asyncio.wait_for(queue.get(), timeout=2.0)
            print(f"Consumer {consumer_id}: processing {item}")
            await asyncio.sleep(random.uniform(0.2, 0.5))   # simulate work
            queue.task_done()
        except asyncio.TimeoutError:
            print(f"Consumer {consumer_id}: no items, shutting down")
            break

async def main():
    queue = asyncio.Queue(maxsize=10)   # bounded queue — backpressure

    producers = [asyncio.create_task(producer(queue, i)) for i in range(3)]
    consumers = [asyncio.create_task(consumer(queue, i)) for i in range(2)]

    # Wait for all producers to finish:
    await asyncio.gather(*producers)

    # Wait for all items to be processed:
    await queue.join()   # blocks until task_done() called for every item

    # Cancel consumers (they're polling with timeout but still running):
    for c in consumers:
        c.cancel()

asyncio.run(main())
```

---

## Q5: Fix race conditions with threading primitives

**Core answer:**

```python
import threading

# RACE CONDITION — not thread-safe:
counter = 0

def unsafe_increment():
    global counter
    for _ in range(100_000):
        counter += 1   # read-modify-write not atomic!

# FIX 1: Lock
counter = 0
lock = threading.Lock()

def safe_increment():
    global counter
    for _ in range(100_000):
        with lock:
            counter += 1   # atomic read-modify-write under lock

# FIX 2: threading.local() for per-thread state (no sharing needed):
local_data = threading.local()

def thread_task():
    local_data.value = 0    # each thread has its own .value
    for _ in range(100_000):
        local_data.value += 1
    # no race — each thread works on its own copy

# Semaphore — limit concurrent access to a resource:
db_semaphore = threading.Semaphore(5)  # max 5 concurrent DB connections

def query_database(query):
    with db_semaphore:   # blocks if 5 connections already active
        return db.execute(query)

# Event — signal between threads:
data_ready = threading.Event()
shared_data = None

def producer_thread():
    global shared_data
    shared_data = fetch_data()
    data_ready.set()   # signal that data is available

def consumer_thread():
    data_ready.wait()   # block until producer signals
    process(shared_data)
```
