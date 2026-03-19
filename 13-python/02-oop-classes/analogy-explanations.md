# Chapter 02 — OOP & Classes: Analogy Explanations

---

## Analogy 1: Class vs Instance — Blueprint vs Building

**The analogy:**
An architect's blueprint for an apartment is a *class*. It defines the number of
rooms, the layout, the plumbing connections, and what the walls should look like.
The blueprint itself is not an apartment — you cannot live in it. It is a plan.

An *instance* is an actual apartment built from that blueprint. You can paint the
walls (set instance attributes), move in furniture (store state), and live in it
(call methods). Two apartments built from the same blueprint are independent — painting
apartment 3B red does not affect apartment 5A.

**Connected to Python:**
```python
class Apartment:                      # Blueprint
    building_name = "Grand Tower"     # Class variable — shared by all apartments

    def __init__(self, floor, number):
        self.floor = floor            # Instance variable — unique per apartment
        self.number = number
        self.color = "white"          # default state

    def repaint(self, new_color):
        self.color = new_color        # mutates only THIS instance

apt_3b = Apartment(3, "B")           # Build apartment 3B
apt_5a = Apartment(5, "A")           # Build apartment 5A — independent!

apt_3b.repaint("red")
print(apt_3b.color)  # red
print(apt_5a.color)  # white — unaffected

# Class variable — shared across ALL instances (the building's name):
apt_3b.building_name  # 'Grand Tower'
Apartment.building_name = "Silver Tower"
apt_5a.building_name  # 'Silver Tower' — all apartments see the change
```

---

## Analogy 2: MRO and C3 Linearization — Inheritance Chain of Command

**The analogy:**
Imagine a military organisation. When a soldier asks for an order, they first ask their
*direct superior*. If the direct superior has no relevant order, they ask the *next
superior up the chain*. The chain is well-defined, and no one is consulted twice.

In a diamond inheritance (`D → B,C → A`), the "naive" chain might visit A twice
(once via B, once via C). C3 ensures each person in the chain is consulted exactly
once and in a sensible order — respecting both the B-branch and C-branch precedence.

**Connected to Python:**
```
        Object (Chief)
           │
           A (Colonel)
          / \
    B (Major) C (Captain)
          \ /
           D (Sergeant)

Chain of command for D: D → B → C → A → object
```

```python
class A:
    def speak(self): return "A"

class B(A):
    def speak(self): return "B → " + super().speak()

class C(A):
    def speak(self): return "C → " + super().speak()

class D(B, C):
    def speak(self): return "D → " + super().speak()

D().speak()       # "D → B → C → A"
D.__mro__         # D, B, C, A, object

# super() doesn't mean "my parent" — it means "next in chain"
# B's super() is C (not A!) when called through D
```

---

## Analogy 3: Descriptors — Smart Property Managers

**The analogy:**
Imagine a luxury apartment building where a *property manager* intercepts all requests
to access or modify specific features. When a tenant wants to adjust the thermostat
(attribute access), the property manager checks their contract (validation logic),
logs the request (side effects), and then either allows or denies the change.

The property manager (descriptor) is hired once by the building (class), but handles
requests from all tenants (instances). The manager's contract applies building-wide.

**Connected to Python:**
```python
class ThermostatDescriptor:
    """Property manager for the 'temperature' attribute."""

    def __set_name__(self, owner, name):
        self.private_name = f"_{name}"  # where we'll actually store the value

    def __get__(self, obj, objtype=None):
        if obj is None:
            return self   # accessed on the class itself
        print(f"Tenant accessed temperature")
        return getattr(obj, self.private_name, 20)

    def __set__(self, obj, value):
        if not (15 <= value <= 30):
            raise ValueError(f"Temperature {value} outside allowed range 15-30°C")
        print(f"Temperature changed to {value}")
        setattr(obj, self.private_name, value)

class Apartment:
    temperature = ThermostatDescriptor()  # property manager hired here

    def __init__(self, unit):
        self.unit = unit

apt = Apartment("3B")
apt.temperature = 22    # "Temperature changed to 22"
apt.temperature = 40    # ValueError: Temperature 40 outside allowed range 15-30°C
```

---

## Analogy 4: __slots__ — Reserved Parking Spaces

**The analogy:**
A regular car park (normal Python instance) is a free-for-all: tenants can park any
number of cars in any unclaimed space. The car park needs a giant open field
(the `__dict__`) to accommodate unpredictable parking needs.

A reserved parking facility (class with `__slots__`) pre-assigns named spaces —
"space A for car X, space B for car Y". Only those specific cars can park, but the
facility needs much less physical space because the layout is known in advance and
can be optimised. You cannot bring an extra car — there is no space for it.

**Connected to Python:**
```python
import sys

class FreeForAll:
    def __init__(self, x, y):
        self.x = x
        self.y = y
    # Has __dict__ — can add any attribute dynamically

class Reserved:
    __slots__ = ("x", "y")   # reserved spaces — only x and y allowed
    def __init__(self, x, y):
        self.x = x
        self.y = y

free = FreeForAll(1, 2)
reserved = Reserved(1, 2)

free.z = 99      # OK — free parking
reserved.z = 99  # AttributeError: 'Reserved' object has no attribute 'z'

# Memory savings (approximate):
print(sys.getsizeof(free.__dict__))  # ~232 bytes for the dict alone
# reserved has no __dict__; attribute access through slot descriptors (C-level pointers)

# When you have millions of instances:
# free:     N × (object_overhead + dict_overhead) = expensive
# reserved: N × (object_overhead + 2 C-level pointers) = much cheaper
```

---

## Analogy 5: Context Managers — Checking In and Checking Out of a Hotel

**The analogy:**
When you check in to a hotel, the front desk (the `with` statement) does two things:
(1) sets up your room — gives you a keycard, activates the minibar, notes your arrival
(`__enter__`); (2) when you check out, they clean the room, deactivate the keycard,
and handle any problems (charges for damages) regardless of whether your stay was
pleasant or you left in a hurry (`__exit__`).

The hotel guarantees that checkout procedures run no matter what happened during your
stay — even if you called the fire department.

**Connected to Python:**
```python
class DatabaseTransaction:
    def __init__(self, connection):
        self.conn = connection
        self.transaction = None

    def __enter__(self):
        # "Check-in" — start the transaction
        self.transaction = self.conn.begin()
        return self.transaction   # what the `as` variable receives

    def __exit__(self, exc_type, exc_val, exc_tb):
        # "Check-out" — always runs
        if exc_type is None:
            # No exception — commit
            self.transaction.commit()
        else:
            # Exception — roll back
            self.transaction.rollback()
        return False  # False = don't suppress the exception

with DatabaseTransaction(conn) as tx:
    tx.execute("INSERT INTO users VALUES (?)", (user,))
    # If this raises, __exit__ rolls back automatically
    tx.execute("UPDATE balance SET amount = amount - 100")

# Both commits, or neither — atomicity guaranteed
```

**The `exc_type, exc_val, exc_tb` triple:**
If `__exit__` returns a truthy value, the exception is *suppressed* (swallowed).
Returning `False` or `None` re-raises it. This lets context managers selectively
handle errors.

---

## Analogy 6: Metaclasses — The Factory That Builds Factories

**The analogy:**
A regular factory makes products (instances). A *meta-factory* makes factories
themselves — it controls how factories are set up, what machines they contain, and
what quality checks they must pass before they can start operating.

In Python: regular classes are factories that make instances. Metaclasses are
factories that make *classes*. When Python processes a `class` statement, it calls
the metaclass to actually construct the class object.

**Connected to Python:**
```python
class RegistryMeta(type):
    """A metaclass that registers every class it creates."""
    _registry = {}

    def __new__(mcs, name, bases, namespace):
        cls = super().__new__(mcs, name, bases, namespace)
        if name != "Base":              # don't register the base itself
            mcs._registry[name] = cls
        return cls

class Base(metaclass=RegistryMeta):
    pass

class PluginA(Base): pass
class PluginB(Base): pass

RegistryMeta._registry
# {'PluginA': <class 'PluginA'>, 'PluginB': <class 'PluginB'>}

# Useful for: plugin systems, ORM model registration, enum-like registries
```

**Metaclass execution order:**
```
1. Python reads `class Foo(Base, metaclass=Meta):` body
2. Calls Meta.__prepare__(name, bases) → empty namespace dict
3. Executes class body, populating namespace
4. Calls Meta(name, bases, namespace) → calls Meta.__new__ → Meta.__init__
5. Returns the new class object
```

---

## Analogy 7: __repr__ vs __str__ — ID Card vs Business Card

**The analogy:**
Every person in a company has two ways to identify themselves:

An **ID card** (`__repr__`) is the unambiguous, official record. It has your employee
number, full legal name, and department code. It is used by HR systems and security
doors — machines that need *exact*, reproducible identification. If you had to recreate
someone's record from scratch, the ID card has everything needed.

A **business card** (`__str__`) is the human-friendly presentation. It says "Alice,
Software Engineer at TechCorp." It is pleasant to hand to a client, but does not
contain your employee ID or payroll details.

**Connected to Python:**
```python
class Money:
    def __init__(self, amount, currency):
        self.amount = amount
        self.currency = currency

    def __repr__(self):
        # Unambiguous, ideally eval-able:  eval(repr(obj)) == obj
        return f"Money({self.amount!r}, {self.currency!r})"

    def __str__(self):
        # Human-readable:
        return f"{self.currency} {self.amount:.2f}"

m = Money(99.5, "USD")

repr(m)   # "Money(99.5, 'USD')"       ← machine-readable, dev-facing
str(m)    # "USD 99.50"                ← human-readable, user-facing
print(m)  # "USD 99.50"               ← print() calls __str__

# In containers, repr() is used:
[m]       # [Money(99.5, 'USD')]       ← list uses repr for elements

# If only __repr__ is defined, __str__ falls back to __repr__
# If only __str__ is defined, repr() shows the default <Money object at 0x...>
```
