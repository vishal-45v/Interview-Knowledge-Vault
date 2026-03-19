# Chapter 07: Performance & Profiling — Analogy Explanations

## Analogy 1: cProfile as a Time-Tracking App for Employees

**The Story:**
A consulting firm uses time-tracking software. Every employee logs which task they're working on and for how long. At the end of the month, the manager sees: Alice spent 40 hours on "client report" but only 2 hours of that was her own writing — the rest was waiting for colleagues to send her data (cumulative vs total time). Bob spent 200 hours on "email replies" (high calls × low per-call time). The manager can now make data-driven decisions: don't optimize Alice's writing speed, fix the data pipeline; don't make Bob type faster, use email templates.

**The Python Connection:**
`cProfile` instruments every function call. `tottime` is how long the employee themselves worked. `cumtime` is the total project time including waiting. `ncalls` is how many times they were interrupted. A high `tottime` with high `ncalls` and low per-call time means "called too often." A high `cumtime` with low `tottime` means "calling something expensive."

```python
import cProfile, pstats, io

pr = cProfile.Profile()
pr.enable()
process_reports()
pr.disable()

buf = io.StringIO()
pstats.Stats(pr, stream=buf).sort_stats("cumulative").print_stats(15)
# Look for: high ncalls × moderate tottime = "called too often"
# Look for: high cumtime, low tottime = "calls something slow"
```

---

## Analogy 2: `lru_cache` as a Waiter's Notepad

**The Story:**
A busy restaurant waiter serves a table of regulars. The first time someone orders "the usual," the waiter goes to the kitchen, waits for the chef, and returns with the dish. He writes the order down on his notepad. The next time that person orders "the usual," the waiter skips the kitchen entirely — he already has it noted. But the notepad only has 128 lines. When it's full, the waiter erases the oldest entry to make room. If a menu item changes (the chef updates the recipe), the waiter's note is stale until it is erased.

**The Python Connection:**
The notepad is the LRU cache. The kitchen trip is the expensive function call. The notepad capacity is `maxsize`. Erasing the oldest entry is the LRU eviction policy. A stale note is the cache invalidation problem.

```python
from functools import lru_cache

@lru_cache(maxsize=128)   # 128-line notepad
def get_user_profile(user_id: int) -> dict:
    return db.query("SELECT * FROM users WHERE id = ?", user_id)  # kitchen trip

# Same user_id → notepad hit, no DB trip
profile = get_user_profile(42)
profile = get_user_profile(42)   # instant

# If user updates their profile in the DB, clear the stale note:
get_user_profile.cache_clear()
```

---

## Analogy 3: `__slots__` as a Hotel Registration Card vs. a Blank Notepad

**The Story:**
A large hotel chain has two types of guest check-in. The old system gave every guest a blank notepad — they could write whatever they wanted (room number, preferences, car plate, loyalty tier, dietary restrictions, etc.). Flexible, but every guest got the full notepad (200 pages) even if they only wrote three lines. The new system uses a pre-printed registration card with specific fields: Name, Room Number, Check-in Date. It fits on one index card. Multiplied across 100,000 annual guests, the storage savings are enormous — but you can't write "has a pet iguana" anywhere because there's no field for it.

**The Python Connection:**
The blank notepad is `__dict__`. The registration card is `__slots__`. Both hold the same core data, but `__slots__` is dramatically smaller when the attribute set is known and fixed.

```python
class GuestWithDict:
    def __init__(self, name, room, checkin):
        self.name = name
        self.room = room
        self.checkin = checkin
        # Can add: self.pet = "iguana" — flexible!

class GuestWithSlots:
    __slots__ = ("name", "room", "checkin")
    def __init__(self, name, room, checkin):
        self.name = name
        self.room = room
        self.checkin = checkin
        # self.pet = "iguana"  → AttributeError — no field on the card
```

---

## Analogy 4: Generators as a Library Book Request System

**The Story:**
A huge university library has 10 million books. A researcher could request "give me all 10 million books" — the staff would spend weeks hauling them out, the research room would be unusable, and most books would never be opened. Instead, the researcher says "I need the first book on topic X, and when I'm done with it, I'll ask for the next one." The librarian fetches one book at a time, from wherever it lives in the stacks, only when the researcher is ready.

**The Python Connection:**
Creating a list is like hauling all 10 million books upfront. A generator is the on-demand request system. The `yield` statement is the librarian handing over the current book and waiting.

```python
# Librarian hauling everything upfront (list comprehension)
all_records = [parse(line) for line in open("10gb_log.txt")]  # OOM risk!

# On-demand: one record at a time
def stream_records(filename):
    with open(filename) as f:
        for line in f:
            yield parse(line)   # hand over this book, wait for next request

for record in stream_records("10gb_log.txt"):
    if is_error(record):
        handle_error(record)
    # Python never holds more than one record in memory at a time
```

---

## Analogy 5: `tracemalloc` as a Carbon Footprint Tracker

**The Story:**
A company wants to reduce its environmental footprint. A carbon footprint tracker instruments every department: "Marketing spent 2 tons on flights, Engineering spent 5 tons on servers, the CEO's office spent 0.5 tons on printing." It can also take a "snapshot before and after" and show you what changed: "After hiring 10 new engineers, server carbon went up by 3 tons — that's where the increase is."

**The Python Connection:**
`tracemalloc` instruments every Python memory allocation and tells you which source line allocated what. Take a snapshot before and after a suspected leak, then compare.

```python
import tracemalloc

tracemalloc.start()
snapshot1 = tracemalloc.take_snapshot()   # before

run_suspected_leaky_code()

snapshot2 = tracemalloc.take_snapshot()   # after
top_stats = snapshot2.compare_to(snapshot1, "lineno")

print("=== Top memory increases ===")
for stat in top_stats[:5]:
    print(stat)
# mymodule.py:88: +5.2 MiB (105,000 allocations) ← the leak
```

---

## Analogy 6: Global vs Local Variable Lookup as a Filing Cabinet vs a Desktop

**The Story:**
An accountant has two places to find documents. Global storage is the filing cabinet across the room: they must walk over, find the right drawer, search alphabetically, and pull the folder — multiple steps each time. Local storage is the documents currently on their desk: they look down and grab it — one motion. If they are doing the same calculation 100,000 times and it always uses the same formula sheet, it makes sense to put that sheet on the desk at the start of the day rather than walking to the filing cabinet 100,000 times.

**The Python Connection:**
`LOAD_GLOBAL` walks to the module's `__dict__` filing cabinet. `LOAD_FAST` grabs from the function's local variable array — a direct index into a C array.

```python
import math

# Walking to the cabinet every iteration
def slow(data):
    return [math.sqrt(x) for x in data]   # LOAD_GLOBAL math, LOAD_ATTR sqrt

# Put it on the desk once
def fast(data):
    sqrt = math.sqrt    # one cabinet trip, then it's on the desk
    return [sqrt(x) for x in data]        # LOAD_FAST sqrt — O(1) array index

# For 10M iterations, this difference is measurable
```

---

## Analogy 7: `weakref` as a Library Catalog Entry for a Book That Might Be Checked Out

**The Story:**
A library catalog has an entry for every book. Some entries are "reserved" — the library guarantees the book stays on the shelf (strong reference). Others are "open loan" — the entry exists in the catalog, but if a student checks out the last copy, the catalog entry cannot prevent the book from leaving. When you look up an open-loan entry and the book is gone, the catalog returns "not available." It doesn't crash; it just tells you the truth.

**The Python Connection:**
A strong Python reference keeps an object alive. A `weakref` is the catalog entry — it knows where the object is, but does not prevent garbage collection. `weakref()` returns `None` if the object has been collected.

```python
import weakref

class DatabaseConnection:
    def __init__(self, url): self.url = url

# Cache connections without preventing GC when the connection goes out of scope elsewhere
connection_cache = weakref.WeakValueDictionary()

conn = DatabaseConnection("postgresql://localhost/mydb")
connection_cache["main_db"] = conn   # weak reference in the cache

print(connection_cache.get("main_db"))  # <DatabaseConnection ...>

del conn      # only strong reference removed
# GC is now free to collect the DatabaseConnection
# connection_cache["main_db"] automatically removed — no stale entries!
```

---

## Analogy 8: `timeit` as a Race with a Pace Car

**The Story:**
Motor racing teams test their car's performance on a closed track, not public roads. They run 10 laps rather than 1 to average out variability. They use a pace car to ensure consistent conditions. The number they care about is the *best lap time*, not the average (because slow laps represent traffic or safety-car interruptions, not the car's true speed). The test tells them nothing about how the car performs in a rain race on a street circuit — it's a controlled comparison.

**The Python Connection:**
`timeit` is the closed track. It runs the code N×M times (number × repeat), disables GC (the pace car — consistent conditions), and you take the minimum time (best lap). Use it for relative comparison between implementations, not as an absolute production benchmark.

```python
import timeit

# 5 runs of 10,000 iterations each → take the minimum
results = timeit.repeat(
    stmt="result = ','.join(str(i) for i in range(1000))",
    repeat=5,
    number=10_000
)
best_per_call = min(results) / 10_000 * 1e6
print(f"Best: {best_per_call:.2f} µs per call")

# Compare two implementations
old = timeit.timeit("old_fn(data)", globals=globals(), number=10_000)
new = timeit.timeit("new_fn(data)", globals=globals(), number=10_000)
print(f"Speedup: {old/new:.2f}x")
```
