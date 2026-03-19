# Chapter 02 — OOP & Classes: Structured Answers

---

## Q1: Explain instance vs class vs static methods

**Core answer:**
These three method types differ in what their first argument is and how they relate
to the class hierarchy.

```python
class Temperature:
    unit = "Celsius"   # class variable

    def __init__(self, value):
        self.value = value   # instance variable

    # INSTANCE METHOD — receives the instance as first arg (self)
    # Can access and modify instance state AND class state
    def to_fahrenheit(self):
        return self.value * 9/5 + 32

    # CLASS METHOD — receives the class as first arg (cls)
    # Cannot access instance state; used for alternative constructors
    @classmethod
    def from_fahrenheit(cls, f_value):
        return cls((f_value - 32) * 5/9)   # cls() creates correct subclass!

    # STATIC METHOD — receives no implicit first arg
    # No access to instance or class state; just a namespaced function
    @staticmethod
    def is_valid(value):
        return -273.15 <= value

# Usage:
t = Temperature(100)
t.to_fahrenheit()            # 212.0  — via instance
Temperature.to_fahrenheit(t) # 212.0  — same call, explicit self

t2 = Temperature.from_fahrenheit(212)   # alternative constructor
t2.value  # 100.0

Temperature.is_valid(-300)   # False
t.is_valid(25)               # also works via instance
```

**Why `@classmethod` for alternative constructors?**
Using `cls(...)` instead of `Temperature(...)` ensures that subclasses get the right
type back:

```python
class MetricTemp(Temperature):
    pass

mt = MetricTemp.from_fahrenheit(212)
type(mt)  # MetricTemp — correct! (would be Temperature if hardcoded)
```

---

## Q2: Explain MRO and C3 Linearization

**Core answer:**
When Python looks up a method in a class hierarchy, it follows the Method Resolution
Order (MRO). Python uses the C3 linearization algorithm to compute a consistent, linear
order that respects all inheritance declarations.

**The diamond problem:**
```
        A
       / \
      B   C
       \ /
        D
```

```python
class A:
    def method(self): print("A")

class B(A):
    def method(self): print("B"); super().method()

class C(A):
    def method(self): print("C"); super().method()

class D(B, C):
    def method(self): print("D"); super().method()

D.__mro__
# (<class 'D'>, <class 'B'>, <class 'C'>, <class 'A'>, <class 'object'>)

D().method()
# D
# B
# C
# A
# A is called only ONCE — not twice as in naive "depth-first left-to-right"
```

**C3 rule summary:**
1. A class always comes before its parents.
2. If a class inherits from multiple parents, they appear in the order listed.
3. Consistent ordering is maintained across the entire hierarchy.

**Inspect the MRO:**
```python
D.__mro__           # tuple of classes
D.mro()             # list of classes (same content)

import inspect
inspect.getmro(D)   # same as __mro__
```

**Cooperative super() — every class must call super():**
```python
class Mixin:
    def setup(self):
        print("Mixin setup")
        super().setup()   # must call super even though Mixin doesn't know what's next

class Base:
    def setup(self):
        print("Base setup")

class App(Mixin, Base):
    def setup(self):
        print("App setup")
        super().setup()

App().setup()
# App setup
# Mixin setup
# Base setup
```

---

## Q3: Implement a complete class with key dunder methods

**Core answer:**

```python
from math import sqrt

class Vector2D:
    """A 2D vector with full operator support."""

    def __init__(self, x: float, y: float):
        self.x = x
        self.y = y

    # String representations
    def __repr__(self) -> str:
        # Should be unambiguous; ideally eval-able
        return f"Vector2D({self.x!r}, {self.y!r})"

    def __str__(self) -> str:
        # Human-readable
        return f"({self.x}, {self.y})"

    # Equality and hashing (must be consistent)
    def __eq__(self, other) -> bool:
        if not isinstance(other, Vector2D):
            return NotImplemented
        return self.x == other.x and self.y == other.y

    def __hash__(self) -> int:
        return hash((self.x, self.y))

    # Arithmetic operators
    def __add__(self, other: "Vector2D") -> "Vector2D":
        return Vector2D(self.x + other.x, self.y + other.y)

    def __sub__(self, other: "Vector2D") -> "Vector2D":
        return Vector2D(self.x - other.x, self.y - other.y)

    def __mul__(self, scalar: float) -> "Vector2D":
        return Vector2D(self.x * scalar, self.y * scalar)

    def __rmul__(self, scalar: float) -> "Vector2D":
        # Called when scalar * vector (float.__mul__ returns NotImplemented)
        return self.__mul__(scalar)

    def __abs__(self) -> float:
        return sqrt(self.x**2 + self.y**2)

    def __bool__(self) -> bool:
        return self.x != 0 or self.y != 0

    def __len__(self) -> int:
        return 2   # a 2D vector always has 2 components

    def __getitem__(self, index: int) -> float:
        return (self.x, self.y)[index]

    def __iter__(self):
        yield self.x
        yield self.y

# Usage:
v1 = Vector2D(1, 2)
v2 = Vector2D(3, 4)
v1 + v2              # Vector2D(4, 6)
3 * v1               # Vector2D(3, 6)  — uses __rmul__
abs(v2)              # 5.0
list(v1)             # [1, 2]  — via __iter__
{v1, v2}             # works — has __hash__
```

---

## Q4: Explain properties and descriptors

**Core answer:**
`@property` is syntactic sugar over the descriptor protocol. A descriptor is any object
that defines `__get__`, `__set__`, or `__delete__`.

```python
class Circle:
    def __init__(self, radius: float):
        self._radius = radius   # private by convention

    @property
    def radius(self) -> float:
        """Getter — called on attribute access."""
        return self._radius

    @radius.setter
    def radius(self, value: float):
        """Setter — called on attribute assignment."""
        if value < 0:
            raise ValueError("Radius cannot be negative")
        self._radius = value

    @radius.deleter
    def radius(self):
        """Deleter — called on del obj.radius."""
        del self._radius

    @property
    def area(self) -> float:
        """Computed property — no setter, so it's read-only."""
        return 3.14159 * self._radius ** 2

c = Circle(5)
c.radius          # 5    — calls getter
c.radius = 10     # calls setter
c.radius = -1     # ValueError
c.area            # 314.159 — computed on demand
c.area = 0        # AttributeError: can't set attribute (no setter)
```

**Raw descriptor:**
```python
class Validated:
    def __set_name__(self, owner, name):
        self.name = "_" + name

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self
        return getattr(obj, self.name, None)

    def __set__(self, obj, value):
        if not isinstance(value, (int, float)):
            raise TypeError(f"{self.name} must be numeric")
        setattr(obj, self.name, value)

class Rectangle:
    width  = Validated()
    height = Validated()

    def __init__(self, w, h):
        self.width  = w   # goes through Validated.__set__
        self.height = h

r = Rectangle(5, 3)
r.width = "oops"   # TypeError: _width must be numeric
```

---

## Q5: Dataclasses — what they generate and when to use them

**Core answer:**

```python
from dataclasses import dataclass, field
from typing import List

@dataclass
class Employee:
    name: str
    department: str
    salary: float = 0.0
    skills: List[str] = field(default_factory=list)  # mutable default — use field()

# What @dataclass generates automatically:
# __init__(self, name, department, salary=0.0, skills=<factory>)
# __repr__(self)  → "Employee(name='Alice', department='Eng', salary=90000.0, skills=[])"
# __eq__(self, other)  → compares all fields

e1 = Employee("Alice", "Engineering", 90000.0, ["Python", "Go"])
e2 = Employee("Alice", "Engineering", 90000.0, ["Python", "Go"])
e1 == e2   # True — field-by-field comparison

# Frozen (immutable + hashable):
@dataclass(frozen=True)
class Point:
    x: float
    y: float

# Ordering:
@dataclass(order=True)
class Version:
    major: int
    minor: int
    patch: int

Version(1, 2, 3) < Version(1, 3, 0)  # True

# Post-init validation:
@dataclass
class PositivePoint:
    x: float
    y: float

    def __post_init__(self):
        if self.x < 0 or self.y < 0:
            raise ValueError("Coordinates must be positive")
```

**Comparison table:**
```
@dataclass     — mutable by default; full class; supports methods; most flexible
namedtuple     — immutable; tuple-like; lighter weight; positional access
frozen=True    — immutable dataclass; hashable; recommended for value objects
TypedDict      — dict at runtime; only type-checking benefit; no methods
```
