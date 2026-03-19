# Chapter 03 — Functional Programming: Structured Answers

---

## Q1: What is a closure and how does it work internally?

**Core answer:**
A closure is a function that retains access to variables from its enclosing scope even
after the enclosing function has returned. The retained variables are called *free
variables* and are stored in *cell objects* attached to the function.

```python
def make_multiplier(factor):
    def multiply(x):
        return x * factor   # 'factor' is a free variable
    return multiply

double = make_multiplier(2)
triple = make_multiplier(3)

double(5)   # 10
triple(5)   # 15

# Each closure has its own cell:
double.__closure__                     # (<cell at 0x...>,)
double.__closure__[0].cell_contents    # 2
triple.__closure__[0].cell_contents    # 3

# The function's code knows which locals are free variables:
double.__code__.co_freevars   # ('factor',)
```

**How cell objects work:**
When Python compiles a nested function that references an outer variable, it converts
that variable into a *cell*. Both the outer function's local namespace and the inner
function's `__closure__` point to the same cell object. The cell holds a reference to
the current value of the variable.

```python
def counter():
    count = 0
    def increment():
        nonlocal count
        count += 1
        return count
    def get():
        return count
    return increment, get

inc, get = counter()
inc()   # 1
inc()   # 2
get()   # 2  — same cell, both functions see the same value
```

---

## Q2: How do you write a well-formed decorator?

**Core answer:**
A complete decorator must: (1) preserve the wrapped function's metadata, (2) forward
all arguments correctly, (3) preserve the return value.

```python
import functools
import time
import logging

def timed(func):
    """Logs execution time of the wrapped function."""
    @functools.wraps(func)   # copies __name__, __doc__, __annotations__, __module__
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        try:
            result = func(*args, **kwargs)
            return result
        finally:
            elapsed = time.perf_counter() - start
            logging.info(f"{func.__name__} took {elapsed:.4f}s")
    return wrapper

@timed
def fetch_data(url: str, timeout: int = 30) -> dict:
    """Fetch data from the given URL."""
    pass

fetch_data.__name__     # 'fetch_data'   — not 'wrapper'
fetch_data.__doc__      # 'Fetch data from the given URL.'
fetch_data.__wrapped__  # <function fetch_data> — functools.wraps sets this
```

**Parametrised decorator (three levels):**
```python
def retry(max_attempts=3, exceptions=(Exception,), delay=1.0):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exc = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exc = e
                    if attempt < max_attempts - 1:
                        time.sleep(delay * (2 ** attempt))  # exponential backoff
            raise last_exc
        return wrapper
    return decorator

@retry(max_attempts=3, exceptions=(IOError, TimeoutError), delay=0.5)
def load_file(path):
    with open(path) as f:
        return f.read()
```

---

## Q3: Explain generators — yield, yield from, and send()

**Core answer:**
A generator function contains at least one `yield` statement. Calling it returns a
generator iterator without executing the body. The body executes in steps, pausing at
each `yield`.

```python
def fibonacci():
    a, b = 0, 1
    while True:
        yield a           # suspend here, return a to caller
        a, b = b, a + b   # resume here after next()

fib = fibonacci()
[next(fib) for _ in range(8)]   # [0, 1, 1, 2, 3, 5, 8, 13]
```

**send() — two-way communication:**
```python
def accumulator():
    total = 0
    while True:
        value = yield total    # yield current total, receive new value via send()
        if value is None:
            break
        total += value

gen = accumulator()
next(gen)       # 0   — prime the generator (advance to first yield)
gen.send(10)    # 10
gen.send(20)    # 30
gen.send(5)     # 35
```

**yield from — delegation and chaining:**
```python
def read_chunks(path, size=1024):
    with open(path, "rb") as f:
        while chunk := f.read(size):
            yield chunk

def read_multiple_files(paths, size=1024):
    for path in paths:
        yield from read_chunks(path, size)  # delegates to sub-generator
        # Equivalent to:
        # for chunk in read_chunks(path, size):
        #     yield chunk
        # BUT also: passes .send() and .throw() through to sub-generator

# Generator pipeline:
def parse_lines(chunks):
    buffer = ""
    for chunk in chunks:
        buffer += chunk.decode("utf-8")
        while "\n" in buffer:
            line, buffer = buffer.split("\n", 1)
            yield line

# Compose:
chunks = read_chunks("large_file.txt")
lines  = parse_lines(chunks)    # lazy — no I/O yet
for line in lines:              # drives the whole pipeline one line at a time
    process(line)
```

---

## Q4: Explain the iterator protocol

**Core answer:**
An *iterable* implements `__iter__()` which returns an *iterator*. An *iterator*
implements both `__iter__()` (returns `self`) and `__next__()` (returns next value or
raises `StopIteration`).

```python
class NumberRange:
    """A custom iterable that produces numbers in a range."""

    def __init__(self, start, stop):
        self.start = start
        self.stop  = stop

    def __iter__(self):
        return NumberRangeIterator(self.start, self.stop)  # return an iterator

class NumberRangeIterator:
    """The stateful iterator for NumberRange."""

    def __init__(self, start, stop):
        self.current = start
        self.stop    = stop

    def __iter__(self):
        return self   # iterators must return themselves

    def __next__(self):
        if self.current >= self.stop:
            raise StopIteration
        value = self.current
        self.current += 1
        return value

nr = NumberRange(1, 5)
list(nr)    # [1, 2, 3, 4]
list(nr)    # [1, 2, 3, 4]  — iterable can be iterated multiple times

it = iter(nr)
next(it)    # 1
next(it)    # 2

# The for loop is syntactic sugar for:
iterator = iter(nr)
while True:
    try:
        item = next(iterator)
        # loop body
    except StopIteration:
        break
```

**Key distinction — iterable vs iterator:**
```
Iterable:  has __iter__, returns a fresh iterator each time
Iterator:  has __iter__ (returns self) + __next__
           — stateful, single-pass, exhaustible

list:       iterable (not an iterator — no __next__)
iter(list): iterator (has __next__)
range(5):   iterable (not an iterator)
generator:  both iterable AND iterator (returns self from __iter__)
```

---

## Q5: Key itertools tools and when to use them

**Core answer:**

```python
import itertools

# chain — flatten one level of nesting:
list(itertools.chain([1,2], [3,4], [5]))   # [1, 2, 3, 4, 5]
list(itertools.chain.from_iterable([[1,2],[3,4]]))  # same

# islice — lazy slicing without materialising:
gen = (x**2 for x in itertools.count())   # infinite!
list(itertools.islice(gen, 5))             # [0, 1, 4, 9, 16] — first 5

# groupby — consecutive groups (MUST BE SORTED FIRST):
data = [("a", 1), ("a", 2), ("b", 3), ("b", 4)]
for key, group in itertools.groupby(data, key=lambda x: x[0]):
    print(key, list(group))
# a [('a', 1), ('a', 2)]
# b [('b', 3), ('b', 4)]

# product — cartesian product (nested loops):
list(itertools.product([1,2], ["a","b"]))
# [(1,'a'), (1,'b'), (2,'a'), (2,'b')]
# equivalent to: [(x,y) for x in [1,2] for y in ["a","b"]]

# combinations — ordered subsets, no repetition:
list(itertools.combinations([1,2,3], 2))
# [(1,2), (1,3), (2,3)]   — C(3,2) = 3

# permutations — ordered arrangements, no repetition:
list(itertools.permutations([1,2,3], 2))
# [(1,2),(1,3),(2,1),(2,3),(3,1),(3,2)]  — P(3,2) = 6

# takewhile / dropwhile:
list(itertools.takewhile(lambda x: x < 5, [1,3,5,2,1]))  # [1,3]
list(itertools.dropwhile(lambda x: x < 5, [1,3,5,2,1]))  # [5,2,1]

# count / cycle / repeat:
list(itertools.islice(itertools.count(10, 2), 5))  # [10,12,14,16,18]
list(itertools.islice(itertools.cycle("AB"), 5))   # ['A','B','A','B','A']
list(itertools.repeat(7, 3))                        # [7, 7, 7]
```
