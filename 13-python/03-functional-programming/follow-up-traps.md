# Chapter 03 — Functional Programming: Follow-Up Traps

---

## Trap 1: "A closure captures the value of the variable at the time the closure is created"

**What most people say:**
"The lambda `lambda: i` captures the value of `i` when the lambda is created."

**Correct answer:**
Closures capture the *variable* (a cell object reference), not the *value*. The value
is read when the closure is *called*. This is why the classic loop closure bug produces
all the same value:

```python
# BUG: all closures reference the SAME variable i
functions = [lambda: i for i in range(5)]
[f() for f in functions]   # [4, 4, 4, 4, 4] — all see i's final value

# FIX 1: default argument captures value at creation time
functions = [lambda i=i: i for i in range(5)]
[f() for f in functions]   # [0, 1, 2, 3, 4]

# FIX 2: functools.partial
from functools import partial
def identity(x): return x
functions = [partial(identity, i) for i in range(5)]
[f() for f in functions]   # [0, 1, 2, 3, 4]
```

---

## Trap 2: "Generators are the same as iterators"

**What most people say:**
"Generator and iterator are interchangeable terms."

**Correct answer:**
All generators are iterators, but not all iterators are generators. An *iterator* is
any object that implements `__iter__` and `__next__`. A *generator* is a specific kind
of iterator created by a generator function (using `yield`) or a generator expression.
Generators also support `.send()`, `.throw()`, and `.close()` — they are coroutines.

```python
# Iterator (non-generator):
class Counter:
    def __init__(self, n): self.n = n; self.current = 0
    def __iter__(self): return self
    def __next__(self):
        if self.current >= self.n:
            raise StopIteration
        self.current += 1
        return self.current

# Generator:
def counter(n):
    for i in range(1, n+1):
        yield i

g = counter(3)
g.send(None)   # advance, same as next(g) — generators support send()
Counter(3).send(None)  # AttributeError — plain iterator has no send()
```

---

## Trap 3: "functools.lru_cache works on any function"

**What most people say:**
"Just slap `@lru_cache` on slow functions and they get faster."

**Correct answer:**
`lru_cache` requires that all arguments be *hashable*. It also caches based on
*argument values*, so functions with side effects or that depend on external state
will return stale cached results. Specifically unsuitable:
- Functions with list, dict, or set arguments
- Functions that read from databases, files, or network (return value changes)
- Instance methods where `self` is unhashable or mutable
- Functions that intentionally return different results on each call (random, time-based)

```python
from functools import lru_cache

@lru_cache(maxsize=128)
def process(data: list):    # TypeError: unhashable type 'list'
    return sum(data)

# Workaround: convert to tuple at call site
@lru_cache(maxsize=128)
def process(data: tuple):   # tuple is hashable
    return sum(data)

process(tuple([1, 2, 3]))

# Bad use — result changes over time:
@lru_cache(maxsize=128)
def get_time():
    import time
    return time.time()   # will always return the first cached result!
```

---

## Trap 4: "yield from is just a loop with yield"

**What most people say:**
"`yield from sub` is the same as `for x in sub: yield x`."

**Correct answer:**
They produce the same values in the simple case, but `yield from` also:
1. Passes `.send()` values through to the sub-generator
2. Passes `.throw()` exceptions through to the sub-generator
3. Collects the sub-generator's `return` value (available as the expression value of
   `yield from`)
4. Closes the sub-generator properly when the outer generator is closed

```python
def sub():
    received = yield "from sub"
    return f"sub done, received: {received}"

def outer():
    result = yield from sub()   # passes send() values down; captures return
    yield f"outer got: {result}"

g = outer()
next(g)               # "from sub"   — advanced to sub's yield
g.send("hello")       # "outer got: sub done, received: hello"

# The for-loop version CANNOT do this:
def outer_bad():
    for x in sub():
        yield x       # send() not passed to sub(); return value lost
```

---

## Trap 5: "Lambda functions can contain any Python expression"

**What most people say:**
"Lambdas are just inline functions, so they can do anything a function can."

**Correct answer:**
Lambda bodies must be a *single expression* — no statements. This means no `=`
assignment (though walrus `:=` works), no `return` (value is the expression result),
no `if/else` blocks (only ternary `x if cond else y`), no `try/except`, no `for`
loops (though comprehensions are expressions). Lambdas also cannot use `yield`.

```python
# OK — ternary expression:
f = lambda x: "even" if x % 2 == 0 else "odd"

# OK — comprehension (expression):
f = lambda xs: [x**2 for x in xs]

# NOT OK — assignment statement:
# f = lambda x: result = x * 2  ← SyntaxError

# Walrus OK (but ugly):
f = lambda x: (y := x * 2, y + 1)[1]   # contrived; use a def

# Cannot yield:
# f = lambda: (yield 1)  ← SyntaxError
```

---

## Trap 6: "functools.partial is the same as a lambda wrapper"

**What most people say:**
"`partial(func, x)` is just shorthand for `lambda *args, **kw: func(x, *args, **kw)`."

**Correct answer:**
Functionally similar, but `partial` objects: (1) are picklable (lambdas are not),
(2) expose `.func`, `.args`, `.keywords` attributes for introspection, (3) work with
`isinstance` checks, (4) avoid the closure variable capture bug. This matters in
multiprocessing (pickling) and in descriptive debugging.

```python
from functools import partial
import pickle

def power(base, exp):
    return base ** exp

square = partial(power, exp=2)
square.func      # <function power>
square.keywords  # {'exp': 2}
square(5)        # 25

# Pickling — works with partial, fails with lambda:
pickle.dumps(square)   # OK
cube = lambda x: power(x, 3)
pickle.dumps(cube)     # AttributeError (can't pickle local object)

# Multiprocessing requires picklable callables:
from multiprocessing import Pool
with Pool() as p:
    p.map(square, range(5))   # works
    p.map(cube, range(5))     # fails with pickle error
```

---

## Trap 7: "itertools.groupby groups all items with the same key"

**What most people say:**
"`groupby(data, key=func)` collects ALL items with the same key value into one group."

**Correct answer:**
`groupby` groups *consecutive* items with the same key. If the data is not sorted by
the key first, you will get multiple groups for the same key value. This is by design
for memory efficiency (streaming), but it catches nearly everyone off guard.

```python
from itertools import groupby

data = [1, 1, 2, 2, 1, 1]   # 1 appears in two non-consecutive runs

# WRONG expectation:
for key, group in groupby(data):
    print(key, list(group))
# 1 [1, 1]
# 2 [2, 2]
# 1 [1, 1]   ← THIRD group! not merged with first

# CORRECT — sort first:
for key, group in groupby(sorted(data)):
    print(key, list(group))
# 1 [1, 1, 1, 1]
# 2 [2, 2]
```

---

## Trap 8: "Decorators with arguments are just decorators that take arguments"

**What most people say:**
"`@decorator(arg)` is just a decorator that accepts an argument."

**Correct answer:**
`@decorator(arg)` means: call `decorator(arg)` first (returns something), then use
that something as a decorator. So you need THREE levels of nesting: the outermost
function receives arguments and returns the actual decorator, which receives the
function and returns the wrapper.

```python
# Two-level (no arguments) — @decorator:
def decorator(func):
    def wrapper(*a, **kw): return func(*a, **kw)
    return wrapper

# Three-level (with arguments) — @decorator(arg):
def decorator(arg):          # level 1: receives decorator args, returns level 2
    def actual_decorator(func):   # level 2: receives function, returns level 3
        def wrapper(*a, **kw):    # level 3: the actual wrapper
            print(f"arg={arg}")
            return func(*a, **kw)
        return wrapper
    return actual_decorator

@decorator("hello")   # same as: func = decorator("hello")(func)
def my_func(): pass
```

---

## Trap 9: "A generator is exhausted when its function returns"

**What most people say:**
"The generator finishes when the generator function returns at the end."

**Correct answer:**
A generator raises `StopIteration` when the generator function either: (1) falls off
the end (implicit `return`), or (2) executes an explicit `return` statement. The
`return` value becomes the `value` attribute of the `StopIteration` exception, which
is accessible via `yield from` or by catching the exception. This is the basis of
PEP 380 coroutine chaining.

```python
def gen():
    yield 1
    yield 2
    return "done"    # value of StopIteration

g = gen()
next(g)   # 1
next(g)   # 2
try:
    next(g)
except StopIteration as e:
    print(e.value)  # "done"

# yield from captures it:
def outer():
    result = yield from gen()  # result = "done"
    print(f"subgenerator returned: {result}")
```

---

## Trap 10: "map() and filter() eagerly evaluate like list comprehensions"

**What most people say:**
"`map()` processes all items immediately when you call it."

**Correct answer:**
In Python 3, `map()`, `filter()`, and `zip()` all return *lazy iterators*, not lists.
They process items one at a time on demand. This is a breaking change from Python 2
where they returned lists. The practical implication: you can pass `map()` results to
any function that accepts an iterable without creating an intermediate list.

```python
result = map(str, range(10**9))  # instant — no computation yet
type(result)   # <class 'map'>   — lazy iterator

# Materialise only if needed:
first_five = list(itertools.islice(result, 5))  # only processes 5 items

# Side-effect pitfall: map is single-pass
result = map(lambda x: x * 2, [1, 2, 3])
list(result)   # [2, 4, 6]
list(result)   # []  ← exhausted! Unlike a list comprehension or range
```
