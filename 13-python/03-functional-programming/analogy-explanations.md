# Chapter 03 — Functional Programming: Analogy Explanations

---

## Analogy 1: Closures — A Backpack That Carries Its Environment

**The analogy:**
Imagine a chef who is trained in a specific kitchen (the outer function). When the chef
graduates and goes to work at a restaurant (the inner function is returned), they carry
a backpack containing the key ingredients and spices they learned to use in their
training kitchen. Even though the training kitchen is now closed, the chef can still
make those recipes — everything they need is in the backpack.

The backpack is the closure: it contains references to the variables from the outer
scope, preserved as *cell objects*.

**Connected to Python:**
```python
def training_kitchen(secret_spice):
    """The training kitchen — outer function."""

    def chef_cook(dish):
        """The chef function — inner function with a 'backpack'."""
        return f"{dish} seasoned with {secret_spice}"

    return chef_cook   # returns the chef WITH their backpack

italian_chef = training_kitchen("basil and oregano")
french_chef  = training_kitchen("thyme and tarragon")

# Training kitchen is "closed" — but chefs still have their spices:
italian_chef("pasta")   # "pasta seasoned with basil and oregano"
french_chef("duck")     # "duck seasoned with thyme and tarragon"

# Inspect the backpack:
italian_chef.__closure__[0].cell_contents  # "basil and oregano"
```

---

## Analogy 2: Decorators — A Coat Check with Added Services

**The analogy:**
At a fancy restaurant, there is a coat check service. When you arrive (call the
function), the host takes your coat (the original function), tags it (wraps it), and
gives you a ticket. When you need your coat (calling the wrapper), they retrieve it
but also: check your VIP status, log the retrieval time, and add a courtesy wrapper.
You get your coat back but the restaurant has done extra things around the interaction.

The original coat (function) is unchanged. The coat check service (decorator) adds
behaviour before and after without touching the coat itself.

**Connected to Python:**
```python
import functools
import time

def coat_check(func):
    """The coat check service — decorator."""
    @functools.wraps(func)
    def service(*args, **kwargs):
        # Before: log arrival
        print(f"Checking in: {func.__name__}")
        start = time.perf_counter()

        # The actual function call (retrieve the coat)
        result = func(*args, **kwargs)

        # After: log departure time
        elapsed = time.perf_counter() - start
        print(f"Checked out: {func.__name__} ({elapsed:.4f}s)")
        return result
    return service

@coat_check
def fetch_user(user_id: int) -> dict:
    """The original coat — unchanged by decoration."""
    return {"id": user_id, "name": "Alice"}

# The decorated function:
user = fetch_user(42)
# Checking in: fetch_user
# Checked out: fetch_user (0.0001s)
```

---

## Analogy 3: Generators — A Bread Slicer That Works On Demand

**The analogy:**
A traditional baker bakes an entire loaf and slices all of it at once before any
customer arrives (list comprehension — all results materialised). A modern bakery
has a slicer with a special sensor: it only cuts the next slice when a customer
explicitly asks for one. The unsliced loaf waits patiently. No slice is wasted.

If a customer only wants 3 slices and leaves, the other 17 slices were never cut
(and never needed to be). The slicer knows where it stopped and picks up exactly
where it left off for the next customer request.

**Connected to Python:**
```python
def bread_slicer(loaf_size):
    """A generator — slices on demand."""
    for slice_number in range(1, loaf_size + 1):
        print(f"  Slicing slice {slice_number}...")
        yield f"Slice {slice_number}"

# Create the slicer — no slicing yet!
slicer = bread_slicer(20)
print("Slicer ready")   # "Slicer ready" — no "Slicing..." messages yet

# Ask for slices one at a time:
next(slicer)   # "Slicing slice 1..." → "Slice 1"
next(slicer)   # "Slicing slice 2..." → "Slice 2"

# Take a few more without slicing the rest:
for slice in itertools.islice(slicer, 3):
    print(f"Customer gets: {slice}")
# Slices 3, 4, 5 are cut. Slices 6-20 remain uncut.
```

---

## Analogy 4: map() and filter() — Assembly Line Stations

**The analogy:**
Picture a factory assembly line. Raw parts come in on a conveyor belt (the input
iterable). The conveyor passes through several stations:

- A **transformation station** (`map`) modifies each part as it passes through —
  painting, shaping, or measuring.
- A **quality control station** (`filter`) inspects each part and diverts defective
  ones off the line.

Neither station needs to wait for all parts to arrive before processing. Each station
handles one part at a time as it arrives. The line is *lazy* — it only runs when
someone at the end is taking finished goods.

**Connected to Python:**
```python
raw_data = [" Alice ", " BOB ", "  charlie  "]

# map: transformation station — strip and titlecase each name
cleaned = map(str.strip, raw_data)          # lazy
titled  = map(str.title, cleaned)           # lazy, chained

# filter: quality control — only names longer than 3 characters
valid = filter(lambda name: len(name) > 3, titled)   # still lazy

# Nothing has run yet. Drive the line:
result = list(valid)   # NOW processes all items
# ["Alice", "Charlie"]  — Bob filtered out after title-casing

# In Python 3 — all lazy:
type(map(str.strip, raw_data))     # <class 'map'>
type(filter(None, [1,0,2,0,3]))    # <class 'filter'>

# With a generator expression — same effect, often clearer:
result = [
    name.strip().title()
    for name in raw_data
    if len(name.strip()) > 3
]
```

---

## Analogy 5: functools.partial — Pre-filling a Form

**The analogy:**
Imagine a government form with 10 fields. You are a department that always fills in
the same value for fields 1, 3, and 7 (the department code, the year, and the category).
Instead of filling those three fields every time, you create a *pre-filled template*
of the form with those fields already completed. Each clerk receives the template and
only fills in the remaining unique fields.

`functools.partial` creates a pre-filled function call — some arguments are baked in,
and the partial object waits for the remaining arguments.

**Connected to Python:**
```python
from functools import partial

def send_notification(recipient, subject, body, priority="normal"):
    print(f"To: {recipient} | {priority.upper()} | {subject}")
    print(f"  {body}")

# Pre-fill the priority and sender context:
send_urgent = partial(send_notification, priority="urgent")
send_to_ops = partial(send_notification, "ops-team@company.com")

# Clerks only fill in what changes:
send_urgent("alice@co.com", "Server down", "DB is unreachable")
# To: alice@co.com | URGENT | Server down

send_to_ops("Deployment complete", "v2.3.1 is now live")
# To: ops-team@company.com | NORMAL | Deployment complete

# Inspect the pre-filled form:
send_urgent.func       # <function send_notification>
send_urgent.keywords   # {'priority': 'urgent'}
```

---

## Analogy 6: yield from — A Telephone Switchboard

**The analogy:**
A company's main telephone switchboard (`outer generator`) receives calls and routes
them to specific departments (`sub-generators`). When the switchboard connects you
to the finance department, your conversation is directly with finance — your words
go straight through, their responses come straight back. The switchboard does not
intercept, mangle, or delay the conversation; it merely established the connection.
When the finance department hangs up, the switchboard reconnects to the main flow.

`yield from` is the switchboard: it creates a transparent two-way channel between the
caller and the sub-generator, passing through `send()` values, exceptions, and return
values.

**Connected to Python:**
```python
def finance_dept():
    query = yield "Finance: ready"
    return f"Finance handled: {query}"

def it_dept():
    query = yield "IT: ready"
    return f"IT handled: {query}"

def switchboard():
    result1 = yield from finance_dept()   # transparent delegation
    print(f"Finance returned: {result1}")
    result2 = yield from it_dept()
    print(f"IT returned: {result2}")
    yield "All departments done"

board = switchboard()
print(next(board))           # "Finance: ready"
print(board.send("invoice")) # Finance returned: Finance handled: invoice
                             # "IT: ready"
print(board.send("ticket"))  # IT returned: IT handled: ticket
                             # "All departments done"
```

---

## Analogy 7: lru_cache — A Memo Pad for Repeated Questions

**The analogy:**
A senior consultant is frequently asked the same questions by different clients.
Instead of re-researching each question from scratch, they keep a memo pad of recent
answers. When asked a question they have answered before, they glance at the pad and
recite the answer immediately. When the pad is full, they erase the oldest entry to
make room for a new question's answer.

The memo pad has a limited number of slots (`maxsize`). When `maxsize=None`, the pad
is unlimited. The "Least Recently Used" policy means that the most recent answers are
kept and the oldest forgotten first.

**Connected to Python:**
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def expensive_lookup(key: str) -> dict:
    """Simulates a slow database lookup."""
    import time
    time.sleep(0.1)   # simulate I/O
    return {"key": key, "data": f"result for {key}"}

expensive_lookup("user_42")   # 0.1s — cache miss, computes and stores
expensive_lookup("user_42")   # instant — cache hit, returns memo
expensive_lookup("user_43")   # 0.1s — new key, cache miss

# Inspect the memo pad:
expensive_lookup.cache_info()
# CacheInfo(hits=1, misses=2, maxsize=128, currsize=2)

# Clear the pad:
expensive_lookup.cache_clear()

# LRU eviction: when maxsize is reached, the least-recently-used entry
# is evicted to make room for a new entry
```
