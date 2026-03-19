# Chapter 01 — Python Basics: Structured Answers

---

## Q1: Explain mutability vs immutability with practical consequences

**Core answer:**
An object is *immutable* if its state cannot be changed after creation. Python's
immutable types are: `int`, `float`, `complex`, `bool`, `str`, `bytes`, `tuple`,
`frozenset`, and `range`. Mutable types include `list`, `dict`, `set`, and `bytearray`.

The practical consequences are significant:

**1. Default argument trap** — only safe with immutable defaults:
```python
# WRONG — list is mutable; shared across all calls
def add_item(val, lst=[]):
    lst.append(val)
    return lst

# CORRECT — None is immutable sentinel
def add_item(val, lst=None):
    if lst is None:
        lst = []
    lst.append(val)
    return lst
```

**2. Hashability** — only immutable objects can be dict keys or set members:
```python
d = {}
d[(1, 2)] = "point"   # OK: tuple is immutable → hashable
d[[1, 2]] = "point"   # TypeError: list is not hashable

# frozenset as dict key:
d[frozenset([1, 2])] = "set as key"  # OK
```

**3. String "mutation" creates new objects**:
```python
s = "hello"
id_before = id(s)
s += " world"
id(s) == id_before  # False — s now points to a new object
```

**4. Tuple with mutable elements is not deeply immutable**:
```python
t = ([1, 2], [3, 4])
t[0].append(99)   # Works! The tuple reference is immutable, not the list contents
print(t)          # ([1, 2, 99], [3, 4])
hash(t)           # TypeError — tuple containing mutable objects is not hashable
```

---

## Q2: Explain the LEGB rule with a complete example

**Core answer:**
LEGB stands for Local → Enclosing → Global → Built-in. When Python encounters a name,
it searches these scopes in order and uses the first match found.

```python
# Built-in: len, print, range, etc.
x = "global"     # Global scope

def outer():
    x = "enclosing"   # Enclosing scope for inner()

    def inner():
        x = "local"   # Local scope
        print(x)      # → "local"  (L wins)

    inner()
    print(x)          # → "enclosing"  (E, since outer's local)

outer()
print(x)              # → "global"  (G)
```

**Modifying outer scopes:**
```python
count = 0

def increment():
    global count      # without this: UnboundLocalError
    count += 1

def make_counter():
    n = 0
    def inc():
        nonlocal n    # refers to enclosing function's n
        n += 1
        return n
    return inc

counter = make_counter()
counter()  # 1
counter()  # 2
```

**Key insight:** Python decides at compile time whether a name is local to a function
(if the name is assigned anywhere in the function body, it is treated as local
throughout that function — even before the assignment):
```python
x = 10
def f():
    print(x)   # UnboundLocalError! Python saw x = ... below and made x local
    x = 20
```

---

## Q3: Explain `is` vs `==` and when they diverge

**Core answer:**
`==` tests value equality (calls `__eq__`). `is` tests identity (same object in memory,
same `id()`). They diverge in three main scenarios:

**Scenario 1 — Integer caching:**
```python
a = 256; b = 256
a is b   # True  — CPython caches integers -5 through 256

a = 257; b = 257
a is b   # False — outside cache range (might be True in some compile units)
```

**Scenario 2 — String interning:**
```python
a = "hello"; b = "hello"
a is b   # True  — simple identifier-like strings are interned by CPython

a = "hello world"; b = "hello world"
a is b   # False in general (though may be True if in same code object)

import sys
a = sys.intern("hello world")
b = sys.intern("hello world")
a is b   # True — explicit interning
```

**Scenario 3 — Custom __eq__:**
```python
class AlwaysEqual:
    def __eq__(self, other): return True

x = AlwaysEqual()
y = AlwaysEqual()
x == y   # True
x is y   # False
x == None  # True  — this is why `if x == None` is wrong; use `if x is None`
```

**Rule:** Use `is` ONLY for: `None`, `True`, `False`, and when you explicitly need
identity. Use `==` for all value comparisons.

---

## Q4: Explain *args and **kwargs thoroughly

**Core answer:**
`*args` collects extra positional arguments into a `tuple`. `**kwargs` collects extra
keyword arguments into a `dict`. The names `args` and `kwargs` are conventions only —
the `*` and `**` are the actual syntax.

```python
def demo(*args, **kwargs):
    print(type(args))   # <class 'tuple'>
    print(type(kwargs)) # <class 'dict'>
    print(args)
    print(kwargs)

demo(1, 2, 3, name="Alice", age=30)
# (1, 2, 3)
# {'name': 'Alice', 'age': 30}
```

**Full parameter ordering (Python 3):**
```python
def f(pos_only, /, normal, *args, kw_only, **kwargs):
    pass
# pos_only — positional-only (before /)
# normal   — positional or keyword
# *args    — variadic positional
# kw_only  — keyword-only (after *args)
# **kwargs — variadic keyword
```

**Unpacking at the call site:**
```python
def add(a, b, c):
    return a + b + c

nums = [1, 2, 3]
add(*nums)         # 6

params = {"a": 1, "b": 2, "c": 3}
add(**params)      # 6

# Combining:
add(*[1, 2], c=3)  # 6
```

**Forwarding args (decorator pattern):**
```python
def log_call(func):
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)   # forward everything unchanged
    return wrapper
```

---

## Q5: Explain Python's truthiness system

**Core answer:**
Every Python object has a boolean value. `bool(x)` calls `x.__bool__()`, or falls back
to `x.__len__() == 0` if `__bool__` is not defined. Objects are *falsy* if their
`__bool__` returns `False` or `__len__` returns 0; everything else is truthy.

**Falsy values by type:**
```python
# None
bool(None)          # False

# Numeric zeros
bool(0)             # False
bool(0.0)           # False
bool(0j)            # False (complex zero)
bool(Decimal("0"))  # False

# Empty sequences/collections
bool("")            # False
bool(b"")           # False
bool([])            # False
bool(())            # False
bool({})            # False
bool(set())         # False
bool(range(0))      # False

# Boolean
bool(False)         # False
```

**Custom __bool__ and __len__:**
```python
class MyContainer:
    def __init__(self, items):
        self.items = items

    def __len__(self):
        return len(self.items)
        # bool(MyContainer([])) → False
        # bool(MyContainer([1])) → True

class AlwaysFalsy:
    def __bool__(self): return False

if not AlwaysFalsy():   # True — object is falsy
    print("falsy!")
```

**Common idiom pitfalls:**
```python
# Checking for empty list vs None — different semantics!
data = []
if not data:           # True for [], None, 0, "" — too broad
    pass
if data is None:       # Only catches None
    pass
if len(data) == 0:     # Only catches empty sequence
    pass
```

---

## Q6: Explain comprehensions and when to use each form

**Core answer:**
Python has four comprehension forms: list (`[]`), dict (`{k:v}`), set (`{v}`), and
generator (`()`). They are syntactic sugar over loops but are often faster because
the loop is executed in C rather than interpreted Python bytecode.

```python
numbers = range(1, 11)

# List comprehension — materialises all results in memory
squares = [x**2 for x in numbers]
# [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# Dict comprehension
sq_map = {x: x**2 for x in numbers}
# {1: 1, 2: 4, 3: 9, ...}

# Set comprehension — deduplicates
remainders = {x % 3 for x in numbers}
# {0, 1, 2}

# Generator expression — lazy, single-pass
gen = (x**2 for x in numbers)
next(gen)   # 1
next(gen)   # 4
```

**Filtering:**
```python
evens = [x for x in numbers if x % 2 == 0]

# Nested (order mirrors nested loops):
pairs = [(x, y) for x in [1, 2, 3] for y in [10, 20] if x != y]
```

**Performance — list comprehension vs loop:**
```python
import timeit

# List comprehension (faster):
timeit.timeit('[x**2 for x in range(1000)]', number=10000)

# Equivalent loop (slower — LOAD_FAST on list.append + method lookup):
timeit.timeit('''
result = []
for x in range(1000):
    result.append(x**2)
''', number=10000)
```

**When to use a loop instead:**
- When the logic is more than one line and the comprehension becomes unreadable
- When you need multiple statements per iteration
- When you need `break` or `continue` logic

**Generator expression tip:** Pass a generator directly to functions that accept
iterables — you save the intermediate list allocation:
```python
total = sum(x**2 for x in numbers)   # no [] needed inside sum()
```
