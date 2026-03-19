# Chapter 04 — Concurrency & Async: Follow-Up Traps

---

## Trap 1: "The GIL means Python threads are useless"

**What most people say:**
"Because of the GIL, Python threads don't actually run in parallel, so there's no
point using them."

**Correct answer:**
The GIL is released during I/O operations and by C extensions that explicitly release
it (e.g., NumPy during computation). For I/O-bound tasks (network calls, disk reads,
database queries), threads ARE useful — one thread can be waiting on I/O while another
runs Python bytecode. The GIL only prevents true parallel *Python bytecode* execution.

```python
import threading, time, requests

def fetch(url):
    requests.get(url)   # GIL is RELEASED during the actual socket I/O

urls = ["https://httpbin.org/delay/1"] * 5

# Sequential: ~5s (each I/O waits)
# With threads: ~1s (I/O overlaps — GIL released during socket wait)
threads = [threading.Thread(target=fetch, args=(url,)) for url in urls]
for t in threads: t.start()
for t in threads: t.join()
# Roughly 1 second — not 5 seconds — because GIL released during I/O
```

---

## Trap 2: "async/await means my code runs in parallel"

**What most people say:**
"`async/await` runs things concurrently like threads, just with less overhead."

**Correct answer:**
`async/await` is *cooperative concurrency*, not parallelism. Only ONE coroutine runs
at any instant on one thread. A coroutine runs until it hits an `await`, at which
point it voluntarily yields control back to the event loop, which picks another
ready coroutine. If a coroutine does CPU-bound work without any `await`, it monopolises
the event loop and starves all other coroutines.

```python
import asyncio

async def cpu_hog():
    # This BLOCKS the event loop — no await, no yield of control
    total = sum(range(10**8))   # ~3s of pure CPU work
    return total

async def quick_task():
    print("quick task running")
    await asyncio.sleep(0)    # yield to event loop
    print("quick task done")

async def main():
    # This does NOT run concurrently — cpu_hog blocks everything:
    await asyncio.gather(cpu_hog(), quick_task())
    # quick_task only gets to run AFTER cpu_hog finishes

# Fix: run CPU-bound work in a thread/process pool executor
async def main_fixed():
    loop = asyncio.get_event_loop()
    cpu_result = await loop.run_in_executor(None, sum, range(10**8))
    await quick_task()
```

---

## Trap 3: "await coroutine() and asyncio.create_task() are equivalent"

**What most people say:**
"I can use `await coro()` or `create_task(coro())`  — both run the coroutine."

**Correct answer:**
`await coro()` runs the coroutine *immediately and waits for it to complete* before
proceeding. `create_task(coro())` *schedules* the coroutine to run concurrently —
it runs while other `await` points yield control. `create_task` is what gives you
actual concurrent execution within a single event loop.

```python
import asyncio, time

async def fetch(n):
    await asyncio.sleep(1)  # simulate 1s I/O
    return n

async def sequential():
    # Runs one at a time — takes 3 seconds total
    a = await fetch(1)
    b = await fetch(2)
    c = await fetch(3)
    return [a, b, c]

async def concurrent():
    # All scheduled immediately — takes ~1 second total
    task_a = asyncio.create_task(fetch(1))
    task_b = asyncio.create_task(fetch(2))
    task_c = asyncio.create_task(fetch(3))
    return [await task_a, await task_b, await task_c]

# asyncio.gather() shorthand for create_task + gather results:
async def concurrent_gather():
    return await asyncio.gather(fetch(1), fetch(2), fetch(3))  # ~1s
```

---

## Trap 4: "multiprocessing is always faster than threading"

**What most people say:**
"Since multiprocessing bypasses the GIL, it's always the better choice."

**Correct answer:**
Processes have significant overhead: each process starts a fresh Python interpreter
(startup cost: 50-200ms), uses separate memory (no sharing), and communicating between
processes requires pickling/serialisation. For tasks where the work is small or data
transfer is large relative to computation, multiprocessing can actually be *slower*
than threading.

```python
import multiprocessing as mp
import threading
import time

# CPU-bound: multiprocessing wins (bypass GIL)
def cpu_task(n): return sum(range(n))

# I/O-bound: threading wins (lower overhead, GIL released during I/O)
def io_task(url): import requests; return requests.get(url).status_code

# Overhead comparison:
start = time.time()
processes = [mp.Process(target=cpu_task, args=(10,)) for _ in range(4)]
for p in processes: p.start()
for p in processes: p.join()
print(f"Processes (trivial work): {time.time()-start:.3f}s")  # ~0.4-0.8s overhead!

start = time.time()
threads = [threading.Thread(target=cpu_task, args=(10,)) for _ in range(4)]
for t in threads: t.start()
for t in threads: t.join()
print(f"Threads (trivial work): {time.time()-start:.3f}s")    # ~0.001s overhead
```

---

## Trap 5: "asyncio.gather() returns results in the order tasks complete"

**What most people say:**
"`gather()` returns results in the order tasks finish, not the order they were given."

**Correct answer:**
`asyncio.gather()` returns results in the ORDER THEY WERE PASSED IN, not in the order
they complete. This makes it easy to match inputs with results. If you need completion
order, use `asyncio.as_completed()`.

```python
import asyncio, random

async def task(name, delay):
    await asyncio.sleep(delay)
    print(f"{name} completed after {delay}s")
    return name

async def main():
    # Tasks complete in order: C, A, B (fastest to slowest)
    results = await asyncio.gather(
        task("A", 0.3),
        task("B", 0.5),
        task("C", 0.1),
    )
    print(results)   # ['A', 'B', 'C'] — INPUT order, not completion order

    # For completion order, use as_completed:
    import asyncio
    coros = [task("A", 0.3), task("B", 0.5), task("C", 0.1)]
    for future in asyncio.as_completed(coros):
        result = await future
        print(f"Completed: {result}")   # C, A, B
```

---

## Trap 6: "A daemon thread is the same as a background thread"

**What most people say:**
"Daemon threads run in the background, so they're just regular threads with less
priority."

**Correct answer:**
A daemon thread is a thread that is automatically killed when the main thread exits,
without waiting for it to finish. A non-daemon thread is waited upon by the Python
interpreter at shutdown — the program stays alive until all non-daemon threads finish.
This has nothing to do with priority or scheduling.

```python
import threading, time

def background_work():
    while True:
        time.sleep(0.1)
        print("still running...")

# Non-daemon: program waits for this thread
t1 = threading.Thread(target=background_work)
t1.daemon = False
t1.start()
# Program will NEVER exit — waits for t1 which loops forever

# Daemon: program exits without waiting
t2 = threading.Thread(target=background_work, daemon=True)
t2.start()
# Program can exit; t2 is killed automatically at shutdown
# t2 might be in the middle of an operation — no cleanup!
```

---

## Trap 7: "You can use threading.Lock() to protect against all race conditions"

**What most people say:**
"Just lock the critical section and you're safe."

**Correct answer:**
A `Lock` protects against data races on a single lock, but: (1) taking multiple locks
in different order in different threads causes deadlock, (2) `Lock` is not reentrant
— the same thread trying to acquire it twice will deadlock (use `RLock` for that),
(3) locks don't help with higher-level logical race conditions (e.g., check-then-act).

```python
import threading

lock = threading.Lock()

# DEADLOCK: two threads, two locks, opposite order
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_1():
    with lock_a:
        with lock_b:   # waits for lock_b, which thread_2 holds
            pass

def thread_2():
    with lock_b:
        with lock_a:   # waits for lock_a, which thread_1 holds
            pass
# Both threads wait forever — classic deadlock

# NON-REENTRANT trap:
lock = threading.Lock()
def outer():
    with lock:
        inner()    # inner also tries to acquire lock → DEADLOCK

def inner():
    with lock:   # same thread trying to re-acquire!
        pass

# Fix: use RLock (reentrant lock)
rlock = threading.RLock()
def outer_safe():
    with rlock:
        inner_safe()

def inner_safe():
    with rlock:   # same thread can acquire RLock multiple times
        pass
```

---

## Trap 8: "asyncio programs can use regular blocking libraries"

**What most people say:**
"I can use `requests` in my asyncio code by just making the function async."

**Correct answer:**
Making a function `async def` does NOT make its internals async. If you call a
blocking library (like `requests`) inside an `async def`, the ENTIRE event loop
blocks for the duration of that call — no other coroutines can run, completely
defeating the purpose of async.

```python
import asyncio, requests

# WRONG — blocks the event loop for the entire request duration:
async def bad_fetch(url):
    response = requests.get(url)  # BLOCKING — freezes all other coroutines
    return response.json()

# CORRECT option 1 — use async library (aiohttp):
import aiohttp
async def good_fetch(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()

# CORRECT option 2 — run blocking call in executor:
async def executor_fetch(url):
    loop = asyncio.get_running_loop()
    response = await loop.run_in_executor(
        None,                    # None = default ThreadPoolExecutor
        requests.get, url
    )
    return response.json()
```

---

## Trap 9: "ProcessPoolExecutor can pickle any function"

**What most people say:**
"ProcessPoolExecutor works just like ThreadPoolExecutor but uses processes."

**Correct answer:**
ProcessPoolExecutor serialises work units using `pickle` to send them to worker
processes. Lambda functions, closures, and locally-defined functions are NOT picklable.
Also, if the function is defined in `__main__`, the worker processes must be able to
import it — on Windows this requires the `if __name__ == "__main__":` guard to prevent
infinite process spawning.

```python
from concurrent.futures import ProcessPoolExecutor

# NOT picklable — will fail:
executor = ProcessPoolExecutor()
f = lambda x: x * 2
executor.submit(f, 5)   # PicklingError

# NOT picklable — locally defined inside a function:
def create_processor():
    def local_func(x): return x * 2   # closure/local — not picklable
    with ProcessPoolExecutor() as ex:
        ex.submit(local_func, 5)   # PicklingError

# CORRECT — module-level function:
def double(x):       # defined at module level — picklable
    return x * 2

if __name__ == "__main__":   # required on Windows
    with ProcessPoolExecutor() as ex:
        results = list(ex.map(double, range(10)))
```

---

## Trap 10: "asyncio.Queue is thread-safe"

**What most people say:**
"asyncio.Queue is a queue, so I can use it from multiple threads."

**Correct answer:**
`asyncio.Queue` is NOT thread-safe. It is designed for use within a single asyncio
event loop from coroutines only. Using it from threads will corrupt its internal state.
For cross-thread communication with asyncio, use `asyncio.Queue` with
`loop.call_soon_threadsafe()`, or use `janus.Queue` (third-party) for a truly hybrid
queue. For pure threading, use `queue.Queue`.

```python
import asyncio, queue, threading

# Thread-safe synchronous queue:
sync_q = queue.Queue()
# Use from threads with queue.put() and queue.get()

# Async queue — coroutines only:
async def async_producer(q: asyncio.Queue):
    await q.put("item")          # must be awaited

async def async_consumer(q: asyncio.Queue):
    item = await q.get()         # must be awaited
    q.task_done()

# Bridging threads and asyncio:
loop = asyncio.new_event_loop()
async_q = asyncio.Queue()

def thread_put(item):
    # Thread-safe way to put item into asyncio queue from a thread:
    asyncio.run_coroutine_threadsafe(async_q.put(item), loop)
```
