# Chapter 07: Performance & Profiling — Follow-Up Traps

## Trap 1: "cProfile shows which function takes the most time"

**What most people say:** "I look for the function with the highest `tottime` — that's my bottleneck."

**Correct answer:** `tottime` (total time, excluding time in called functions) tells you where time is *spent*, not necessarily where your fix should go. A function with high `tottime` might be called 10 million times doing trivial work — the fix is to call it less, not to optimize the function body. Always cross-reference `tottime` with `ncalls`. A function called 1,000,000 times at 0.0001 seconds each is a different problem than one called once at 5 seconds. Also, `tottime` excludes callees, so a high `cumtime` with low `tottime` tells you the function itself is fast but calls something slow.

```python
import cProfile, pstats, io

pr = cProfile.Profile()
pr.enable()
your_function()
pr.disable()

s = io.StringIO()
ps = pstats.Stats(pr, stream=s).sort_stats("cumulative")
ps.print_stats(20)   # top 20 by cumulative time
print(s.getvalue())
```

---

## Trap 2: "timeit gives me accurate real-world performance numbers"

**What most people say:** "I ran timeit and got 50 microseconds, so my function takes 50 microseconds."

**Correct answer:** `timeit` disables garbage collection by default (to reduce noise), runs in a tight loop, and measures CPU time in a warm cache state. Real-world performance differs due to: (1) cold caches — your hot path may be cold the first time; (2) GC pauses — disabling GC in `timeit` hides allocation costs; (3) branch predictor and CPU cache effects that differ in production. `timeit` is useful for *relative* comparisons between implementations, not for absolute production estimates. Also, `timeit.repeat()` returns a list of times — use `min()`, not `mean()`, because outliers are from noise (OS interrupts), not representative.

```python
import timeit

# repeat returns a list of 5 runs of 1000 iterations each
results = timeit.repeat("','.join(str(n) for n in range(100))", repeat=5, number=1000)
print(f"Best: {min(results)/1000*1e6:.2f} µs")  # use min, not mean
```

---

## Trap 3: "lru_cache always improves performance"

**What most people say:** "I'll add @lru_cache to speed up this function."

**Correct answer:** `lru_cache` has overhead: hashing the arguments, dictionary lookup, and storing the result. For functions that are called infrequently or where the computation is cheaper than the cache overhead, `lru_cache` can be *slower*. It also holds strong references to all arguments and return values — for functions that return large objects or take mutable containers (which can't be cached at all), this causes memory growth. Additionally, `lru_cache` is not thread-safe for concurrent modification in all edge cases. Only use it when: (1) the function is pure (same inputs always produce same outputs), (2) arguments are hashable, (3) the function is called repeatedly with the same arguments, and (4) the computation is significantly more expensive than dict lookup.

```python
import functools
import timeit

@functools.lru_cache(maxsize=128)
def cached_add(a, b):
    return a + b

def plain_add(a, b):
    return a + b

# For a trivial operation, lru_cache is SLOWER due to overhead
print(timeit.timeit("cached_add(1, 2)", globals=globals(), number=1_000_000))
print(timeit.timeit("plain_add(1, 2)", globals=globals(), number=1_000_000))
```

---

## Trap 4: "`sys.getsizeof` tells me how much memory my object uses"

**What most people say:** "I checked `sys.getsizeof(my_dict)` and it's only 200 bytes."

**Correct answer:** `sys.getsizeof` returns the *shallow* size — the size of the container object itself, not the objects it references. A list of 1000 strings will show as ~8000 bytes (the list's internal pointer array), even though the strings themselves might use megabytes. To measure deep (recursive) size, you must walk the object graph yourself or use a library like `pympler`.

```python
import sys

data = ["x" * 1000 for _ in range(1000)]
print(sys.getsizeof(data))          # ~8056 bytes — just the list's pointer array!
print(sys.getsizeof(data[0]))       # 1049 bytes — one string

# Deep size (manual)
def deep_size(obj, seen=None):
    if seen is None: seen = set()
    obj_id = id(obj)
    if obj_id in seen: return 0
    seen.add(obj_id)
    size = sys.getsizeof(obj)
    if isinstance(obj, (list, tuple, set, frozenset)):
        size += sum(deep_size(item, seen) for item in obj)
    elif isinstance(obj, dict):
        size += sum(deep_size(k, seen) + deep_size(v, seen) for k, v in obj.items())
    return size

print(deep_size(data))  # ~1,056,000 bytes — true picture
```

---

## Trap 5: "`__slots__` makes everything faster"

**What most people say:** "I'll add `__slots__` to all my classes for better performance."

**Correct answer:** `__slots__` reduces per-instance memory by eliminating the `__dict__` (which is a hash table ~200 bytes per instance). It also gives slightly faster attribute access because slot descriptors use a fixed offset instead of a dict lookup. However: (1) You lose the ability to add arbitrary attributes to instances. (2) Pickling and copying need special handling. (3) Multiple inheritance with `__slots__` requires careful design — if a parent class does not define `__slots__`, the subclass still has a `__dict__`. (4) `__weakref__` must be explicitly included in `__slots__` if you need weak references. The benefit is most significant when you create tens of thousands of instances with a small, fixed set of attributes.

```python
import sys

class WithDict:
    def __init__(self, x, y):
        self.x = x
        self.y = y

class WithSlots:
    __slots__ = ("x", "y")
    def __init__(self, x, y):
        self.x = x
        self.y = y

a = WithDict(1, 2)
b = WithSlots(1, 2)
print(sys.getsizeof(a) + sys.getsizeof(a.__dict__))  # ~360 bytes
print(sys.getsizeof(b))                               # ~56 bytes
# ~6x memory reduction — significant when creating millions of instances
```

---

## Trap 6: "generators are always better than lists"

**What most people say:** "I always use generators instead of lists to save memory."

**Correct answer:** Generators are lazily evaluated — they produce values one at a time without storing the full sequence. This is a huge win when processing large sequences that fit one element in memory. But generators have overhead per `next()` call (context switching back to the generator frame), and they can only be iterated *once*. A list comprehension that fits in memory and is iterated multiple times will be significantly faster than re-generating the sequence. For single-pass, large-data processing: use generators. For repeated access or small data: use lists.

```python
import timeit

# Generator: low memory, but each iteration has overhead
gen_time = timeit.timeit(
    "sum(x*x for x in range(1000))", number=10000
)

# List comprehension: all in memory, but C-speed sum()
list_time = timeit.timeit(
    "sum([x*x for x in range(1000)])", number=10000
)

print(f"Generator: {gen_time:.3f}s")
print(f"List comp:  {list_time:.3f}s")   # often 20-30% faster for small sizes
```

---

## Trap 7: "weakref prevents garbage collection from happening"

**What most people say:** "I used weakref so Python won't garbage collect my object."

**Correct answer:** It is the exact opposite. A `weakref` is a reference that does *not* prevent garbage collection. When the only references to an object are weak references, the garbage collector is free to collect it. The `weakref.ref` callback is then invoked (optionally), and accessing the dead weak reference returns `None`. Use `weakref.WeakValueDictionary` for caches where values should be eligible for GC when nothing else holds a strong reference — this prevents the cache from keeping objects alive indefinitely.

```python
import weakref

class Resource:
    def __init__(self, name):
        self.name = name

r = Resource("db_connection")
weak = weakref.ref(r)

print(weak())          # <Resource object at ...>
del r                  # strong reference gone
print(weak())          # None — object was GC'd
```

---

## Trap 8: "`tracemalloc` finds all memory leaks"

**What most people say:** "I ran tracemalloc and didn't find anything growing, so there's no leak."

**Correct answer:** `tracemalloc` tracks Python object allocations. It will not find leaks in: (1) C extension allocations that bypass Python's memory allocator, (2) references held in data structures that `tracemalloc` itself cannot trace deeply (e.g., objects in a callback list), (3) OS-level resources like file descriptors or sockets (Python objects might be collected but the underlying FD may leak if `__del__` or `close()` is not called). For C extension leaks, use `valgrind` with the Python suppressions file. Also, `tracemalloc` adds overhead (~10-30% slowdown) and should not run in production profiling.

---

## Trap 9: "PyPy is always faster than CPython"

**What most people say:** "PyPy is faster, so I should use it for everything."

**Correct answer:** PyPy's JIT compiler excels at long-running, loop-heavy pure Python code — it can be 10-50x faster in those cases. But PyPy is often slower than CPython for: (1) short scripts (JIT warmup time dominates), (2) code that heavily uses C extensions (NumPy, Pandas, SQLAlchemy — these call into CPython-specific C API), (3) code with lots of I/O where Python execution time is not the bottleneck. PyPy also has historically lagged on Python version support. For scientific computing using NumPy, CPython + vectorized NumPy operations almost always beats PyPy + loops.

---

## Trap 10: "global variable lookup is negligible overhead"

**What most people say:** "It's just a dictionary lookup, it can't matter at scale."

**Correct answer:** In a tight loop called millions of times, the difference between `LOAD_GLOBAL` (dictionary lookup in the module's `__dict__`) and `LOAD_FAST` (direct index into the frame's local variable array, a C array access) is measurable. Caching frequently accessed globals as locals is a standard Python optimization used in CPython's own standard library.

```python
import math

# SLOW: math.sqrt is a LOAD_GLOBAL + LOAD_ATTR on every iteration
def compute_slow(data):
    return [math.sqrt(x) for x in data]

# FAST: sqrt is a LOAD_FAST after the assignment
def compute_fast(data):
    sqrt = math.sqrt         # cache in local variable
    return [sqrt(x) for x in data]

# Also applies to method lookups in loops
def process_slow(items):
    result = []
    for item in items:
        result.append(item * 2)   # LOAD_FAST result, LOAD_ATTR append — per iter

def process_fast(items):
    result = []
    append = result.append    # cache method in local
    for item in items:
        append(item * 2)      # LOAD_FAST append — per iter
```
