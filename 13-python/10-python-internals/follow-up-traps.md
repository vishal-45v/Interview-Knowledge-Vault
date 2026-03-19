# Chapter 10: Python Internals — Follow-Up Traps

## Trap 1: "The GIL makes Python thread-safe"

**What most people say:** "Python is thread-safe because of the GIL."

**Correct answer:** The GIL protects CPython's *internal data structures* (reference counts, the memory allocator, the bytecode interpreter) from concurrent modification. It does NOT make your Python code thread-safe. Operations that appear atomic at the Python level (like `counter += 1`) compile to multiple bytecode instructions (`LOAD_GLOBAL`, `LOAD_CONST`, `BINARY_OP`, `STORE_GLOBAL`), and a thread switch can happen between any two instructions. The GIL is released at each bytecode instruction boundary (safepoint), so another thread can run at any point between your instructions.

```python
import dis
def increment():
    global counter
    counter += 1   # looks atomic, NOT atomic

dis.dis(increment)
# LOAD_GLOBAL  0 (counter)   ← thread switch can happen HERE
# LOAD_CONST   1 (1)
# BINARY_OP    0 (+)         ← thread switch can happen HERE
# STORE_GLOBAL 0 (counter)   ← thread switch can happen HERE

# Fix: use threading.Lock
lock = threading.Lock()
def safe_increment():
    with lock:
        counter += 1
# OR: use threading.local() for per-thread state
# OR: use atomic types from queue.Queue, collections.deque
```

---

## Trap 2: "del x frees the memory immediately"

**What most people say:** "I called `del x` so the memory is freed."

**Correct answer:** `del x` removes the name `x` from the current namespace and decrements the object's reference count by 1. If the reference count reaches 0, CPython frees the object *immediately* (this is a CPython implementation detail; other Python implementations like PyPy use tracing GC with no immediate deallocation). But if other references to the object exist, `del x` only removes that one name. Common mistake: `del x` inside a function where `x` was passed in — the caller still holds a reference.

```python
import sys

a = [1, 2, 3]
b = a           # two references to the same list
print(sys.getrefcount(a))  # 3 (a, b, and the getrefcount argument itself)

del a           # removes name 'a', refcount drops to 2
# List still alive! b still holds a reference.
print(b)        # [1, 2, 3] — still accessible

del b           # refcount drops to 1 (only getrefcount arg)
# After this line, refcount hits 0, list is freed immediately in CPython
```

---

## Trap 3: "Python strings are interned automatically"

**What most people say:** "Python interns all strings, so `a is b` always works for equal strings."

**Correct answer:** CPython interns strings that look like Python identifiers (only letters, digits, underscores) and are created as compile-time constants. Strings with spaces, special characters, or created at runtime are NOT automatically interned. The small integer cache guarantees `is` for integers -5 to 256. String interning is an optimization, not a language guarantee — never use `is` to compare string values. Always use `==`.

```python
# Compile-time constants — may be interned (CPython implementation detail)
a = "hello"
b = "hello"
print(a is b)   # True (probably — but don't rely on this!)

# Runtime strings — NOT interned
a = "".join(["hel", "lo"])
b = "hello"
print(a is b)   # False
print(a == b)   # True — always use == for value comparison

# Force interning when you explicitly need identity (e.g., for dict key optimization)
import sys
a = sys.intern("my_key")
b = sys.intern("my_key")
print(a is b)   # True — guaranteed
```

---

## Trap 4: "The cyclic GC always catches reference cycles"

**What most people say:** "Don't worry about reference cycles; the garbage collector handles them."

**Correct answer:** CPython's cyclic GC collects *pure Python* reference cycles. But it does NOT collect cycles involving objects with `__del__` methods (prior to Python 3.4). In Python 3.4+, PEP 442 allows safe finalization of objects with `__del__` in cycles, but the order of `__del__` calls is undefined. Additionally, if a `__del__` method creates a new reference to a dying object (object resurrection), the collector must handle it carefully. More critically: cycles involving C extension objects that do not support the GC protocol (`tp_traverse` not implemented) will NEVER be collected by the cyclic GC.

```python
import gc

class Node:
    def __init__(self):
        self.child = None
    def __del__(self):
        print(f"__del__ called for {id(self)}")

a = Node()
b = Node()
a.child = b
b.child = a   # cycle

del a, del b  # refcounts don't reach 0 due to cycle

gc.collect()  # Python 3.4+: __del__ is called, cycle collected
              # Python 3.3 and earlier: NOT collected! (gc.garbage list fills up)
print(gc.garbage)  # pre-3.4: [a, b]; 3.4+: []
```

---

## Trap 5: "The descriptor protocol only applies to properties"

**What most people say:** "Descriptors are for when you want to use `@property`."

**Correct answer:** `@property` is a built-in descriptor, but the descriptor protocol is far more general. `classmethod`, `staticmethod`, instance methods (function objects ARE descriptors), `__slots__` slot descriptors, `super()`, and many ORM field implementations all use the descriptor protocol. Any object that implements `__get__` (and optionally `__set__`, `__delete__`) is a descriptor. When you write `obj.attr`, Python's attribute lookup calls `type(obj).__mro__` to find the descriptor and invokes `__get__`.

```python
class Validator:
    """A data descriptor that validates before storing."""
    def __set_name__(self, owner, name):
        self.name = f"_{name}"

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.name, None)

    def __set__(self, obj, value):
        if not isinstance(value, int) or value < 0:
            raise ValueError(f"{self.name} must be a non-negative integer")
        setattr(obj, self.name, value)

class Order:
    quantity = Validator()   # descriptor used like a field
    def __init__(self, quantity):
        self.quantity = quantity   # calls Validator.__set__

o = Order(5)
print(o.quantity)   # calls Validator.__get__ → 5
o.quantity = -1     # calls Validator.__set__ → ValueError
```

---

## Trap 6: "MRO (C3 linearization) is the same as depth-first search"

**What most people say:** "Python uses depth-first search to resolve method names in multiple inheritance."

**Correct answer:** Python 2 used depth-first left-to-right, which produces inconsistent ordering in diamond inheritance. Python 3 uses **C3 linearization**, which satisfies three properties: (1) children come before parents, (2) local precedence order is preserved, (3) the monotonicity constraint (a class's MRO is a consistent extension of all its bases' MROs). These properties guarantee that `super()` in a cooperative multiple-inheritance chain calls each parent class exactly once in a predictable order.

```python
class A:
    def method(self): return "A"

class B(A):
    def method(self): return "B → " + super().method()

class C(A):
    def method(self): return "C → " + super().method()

class D(B, C):   # diamond: D→B→C→A
    def method(self): return "D → " + super().method()

print(D.__mro__)
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)
# C3: D comes before B, B before C, C before A — depth-first WOULD give D,B,A,C,A (wrong!)

print(D().method())
# "D → B → C → A" — each class called exactly once
```

---

## Trap 7: "`sys.modules` caching means imports are free after the first one"

**What most people say:** "Module imports are cached, so calling `import mymodule` in a loop is fine."

**Correct answer:** `import mymodule` at the top of a function body does hit `sys.modules` (a dict lookup) on every call — not free, but cheap. The heavier issue is that `import` inside a hot loop is a dict lookup + attribute lookup + name binding *on every iteration*, even if cached. The real cost is the attribute lookup pattern: doing `import json; json.dumps(...)` in a loop is two attribute accesses per iteration. Optimize by binding to a local:

```python
# Mildly wasteful in a tight loop
def process_many(items):
    import json   # sys.modules dict lookup on every call
    return [json.dumps(i) for i in items]   # json.dumps LOAD_GLOBAL + LOAD_ATTR per item

# Optimized
import json
_dumps = json.dumps   # bind once at module level

def process_many_fast(items):
    dumps = _dumps    # local binding — LOAD_FAST in loop
    return [dumps(i) for i in items]
```

---

## Trap 8: "Structural pattern matching is just a fancy switch statement"

**What most people say:** "Python's `match` is like switch/case in other languages."

**Correct answer:** `match`/`case` in Python is fundamentally different from C/Java `switch`. It is a pattern matching expression, not a lookup table. It can: (1) destructure sequences and mappings, (2) capture values into new names, (3) apply guard conditions, (4) match class instances by attribute, (5) use OR patterns (`case A | B`). A critical trap: a bare name in a `case` clause is a *capture pattern* (always matches, binds the value to that name), NOT a comparison against an existing variable. This surprises everyone coming from C/Java/Rust.

```python
status = "active"

# TRAP: this does NOT compare against the variable `status`
case_value = "active"
match some_input:
    case case_value:   # WRONG INTENT — this is a capture, always matches!
        print("matched!")

# CORRECT: use literal values in case clauses for comparison
match some_input:
    case "active":     # literal match — correct
        print("active")
    case "inactive":
        print("inactive")

# CORRECT: use guard for variable comparison
match some_input:
    case x if x == case_value:   # x captures the value, guard checks it
        print("matched!")
```

---

## Trap 9: "Python's `import` statement creates a new module object each time"

**What most people say:** "Every `import` loads the module fresh."

**Correct answer:** Python maintains `sys.modules` as a cache. After the first import, subsequent `import` statements return the *same module object* from cache. This means: module-level code runs only once. Module-level state (global variables, singleton instances) is shared across all importers. This can cause surprising behavior when tests import a module, modify its state, and the next test sees the modified state. Also, `from mymodule import X` binds the name `X` in the importer's namespace at import time — later changes to `mymodule.X` are not reflected in the bound name.

```python
# module_a.py
counter = 0

# module_b.py
from module_a import counter   # binds 'counter' to 0 at import time

# main.py
import module_a
import module_b

module_a.counter = 42
print(module_a.counter)   # 42 — the attribute on the module object
print(module_b.counter)   # 0 — the name in module_b is still bound to original 0!
```

---

## Trap 10: "Python 3.13 free-threaded mode removes all GIL constraints"

**What most people say:** "Python 3.13 no longer has a GIL, so threading is now truly parallel."

**Correct answer:** Python 3.13 introduced an *experimental* free-threaded build (PEP 703, `python3.13t`) that can be enabled with `--disable-gil`. This is opt-in and not the default. Key limitations: (1) Many C extension packages (NumPy, Pandas older versions) are not yet thread-safe without the GIL and may crash or corrupt data. (2) Performance regressions exist in single-threaded code due to per-object locking overhead replacing the GIL. (3) The standard library itself has threading assumptions that are being audited. (4) It is marked as experimental with no stability guarantees in 3.13. The community expects several more Python releases before free-threading is considered production-ready.
