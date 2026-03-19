# Chapter 07: Performance & Profiling — Diagram Explanations

## Diagram 1: Profiling Funnel — From Macro to Micro

```
PROFILING METHODOLOGY (narrow the search space)

  ┌───────────────────────────────────────────────┐
  │  STEP 1: Is it slow?                          │
  │  time.perf_counter() or /usr/bin/time         │
  │  "Total run: 45 seconds"                      │
  └───────────────────────┬───────────────────────┘
                          │ Yes, it's slow
                          ▼
  ┌───────────────────────────────────────────────┐
  │  STEP 2: Which function?                      │
  │  cProfile + pstats                            │
  │  "process_rows() accounts for 38 of 45s"      │
  └───────────────────────┬───────────────────────┘
                          │ Found hot function
                          ▼
  ┌───────────────────────────────────────────────┐
  │  STEP 3: Which line?                          │
  │  line_profiler (@profile decorator)           │
  │  "Line 47: regex.match() takes 30s"           │
  └───────────────────────┬───────────────────────┘
                          │ Found hot line
                          ▼
  ┌───────────────────────────────────────────────┐
  │  STEP 4: Fix and verify                       │
  │  timeit.repeat() before vs after              │
  │  "Speedup: 8.3x"                              │
  └───────────────────────┬───────────────────────┘
                          │ Verify no regression
                          ▼
  ┌───────────────────────────────────────────────┐
  │  STEP 5: Memory?                              │
  │  tracemalloc.take_snapshot()                  │
  │  "Line 47: +200 MB from compiled patterns"    │
  └───────────────────────────────────────────────┘
```

---

## Diagram 2: `lru_cache` Internals — LRU Eviction

```
CACHE STATE (maxsize=4)

After calls: f(1), f(2), f(3), f(4)

  ┌────────────────────────────────────┐
  │  LRU Cache (Most → Least Recent)  │
  │                                   │
  │  [4] → [3] → [2] → [1]           │
  │  MRU                        LRU   │
  └────────────────────────────────────┘
  Cache full. Next new call evicts [1].

Call f(5):  (cache miss, new entry)
  ┌────────────────────────────────────┐
  │  [5] → [4] → [3] → [2]           │
  │  [1] EVICTED                      │
  └────────────────────────────────────┘

Call f(2):  (cache HIT, promoted to MRU)
  ┌────────────────────────────────────┐
  │  [2] → [5] → [4] → [3]           │
  │  [2] moved from position 4 to 1   │
  └────────────────────────────────────┘

CACHE STATISTICS:
  hits     = times returned from cache (no function call)
  misses   = times function was actually called
  maxsize  = capacity
  currsize = current entries stored

hit_rate = hits / (hits + misses)
Target: hit_rate > 0.80 for cache to be worthwhile
```

---

## Diagram 3: Memory Layout — `__dict__` vs `__slots__`

```
INSTANCE WITH __dict__ (default)
──────────────────────────────────────────────────────

Python Heap:
  ┌─────────────────────────────┐
  │  Point instance             │  48 bytes
  │  ┌─────────────────────┐   │
  │  │ ob_refcnt: 1        │   │
  │  │ ob_type: → Point    │   │
  │  │ __dict__: → ────────┼───┼──► dict object: 232 bytes
  │  └─────────────────────┘   │      { "x": 1.0, "y": 2.0, "z": 3.0 }
  └─────────────────────────────┘  │
                                    ├──► float 1.0: 24 bytes
  Total per instance: ~304 bytes    ├──► float 2.0: 24 bytes
                                    └──► float 3.0: 24 bytes


INSTANCE WITH __slots__
──────────────────────────────────────────────────────

Python Heap:
  ┌──────────────────────────────────────┐
  │  PointSlots instance          72 bytes │
  │  ┌──────────────────────────┐        │
  │  │ ob_refcnt: 1             │        │
  │  │ ob_type: → PointSlots   │        │
  │  │ slot[0] x: → ───────────┼────────┼──► float 1.0: 24 bytes
  │  │ slot[1] y: → ───────────┼────────┼──► float 2.0: 24 bytes
  │  │ slot[2] z: → ───────────┼────────┼──► float 3.0: 24 bytes
  │  └──────────────────────────┘        │
  └──────────────────────────────────────┘

  Total per instance: ~144 bytes (no __dict__ overhead!)
  Savings: ~160 bytes per instance
  At 1,000,000 instances: saves ~160 MB
```

---

## Diagram 4: Generator Pipeline vs Eager List Pipeline

```
EAGER (list) PIPELINE — all data loaded at each stage

  File (10 GB)
       │ read ALL
       ▼
  ┌─────────────────┐
  │ all_lines       │  10 GB in memory
  │ [line, line...] │
  └────────┬────────┘
           │ process ALL
           ▼
  ┌─────────────────┐
  │ parsed_rows     │  10 GB in memory (different shape)
  │ [row, row, ...] │
  └────────┬────────┘
           │ filter ALL
           ▼
  ┌─────────────────┐
  │ errors          │  maybe 1 MB
  │ [err, err, ...] │
  └─────────────────┘
  Peak memory: ~20 GB (two intermediate lists)


LAZY (generator) PIPELINE — one item flows through at a time

  File (10 GB)
       │ read 1 line
       ▼
  ┌─────────────────┐
  │ line generator  │  ~112 bytes (generator object)
  └────────┬────────┘
           │ parse 1 item
           ▼
  ┌─────────────────┐
  │ row generator   │  ~112 bytes
  └────────┬────────┘
           │ filter 1 item
           ▼
  ┌─────────────────┐  ← only error items accumulate
  │ errors list     │  1 MB
  └─────────────────┘
  Peak memory: ~1 MB + one row in flight
```

---

## Diagram 5: `tracemalloc` Snapshot Comparison

```
tracemalloc.start()
     │
     ▼
snapshot1 = take_snapshot()  ← BEFORE: baseline
     │
     │  [run suspected leaky code]
     │
     ▼
snapshot2 = take_snapshot()  ← AFTER
     │
     ▼
top_stats = snapshot2.compare_to(snapshot1, "lineno")

OUTPUT:
─────────────────────────────────────────────────────────────
File                  Line  Size Diff    Count  Count Diff
─────────────────────────────────────────────────────────────
myapp/cache.py          88  +52.4 MiB   +524k  allocs  ← LEAK
myapp/parser.py         31  +1.2 MiB    +12k   allocs  normal
myapp/models.py         15  +0.3 MiB    +3k    allocs  normal
─────────────────────────────────────────────────────────────

Investigate myapp/cache.py line 88:

  88: self._event_log.append(Event(timestamp, data))
  ↑
  The list is never cleared. Each request appends a new Event.
  Fix: use a deque(maxlen=1000) or clear periodically.
```

---

## Diagram 6: `LOAD_FAST` vs `LOAD_GLOBAL` — Bytecode Comparison

```
FUNCTION: uses global math.sqrt
───────────────────────────────────────────────────────────

def slow(x):
    return math.sqrt(x)

BYTECODE (dis.dis output):
  LOAD_GLOBAL  0 (math)        # look up "math" in module.__dict__
  LOAD_ATTR    1 (sqrt)        # look up "sqrt" in math.__dict__
  LOAD_FAST    0 (x)           # direct array index [0]
  CALL         1               # call sqrt(x)
  RETURN_VALUE

COST per iteration:
  LOAD_GLOBAL → hash("math") → bucket lookup → found → PyObject*
  LOAD_ATTR   → hash("sqrt") → bucket lookup → found → PyObject*
  Total: 2 hash table lookups per call


FUNCTION: caches sqrt as local
───────────────────────────────────────────────────────────

def fast(data):
    sqrt = math.sqrt    # LOAD_GLOBAL + LOAD_ATTR done ONCE
    return [sqrt(x) for x in data]

BYTECODE inside the list comprehension:
  LOAD_FAST    0 (sqrt)        # direct array index [0] — O(1) C array access
  LOAD_FAST    1 (x)           # direct array index [1]
  CALL         1
  (no hash lookups per iteration)


SPEEDUP IN TIGHT LOOP (1M iterations):
  slow():  ~0.45s  (2 hash lookups × 1M = 2M lookups)
  fast():  ~0.28s  (0 hash lookups after setup)
  Speedup: ~1.6x
```
