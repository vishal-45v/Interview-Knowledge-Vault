# Chapter 07: Performance & Profiling — Structured Answers

## Q1: How do you profile a Python application end-to-end?

**Answer:**

Profiling is a three-step process: measure, identify, optimize, then verify the improvement.

**Step 1: Macro timing with `timeit` or wall-clock time**
```python
import time

start = time.perf_counter()
result = expensive_function()
elapsed = time.perf_counter() - start
print(f"Elapsed: {elapsed:.4f}s")
```

**Step 2: Function-level profiling with `cProfile`**
```python
import cProfile
import pstats
import io

profiler = cProfile.Profile()
profiler.enable()

run_application()

profiler.disable()

stream = io.StringIO()
stats = pstats.Stats(profiler, stream=stream)
stats.sort_stats("cumulative")      # sort by cumulative time
stats.print_stats(20)               # top 20 functions
print(stream.getvalue())
```

Output columns:
- `ncalls`: number of calls (5/3 means 5 total, 3 primitive/non-recursive)
- `tottime`: time in the function itself, excluding callees
- `cumtime`: time in the function + all functions it called
- `percall`: tottime/ncalls and cumtime/ncalls

**Step 3: Line-level profiling with `line_profiler`**

Once `cProfile` identifies a hot function, use `line_profiler` to see which *lines* within that function are slow:

```python
# Install: pip install line_profiler
# Decorate target function:
from line_profiler import LineProfiler

def slow_transform(data):
    result = []
    for row in data:
        val = complex_parse(row)        # ← cProfile showed this is hot
        val = normalize(val)
        result.append(val)
    return result

lp = LineProfiler(slow_transform, complex_parse)
lp.runcall(slow_transform, my_data)
lp.print_stats()
```

**Step 4: Memory profiling with `tracemalloc`**
```python
import tracemalloc

tracemalloc.start()

run_application()

snapshot = tracemalloc.take_snapshot()
top_stats = snapshot.statistics("lineno")

for stat in top_stats[:10]:
    print(stat)
# Output: mymodule.py:47: size=10.5 MiB, count=105000, average=105 B
```

**Step 5: Verify the fix with `timeit`**
```python
import timeit

before = timeit.timeit("old_function(data)", globals=globals(), number=1000)
after  = timeit.timeit("new_function(data)", globals=globals(), number=1000)
print(f"Speedup: {before/after:.1f}x")
```

---

## Q2: Explain `lru_cache`, when it helps, when it hurts, and the thread-safety story

**Answer:**

`functools.lru_cache` (Least Recently Used cache) memoizes a function's return value keyed by its arguments. When called with the same arguments again, the cached value is returned immediately without re-executing the function.

```python
from functools import lru_cache
import time

@lru_cache(maxsize=128)
def fetch_user_permissions(user_id: int) -> frozenset:
    # Expensive DB query
    time.sleep(0.1)
    return frozenset(["read", "write"])

# First call: 0.1 seconds
fetch_user_permissions(42)

# Subsequent calls: microseconds (cache hit)
fetch_user_permissions(42)

# Inspect cache statistics
print(fetch_user_permissions.cache_info())
# CacheInfo(hits=1, misses=1, maxsize=128, currsize=1)

# Clear cache (e.g., when DB is updated)
fetch_user_permissions.cache_clear()
```

**`maxsize` tradeoffs:**
- `maxsize=None` (or `functools.cache`): unlimited cache, never evicts — memory grows without bound
- `maxsize=128`: evicts least recently used entries when full — bounded memory
- `maxsize=1`: only caches the most recent call — useful for "if same args as last time" pattern

**When it hurts:**
1. Function arguments are not hashable (lists, dicts) → `TypeError`
2. Function has side effects (logging, DB writes) → side effects only happen on cache miss
3. Return values are large objects → memory held indefinitely
4. Function result changes over time (current time, DB state) → stale cache

**Thread safety:** `lru_cache` is thread-safe for reads and writes in CPython (the GIL provides protection). However, it is possible for multiple threads to simultaneously call the function on a cache miss (the GIL is released during the function's execution), causing duplicate work. This is "stampede" behavior, not a correctness bug in most cases.

**Python 3.9+ simplification:**
```python
from functools import cache  # equivalent to lru_cache(maxsize=None)

@cache
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

---

## Q3: How do `__slots__` reduce memory and what do you lose?

**Answer:**

By default, every Python instance has a `__dict__` — a hash table that stores its attribute names and values. For a class with 3 attributes, this dict costs ~200-240 bytes per instance regardless of the actual values. When you create 100,000 instances, the dict overhead alone is 20-24 MB.

`__slots__` replaces the per-instance `__dict__` with a fixed-size C array of pointers (one per slot). Attribute access becomes a direct offset lookup instead of a hash table lookup.

```python
import sys

class Point:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class PointSlots:
    __slots__ = ("x", "y", "z")
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

p1 = Point(1.0, 2.0, 3.0)
p2 = PointSlots(1.0, 2.0, 3.0)

print(sys.getsizeof(p1))              # 48 bytes (object header)
print(sys.getsizeof(p1.__dict__))     # 232 bytes (the dict)
# Total: ~280 bytes

print(sys.getsizeof(p2))              # 72 bytes (object + 3 slots)
# Total: 72 bytes — ~4x smaller

# With 1,000,000 instances:
# Point:      ~280 MB
# PointSlots: ~72 MB
```

**What you lose:**
1. Cannot add arbitrary attributes: `p2.color = "red"` → `AttributeError`
2. `__dict__` is absent (breaks some generic serializers, `vars()` fails)
3. `__weakref__` support must be explicitly included: `__slots__ = ("x", "y", "__weakref__")`
4. Multiple inheritance requires all bases to define `__slots__` OR one of them to have `__dict__`
5. Some metaprogramming (dynamically setting attributes, `__getattr__`-based patterns) breaks

**When to use:** Classes that are instantiated in very large quantities (data rows, game entities, event records) with a known, fixed set of attributes.

---

## Q4: Explain lazy evaluation and generators vs lists for performance

**Answer:**

**Lazy evaluation** means values are computed only when requested, not all upfront. Python generators implement lazy evaluation: the function body is paused at each `yield`, and execution resumes only when the next value is requested via `next()` or a `for` loop.

```python
import sys

# Eager — entire sequence materialized in memory immediately
eager = [x**2 for x in range(10_000_000)]
print(sys.getsizeof(eager))     # ~80,000,056 bytes (~76 MB)

# Lazy — only the generator object exists; values computed on demand
lazy = (x**2 for x in range(10_000_000))
print(sys.getsizeof(lazy))      # 112 bytes — just the generator object!

# Memory usage during iteration:
# eager: all 76 MB loaded upfront
# lazy: only the current x is in memory at any time
```

**When generators win:**
- Data does not fit in memory (streaming file parsing, database result sets)
- Early termination is common (searching for first match, `itertools.takewhile`)
- Pipeline of transformations (chained generators compose without intermediate lists)

```python
# Pipeline: no intermediate lists created
def parse_lines(filename):
    with open(filename) as f:
        for line in f:          # lazy file read
            yield line.strip()

def filter_errors(lines):
    for line in lines:
        if "ERROR" in line:     # lazy filter
            yield line

def extract_timestamps(lines):
    for line in lines:
        yield line.split()[0]   # lazy transform

# Each line flows through the whole pipeline before the next is read
timestamps = list(extract_timestamps(filter_errors(parse_lines("app.log"))))
```

**When lists win:**
- Multiple iterations over the same data (a generator is exhausted after one pass)
- Random access (`data[500]` — generators don't support indexing)
- Built-in functions like `sum()`, `max()` on small sequences are faster with list input
- When you need `len()` (generators have no length)

**Generator overhead:**
Each `next()` call involves a Python frame context switch. For very small, tight loops, the overhead per element can make generators 20-50% slower than list comprehensions on equivalent data. Profile before assuming generators are always faster.

---

## Q5: How do you optimize a hot loop in Python?

**Answer:**

Five concrete techniques, in order of impact:

**1. Move invariant operations out of the loop:**
```python
# BAD: len(data) computed each iteration
for i in range(len(data)):
    process(data[i])

# GOOD: computed once
n = len(data)
for i in range(n):
    process(data[i])
```

**2. Cache global and attribute lookups as locals:**
```python
# BAD: global lookup + attribute lookup on every iteration
import math
def compute_distances(points):
    return [math.sqrt(p[0]**2 + p[1]**2) for p in points]

# GOOD: bind to local variables
def compute_distances_fast(points):
    sqrt = math.sqrt       # LOAD_FAST instead of LOAD_GLOBAL + LOAD_ATTR
    return [sqrt(p[0]**2 + p[1]**2) for p in points]
```

**3. Use list comprehensions and map/filter over explicit for-loops:**
```python
# map() and list comprehensions run the inner loop in C
values = [x * 2 for x in data]       # faster than explicit for
values = list(map(lambda x: x*2, data))  # fastest for simple transforms
```

**4. Avoid string concatenation — use `join`:**
```python
# BAD: O(n²) — creates new string object each time
result = ""
for word in words:
    result += word + " "

# GOOD: O(n) — builds list then joins once
result = " ".join(words)
```

**5. Use NumPy for numerical bulk operations:**
```python
import numpy as np

data = list(range(1_000_000))
np_data = np.array(data)

# Python loop: ~0.5s
result_py = [x ** 2 for x in data]

# NumPy vectorized: ~0.002s — 250x faster
result_np = np_data ** 2
```

NumPy operations are implemented in C and apply the operation to the entire array without a Python-level loop. The GIL is often released during NumPy operations, enabling true parallel execution.
