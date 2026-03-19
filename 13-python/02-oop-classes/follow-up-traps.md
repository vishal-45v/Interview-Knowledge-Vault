# Chapter 02 — OOP & Classes: Follow-Up Traps

---

## Trap 1: "__init__ is Python's constructor"

**What most people say:**
"`__init__` is the constructor — it creates the object."

**Correct answer:**
`__new__` is the actual constructor — it allocates and returns the new instance.
`__init__` is the *initialiser* — it receives the already-created instance and sets
up its state. When you call `MyClass(args)`, Python calls `MyClass.__new__(MyClass)`
first, then calls `__init__` on the returned instance. Overriding `__new__` is
necessary for immutable types (since you cannot set attributes after construction)
and for singleton patterns.

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance   # __init__ still runs on every call!

    def __init__(self, value):
        self.value = value

a = Singleton(1)
b = Singleton(2)
a is b       # True — same object
a.value      # 2    — __init__ ran again and overwrote value!
```

---

## Trap 2: "super() calls the parent class"

**What most people say:**
"`super()` calls the immediate parent class."

**Correct answer:**
`super()` calls the NEXT class in the MRO, which is not always the direct parent.
In multiple inheritance, `super()` follows the C3 linearization order. This is
why consistent `super()` usage is crucial in cooperative multiple inheritance — each
class in the chain passes the call along.

```python
class A:
    def greet(self):
        print("A")
        super().greet()   # calls B (next in MRO), not object

class B:
    def greet(self):
        print("B")
        super().greet()   # calls object.greet() — but object has no greet!

class C(A, B):
    def greet(self):
        print("C")
        super().greet()

C.__mro__   # (C, A, B, object)
C().greet() # C → A → B  (cooperative chain)
```

---

## Trap 3: "Defining __eq__ doesn't affect __hash__"

**What most people say:**
"I can define `__eq__` and my class will still be hashable."

**Correct answer:**
When you define `__eq__` in a class that does not already define `__hash__`, Python
**sets `__hash__` to `None`**, making instances unhashable. This enforces the
mathematical requirement: if `a == b`, then `hash(a) == hash(b)`. If you define
`__eq__`, you must also define `__hash__` if you want instances in sets or as dict keys.

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)
    # __hash__ is now None!

p = Point(1, 2)
{p}      # TypeError: unhashable type: 'Point'

# Fix: define __hash__ consistent with __eq__
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __eq__(self, other):
        return (self.x, self.y) == (other.x, other.y)
    def __hash__(self):
        return hash((self.x, self.y))
```

---

## Trap 4: "Class variables and instance variables are separate unless they share a name"

**What most people say:**
"Class variables are shared; instance variables are per-instance. They don't interfere."

**Correct answer:**
Reading a name on an instance searches the instance `__dict__` first, then the class
`__dict__`. So reading a class variable through an instance works. But *writing* to an
instance attribute with the same name as a class variable creates a new entry in the
**instance** `__dict__`, shadowing (not modifying) the class variable. This causes a
subtle bug with mutable class attributes:

```python
class Config:
    tags = []       # class variable — SHARED

c1 = Config()
c2 = Config()

c1.tags.append("fast")   # mutates the SHARED class-level list!
print(c2.tags)            # ['fast'] — c2 also sees it

c1.tags = ["new"]         # NOW c1 has its own instance variable
print(c2.tags)            # ['fast'] — c2 still on the class variable
```

---

## Trap 5: "Descriptors are an advanced feature only used by framework authors"

**What most people say:**
"Descriptors are too advanced; I only need them for writing frameworks."

**Correct answer:**
You use descriptors every day: `@property`, `@classmethod`, `@staticmethod`, and
`@functools.cached_property` are all implemented as descriptors. Understanding
the descriptor protocol (`__get__`, `__set__`, `__delete__`) lets you understand
exactly how these built-ins work and build validation attributes concisely.

```python
class Positive:
    """Descriptor that enforces positive values."""
    def __set_name__(self, owner, name):
        self.name = name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return obj.__dict__.get(self.name)

    def __set__(self, obj, value):
        if value <= 0:
            raise ValueError(f"{self.name} must be positive, got {value}")
        obj.__dict__[self.name] = value

class Circle:
    radius = Positive()   # one descriptor instance, shared by all Circle instances

c = Circle()
c.radius = 5    # OK
c.radius = -1   # ValueError: radius must be positive, got -1
```

---

## Trap 6: "__slots__ disables everything dynamic about instances"

**What most people say:**
"Using __slots__ means you can never add new attributes."

**Correct answer:**
`__slots__` removes the per-instance `__dict__` and `__weakref__` by default, so you
cannot add arbitrary attributes. However: (1) you can still add class-level attributes
normally, (2) a subclass without `__slots__` regains `__dict__`, (3) you can include
`"__dict__"` in `__slots__` to opt back into dynamic attributes for specific classes.
The main benefit is reduced memory per instance — roughly 40-50% for objects with
many instances.

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

sys.getsizeof(WithDict(1, 2))   # ~56 bytes (object) + dict overhead (~232 bytes)
sys.getsizeof(WithSlots(1, 2))  # ~56 bytes — no dict!

# Subclass without __slots__ regains __dict__:
class Sub(WithSlots):
    pass
Sub(1, 2).extra = "allowed"  # OK — Sub has __dict__
```

---

## Trap 7: "@dataclass generates __init__ but not __hash__ by default"

**What most people say:**
"@dataclass auto-generates __init__, __repr__, and __eq__, so instances are
automatically usable as dict keys."

**Correct answer:**
When `@dataclass` generates `__eq__` (which it does by default), it also sets
`__hash__` to `None` (same rule as manually defining `__eq__`). To get hashable
dataclass instances, use `@dataclass(frozen=True)` (makes the instance immutable
AND generates a proper `__hash__`) or `@dataclass(unsafe_hash=True)` (generates a
hash even on a mutable class — use with caution).

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

p = Point(1.0, 2.0)
hash(p)   # TypeError: unhashable type: 'Point'

@dataclass(frozen=True)
class FrozenPoint:
    x: float
    y: float

fp = FrozenPoint(1.0, 2.0)
hash(fp)  # Works!
fp.x = 5  # FrozenInstanceError — truly immutable
```

---

## Trap 8: "Abstract methods enforce interface at import time"

**What most people say:**
"If I use ABC, Python will refuse to import a module that has an incomplete subclass."

**Correct answer:**
Python raises `TypeError` when you try to *instantiate* an incomplete abstract subclass,
not at class definition time or import time. The class can be defined and imported
successfully — the error only surfaces when you call `MyIncompleteClass()`.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class BadCircle(Shape):
    pass   # no area() defined — no error yet!

# Module imports fine. Error only at instantiation:
BadCircle()  # TypeError: Can't instantiate abstract class BadCircle
             # with abstract method area

# Correct:
class GoodCircle(Shape):
    def __init__(self, r): self.r = r
    def area(self): return 3.14159 * self.r ** 2

GoodCircle(5)  # OK
```

---

## Trap 9: "Metaclasses are the same as class decorators"

**What most people say:**
"A metaclass and a class decorator both modify a class, so they're equivalent."

**Correct answer:**
Both can modify classes, but they operate at different times and with different power.
A class decorator runs *after* the class is fully created and receives the class object.
A metaclass controls *how the class is created* — it is called during class creation and
can modify the class's `__dict__` before the class object exists. Metaclasses are also
inherited by subclasses automatically; class decorators are not.

```python
# Class decorator — applied after creation
def add_repr(cls):
    def __repr__(self):
        return f"{cls.__name__}({self.__dict__})"
    cls.__repr__ = __repr__
    return cls

@add_repr
class Foo:
    def __init__(self, x): self.x = x

# Metaclass — controls creation; inherited
class AutoReprMeta(type):
    def __new__(mcs, name, bases, namespace):
        namespace["__repr__"] = lambda self: f"{name}({self.__dict__})"
        return super().__new__(mcs, name, bases, namespace)

class Bar(metaclass=AutoReprMeta):
    def __init__(self, x): self.x = x

class Baz(Bar):  # Baz ALSO gets AutoReprMeta — inheritance!
    def __init__(self, y): self.y = y
```

---

## Trap 10: "__del__ is a reliable destructor like in C++"

**What most people say:**
"`__del__` is called when an object is destroyed, so I can use it for cleanup."

**Correct answer:**
`__del__` is unreliable in Python. It is called when the reference count drops to zero
(CPython) or when the garbage collector decides to collect (PyPy, Jython). In CPython,
reference cycles involving objects with `__del__` used to prevent GC collection entirely
(fixed in Python 3.4 with PEP 442). `__del__` may never be called if the program exits
abnormally, or may be called at interpreter shutdown when globals are already `None`.
Use context managers (`with`) for deterministic cleanup.

```python
class Resource:
    def __del__(self):
        # UNRELIABLE — may be called too late, or not at all
        close_connection()

# RELIABLE:
class Resource:
    def __enter__(self):
        self.conn = open_connection()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.conn.close()   # ALWAYS called when with block exits
        return False        # don't suppress exceptions

with Resource() as r:
    use(r)
```
